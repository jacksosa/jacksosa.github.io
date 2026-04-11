---
title: "Betfair | Weight of Money — Implementing WoM Calculations in Java"
date: 2026-05-16
tags: [Java, Betfair, Trading, Market Analysis, Spring Boot]
style: fill
color: danger
description: >-
  How to calculate Weight of Money (WoM) from live Betfair Streaming data in Java — including the formula, rolling trend signals, Spring Boot wiring, and when the signal is and isn't reliable.
---

Weight of Money is one of the most widely discussed pre-race trading signals on Betfair, and one of the most widely misunderstood. I've seen traders treat it as gospel and get burned; I've also seen it provide a genuinely useful edge when applied with the right context. The signal itself is simple to compute — the insight is knowing when to trust it and when to ignore it.

## What Weight of Money Is

WoM compares the money available to back a selection with the money available to lay it. When there's significantly more money queued on the lay side, it suggests sellers (layers) are more aggressive than buyers (backers) — the price may drift out. When back money dominates, the price may shorten.

The formula I use:

```
WoM = availableToBack / (availableToBack + availableToLay)
```

A value above 0.5 means back money dominates. Below 0.5 means lay money dominates. The distance from 0.5 indicates the strength of the imbalance.

You can compute this across all ladder levels or restrict to the best few levels. I typically use the top 3 levels on each side — deeper levels are less reliable because large chunks of queued money may never be matched.

## Getting the Data from the Streaming API

The Streaming API provides available-to-back (`batb`) and available-to-lay (`batl`) as ladder arrays in each `RunnerChange`. Each element is a `[price, size]` pair:

```java
public class WomCalculator {

    /**
     * Calculate WoM for a runner using the top N ladder levels on each side.
     *
     * @param batb  available-to-back ladder: list of [price, size] pairs
     * @param batl  available-to-lay ladder: list of [price, size] pairs
     * @param levels number of ladder levels to include
     * @return WoM ratio in [0, 1], or empty if insufficient data
     */
    public OptionalDouble calculate(
            List<List<Double>> batb,
            List<List<Double>> batl,
            int levels) {

        if (batb == null || batl == null) return OptionalDouble.empty();

        double backTotal = batb.stream()
            .limit(levels)
            .mapToDouble(pair -> pair.get(1)) // index 1 = size
            .sum();

        double layTotal = batl.stream()
            .limit(levels)
            .mapToDouble(pair -> pair.get(1))
            .sum();

        double combined = backTotal + layTotal;
        if (combined == 0) return OptionalDouble.empty();

        return OptionalDouble.of(backTotal / combined);
    }
}
```

Note that `batb` is sorted best-back-first (lowest price first — shortest odds at the top), and `batl` is sorted best-lay-first (lowest lay price first). You want the top of each side for the most relevant signal.

## Building a Rolling WoM Trend

A single WoM reading is noisy. What matters is the direction and consistency of the signal over time. I maintain a rolling window of WoM observations for each runner:

```java
public class WomTrendTracker {

    private final int windowSize;
    private final Deque<TimestampedWom> window = new ArrayDeque<>();

    public WomTrendTracker(int windowSize) {
        this.windowSize = windowSize;
    }

    public void record(double wom) {
        window.addLast(new TimestampedWom(Instant.now(), wom));
        while (window.size() > windowSize) {
            window.removeFirst();
        }
    }

    /**
     * Average WoM across the window.
     */
    public OptionalDouble averageWom() {
        if (window.isEmpty()) return OptionalDouble.empty();
        return window.stream()
            .mapToDouble(TimestampedWom::wom)
            .average();
    }

    /**
     * Trend: positive means WoM moving in favour of backs (price shortening signal),
     * negative means moving in favour of lays (drift signal).
     */
    public OptionalDouble trend() {
        if (window.size() < 2) return OptionalDouble.empty();

        List<TimestampedWom> entries = new ArrayList<>(window);
        int half = entries.size() / 2;

        double earlyAvg = entries.subList(0, half).stream()
            .mapToDouble(TimestampedWom::wom).average().orElse(0.5);
        double recentAvg = entries.subList(half, entries.size()).stream()
            .mapToDouble(TimestampedWom::wom).average().orElse(0.5);

        return OptionalDouble.of(recentAvg - earlyAvg);
    }

    record TimestampedWom(Instant timestamp, double wom) {}
}
```

A trend of `+0.1` over a 20-observation window is meaningful. A single reading jumping from 0.4 to 0.7 is likely noise — one large order landed.

## Wiring into a Spring Boot Strategy Engine

In a Spring Boot trading framework, I compute WoM in a market state component that the strategy engine queries:

```java
@Component
public class MarketSignalService {

    private final WomCalculator womCalculator;
    private final Map<Long, WomTrendTracker> trackers = new ConcurrentHashMap<>();

    public void onRunnerChange(long selectionId, RunnerChange rc) {
        WomTrendTracker tracker = trackers.computeIfAbsent(
            selectionId, id -> new WomTrendTracker(20));

        OptionalDouble wom = womCalculator.calculate(
            rc.getBatb(), rc.getBatl(), 3);

        wom.ifPresent(tracker::record);
    }

    public WomSignal getSignal(long selectionId) {
        WomTrendTracker tracker = trackers.get(selectionId);
        if (tracker == null) return WomSignal.NEUTRAL;

        OptionalDouble trend = tracker.trend();
        if (trend.isEmpty()) return WomSignal.NEUTRAL;

        double t = trend.getAsDouble();
        if (t > 0.05) return WomSignal.STEAM;    // backs dominating — price may shorten
        if (t < -0.05) return WomSignal.DRIFT;   // lays dominating — price may drift
        return WomSignal.NEUTRAL;
    }
}
```

The strategy engine consults `getSignal()` when deciding whether to enter a trade. WoM alone is rarely sufficient — it works best combined with LTP velocity (covered in a later post).

## When the Signal Is and Isn't Reliable

WoM is strongest when:
- The market has good liquidity (£50k+ matched volume)
- The reading is consistent over multiple observations
- The signal aligns with LTP movement (WoM says steam, LTP is shortening)

WoM is unreliable when:
- It's more than ~15 minutes before the off — large amounts of queued money appear and disappear as traders position
- A single large bet creates a spike — one £2000 back at a specific price inflates the reading without signalling genuine steam
- The market is in-play — WoM dynamics are completely different in running markets

## ProTips

- **Don't use WoM in isolation.** Treat it as one input to a composite signal. Alone, its false positive rate is high.
- **Normalise by traded volume.** A £500 WoM imbalance in a £200,000 matched market is insignificant. Scale the signal to the market's liquidity.
- **Record WoM data with timestamps.** After trading sessions, replaying WoM signals against outcomes tells you quickly whether the signal is predictive for specific race types or not.
- **Watch for market manipulation.** Experienced traders place large orders near the off and cancel before matching, artificially creating WoM signals. Look for orders that appear and vanish repeatedly — a sign of layering.

If you're looking for a Java contractor who knows this space inside out, [get in touch](/hire/).
