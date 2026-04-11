---
title: "Betfair | Last Traded Price Dynamics — Reading Market Direction"
date: 2026-05-30
tags: [Java, Betfair, Trading, LTP, Market Analysis]
style: fill
color: primary
description: >-
  How Last Traded Price data from the Betfair Streaming API reveals market momentum — building a sliding window LTP analyser in Java to calculate price velocity and identify drift patterns.
---

The Last Traded Price (LTP) is the odds at which the most recent bet was matched on Betfair. In isolation it's just a number. In sequence, it tells a story about where the market is going and how fast it's moving. LTP analysis is the closest thing to reading tape in traditional financial markets — and in my experience it's more reliable than Weight of Money as a directional signal, particularly in the final 10 minutes before the off.

## LTP vs Best Available Price

There's an important distinction between LTP and best available price. The best available back price is what you'd get if you placed a back bet right now. LTP is where the last matched bet occurred. In a liquid market these will be close together. In a thin or fast-moving market, there can be several ticks of difference.

For directional analysis, LTP is more informative — it represents actual agreement between buyers and sellers, not just the best advertised price. A sequence of shortening LTPs means backs and lays are being matched at progressively lower odds: genuine steam, not just queued orders.

## Capturing LTP from the Streaming API

The Streaming API delivers LTP as `ltp` in the `RunnerChange` object. It arrives whenever a bet is matched:

```java
public void onRunnerChange(long selectionId, RunnerChange rc) {
    if (rc.getLtp() != null) {
        ltpTracker.record(selectionId, rc.getLtp(), Instant.now());
    }
}
```

Not every `RunnerChange` contains an LTP — if no bets matched in that delta, the field is absent. You need to hold the last known LTP in your cache and only update it when a new one arrives.

## Building the Sliding Window LTP Analyser

I track LTP history in a fixed-size sliding window per runner:

```java
public class LtpAnalyser {

    private final int windowSize;
    private final Map<Long, Deque<LtpPoint>> windows = new ConcurrentHashMap<>();

    public LtpAnalyser(int windowSize) {
        this.windowSize = windowSize;
    }

    public void record(long selectionId, double ltp) {
        Deque<LtpPoint> window = windows.computeIfAbsent(
            selectionId, id -> new ArrayDeque<>());

        window.addLast(new LtpPoint(Instant.now(), ltp));
        while (window.size() > windowSize) {
            window.removeFirst();
        }
    }

    /**
     * Price velocity: change in LTP per second over the window.
     * Negative = shortening (price drifting lower = steam).
     * Positive = drifting (price moving higher).
     */
    public OptionalDouble velocity(long selectionId) {
        Deque<LtpPoint> window = windows.get(selectionId);
        if (window == null || window.size() < 2) return OptionalDouble.empty();

        LtpPoint first = window.peekFirst();
        LtpPoint last  = window.peekLast();

        double priceChange = last.ltp() - first.ltp();
        double seconds = Duration.between(first.timestamp(), last.timestamp()).toMillis() / 1000.0;
        if (seconds == 0) return OptionalDouble.empty();

        return OptionalDouble.of(priceChange / seconds);
    }

    /**
     * Simple linear regression slope across the window — more stable than first/last delta.
     */
    public OptionalDouble trend(long selectionId) {
        List<LtpPoint> points = new ArrayList<>(
            windows.getOrDefault(selectionId, new ArrayDeque<>()));

        if (points.size() < 3) return OptionalDouble.empty();

        long baseTime = points.get(0).timestamp().toEpochMilli();
        int n = points.size();

        double sumX = 0, sumY = 0, sumXY = 0, sumX2 = 0;
        for (LtpPoint p : points) {
            double x = (p.timestamp().toEpochMilli() - baseTime) / 1000.0;
            double y = p.ltp();
            sumX  += x;
            sumY  += y;
            sumXY += x * y;
            sumX2 += x * x;
        }

        double denom = n * sumX2 - sumX * sumX;
        if (denom == 0) return OptionalDouble.empty();

        double slope = (n * sumXY - sumX * sumY) / denom;
        return OptionalDouble.of(slope);
    }

    record LtpPoint(Instant timestamp, double ltp) {}
}
```

