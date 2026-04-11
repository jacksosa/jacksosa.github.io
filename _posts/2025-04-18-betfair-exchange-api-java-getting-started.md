---
title: Betfair | Getting Started with the Betfair Exchange API in Java
tags: [Java, Betfair, Betting Exchange, Spring Boot, API]
style: fill
color: primary
description: A practical Java guide to the Betfair Exchange API — covering SSOID authentication, the non-interactive login flow, listing markets, retrieving runner prices, and placing your first bet.
---

The Betfair Exchange API is one of the most powerful — and most misunderstood — sports betting APIs available to developers. I've been building production trading systems on top of it for years, and the number one problem I see is developers diving into order placement before they've properly understood authentication and market data. Get these foundations right, and the rest follows naturally.

## Authentication — SSOID and the Non-Interactive Login

Betfair uses a session token called an SSOID (Single Sign-On ID). Every API request requires a valid SSOID in the `X-Authentication` header. There are two login flows: interactive (browser-based) and non-interactive (certificate-based). For automated systems, you want non-interactive login.

The non-interactive flow requires:
- An approved Application Key from the Developer Portal
- A self-signed X.509 certificate uploaded to your Betfair account
- The corresponding private key on your server

The login endpoint is `https://identitysso-cert.betfair.com/api/certlogin` and accepts a POST with your username and password as form parameters, with your certificate and private key presented as mutual TLS.

```java
public class BetfairAuthClient {

    private static final String LOGIN_URL =
        "https://identitysso-cert.betfair.com/api/certlogin";

    private final HttpClient httpClient;

    public BetfairAuthClient(SSLContext sslContext) {
        this.httpClient = HttpClient.newBuilder()
            .sslContext(sslContext)
            .build();
    }

    public String login(String username, String password) throws Exception {
        String body = "username=" + URLEncoder.encode(username, StandardCharsets.UTF_8)
            + "&password=" + URLEncoder.encode(password, StandardCharsets.UTF_8);

        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(LOGIN_URL))
            .header("Content-Type", "application/x-www-form-urlencoded")
            .header("X-Application", "your-app-key")
            .POST(HttpRequest.BodyPublishers.ofString(body))
            .build();

        HttpResponse<String> response = httpClient.send(
            request, HttpResponse.BodyHandlers.ofString());

        // Response: {"token":"SSOID...","status":"SUCCESS","error":""}
        JsonNode node = new ObjectMapper().readTree(response.body());
        if (!"SUCCESS".equals(node.get("status").asText())) {
            throw new RuntimeException("Login failed: " + node.get("error").asText());
        }
        return node.get("token").asText();
    }
}
```

Build the `SSLContext` using `KeyManagerFactory` loaded with your PKCS12 or PEM certificate. Store the SSOID and refresh it before it expires (sessions last approximately 8 hours).

## Listing Event Types and Markets

Once authenticated, all requests go to `https://api.betfair.com/exchange/betting/json-rpc/v1` (JSON-RPC) or the REST endpoints at `https://api.betfair.com/exchange/betting/rest/v1.0/`. I prefer the REST endpoints — they're cleaner to work with in Java.

To find Horse Racing markets for today:

```java
public List<MarketCatalogue> getHorseRacingMarkets(String ssoid) throws Exception {
    MarketFilter filter = new MarketFilter();
    filter.setEventTypeIds(Set.of("7")); // 7 = Horse Racing
    filter.setMarketStartTime(new TimeRange(Instant.now(), Instant.now().plus(1, ChronoUnit.DAYS)));
    filter.setMarketCountries(Set.of("GB", "IE"));

    ListMarketCatalogueRequest request = new ListMarketCatalogueRequest();
    request.setFilter(filter);
    request.setMarketProjection(Set.of(
        MarketProjection.MARKET_START_TIME,
        MarketProjection.RUNNER_DESCRIPTION,
        MarketProjection.EVENT
    ));
    request.setMaxResults(100);

    // POST to /listMarketCatalogue
    return betfairRestClient.listMarketCatalogue(ssoid, request);
}
```

