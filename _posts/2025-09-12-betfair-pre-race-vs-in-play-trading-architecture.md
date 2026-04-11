---
title: "Betfair | Pre-Race vs In-Play Trading — Technical Architecture"
tags: [Java, Betfair, Trading, In-Play, Architecture]
style: fill
color: primary
description: The technical and strategic differences between pre-race and in-play Betfair trading — market state transitions, streaming API indicators, latency requirements, and Java architecture for both modes.
---

Pre-race and in-play trading on Betfair are fundamentally different problems. Pre-race trading operates over minutes, prices move predictably enough to apply analytical signals, and the risk of catastrophic loss on a single decision is manageable. In-play trading operates in seconds or fractions of seconds, prices can halve or double on a single event, and the consequences of a position left unhedged during a suspension cascade can be severe. Building a system that handles both modes cleanly — without either polluting the other — requires deliberate architectural choices.

## Market State Transitions

A Betfair market moves through several states, visible in the Streaming API's `MarketDefinition`:

```java
public enum MarketStatus {
    INACTIVE,       // market exists but not yet accepting bets
    OPEN,           // pre-race, accepting bets, match-time clock running
    SUSPENDED,      // temporarily suspended (VAR check, injury, etc.)
    CLOSED          // market settled
}
```

The transition that matters most for trading architecture is `OPEN → IN_PLAY`. This happens at the event start and is signalled in the streaming API via `MarketDefinition.inPlay = true`.

```java
public void onMarketDefinitionChange(String marketId, MarketDefinition definition) {
    MarketState current = markets.get(marketId);
    if (current == null) return;

    boolean wasInPlay = current.isInPlay();
    boolean isNowInPlay = Boolean.TRUE.equals(definition.isInPlay());

    if (!wasInPlay && isNowInPlay) {
        onMarketGoesInPlay(marketId);
    }

    if (definition.getStatus() == MarketStatus.SUSPENDED) {
        onMarketSuspended(marketId, wasInPlay);
    }

    current.update(definition);
}
```

The `onMarketGoesInPlay` event is your trigger to switch from pre-race mode (signal-based, deliberate entry) to in-play mode (position management, hedging, emergency exits).

## Pre-Race Architecture

Pre-race trading has time on its side. Your analytical loop runs every 500ms–1s, evaluating WoM, LTP, and OFI signals. The latency requirements are lenient — a 50ms delay in signal processing rarely changes the outcome.

```java
@Component
public class PreRaceStrategyEngine {

    private final Map<String, MarketSignalBuffer> signalBuffers = new ConcurrentHashMap<>();

    // Called on each Streaming API delta
    public void onMarketUpdate(String marketId, MarketChangeMessage mcm) {
        MarketSignalBuffer buffer = signalBuffers.get(marketId);
        if (buffer == null) return;

        buffer.addDelta(mcm);

        // Evaluate at most once per 500ms
        if (buffer.shouldEvaluate()) {
            MarketSignals signals = signalCalculator.calculate(buffer.getState());
            strategies.forEach(s -> s.onSignalUpdate(context(marketId), signals));
        }
    }

    public void onMarketGoesInPlay(String marketId) {
        // Stop pre-race evaluation
        signalBuffers.remove(marketId);
        // Transfer to in-play engine
        inPlayEngine.trackMarket(marketId);
    }
}
```

The `MarketSignalBuffer` rate-limits evaluation — no point running signal calculations on every Streaming API tick when the signals require 20-observation windows to be meaningful.

## In-Play Architecture

In-play is a different beast. Prices move at sports event pace — a goal in football can move a selection from 2.5 to 1.2 in under a second. The in-play engine must respond in milliseconds, not seconds. In-play strategy is primarily about position management:

