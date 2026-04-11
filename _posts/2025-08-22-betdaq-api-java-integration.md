---
title: "Betdaq | Betdaq API Integration for Java Developers"
tags: [Java, Betdaq, Betting Exchange, API, Spring Boot]
style: fill
color: warning
description: A practical guide to integrating with the Betdaq Exchange API in Java — authentication, market browsing, placing orders, and building an abstraction layer for multi-exchange trading systems.
---

Betdaq is the second-largest betting exchange in the world, and it remains significantly underexplored by Java developers compared to Betfair. That's partly because the Betfair API has better documentation and a larger developer community, and partly because Betdaq's liquidity is lower in most markets. But Betdaq has a commission structure that's meaningfully cheaper than Betfair's, genuine liquidity in certain sport types, and an API that's straightforward to work with once you understand its structure. If you're building a serious exchange trading system, you should evaluate Betdaq alongside Betfair.

## Betdaq API Overview

Betdaq provides a SOAP/WSDL API — older than Betfair's REST API, but functional. The core operations you'll need are:

- `GetListTopLevelEvents` — top-level event categories
- `GetEventSubTree` — markets within an event type  
- `GetPrices` — current prices for one or more markets
- `SubmitOrdersNoReceipt` / `SubmitOrders` — order placement
- `GetOrderDetails` — check order status
- `CancelOrders` / `UpdateOrders` — order management

The SOAP endpoint is `https://api.betdaq.com/v2.0/BetDAQAPI.asmx`.

## Generating a Java Client

With Maven and the JAX-WS plugin, generate a Java client from the Betdaq WSDL:

```xml
<plugin>
    <groupId>com.sun.xml.ws</groupId>
    <artifactId>jaxws-maven-plugin</artifactId>
    <version>4.0.1</version>
    <executions>
        <execution>
            <goals><goal>wsimport</goal></goals>
            <configuration>
                <wsdlUrls>
                    <wsdlUrl>https://api.betdaq.com/v2.0/BetDAQAPI.asmx?WSDL</wsdlUrl>
                </wsdlUrls>
                <packageName>com.example.betdaq.client</packageName>
                <sourceDestDir>${project.build.directory}/generated-sources</sourceDestDir>
            </configuration>
        </execution>
    </executions>
</plugin>
```

The generated client provides strongly-typed request/response objects. Wrap it in a Spring `@Service` that handles authentication and error translation.

## Authentication

Betdaq uses username/password authentication with a session token header. Unlike Betfair's certificate-based non-interactive login, Betdaq authentication is simpler — pass credentials in the SOAP header:

```java
@Service
public class BetdaqAuthService {

    @Value("${betdaq.username}")
    private String username;

    @Value("${betdaq.password}")
    private String password;

    public void addAuthHeader(BindingProvider provider) {
        Map<String, Object> headers = new HashMap<>();
        headers.put("username", username);
        headers.put("password", password);

        provider.getRequestContext().put(
            MessageContext.HTTP_REQUEST_HEADERS,
            Map.of("username", List.of(username),
                   "password", List.of(password))
        );
    }
}
```

In practice, I wrap the generated SOAP stub in an authenticated proxy:

```java
@Bean
public BetDAQAPI betdaqApi() {
    BetDAQAPI service = new BetDAQAPI();
    BetDAQAPIService port = service.getBetDAQAPIServiceSoap();

    BindingProvider provider = (BindingProvider) port;
    provider.getRequestContext().put(
        BindingProvider.ENDPOINT_ADDRESS_PROPERTY,
        "https://api.betdaq.com/v2.0/BetDAQAPI.asmx"
    );

    authService.addAuthHeader(provider);
    return service;
}
```

## Browsing Markets and Retrieving Prices

Fetch markets for a specific event type:

```java
public List<EventClassifier> getHorseRacingMarkets() {
    GetEventSubTreeRequest request = new GetEventSubTreeRequest();
    request.setEventClassifierIds(List.of(100004L)); // 100004 = Horse Racing on Betdaq
    request.setWantDirectDescendentsOnly(false);

    GetEventSubTreeResponse response = api.getEventSubTree(request);

    return response.getEventClassifiers().stream()
        .filter(e -> e.getMarkets() != null)
        .flatMap(e -> e.getMarkets().stream())
        .filter(m -> m.getStatus() == MarketStatusEnum.ACTIVE)
        .toList();
}
```

