---
title: "Betfair | Placing and Managing Orders with the Betfair API in Java"
date: 2026-07-25
tags: [Java, Betfair, Exchange API, Order Management, Trading]
style: fill
color: danger
description: >-
  How to place, update, cancel, and replace orders on Betfair in Java — covering placeOrders, cancelOrders, replaceOrders, listCurrentOrders, error handling, and reconciliation.
---

Order management is where the Betfair API gets serious. Reading markets is relatively forgiving — a failed request just means stale data. A failed order management call can mean unmatched positions, double-placed bets, or trades left open when they should be closed. I've built production order management into my Betfair trading framework over several years, and the patterns below represent the defensive approach that keeps things clean under pressure.

## placeOrders — The Core Call

`placeOrders` supports placing one or more bet instructions in a single API call:

```java
public PlaceExecutionReport placeOrders(
        String ssoid, String marketId,
        List<PlaceInstruction> instructions) throws Exception {

    PlaceOrdersRequest request = new PlaceOrdersRequest();
    request.setMarketId(marketId);
    request.setInstructions(instructions);
    request.setCustomerRef(UUID.randomUUID().toString()); // idempotency key
    request.setMarketVersion(null); // null = no version check; set for concurrency control

    PlaceExecutionReport report = betfairClient.placeOrders(ssoid, request);

    if (report.getStatus() == ExecutionReportStatus.FAILURE) {
        throw new OrderPlacementException(
            "placeOrders failed: " + report.getErrorCode(),
            report.getErrorCode()
        );
    }

    return report;
}
```

The `PlaceExecutionReport` has a top-level `status` and individual `PlaceInstructionReport` objects per instruction. Check both levels — the top-level can be SUCCESS while individual instructions have `FAILURE` status:

```java
public void processReport(PlaceExecutionReport report, List<PlaceInstruction> instructions) {
    for (int i = 0; i < report.getInstructionReports().size(); i++) {
        PlaceInstructionReport ir = report.getInstructionReports().get(i);
        PlaceInstruction instruction = instructions.get(i);

        if (ir.getStatus() == InstructionReportStatus.SUCCESS) {
            log.info("Bet placed: betId={}, matchedSize={}, averagePrice={}",
                ir.getBetId(), ir.getSizeMatched(), ir.getAveragePriceMatched());
            orderStore.record(ir.getBetId(), instruction, ir);

        } else {
            log.error("Instruction failed: errorCode={}, instruction={}",
                ir.getErrorCode(), instruction);
            handleInstructionFailure(ir.getErrorCode(), instruction);
        }
    }
}
```

## Bet Types — Back vs Lay and Persistence Types

```java
// Back bet — backing a runner to win
PlaceInstruction backInstruction = new PlaceInstruction();
backInstruction.setOrderType(OrderType.LIMIT);
backInstruction.setSelectionId(selectionId);
backInstruction.setSide(Side.BACK);

LimitOrder backLimit = new LimitOrder();
backLimit.setSize(10.0);       // £10 stake
backLimit.setPrice(4.5);       // decimal odds
backLimit.setPersistenceType(PersistenceType.LAPSE); // cancel unmatched at off
backInstruction.setLimitOrder(backLimit);

// Lay bet — laying a runner (acting as bookmaker)
PlaceInstruction layInstruction = new PlaceInstruction();
layInstruction.setOrderType(OrderType.LIMIT);
layInstruction.setSelectionId(selectionId);
layInstruction.setSide(Side.LAY);

LimitOrder layLimit = new LimitOrder();
layLimit.setSize(10.0);        // £10 backer's stake (liability = size * (price - 1))
layLimit.setPrice(4.5);
layLimit.setPersistenceType(PersistenceType.LAPSE);
layInstruction.setLimitOrder(layLimit);
```