```java
@Component
public class InPlayPositionManager {

    private final Map<String, InPlayPosition> openPositions = new ConcurrentHashMap<>();
    private final OrderManager orderManager;
    private final RiskController riskController;

    // Called on every Streaming API delta in-play
    public void onInPlayUpdate(String marketId, MarketChangeMessage mcm) {
        InPlayPosition position = openPositions.get(marketId);
        if (position == null) return;

        for (RunnerChange rc : mcm.getMarketChanges().get(0).getRunnerChanges()) {
            double currentLtp = extractLtp(rc);
            position.updateLtp(rc.getId(), currentLtp);

            // Check hedge conditions
            if (shouldHedge(position, rc.getId(), currentLtp)) {
                hedgePosition(marketId, position, rc.getId(), currentLtp);
            }
        }
    }

    private boolean shouldHedge(InPlayPosition position, long selectionId, double currentLtp) {
        double entryPrice = position.getEntryPrice(selectionId);
        double currentPnl = position.calculatePnl(selectionId, currentLtp);

        // Hedge if profit target hit
        if (currentPnl >= position.getProfitTarget()) return true;

        // Hedge if stop loss hit
        if (currentPnl <= position.getStopLoss()) return true;

        return false;
    }
}
```

The critical difference: no signal buffering in-play. Every delta is evaluated immediately.

## API Rate Limits In-Play

Betfair applies tighter API rate limits in-play. The REST API allows around 5 calls per second per application key for order operations in-play — roughly half the pre-race limit in some markets. This is why in-play order management must be batched: multiple position hedges in the same API call wherever possible:

```java
public void hedgeMultiplePositions(String marketId, List<HedgeInstruction> hedges) {
    List<PlaceInstruction> instructions = hedges.stream()
        .map(h -> buildLayInstruction(h.selectionId(), h.price(), h.size()))
        .toList();

    // One API call for all hedges
    PlaceExecutionReport report = orderManager.placeOrders(marketId, instructions);
    processReport(report, hedges);
}
```

## Suspension Handling

Suspension is particularly dangerous in-play. When a market suspends, open positions can't be hedged, but liability from unmatched lays remains. Your architecture must:

1. Detect suspension immediately (streaming API `MarketDefinition.status = SUSPENDED`)
2. Record all open position states
3. Attempt to cancel unmatched portions (may fail if market is suspended)
4. Wait for resumption
5. Reassess positions at the new prices

```java
public void onMarketSuspended(String marketId, boolean wasInPlay) {
    log.warn("Market {} suspended (in-play: {})", marketId, wasInPlay);

    if (wasInPlay) {
        // In-play suspension — prices will change on resume
        InPlayPosition position = openPositions.get(marketId);
        if (position != null) {
            position.flagSuspended();
            // Attempt cancel but expect potential failure
            try {
                orderManager.cancelAllOpenOrders(marketId);
            } catch (Exception e) {
                log.error("Cancel on suspension failed — will retry on resume", e);
                position.setPendingCancel(true);
            }
        }
    }
}

public void onMarketResumed(String marketId) {
    InPlayPosition position = openPositions.get(marketId);
    if (position == null) return;

    position.clearSuspended();

    if (position.isPendingCancel()) {
        // Retry cancel now market is live again
        orderManager.cancelAllOpenOrders(marketId);
    }

    // Reconcile — prices may have moved significantly during suspension
    orderManager.reconcileOpenOrders(marketId);
}
```

## ProTips

- **Don't carry pre-race positions into in-play unless you intend to.** Use `PersistenceType.LAPSE` on pre-race orders — they'll cancel at the off unless you explicitly convert them. An accidental in-play position at bad odds can ruin a profitable session.
- **In-play latency is network, not application, limited.** Even with a well-optimised Java application, the Betfair API introduces 50–150ms round-trip latency from most UK locations. In-play execution should acknowledge this and only target markets where 150ms round-trip is acceptable.
- **Test suspension recovery in staging.** Simulate a suspension by force-closing the streaming connection mid-position and verify your recovery logic handles it correctly. Suspension in a live race happens fast and the recovery window is short.
- **Log every in-play decision with microsecond timestamps.** When reviewing a session, you need to know whether your hedge hit at the right price and in the right time window. Millisecond timestamps are often not granular enough for in-play analysis.

If you're looking for a Java contractor who knows this space inside out, [get in touch](/hire/).