Retrieve prices for specific markets:

```java
public List<MarketPrices> getPrices(List<Long> marketIds) {
    GetPricesRequest request = new GetPricesRequest();
    request.setMarketIds(marketIds);
    request.setThresholdAmount(0.0);  // minimum size to include in price ladder
    request.setNumberOfRunners(20);
    request.setNumberForPricesRequired(3); // top 3 back/lay levels

    GetPricesResponse response = api.getPrices(request);
    return response.getMarketPrices();
}
```

The `MarketPrices` object contains `RunnerPrices` with `ForSidePrices` (lay, from Betdaq's bookmaker perspective) and `AgainstSidePrices` (back). Note that Betdaq uses the opposite naming convention to Betfair — "For" means lay, "Against" means back — which is a source of confusion for developers moving between APIs.

## Placing Orders

```java
public SubmitOrdersResponse submitBackBet(
        long marketId, long selectionId,
        double price, double stake) {

    Order order = new Order();
    order.setSelectionId(selectionId);
    order.setStake(stake);          // backer's stake
    order.setPrice(price);          // decimal odds
    order.setPolarity(OrderPolarity.FOR); // FOR = lay, AGAINST = back — confusing but correct
    order.setExpectedSelectionResetCount(0);
    order.setExpectedWithdrawalSequenceNumber(0);

    SubmitOrdersNoReceiptRequest request = new SubmitOrdersNoReceiptRequest();
    request.setMarketId(marketId);
    request.setOrders(List.of(order));
    request.setWantAllOrNothingBehaviour(false);

    return api.submitOrdersNoReceipt(request);
}
```

`SubmitOrdersNoReceipt` is the fast order submission path — it returns immediately without waiting for order confirmation. Use `SubmitOrders` if you need a confirmed receipt, at the cost of slightly higher latency.

## Building a Multi-Exchange Abstraction

If you're running strategies on both Betfair and Betdaq, a common abstraction prevents your strategy code from caring about exchange-specific API details:

```java
public interface ExchangeAdapter {
    String exchangeName();
    List<ExchangeMarket> getMarkets(MarketFilter filter);
    MarketPriceView getPrices(String marketId);
    OrderResult placeOrder(String marketId, String selectionId, Side side, double price, double stake);
    void cancelOrder(String orderId);
}

@Component("betdaqAdapter")
public class BetdaqAdapter implements ExchangeAdapter {
    // translate ExchangeMarket to/from Betdaq types
}

@Component("betfairAdapter")
public class BetfairAdapter implements ExchangeAdapter {
    // translate ExchangeMarket to/from Betfair types
}
```

The abstraction normalises market data across exchanges. The strategy layer works against `ExchangeAdapter` — it never imports Betfair or Betdaq specific types.

## Betdaq vs Betfair — Key Differences

| Aspect | Betfair | Betdaq |
|---|---|---|
| Commission | 2–5% (tiered by activity) | 2% standard, lower for high volume |
| API style | REST + WebSocket streaming | SOAP (older) |
| Liquidity | Market leader in most sports | Strong in football, horse racing (lower than BF) |
| Market coverage | Broader | Narrower |
| Streaming API | Yes (ESA) | No — polling only |

The lack of a streaming API on Betdaq is the biggest operational difference. You must poll `GetPrices` at intervals rather than receiving pushed updates, which increases latency and API call volume.

## ProTips

- **Betdaq's `FOR`/`AGAINST` naming is back-to-front.** `FOR` means you're laying a selection (you want it to lose). `AGAINST` means you're backing it. Check this three times before deploying order placement code.
- **Poll at modest intervals.** With no streaming API, polling `GetPrices` every 2–5 seconds is reasonable for pre-race markets. Polling faster risks API throttling.
- **Leverage Betdaq's commission for profitable strategies.** For strategies that are marginally profitable on Betfair after commission, Betdaq's lower commission may make them viable.
- **Check liquidity before deploying.** The same race will have 10× more matched volume on Betfair than Betdaq in most cases. Verify the Betdaq market has enough depth for your stake sizes before running live.

If you're looking for a Java contractor who knows this space inside out, [get in touch](/hire/).
