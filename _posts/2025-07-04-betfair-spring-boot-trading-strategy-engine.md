---
title: "Betfair | Building a Spring Boot Trading Strategy Engine"
date: 2026-07-04
tags: [Java, Spring Boot, Betfair, Trading, Architecture]
style: fill
color: primary
description: >-
  How to architect a pluggable trading strategy engine in Spring Boot for Betfair — strategy interfaces, event-driven signal evaluation, lifecycle management, and risk controls.
---

A trading strategy engine is the core of any automated Betfair system. Get the architecture right and adding, removing, or tweaking strategies becomes a low-risk configuration change. Get it wrong and every strategy change is a deployment risk, strategies share state they shouldn't, and risk controls are a bolt-on afterthought. I've built production trading frameworks and the patterns that follow represent what actually works under live market conditions.

## The Strategy Interface

Every strategy implements a common interface. The interface is deliberately narrow:

```java
public interface TradingStrategy {

    /**
     * Unique identifier for this strategy. Used in logs, metrics, and config.
     */
    String strategyId();

    /**
     * Market filter — which markets should this strategy receive signals for?
     */
    MarketFilter marketFilter();

    /**
     * Called when a new market matching the filter becomes available.
     */
    void onMarketOpen(MarketContext market);

    /**
     * Called when market signals update (typically on each streaming delta).
     */
    void onSignalUpdate(MarketContext market, MarketSignals signals);

    /**
     * Called when a market closes (settled, suspended, or system shutdown).
     */
    void onMarketClose(MarketContext market);
}
```

`MarketContext` provides read-only access to the current market state: runners, LTP, matched volume, time to off, current positions. `MarketSignals` contains pre-calculated WoM, LTP pattern, and OFI values. The strategy reads signals and market context; it does not interact with the Betfair API directly. All order placement goes through the `OrderManager`.

## Spring Injection and Strategy Registration

Spring's dependency injection makes strategies swappable without touching the engine:

```java
@Component
public class SteamFollowerStrategy implements TradingStrategy {

    private final OrderManager orderManager;
    private final RiskController riskController;

    @Override
    public String strategyId() { return "steam-follower-v2"; }

    @Override
    public MarketFilter marketFilter() {
        return MarketFilter.builder()
            .eventTypes(Set.of("7"))        // horse racing
            .countries(Set.of("GB", "IE"))
            .minTotalMatched(50_000)
            .maxMinutesToOff(30)
            .minMinutesToOff(3)
            .build();
    }

    @Override
    public void onSignalUpdate(MarketContext market, MarketSignals signals) {
        signals.getRunnerSignals().forEach((selectionId, signal) -> {
            if (signal.compositeSignal() == TradingSignal.STRONG_STEAM
                    && !market.hasOpenPosition(selectionId)
                    && riskController.canEnter(market, selectionId)) {

                double backPrice = market.getBestBackPrice(selectionId);
                orderManager.placeBackBet(market.getMarketId(), selectionId, backPrice, 10.0);
            }
        });
    }
}
```

The engine collects all `TradingStrategy` beans at startup:

```java
@Service
public class StrategyEngine {

    private final List<TradingStrategy> strategies;
    private final Map<String, MarketContext> activeMarkets = new ConcurrentHashMap<>();

    public StrategyEngine(List<TradingStrategy> strategies) {
        this.strategies = strategies;
        log.info("Loaded {} strategies: {}", strategies.size(),
            strategies.stream().map(TradingStrategy::strategyId).toList());
    }

    public void onSignalUpdate(String marketId, MarketSignals signals) {
        MarketContext market = activeMarkets.get(marketId);
        if (market == null) return;

        strategies.stream()
            .filter(s -> s.marketFilter().matches(market))
            .forEach(s -> s.onSignalUpdate(market, signals));
    }
}
```

Spring's `List<TradingStrategy>` injection automatically collects every `TradingStrategy` bean in the context. Add a new strategy class with `@Component` and it's live on next deployment.

## Market State Management

Each active market has a `MarketContext` that maintains current state:

```java
public class MarketContext {

    private final String marketId;
    private final MarketDefinition definition;
    private final Map<Long, Position> positions = new ConcurrentHashMap<>();
    private volatile MarketState currentState;

    public boolean hasOpenPosition(long selectionId) {
        Position pos = positions.get(selectionId);
        return pos != null && pos.isOpen();
    }

    public double getBestBackPrice(long selectionId) {
        return currentState.getRunner(selectionId)
            .flatMap(r -> r.getBestBackPrice())
            .orElse(0.0);
    }

    public int minutesUntilOff() {
        return (int) ChronoUnit.MINUTES.between(
            Instant.now(), definition.getMarketTime());
    }
}
```

`MarketContext` is the read side. Strategies never write directly to it — mutations go through the engine which coordinates updates.

## Risk Controls

Risk controls live in a `RiskController` that sits between strategies and order placement. Strategies call `riskController.canEnter()` before placing any order; the controller enforces the guardrails:

```java
@Component
public class RiskController {

    @Value("${risk.max-stake-per-bet:20.0}")
    private double maxStakePerBet;

    @Value("${risk.max-liability-per-market:100.0}")
    private double maxLiabilityPerMarket;

    @Value("${risk.max-daily-loss:500.0}")
    private double maxDailyLoss;

    private final AtomicBoolean killSwitchActive = new AtomicBoolean(false);
    private volatile double dailyPnl = 0.0;

    public boolean canEnter(MarketContext market, long selectionId) {
        if (killSwitchActive.get()) {
            log.warn("Kill switch active — blocking all new positions");
            return false;
        }
        if (dailyPnl < -maxDailyLoss) {
            log.warn("Daily loss limit hit (£{:.2f}) — blocking new positions", -dailyPnl);
            return false;
        }
        double currentLiability = market.getTotalLiability();
        if (currentLiability >= maxLiabilityPerMarket) {
            return false;
        }
        return true;
    }

    public void activateKillSwitch(String reason) {
        log.error("KILL SWITCH ACTIVATED: {}", reason);
        killSwitchActive.set(true);
        // Alert — Slack, PagerDuty, email
    }

    public void updatePnl(double pnl) {
        this.dailyPnl += pnl;
    }
}
```

The kill switch is accessible via an Actuator endpoint for manual intervention:

```java
@RestController
@RequestMapping("/actuator/trading")
public class TradingActuatorEndpoint {

    private final RiskController riskController;

    @PostMapping("/kill-switch")
    public ResponseEntity<String> activateKillSwitch(@RequestBody KillSwitchRequest req) {
        riskController.activateKillSwitch(req.reason());
        return ResponseEntity.ok("Kill switch activated");
    }
}
```

## ProTips

- **Strategies must be stateless or own their state explicitly.** The engine calls strategies concurrently for different markets. If a strategy holds mutable fields that aren't thread-safe, you'll get data corruption. Use `ConcurrentHashMap` or confine mutable state to `@Prototype` scoped helper beans.
- **Log every decision with context.** When a strategy declines to trade, log why (risk limit, conflicting signals, no position available). Post-session review of rejected signals is as important as review of taken positions.
- **Test strategies with replayed streaming data.** Build a replay harness that feeds recorded streaming data through the engine. You can test strategy logic without a live Betfair connection and reproduce exactly what happened in past races.
- **Separate paper trading from live trading.** Implement a `PaperOrderManager` that logs order decisions without sending them to Betfair. Run every new strategy in paper trading mode for at least two weeks before going live.

If you're looking for a Java contractor who knows this space inside out, [get in touch](/hire/).
