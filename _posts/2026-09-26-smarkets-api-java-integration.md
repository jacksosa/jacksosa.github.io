---
title: "Smarkets | Smarkets API Integration for Java Developers"
date: 2026-09-26
tags: [Java, Smarkets, Betting Exchange, API, Spring Boot]
style: fill
color: info
description: >-
  Integrating with the Smarkets Exchange API in Java — REST and streaming authentication, market browsing, order placement, and building a multi-exchange abstraction alongside Betfair.
---

Smarkets is the third major UK betting exchange, and it has a few characteristics that make it worth adding to a multi-exchange trading system. The commission structure is among the lowest in the market — 2% flat. The REST and streaming APIs are well-documented and genuinely modern. And for certain political events and US sports markets, Smarkets often has comparable or superior liquidity to Betfair. If you've built Betfair integration in Java, adding Smarkets is a few days' work — and the incremental edge it provides can be meaningful.

## Smarkets API Overview

Smarkets provides two API surfaces:
- **REST API** (`api.smarkets.com/v3/`) — market browsing, order management, account operations
- **Streaming API** — WebSocket-based market data stream for live prices

Authentication uses OAuth2 tokens. API access is available via the developer portal; you'll need to apply for a trading account with API access.

## Authentication

Smarkets uses a session token obtained via login:

```java
@Service
public class SmarketsAuthService {

    private static final String AUTH_URL = "https://api.smarkets.com/v3/sessions/";

    private final RestClient restClient;

    public String login(String username, String password) {
        LoginRequest loginRequest = new LoginRequest(username, password);

        LoginResponse response = restClient.post()
            .uri(AUTH_URL)
            .contentType(MediaType.APPLICATION_JSON)
            .body(loginRequest)
            .retrieve()
            .body(LoginResponse.class);

        return response.token(); // bearer token for subsequent requests
    }
}
```

Include the token in subsequent requests:

```java
HttpHeaders headers = new HttpHeaders();
headers.setBearerAuth(token);
headers.setContentType(MediaType.APPLICATION_JSON);
```

Sessions have limited TTLs; refresh tokens before they expire, or handle 401 responses with automatic re-authentication.

## Browsing Events and Markets

```java
public List<SmarketsEvent> getHorseRacingEvents() {
    // List event types
    String response = restClient.get()
        .uri("/v3/events/?type=race&state=upcoming&sort=start_datetime")
        .header("Authorization", "Bearer " + token)
        .retrieve()
        .body(String.class);

    // Parse events from JSON response
    return objectMapper.readValue(response, EventsResponse.class).getEvents();
}

public List<SmarketsMarket> getMarketsForEvent(String eventId) {
    MarketsResponse response = restClient.get()
        .uri("/v3/events/" + eventId + "/markets/")
        .header("Authorization", "Bearer " + token)
        .retrieve()
        .body(MarketsResponse.class);

    return response.getMarkets();
}
```

The Smarkets API uses string IDs throughout. A market has a `contractId` per runner — the equivalent of Betfair's `selectionId`.

## Retrieving Prices

```java
public List<Quote> getPrices(String marketId) {
    QuotesResponse response = restClient.get()
        .uri("/v3/markets/" + marketId + "/quotes/")
        .header("Authorization", "Bearer " + token)
        .retrieve()
        .body(QuotesResponse.class);

    return response.getQuotes();
}
```