The linear regression slope is more stable than the simple first-to-last velocity because it's less sensitive to a single outlier trade. A slope of `-0.02` (odds per second) on a runner at 4.0 that's been running for 3 minutes is significant. The same slope on a runner at 1.5 is barely meaningful.

## Identifying Drift Patterns

Beyond simple velocity, specific LTP patterns have distinct interpretations:

```java
public enum LtpPattern {
    SUSTAINED_STEAM,   // consistently shortening over 5+ minutes
    RECENT_STEAM,      // shortening in last 60s but not before
    SUSTAINED_DRIFT,   // consistently lengthening
    LATE_DRIFT,        // drifting in last 60s after being stable
    VOLATILE,          // large swings in both directions
    STABLE             // minimal movement
}

public LtpPattern classify(long selectionId) {
    List<LtpPoint> all = new ArrayList<>(
        windows.getOrDefault(selectionId, new ArrayDeque<>()));

    if (all.size() < 10) return LtpPattern.STABLE;

    int half = all.size() / 2;
    List<LtpPoint> early  = all.subList(0, half);
    List<LtpPoint> recent = all.subList(half, all.size());

    double earlySlope  = slopeOf(early);
    double recentSlope = slopeOf(recent);
    double maxChange   = range(all);

    if (maxChange > 0.5) return LtpPattern.VOLATILE;
    if (recentSlope < -0.01 && earlySlope < -0.01) return LtpPattern.SUSTAINED_STEAM;
    if (recentSlope < -0.015 && earlySlope >= -0.005) return LtpPattern.RECENT_STEAM;
    if (recentSlope > 0.01 && earlySlope > 0.01) return LtpPattern.SUSTAINED_DRIFT;
    if (recentSlope > 0.015 && earlySlope <= 0.005) return LtpPattern.LATE_DRIFT;

    return LtpPattern.STABLE;
}
```

`SUSTAINED_STEAM` is the pattern I act on most confidently. `RECENT_STEAM` gets a smaller position because it might be a temporary blip. `LATE_DRIFT` on a previously stable runner less than 5 minutes to the off is often a sign of breaking news — scratching, jockey change, or late market information.

## Combining with WoM

LTP velocity paired with WoM creates a two-factor signal:

```java
public TradingSignal combinedSignal(long selectionId) {
    LtpPattern ltpPattern = ltpAnalyser.classify(selectionId);
    WomSignal womSignal = womService.getSignal(selectionId);

    if (ltpPattern == LtpPattern.SUSTAINED_STEAM && womSignal == WomSignal.STEAM) {
        return TradingSignal.STRONG_STEAM;
    }
    if (ltpPattern == LtpPattern.SUSTAINED_DRIFT && womSignal == WomSignal.DRIFT) {
        return TradingSignal.STRONG_DRIFT;
    }
    // Conflicting signals — stay out
    return TradingSignal.NEUTRAL;
}
```

When both signals agree, the confidence is higher. When they conflict, the market is uncertain — I don't trade.

## ProTips

- **Normalise velocity by odds range.** A 0.1 tick move on a 2.0 shot is very different from a 0.1 tick move on a 10.0 shot. Consider expressing velocity as a percentage of current LTP rather than absolute odds.
- **Ignore the first 30 minutes of trading.** Early market formation is noisy. Meaningful LTP trends typically emerge in the last 30–60 minutes before the off.
- **Record everything.** Store every LTP point with its timestamp. Post-session analysis of recorded data against outcomes will refine your interpretation far better than any theory.
- **Watch matched volume alongside LTP.** A strongly shortening LTP with very low matched volume could be one punter moving the market with a small stake. High volume confirming the move is a much stronger signal.

If you're looking for a Java contractor who knows this space inside out, [get in touch](/hire/).