**Persistence types:**
- `LAPSE` — cancel any unmatched portion when market goes in-play (safe default for pre-race)
- `PERSIST` — keep unmatched portion in-play (dangerous if you don't intend to trade in-running)
- `MARKET_ON_CLOSE` — BSP (Betfair Starting Price) order, matches at the BSP

## cancelOrders and updateOrders

Cancel unmatched portions:

```java
public CancelExecutionReport cancelBet(String ssoid, String marketId, String betId) {
    CancelInstruction instruction = new CancelInstruction();
    instruction.setBetId(betId);
    // instruction.setSizeReduction(5.0); // partial cancel — reduce by £5

    CancelOrdersRequest request = new CancelOrdersRequest();
    request.setMarketId(marketId);
    request.setInstructions(List.of(instruction));

    return betfairClient.cancelOrders(ssoid, request);
}
```

Update the price or size of an unmatched bet:

```java
public UpdateExecutionReport updateBetPrice(
        String ssoid, String marketId, String betId, double newPrice) {

    UpdateInstruction instruction = new UpdateInstruction();
    instruction.setBetId(betId);
    instruction.setNewPersistenceType(PersistenceType.LAPSE);
    // Note: updateOrders changes persistence type only — you cannot change price
    // Use replaceOrders to change price

    // ...
}
```

To change a bet's price, use `replaceOrders` — it cancels the old bet and places a new one atomically:

```java
public ReplaceExecutionReport replaceBetPrice(
        String ssoid, String marketId, String betId, double newPrice) {

    ReplaceInstruction instruction = new ReplaceInstruction();
    instruction.setBetId(betId);
    instruction.setNewPrice(newPrice);

    ReplaceOrdersRequest request = new ReplaceOrdersRequest();
    request.setMarketId(marketId);
    request.setInstructions(List.of(instruction));
    request.setCustomerRef(UUID.randomUUID().toString());

    return betfairClient.replaceOrders(ssoid, request);
}
```

## Listing Current Orders — Reconciliation

`listCurrentOrders` lets you see your open and recently matched bets:

```java
public CurrentOrderSummaryReport listCurrentOrders(String ssoid) {
    ListCurrentOrdersRequest request = new ListCurrentOrdersRequest();
    request.setOrderStatus(Set.of(OrderStatus.EXECUTABLE, OrderStatus.EXECUTION_COMPLETE));
    request.setDateRange(new TimeRange(Instant.now().minus(24, ChronoUnit.HOURS), Instant.now()));
    request.setRecordCount(200);

    return betfairClient.listCurrentOrders(ssoid, request);
}
```

On startup, always call `listCurrentOrders` to reconcile your in-memory order state with what Betfair actually has. If your application crashed mid-session, there may be open bets from the previous session that you're not tracking.

## Error Handling

Common error codes and appropriate responses:

```java
private void handleInstructionFailure(InstructionReportErrorCode code, PlaceInstruction instruction) {
    switch (code) {
        case INSUFFICIENT_FUNDS -> {
            log.error("Insufficient funds — activating kill switch");
            riskController.activateKillSwitch("Insufficient funds");
        }
        case MARKET_SUSPENDED -> {
            log.warn("Market suspended — order rejected, will retry when market resumes");
            // Queue for retry after suspension clears
        }
        case PRICE_TOO_HIGH, PRICE_TOO_LOW -> {
            log.warn("Price {} invalid for selection {}", 
                instruction.getLimitOrder().getPrice(), instruction.getSelectionId());
        }
        case BET_ACTION_ERROR -> {
            log.error("Bet action error — possible duplicate customerRef or invalid bet state");
        }
        default -> log.error("Unhandled error code: {}", code);
    }
}
```

## ProTips

- **Always set `customerRef`.** A UUID per order request is your idempotency key. If a request times out and you're unsure whether it was received, resend with the same `customerRef` — Betfair will not duplicate it.
- **Set `marketVersion` for safety-critical orders.** `marketVersion` lets you specify the market version you're acting on. If the market has changed (suspension, runner withdrawal) between your last price read and your order, Betfair will reject the order. Use this when entering positions based on stale data is dangerous.
- **Reconcile on startup and after disconnection.** Never assume your in-memory order state is accurate after a reconnect. Call `listCurrentOrders` and rebuild from the API response.
- **Log betId immediately.** As soon as a bet is placed and you receive a `betId`, log it. If the process crashes before you store it, you'll need the betId to cancel the position. The Betfair console and `listCurrentOrders` are your fallback.

If you're looking for a Java contractor who knows this space inside out, [get in touch](/hire/).
