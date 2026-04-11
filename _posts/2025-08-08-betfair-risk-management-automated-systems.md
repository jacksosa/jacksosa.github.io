---
title: "Betfair | Risk Management in Automated Betting Systems"
tags: [Java, Betfair, Risk Management, Trading, Architecture]
style: fill
color: primary
description: Risk management architecture for automated Betfair trading in Java — stake limits, liability caps, drawdown monitoring, kill switch implementation, and graceful shutdown on API disconnection.
---

Of everything involved in building automated trading systems, risk management is the topic traders most often underinvest in — until the day they wish they hadn't. I've spoken to developers who built strategies that worked beautifully in testing, deployed them live, and then watched them place hundreds of bets in under a minute when an edge case triggered a strategy loop. The losses were significant and entirely preventable. Good risk architecture isn't optional — it's the foundation everything else sits on.

## The Risk Layer Architecture

Risk controls should be a distinct layer in your architecture, not scattered conditionals in strategy code. Every order request passes through `RiskController` before reaching the Betfair API. Nothing bypasses it:

```
Strategy → RiskController → OrderManager → Betfair API
                ↑
         (blocks or allows)
```

The strategy is responsible for signal evaluation and position sizing. The risk layer is responsible for enforcing hard limits regardless of what the strategy wants to do.

## Per-Bet Stake Limits

The simplest and most important control: cap the maximum stake on any single bet:

```java
@Component
public class RiskController {

    @Value("${risk.max-stake-per-bet:20.0}")
    private double maxStakePerBet;

    @Value("${risk.max-lay-liability-per-bet:100.0}")
    private double maxLayLiabilityPerBet;

    public OrderValidationResult validateOrder(PlaceInstruction instruction) {
        double size = instruction.getLimitOrder().getSize();
        double price = instruction.getLimitOrder().getPrice();

        if (size > maxStakePerBet) {
            return OrderValidationResult.rejected(
                "Stake £" + size + " exceeds limit £" + maxStakePerBet);
        }

        if (instruction.getSide() == Side.LAY) {
            double liability = size * (price - 1);
            if (liability > maxLayLiabilityPerBet) {
                return OrderValidationResult.rejected(
                    "Lay liability £" + liability + " exceeds limit £" + maxLayLiabilityPerBet);
            }
        }

        return OrderValidationResult.approved();
    }
}
```

Lay bets have liability = stake × (price - 1). At price 10.0 with a £10 stake, your liability is £90 — always calculate this explicitly.

## Per-Market Liability Caps

Track accumulated liability per market and cap it:

```java
@Component
public class MarketExposureTracker {

    // marketId -> net liability (positive = at risk)
    private final ConcurrentHashMap<String, AtomicLong> liabilityByMarket =
        new ConcurrentHashMap<>();

    @Value("${risk.max-liability-per-market:200}")
    private long maxLiabilityPerMarketPence; // store in pence to avoid floating point issues

    public boolean canAcceptLiability(String marketId, double additionalLiabilityGbp) {
        long additionalPence = Math.round(additionalLiabilityGbp * 100);
        long current = liabilityByMarket
            .getOrDefault(marketId, new AtomicLong(0))
            .get();

        return (current + additionalPence) <= maxLiabilityPerMarketPence;
    }

    public void recordOrderPlaced(String marketId, double liabilityGbp) {
        liabilityByMarket
            .computeIfAbsent(marketId, id -> new AtomicLong(0))
            .addAndGet(Math.round(liabilityGbp * 100));
    }

    public void recordOrderSettled(String marketId, double pnlGbp) {
        // Reduce liability by the settled amount
        liabilityByMarket
            .computeIfAbsent(marketId, id -> new AtomicLong(0))
            .addAndGet(-Math.round(Math.abs(pnlGbp) * 100));
    }
}
```

Store monetary values in pence (integer) rather than pounds (double) to avoid floating-point rounding errors accumulating over thousands of calculations.

## Drawdown Monitoring and Daily Loss Limits

Track P&L from settled bets and stop trading when the daily loss limit is hit:

