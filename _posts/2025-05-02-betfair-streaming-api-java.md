---
title: "Betfair | Understanding the Betfair Streaming API in Java"
tags: [Java, Betfair, Streaming API, WebSocket, Spring Boot]
style: fill
color: warning
description: How the Betfair Streaming API (ESA) works and how to build a robust Java WebSocket client — covering the protocol, market change messages, local state caching, and reconnection logic.
---

If you're polling `listMarketBook` to get Betfair prices, you're already behind. Every poll costs an API call against your quota, introduces latency proportional to your polling interval, and gets you a full snapshot when you only need the delta. The Streaming API — formally known as the Exchange Streaming API (ESA) — solves all three problems. It pushes changes to you the moment they occur on the exchange. In my trading framework, switching from polling to streaming cut market data latency from ~500ms to under 50ms.

## How the ESA Protocol Works

The Streaming API is a persistent TCP socket connection (TLS on port 443). The protocol is line-delimited JSON — each message is a JSON object terminated by a newline. There are two sides: messages you send (authentication, subscription), and messages you receive (connection, market change, heartbeat).

Connection sequence:
1. Open a TLS socket to `stream-api.betfair.com:443`
2. Receive a `connection` message with a `connectionId`
3. Send an `authentication` message with your SSOID and app key
4. Receive a `status` message confirming authentication
5. Send a `marketSubscription` message specifying which markets and data fields you want
6. Receive a `marketChange` message with full initial state (image), then deltas thereafter

## Building the Java Client

I use a raw `SSLSocket` rather than a WebSocket library — the ESA is TCP, not WebSocket. Here's the connection setup:

```java
public class BetfairStreamClient implements Runnable {

    private static final String HOST = "stream-api.betfair.com";
    private static final int PORT = 443;

    private final String ssoid;
    private final String appKey;
    private final MarketStateCache cache;
    private final ObjectMapper mapper = new ObjectMapper();

    private SSLSocket socket;
    private BufferedReader reader;
    private PrintWriter writer;

    public void connect() throws IOException {
        SSLSocketFactory factory = (SSLSocketFactory) SSLSocketFactory.getDefault();
        socket = (SSLSocket) factory.createSocket(HOST, PORT);
        socket.setSoTimeout(30_000); // 30s read timeout — heartbeat arrives every 5s

        reader = new BufferedReader(new InputStreamReader(
            socket.getInputStream(), StandardCharsets.UTF_8));
        writer = new PrintWriter(new OutputStreamWriter(
            socket.getOutputStream(), StandardCharsets.UTF_8), true);
    }

    @Override
    public void run() {
        try {
            connect();
            authenticate();
            subscribe();
            readLoop();
        } catch (Exception e) {
            log.error("Stream client error — scheduling reconnect", e);
            scheduleReconnect();
        }
    }
}
```

Authentication sends your SSOID and app key as a JSON message:

```java
private void authenticate() throws IOException {
    AuthenticationMessage auth = new AuthenticationMessage();
    auth.setOp("authentication");
    auth.setId(1);
    auth.setSession(ssoid);
    auth.setAppKey(appKey);
    writer.println(mapper.writeValueAsString(auth));

    String response = reader.readLine();
    StatusMessage status = mapper.readValue(response, StatusMessage.class);
    if (status.getStatusCode() != 200) {
        throw new RuntimeException("Authentication failed: " + status.getErrorMessage());
    }
}
```

## Subscribing to Markets

A subscription message tells Betfair which markets you care about and what data you want:

```java
private void subscribe() throws IOException {
    MarketSubscriptionMessage sub = new MarketSubscriptionMessage();
    sub.setOp("marketSubscription");
    sub.setId(2);

    // Which markets
    MarketFilter filter = new MarketFilter();
    filter.setMarketIds(List.of("1.234567890", "1.234567891"));
    sub.setMarketFilter(filter);

    // What data fields
    MarketDataFilter dataFilter = new MarketDataFilter();
    dataFilter.setFields(List.of(
        "EX_BEST_OFFERS",    // best back/lay prices
        "EX_LTP",            // last traded price
        "EX_TRADED_VOL"      // traded volume
    ));
    dataFilter.setLadderLevels(3); // top 3 levels of the ladder
    sub.setMarketDataFilter(dataFilter);

    writer.println(mapper.writeValueAsString(sub));
}
```

## Maintaining Local State from Delta Messages

This is where most implementations go wrong. ESA sends an initial image (`img: true`) containing the full state of each market, followed by delta messages containing only what changed. You must apply each delta to your local cache to maintain the current view.

```java
public class MarketStateCache {

    private final Map<String, MarketState> markets = new ConcurrentHashMap<>();

    public void applyMarketChange(MarketChangeMessage mcm) {
        if (mcm.getMarketChanges() == null) return;

        for (MarketChange mc : mcm.getMarketChanges()) {
            String marketId = mc.getId();
            MarketState state = markets.computeIfAbsent(marketId, MarketState::new);

            if (Boolean.TRUE.equals(mc.getImg())) {
                // Full image — replace state entirely
                state.reset();
            }

            // Apply runner changes
            if (mc.getRunnerChanges() != null) {
                for (RunnerChange rc : mc.getRunnerChanges()) {
                    state.applyRunnerChange(rc);
                }
            }

            // Apply market definition changes (in-play status, status, etc.)
            if (mc.getMarketDefinition() != null) {
                state.applyDefinition(mc.getMarketDefinition());
            }
        }
    }

    public Optional<MarketState> getMarket(String marketId) {
        return Optional.ofNullable(markets.get(marketId));
    }
}
```

The `RunnerChange` contains price ladder updates as a list of `[price, size]` pairs. A size of `0` means remove that price level. Applying this correctly to maintain a sorted ladder is the most error-prone part of the implementation — test it carefully against known streaming data.

## Reconnection Logic

The stream will disconnect. Network hiccups, Betfair maintenance windows, server restarts — they all happen. Your client must reconnect and re-subscribe automatically:

```java
private void scheduleReconnect() {
    int delaySeconds = Math.min(30, reconnectAttempts * 5);
    scheduler.schedule(() -> {
        reconnectAttempts++;
        try {
            closeQuietly();
            connect();
            authenticate();
            // Re-subscribe with the same markets
            // But handle the case where markets may have closed
            resubscribe();
            reconnectAttempts = 0;
        } catch (Exception e) {
            log.error("Reconnect attempt {} failed", reconnectAttempts, e);
            scheduleReconnect();
        }
    }, delaySeconds, TimeUnit.SECONDS);
}
```

Use exponential backoff — hammering a failed connection achieves nothing and burns API quota. Also refresh your SSOID before reconnecting — it may have expired during a long outage.

## ProTips

- **Handle heartbeats.** Betfair sends a heartbeat message every 5 seconds. If you don't receive one within 30 seconds, assume the connection is dead and reconnect. Set `socket.setSoTimeout(30_000)` to get an automatic `SocketTimeoutException` if the stream goes silent.
- **Subscribe to multiple markets in one message.** A single subscription message can cover many markets — do not send one subscription per market, you'll hit rate limits.
- **Don't process change messages on the IO thread.** Put received messages onto a `LinkedBlockingQueue` and process them on a separate thread. IO thread slowdowns will cause you to miss messages otherwise.
- **Test reconnection in staging.** Close the socket forcibly (`socket.close()`) while it's running and verify your reconnect logic recovers cleanly. Most reconnection bugs only surface under production conditions.

If you're looking for a Java contractor who knows this space inside out, [get in touch](/hire/).
