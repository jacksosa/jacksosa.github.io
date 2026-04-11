---
title: "Java | Java Memory Management and GC Tuning for Low-Latency Systems"
tags: [Java, JVM, GC, Performance, Low Latency]
style: fill
color: warning
description: JVM memory management and GC tuning for Java low-latency systems — G1GC vs ZGC, key JVM flags, allocation pressure reduction, and GC monitoring with JFR for Betfair trading and financial data pipelines.
---

Most Java applications never need to think about GC tuning. The defaults are reasonable, GC pauses are short, and the overhead is negligible. But for systems with strict latency requirements — a Betfair trading framework where a 500ms GC pause causes you to miss a pre-race steam, or a financial data pipeline where consistent sub-10ms processing is part of the SLA — GC behaviour matters and understanding it is the difference between a system that works and one that occasionally doesn't.

## The GC Pause Problem

Every garbage collector must periodically pause application threads to collect unreachable objects. For most applications, a 50ms pause every 30 seconds is irrelevant. For a system that evaluates trading signals from a live streaming feed, a 50ms pause is a missed market event. For an in-play trading system, it's a potentially unhedged position.

Understanding the trade-off: **throughput vs pause time**. G1GC optimises for throughput with bounded pauses. ZGC optimises for minimal pause time. Serial/Parallel GC maximise throughput with no pause-time guarantees. For low-latency systems, ZGC is almost always the right choice on Java 21.

## G1GC — The Default

G1GC is the default collector since Java 9. It divides the heap into regions and prioritises collection of regions with the most garbage ("garbage-first"). Key configuration:

```bash
-XX:+UseG1GC                         # explicit (default on Java 9+)
-XX:MaxGCPauseMillis=100             # target max pause (not guaranteed)
-XX:G1NewSizePercent=20              # young generation minimum
-XX:G1MaxNewSizePercent=40           # young generation maximum
-XX:G1HeapRegionSize=8m              # region size (power of 2, 1m-32m)
-XX:InitiatingHeapOccupancyPercent=35 # start concurrent marking at 35% heap
```

G1GC's `MaxGCPauseMillis` is a target, not a hard limit. Under allocation pressure it may be breached. For my Betfair streaming data processing, G1GC worked well for pre-race analysis (latency requirements ~100ms) but was too unpredictable for in-play.

## ZGC — Pause Times Under 1ms

ZGC (available since Java 15, production-ready Java 21) is a concurrent collector designed for sub-millisecond pause times regardless of heap size. Most GC work happens concurrently with application threads:

```bash
-XX:+UseZGC
-XX:SoftMaxHeapSize=4g               # soft upper bound — ZGC may exceed in GC pressure
-Xmx6g                               # hard upper bound
-XX:ZCollectionInterval=0            # let ZGC choose collection frequency
-XX:ZUncommitDelay=300               # return memory to OS after 300s of inactivity
```

ZGC's stop-the-world pauses are limited to root scanning — typically under 1ms, often under 500μs on modern hardware. For a Betfair in-play trading system, this is transformative. The trading loop continues running while GC collects concurrently.

The trade-off: ZGC uses more CPU for concurrent work. If your system is CPU-bound, ZGC's background threads compete with application work. For IO-bound workloads (most trading systems), this is rarely a problem.

## Monitoring GC Behaviour

Don't tune without measuring first. Enable GC logging:

```bash
-Xlog:gc*:file=/var/log/app/gc.log:time,uptime,level,tags:filecount=5,filesize=20m
```

This writes structured GC logs you can analyse with tools like GCViewer or GCEasy. Look for:
- Pause times (`GC(0) Pause Young (Normal) 150ms` — too high for low-latency)
- Frequency of full GC events (`GC(5) Pause Full (Ergonomics)` — should be rare)
- Heap occupancy trends (growing heap means memory leak or insufficient size)

JDK Flight Recorder is even more powerful:

```bash
-XX:+FlightRecorder
-XX:StartFlightRecording=duration=60s,filename=/tmp/recording.jfr,settings=profile
```

In JDK Mission Control, the GC events view shows pause distribution, allocation rates, and top allocation sites. The allocation profiler identifies which code paths allocate most — the first step in reducing GC pressure.

## Reducing Allocation Pressure

The best GC optimisation is allocating less. Common patterns in high-throughput Java systems:

**Object pooling for frequently created/discarded objects:**

```java
public class MarketUpdatePool {

    private final Queue<MarketUpdate> pool = new ConcurrentLinkedQueue<>();

    public MarketUpdate acquire() {
        MarketUpdate update = pool.poll();
        return update != null ? update.reset() : new MarketUpdate();
    }

    public void release(MarketUpdate update) {
        pool.offer(update);
    }
}
```

**Avoid boxing — use primitive collections:**

```java
// Bad — boxes every long to Long, creates pressure
Map<Long, Double> prices = new HashMap<>();

// Better — primitive long-to-double map (Eclipse Collections or Trove)
LongDoubleHashMap prices = new LongDoubleHashMap();
```

**Reuse buffers in the streaming data path:**

```java
// Reuse ByteBuffer across JSON parses rather than allocating per message
private final byte[] parseBuffer = new byte[64 * 1024];
private final JsonParser parser = jsonFactory.createParser(parseBuffer);
```

## JVM Flags for Betfair Trading Systems

The flags I use in production for a Betfair trading framework on a 16-core server with 32GB RAM:

```bash
# Java 21, ZGC, 8GB heap for the trading service
java \
  -XX:+UseZGC \
  -Xms4g -Xmx8g \
  -XX:SoftMaxHeapSize=6g \
  -XX:+ZGenerational \         # generational ZGC (Java 21+) — better throughput
  -XX:+AlwaysPreTouch \        # pre-touch pages at startup — avoids OS page faults at runtime
  -XX:+DisableExplicitGC \     # ignore System.gc() calls from libraries
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/var/dumps/ \
  -Xlog:gc*:file=/var/log/gc.log:time,uptime:filecount=5,filesize=10m \
  -jar trading-framework.jar
```

`-XX:+ZGenerational` is available in Java 21 and makes ZGC significantly more efficient by separating young and old generation collection — recommended for most applications.

## ProTips

- **Start with ZGC on Java 21 for any latency-sensitive system.** The performance characteristics are predictable and the configuration is simpler than G1GC tuning.
- **Monitor allocation rate, not just heap size.** A system allocating 500MB/s with a 4GB heap will GC constantly. A system allocating 50MB/s with the same heap will GC rarely. `jcmd <pid> GC.run` + allocation profiling reveals the hottest allocation sites.
- **Don't increase heap size as a first resort.** More heap delays full GC but doesn't fix the underlying allocation pattern. Fix the root cause; use heap sizing to tune the frequency.
- **`-XX:+AlwaysPreTouch` is worth it for long-running services.** Without it, the OS lazily allocates physical pages, causing page fault latency spikes early in the application lifecycle. Pre-touch eliminates this at the cost of longer startup.

If you're looking for a Java contractor who knows this space inside out, [get in touch](/hire/).
