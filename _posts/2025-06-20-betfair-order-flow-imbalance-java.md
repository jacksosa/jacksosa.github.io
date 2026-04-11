---
title: "Betfair | Order Flow Imbalance — A Java Implementation"
tags: [Java, Betfair, Trading, Order Flow, Market Microstructure]
style: fill
color: danger
description: Order flow imbalance (OFI) as a trading signal on Betfair — what it is, how to calculate it from streaming data, and how to build a real-time OFI indicator in Java.
---

Order flow imbalance is a concept borrowed from equity market microstructure research, and it translates surprisingly well to Betfair. The core idea: at any moment, there's a flow of money arriving on the back side and a flow arriving on the lay side. If backs are arriving faster than lays, the price is likely to shorten. If lays are arriving faster, the price is likely to drift. OFI captures this directional pressure more precisely than a static snapshot of available money.

## What OFI Measures

OFI measures the net directional pressure at the best available prices over a time window. The intuition in equities terms: are more people buying than selling? On Betfair: are more people backing than laying at the best prices?

The formal definition I use adapts the Cont, Kukanov, and Stoikov (2014) OFI measure:

```
OFI(t) = ΔBestBack(t) - ΔBestLay(t)
```

Where:
- `ΔBestBack(t)` = change in size at best back price (increases when money arrives to back, decreases when it's taken or cancelled)
- `ΔBestLay(t)` = change in size at best lay price (increases when money arrives to lay, decreases when it's taken or cancelled)

Positive OFI → backing pressure dominating → price likely to shorten.
Negative OFI → laying pressure dominating → price likely to drift.

## Extracting Delta Data from the Streaming API

The Streaming API sends `batb` (best available to back) and `batl` (best available to lay) as ladder changes. Each update contains only the changed levels, not the full ladder. To calculate OFI you need to track changes at the best price level specifically:

```java
public class OfiTracker {

    // Map: selectionId -> (bestBackPrice -> bestBackSize)
    private final Map<Long, double[]> bestBack = new ConcurrentHashMap<>();
    private final Map<Long, double[]> bestLay  = new ConcurrentHashMap<>();

    // Cumulative OFI values for rolling window
    private final Map<Long, Deque<Double>> ofiWindow = new ConcurrentHashMap<>();
    private final int windowSize;

    public OfiTracker(int windowSize) {
        this.windowSize = windowSize;
    }

    public void onRunnerChange(long selectionId, RunnerChange rc) {
        double ofi = 0.0;

        if (rc.getBatb() != null && !rc.getBatb().isEmpty()) {
            // Best back = first entry in batb (lowest price = best odds for backer)
            List<Double> bestBackLevel = rc.getBatb().get(0);
            double newPrice = bestBackLevel.get(0);
            double newSize  = bestBackLevel.get(1);

            double[] prev = bestBack.get(selectionId);
            if (prev != null) {
                if (newPrice == prev[0]) {
                    // Same price level — size change represents new/cancelled orders
                    ofi += (newSize - prev[1]);
                } else if (newSize > 0) {
                    // New best price — treat as fresh arrival
                    ofi += newSize;
                }
            }
            bestBack.put(selectionId, new double[]{newPrice, newSize});
        }

        if (rc.getBatl() != null && !rc.getBatl().isEmpty()) {
            // Best lay = first entry in batl (lowest lay price)
            List<Double> bestLayLevel = rc.getBatl().get(0);
            double newPrice = bestLayLevel.get(0);
            double newSize  = bestLayLevel.get(1);

            double[] prev = bestLay.get(selectionId);
            if (prev != null) {
                if (newPrice == prev[0]) {
                    ofi -= (newSize - prev[1]); // subtract lay arrivals
                } else if (newSize > 0) {
                    ofi -= newSize;
                }
            }
            bestLay.put(selectionId, new double[]{newPrice, newSize});
        }

        // Add to rolling window
        Deque<Double> window = ofiWindow.computeIfAbsent(
            selectionId, id -> new ArrayDeque<>());
        window.addLast(ofi);
        while (window.size() > windowSize) window.removeFirst();
    }
}
```

## Normalising OFI

Raw OFI values are highly liquidity-dependent. A £500 order flow imbalance in a £500,000 matched market is noise. Normalise by dividing by total size at best prices:

```java
public OptionalDouble normalisedOfi(long selectionId) {
    Deque<Double> window = ofiWindow.get(selectionId);
    if (window == null || window.isEmpty()) return OptionalDouble.empty();

    double cumulativeOfi = window.stream().mapToDouble(Double::doubleValue).sum();

    double[] back = bestBack.get(selectionId);
    double[] lay  = bestLay.get(selectionId);

    if (back == null || lay == null) return OptionalDouble.empty();

    double totalAtBest = back[1] + lay[1];
    if (totalAtBest == 0) return OptionalDouble.empty();

    return OptionalDouble.of(cumulativeOfi / totalAtBest);
}
```

The normalised OFI ranges from roughly -1 to +1. Values above +0.3 suggest sustained buying pressure; below -0.3 suggest selling pressure. These thresholds are starting points — calibrate against your own recorded data.

## Combining OFI with WoM and LTP

OFI, WoM, and LTP velocity are complementary. OFI captures the dynamics at the best price level — the most liquid and price-relevant point of the book. WoM captures the broader depth. LTP captures what's actually been matched.

```java
public TradingSignal compositeSignal(long selectionId) {
    OptionalDouble ofi  = ofiTracker.normalisedOfi(selectionId);
    WomSignal      wom  = womService.getSignal(selectionId);
    LtpPattern     ltp  = ltpAnalyser.classify(selectionId);

    int steamScore = 0;
    if (ofi.isPresent() && ofi.getAsDouble() > 0.25) steamScore++;
    if (wom == WomSignal.STEAM)                       steamScore++;
    if (ltp == LtpPattern.SUSTAINED_STEAM)            steamScore++;

    int driftScore = 0;
    if (ofi.isPresent() && ofi.getAsDouble() < -0.25) driftScore++;
    if (wom == WomSignal.DRIFT)                        driftScore++;
    if (ltp == LtpPattern.SUSTAINED_DRIFT)             driftScore++;

    if (steamScore >= 2) return TradingSignal.STEAM;
    if (driftScore >= 2) return TradingSignal.DRIFT;
    return TradingSignal.NEUTRAL;
}
```

Requiring at least two of three signals to agree before acting materially reduces false positives. The cost is missing some genuine moves — an acceptable trade-off if you're managing a portfolio of positions across multiple races simultaneously.

## False Positives in Thin Markets

OFI is least reliable in markets with low liquidity. In a market with £10,000 total matched volume, one trader can create a sustained OFI signal with a £200 resting order. Some filters I apply:

```java
public boolean isMarketSuitableForOfi(String marketId) {
    MarketState state = cache.getMarket(marketId).orElse(null);
    if (state == null) return false;

    double totalMatched = state.getTotalMatched();
    int minutesToOff = state.minutesUntilOff();

    return totalMatched > 50_000       // minimum liquidity
        && minutesToOff <= 30          // close enough to off for signal to matter
        && minutesToOff >= 2;          // not so close that execution is infeasible
}
```

Pre-race horse racing markets with £100k+ matched volume 10–20 minutes before the off are where OFI has been most reliable in my experience. Very early market formation, niche events, and sports with sporadic betting patterns are where I don't apply OFI.

## ProTips

- **Store OFI observations with timestamps.** After a trading session, you can calculate the correlation between OFI signals and subsequent LTP movement. This is how you validate that the signal is genuinely predictive in your target markets.
- **Reset the window at market phase changes.** When a market goes in-play, OFI dynamics change completely. Clear the window and don't act on pre-race OFI history in running markets.
- **Beware cancellation spikes.** A trader cancelling a large resting order creates a negative OFI spike that doesn't reflect genuine selling pressure. Filter out extreme single-delta values that are clearly outliers.
- **OFI works best at short odds.** Selections at 2.0–6.0 tend to have enough liquidity at the best price for OFI to be meaningful. Very long shots (20.0+) have thin best-price liquidity and noisy OFI readings.

If you're looking for a Java contractor who knows this space inside out, [get in touch](/hire/).