The response gives you `MarketCatalogue` objects containing the `marketId` (e.g. `"1.234567890"`), market name, start time, and runner names. Hold onto those `marketId` values — everything else hangs off them.

## Retrieving Runner Prices

With a `marketId`, you can retrieve live prices:

```java
public MarketBook getMarketBook(String ssoid, String marketId) throws Exception {
    PriceProjection projection = new PriceProjection();
    projection.setPriceData(Set.of(PriceData.EX_BEST_OFFERS));

    ListMarketBookRequest request = new ListMarketBookRequest();
    request.setMarketIds(List.of(marketId));
    request.setPriceProjection(projection);

    List<MarketBook> books = betfairRestClient.listMarketBook(ssoid, request);
    return books.isEmpty() ? null : books.get(0);
}
```

Each `Runner` in the `MarketBook` has an `ExchangePrices` object with:
- `availableToBack` — the best back prices on offer
- `availableToLay` — the best lay prices on offer
- `tradedVolume` — cumulative matched amounts at each price

For pre-race monitoring, I poll `listMarketBook` every 500ms–1s for the markets I'm watching. For real-time data at scale, switch to the Streaming API (covered separately) — it pushes delta updates and has no per-call overhead.

## Placing Your First Bet

Order placement uses `placeOrders`:

```java
public PlaceExecutionReport placeBackBet(
        String ssoid, String marketId, long selectionId,
        double price, double size) throws Exception {

    PlaceInstruction instruction = new PlaceInstruction();
    instruction.setOrderType(OrderType.LIMIT);
    instruction.setSelectionId(selectionId);
    instruction.setSide(Side.BACK);

    LimitOrder limitOrder = new LimitOrder();
    limitOrder.setSize(size);               // stake in GBP
    limitOrder.setPrice(price);             // decimal odds, e.g. 3.5
    limitOrder.setPersistenceType(PersistenceType.LAPSE); // cancel at off

    instruction.setLimitOrder(limitOrder);

    PlaceOrdersRequest request = new PlaceOrdersRequest();
    request.setMarketId(marketId);
    request.setInstructions(List.of(instruction));
    request.setCustomerRef(UUID.randomUUID().toString()); // idempotency key

    return betfairRestClient.placeOrders(ssoid, request);
}
```

Always check `PlaceExecutionReport.getStatus()` — `SUCCESS` means Betfair accepted the instruction, but check each `PlaceInstructionReport` for the individual bet outcome. A `FAILURE` at the report level with `ERROR_CODE` `INSUFFICIENT_FUNDS` or `MARKET_SUSPENDED` needs specific handling.

## Structuring Your Spring Boot Integration

In a Spring Boot application, I wrap the Betfair client as a `@Service` with injected configuration:

```java
@Service
public class BetfairService {

    @Value("${betfair.app-key}")
    private String appKey;

    private final BetfairAuthClient authClient;
    private final BetfairRestClient restClient;

    private volatile String currentSsoid;
    private volatile Instant sessionExpiry;

    public synchronized String getSsoid() throws Exception {
        if (currentSsoid == null || Instant.now().isAfter(sessionExpiry.minusSeconds(300))) {
            currentSsoid = authClient.login(username, password);
            sessionExpiry = Instant.now().plus(8, ChronoUnit.HOURS);
        }
        return currentSsoid;
    }
}
```

Keep the SSOID refresh logic in one place. Do not scatter auth calls across the application — you'll burn through sessions and hit rate limits fast.

## ProTips

- **Use the sandbox endpoint first.** Betfair provides `api.betfair.com` for production and `api.betfair.com` with a test SSOID via the vendor testing portal. Test order placement logic there before touching a live account.
- **Rate limits are per application key, not per account.** If you share an app key across multiple accounts in a testing setup, all calls count against the same quota.
- **`customerRef` is your idempotency key.** If a `placeOrders` call times out, resend with the same `customerRef`. Betfair will not duplicate the bet if it was already accepted.
- **Log the raw JSON.** Early in development, log every request and response body. When something goes wrong at 2am on a Saturday before a race, you will be glad you did.

If you're looking for a Java contractor who knows this space inside out, [get in touch](/hire/).