Each `Quote` contains `bid` prices (equivalent to Betfair's available-to-lay from the customer perspective) and `offer` prices (equivalent to available-to-back). The terminology maps to traditional financial markets — Smarkets was founded with a financial markets background and it shows.

## Placing Orders

```java
public OrderResponse placeBackOrder(
        String marketId, String contractId,
        String price, String quantity) {

    // Smarkets uses fractional odds represented as rationals (e.g. "4/1")
    // or decimal expressed as numerator/denominator pairs
    PlaceOrderRequest request = PlaceOrderRequest.builder()
        .marketId(marketId)
        .contractId(contractId)
        .side("buy")             // "buy" = back, "sell" = lay (financial convention)
        .price(price)            // e.g. "4.5" for decimal odds
        .quantity(quantity)      // stake in pence as a string, e.g. "1000" = £10
        .type("limit")
        .build();

    return restClient.post()
        .uri("/v3/orders/")
        .header("Authorization", "Bearer " + token)
        .contentType(MediaType.APPLICATION_JSON)
        .body(request)
        .retrieve()
        .body(OrderResponse.class);
}
```

Note the `buy`/`sell` terminology — consistent with financial markets, opposite to Betfair's `BACK`/`LAY` convention. `buy` means you want the selection to win; `sell` means you want it to lose.

## WebSocket Streaming

Smarkets provides real-time price updates via WebSocket:

```java
@Component
public class SmarketsStreamClient {

    private static final String STREAM_URL = "wss://stream.smarkets.com/";

    private WebSocketSession session;

    public void connect(List<String> marketIds) throws Exception {
        WebSocketClient client = new StandardWebSocketClient();

        session = client.execute(new TextWebSocketHandler() {
            @Override
            public void handleTextMessage(WebSocketSession session, TextMessage message) {
                processMessage(message.getPayload());
            }

            @Override
            public void afterConnectionClosed(WebSocketSession session, CloseStatus status) {
                log.warn("Smarkets stream closed: {}", status);
                scheduleReconnect();
            }
        }, STREAM_URL).get();

        // Subscribe to markets
        SubscribeRequest sub = new SubscribeRequest(marketIds);
        session.sendMessage(new TextMessage(objectMapper.writeValueAsString(sub)));
    }

    private void processMessage(String payload) {
        // Parse market update and apply to local state cache
        SmarketsMarketUpdate update = objectMapper.readValue(payload, SmarketsMarketUpdate.class);
        stateCache.apply(update);
    }
}
```

## Multi-Exchange Abstraction

A common adapter interface normalises Smarkets and Betfair behind a shared model:

```java
public interface ExchangeAdapter {

    String exchangeId();

    List<ExchangeMarket> findMarkets(ExchangeMarketFilter filter);

    ExchangePriceView getPrices(String exchangeMarketId);

    ExchangeOrderResult placeOrder(ExchangeOrder order);

    void cancelOrder(String exchangeOrderId);
}

@Component
public class SmarketsAdapter implements ExchangeAdapter {

    @Override
    public String exchangeId() { return "SMARKETS"; }

    @Override
    public ExchangeOrderResult placeOrder(ExchangeOrder order) {
        // Translate ExchangeOrder to Smarkets request
        // Handle buy/sell convention mapping
        String side = order.getSide() == Side.BACK ? "buy" : "sell";
        OrderResponse response = smarketsClient.placeOrder(
            order.getMarketId(), order.getSelectionId(),
            String.valueOf(order.getPrice()), String.valueOf((int)(order.getStake() * 100))
        );
        return ExchangeOrderResult.fromSmarkets(response);
    }
}
```

The strategy layer works only against `ExchangeAdapter`. Arbitrage opportunities between Betfair and Smarkets become detectable when both adapters report prices for correlated markets.

## Smarkets vs Betfair — Practical Notes

| Aspect | Smarkets | Betfair |
|---|---|---|
| Commission | 2% flat | 2–5% tiered |
| API style | REST + WebSocket | REST + TCP streaming |
| Streaming | WebSocket (standard) | Custom TCP/ESA protocol |
| Terminology | Financial (buy/sell) | Exchange (back/lay) |
| Liquidity | Lower overall, strong in politics/US sports | Highest in UK horse racing |
| Developer docs | Good | Excellent |

## ProTips

- **Map terminology carefully.** `buy` = back, `sell` = lay, `bid` = available-to-lay, `offer` = available-to-back — the financial-market naming is internally consistent but opposite to what Betfair developers expect.
- **Prices are decimal in the REST API.** Don't try to use fractional odds strings unless the API specifically requires them.
- **Check liquidity before deploying.** Smarkets liquidity on individual horse racing markets is typically 10–20% of Betfair's. Verify that your stake sizes are proportionate to available depth.
- **Smarkets has better API rate limits for streaming.** The WebSocket stream has no message rate limits equivalent to Betfair's ESA data charges. For heavily monitored markets, this can reduce costs compared to Betfair streaming.

If you're looking for a Java contractor who knows this space inside out, [get in touch](/hire/).