```java
@Component
public class DrawdownMonitor {

    @Value("${risk.max-daily-loss-gbp:500.0}")
    private double maxDailyLossGbp;

    @Value("${risk.max-drawdown-gbp:200.0}")  // from peak
    private double maxDrawdownGbp;

    private volatile double dailyPnl = 0.0;
    private volatile double peakDailyPnl = 0.0;
    private final RiskController riskController;

    public void recordSettledBet(double pnlGbp) {
        dailyPnl += pnlGbp;
        peakDailyPnl = Math.max(peakDailyPnl, dailyPnl);

        double drawdown = peakDailyPnl - dailyPnl;

        if (dailyPnl < -maxDailyLossGbp) {
            riskController.activateKillSwitch(
                String.format("Daily loss limit: P&L = -£%.2f", -dailyPnl));
        }

        if (drawdown > maxDrawdownGbp) {
            riskController.activateKillSwitch(
                String.format("Drawdown limit: Peak P&L = £%.2f, Current = £%.2f",
                    peakDailyPnl, dailyPnl));
        }
    }

    @Scheduled(cron = "0 0 8 * * *")  // Reset at 8am each morning
    public void resetDailyMetrics() {
        dailyPnl = 0.0;
        peakDailyPnl = 0.0;
        log.info("Daily P&L metrics reset");
    }
}
```

## The Kill Switch

The kill switch must be fast, reliable, and recoverable:

```java
@Component
public class RiskController {

    private final AtomicBoolean killSwitchActive = new AtomicBoolean(false);
    private volatile String killSwitchReason;
    private final OrderManager orderManager;
    private final AlertService alertService;

    public void activateKillSwitch(String reason) {
        if (killSwitchActive.compareAndSet(false, true)) {
            this.killSwitchReason = reason;
            log.error("KILL SWITCH ACTIVATED: {}", reason);
            alertService.sendAlert("KILL SWITCH", reason);

            // Cancel all open orders
            orderManager.cancelAllOpenOrders();
        }
    }

    public boolean isKillSwitchActive() { return killSwitchActive.get(); }

    // Manual reset via Actuator — requires deliberate operator action
    public void resetKillSwitch(String operatorId) {
        killSwitchActive.set(false);
        log.warn("Kill switch reset by operator: {}", operatorId);
        alertService.sendAlert("KILL SWITCH RESET", "Reset by " + operatorId);
    }
}
```

`compareAndSet` ensures the kill switch activates exactly once — multiple threads hitting the limit simultaneously won't trigger multiple alert floods.

## Graceful Shutdown on API Disconnection

When the streaming connection drops, your position state is unknown until you reconnect. On disconnection:

```java
@EventListener
public void onStreamDisconnected(StreamDisconnectedEvent event) {
    log.error("Betfair stream disconnected — suspending strategy evaluation");
    strategyEngine.suspend(); // stop evaluating signals, no new orders

    if (event.getDuration().toSeconds() > 60) {
        // Long disconnection — cancel all open orders via REST API
        orderManager.cancelAllOpenOrders();
        riskController.activateKillSwitch("Prolonged stream disconnection");
    }
    // Short disconnection — reconnect and reconcile before resuming
}

@EventListener
public void onStreamReconnected(StreamReconnectedEvent event) {
    orderManager.reconcileOpenOrders(); // rebuild state from listCurrentOrders
    strategyEngine.resume();
    log.info("Stream reconnected and reconciled — strategy evaluation resumed");
}
```

## ProTips

- **Risk parameters in configuration, not code.** `${risk.max-stake-per-bet}` means you can change limits without a deployment. Use `@ConfigurationProperties` with `@Validated` to catch misconfiguration at startup.
- **Persist kill switch state.** If the application crashes, the kill switch should still be active on restart. Store it in a file or database rather than purely in memory.
- **Alert on every risk limit approach, not just breach.** An alert when daily loss reaches 80% of the limit gives you time to intervene manually before the hard stop triggers.
- **Simulate failure modes in testing.** Inject artificial losses, simulate stream disconnections, and verify the kill switch and order cancellation logic work exactly as designed. Don't wait for a live incident to discover a gap.

If you're looking for a Java contractor who knows this space inside out, [get in touch](/hire/).
