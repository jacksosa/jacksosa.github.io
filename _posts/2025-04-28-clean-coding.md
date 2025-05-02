---
title: Clean Coding | Four Principles to Boost Your Team’s Productivity
tags: [ Software Development, Clean Code, Junit, Java, TDD, testcontainers, Mockito, Spring Boot, Apache Kafka ]
style: fill
color: info
description: Four clean code principle - testing, naming, SRP, & side-effect-free functions, that helped transform my projects and doubled productivity.
---

---
As a Java developer with a decade of experience delivering projects like the Ribby Hall Village data warehouse, Co-op’s
competitor pricing reports, ESG Global’s BOL Engine, and a real-time API ingestion pipeline for Mosaic Smart Data, I’ve
wrestled with codebases that were more liability than asset. Early in my career, I encountered systems plagued by
spaghetti logic, rampant duplication, and nonexistent tests. A simple bug fix could spiral into hours of frustration,
spiking our “WTFs per minute” and tanking productivity.

Inspired by Robert Martin’s Clean Code, I made clean coding a cornerstone of my work, transforming chaotic systems into
maintainable ones. For instance, at Mosaic Smart Data, I built a real-time API ingestion pipeline for financial market
analytics, processing high-velocity transaction and communication data with Spring Boot and Apache Kafka. By applying
clean code principles, I ensured the pipeline was robust, scalable, and easy to maintain, delivering actionable insights
to FICC desks with sub-second latency. These practices doubled productivity across my projects and made development a
joy. Here are the four principles that drove 80% of the gains.

## 1. “If It Isn’t Tested, It’s Broken”

I’ve lived the pain of untested code breaking in production. At Mosaic Smart Data, where downtime could cost millions, I
adopted the mantra: “If it isn’t tested, it’s broken.” Building the real-time API ingestion pipeline, I used JUnit,
Mockito & Test Containers to write unit & IT tests for every component, from Kafka message consumers to data
normalization logic. This caught issues like malformed JSON payloads early, ensuring data integrity for downstream
analytics.

Similarly, at Co-op, I wrote unit tests for parsing 50,000 product prices from Assosia’s APIs, preventing errors from
reaching the customers reports. Integration tests validated the entire pipeline, from API calls to database writes. I
push teams to automate tests in CI/CD pipelines (e.g., Jenkins, GitLab) and cover core logic thoroughly. Skipping tests
is like coding blind, you’ll crash eventually, and your users will pay the price.

## 2. Choose Meaningful Names

Naming is a deceptively hard problem in coding. Early on, I used generic names like data or manager, which made
debugging a nightmare in complex systems like ESG’s BOL Engine. Now, I prioritize short, precise names that scream
intent. For Mosaic Smart Data’s pipeline, I named classes like `MarketDataConsumer` and methods like
`normalizeTransactionData`, instantly clarifying their role in processing real-time market feeds.

In the Ribby Hall project, I renamed a vague Processor class to `XledgerDataSync`, reflecting its job of syncing
accountancy data. Variables like `tradeVolume` trump `myNumber`, and methods like `aggregateMarketEvents` beat
`process`. Duringcode reviews, I’m relentless about naming, bad names waste everyone’s time. Good names act as
documentation, speeding up onboarding and IDE searches (e.g., Ctrl+Shift+F).

## 3. Classes and Functions Should Be Small and Obey the Single Responsibility Principle (SRP)

Small, single-purpose code units are my secret weapon. At Mosaic Smart Data, I refactored a monolithic method handling
market data ingestion into smaller functions, `consumeAwsSqsTransactions`, `validateTradeTransactions`,
`persistTradeTransactionsToDruid`, each under 10lines. This made the pipeline easier to test and debug, cutting
maintenance time significantly. At Co-op, breaking a 50-line price-parsing method into bite-sized pieces slashed
bug-fixing time by half.

I aim for functions to fit in a glance (4-10 lines) and classes to stay under 100 lines, adhering to the Single
Responsibility Principle (SRP): one reason to change. In ESG’s Activiti workflows, I split a class managing both meter
data parsing and reporting into `MeterDataParser` and `MeterReportGenerator`, isolating changes to data formats or
report logic. Here’s an example from my Mosaic work, building a URL for market data queries:

```java
public String buildMarketQueryUrl(String base, String path, Map<String, String> params, String hash) {
    StringBuilder url = new StringBuilder(base);
    if (path != null) url.append("/").append(path);
    if (params != null) {
        url.append("?");
        for (Map.Entry<String, String> param : params.entrySet()) {
            url.append(param.getKey()).append("=").append(param.getValue()).append("&");
        }
        url.setLength(url.length() - 1); // Remove trailing &
    }
    if (hash != null) url.append("#").append(hash);
    return url.toString();
}
```

It’s functional but bloated. Here’s the refactored version:

```java
public String buildMarketQueryUrl(String base, String path, Map<String, String> params, String hash) {
    StringBuilder url = initBaseUrl(base);
    appendPath(url, path);
    appendQueryParams(url, params);
    appendHash(url, hash);
    return url.toString();
}

private StringBuilder initBaseUrl(String base) {
    return new StringBuilder(base);
}

private void appendPath(StringBuilder url, String path) {
    if (path != null) url.append("/").append(path);
}

private void appendQueryParams(StringBuilder url, Map<String, String> params) {
    if (params == null) return;
    url.append("?");
    for (Map.Entry<String, String> param : params.entrySet()) {
        url.append(param.getKey()).append("=").append(param.getValue()).append("&");
    }
    url.setLength(url.length() - 1);
}

private void appendHash(StringBuilder url, String hash) {
    if (hash != null) url.append("#").append(hash);
}

```

The refactored code is longer (18 lines) but clearer, testable, and maintainable. Each method has one job, and names
like `appendQueryParams` are self-explanatory. I use a top-down approach: list steps, stub functions, and refine
recursively.

## 4. Functions Should Have No Side Effects

Side effects, when functions modify external state unexpectedly are bug magnets. At Mosaic Smart Data, I ensured the
pipeline’s data processing methods were side-effect-free, returning transformed data without altering inputs or
databases. This made debugging easier, especially when handling millions of market events daily. In contrast, an early
project had a method that fetched data and updated a cache, causing race conditions that took days to resolve.

Consider this flawed example:

```java
public TradeEvent getTradeEvent(String id, Cache cache) {
    TradeEvent event = tradeRepo.findById(id);
    if (event != null) {
        cache.put(id, event);
    }
    return event;
}
```

The name suggests it retrieves a trade event, but it silently updates the cache, creating a side effect. This confuses
callers, complicates testing (you need to mock the cache), and limits reuse. Here’s the fix:

```java
public TradeEvent getTradeEvent(String id) {
    return tradeRepo.findById(id);
}

public void cacheTradeEvent(String id, TradeEvent event, Cache cache) {
    if (event != null) {
        cache.put(id, event);
    }
}
```

Now, `getTradeEvent` does one thing, fetch data, and `cacheTradeEvent` handles caching. This is clearer, testable, and
flexible. In my Mosaic pipeline, I applied this principle to Kafka consumers, ensuring they processed messages
immutably, which was critical for scalability and reliability.

## Conclusion

These four principles - rigorous testing, meaningful names, small/SRP code units, and side-effect-free functions, have
been my north star. They turned chaotic codebases into maintainable systems, from Co-op’s pricing reports to Mosaic
Smart Data’s real-time pipeline, which empowered FICC desks with instant market insights. Clean code doubled
productivity, reduced stress, and let me focus on solving problems, not fighting bugs.

Start small: write a test, rename a variable, or split a method. Over time, these habits will transform your team’s
output. For a deeper dive, grab Robert Martin’s Clean Code, it reshaped how I code.

Have you battled messy code? Share your clean code wins with me [here]({{ site.baseurl }}/contact/), or ping me for help
applying these to your project!


<p class="text-center">
{% include elements/button.html link="https://www.amazon.com/Clean-Code-Handbook-Software-Craftsmanship/dp/0132350882"
text="Read Clean Code" %}
{% include elements/button.html link="https://www.mosaicsmartdata.com/" text="Mosaic Smart Data" %}
{% include elements/button.html link="https://www.coop.co.uk/" text="Co-op" %}
{% include elements/button.html link="https://www.esgglobal.com/" text="ESG Global" %}
{% include elements/button.html link="https://www.ribbyhall.co.uk/" text="Ribby Hall Village" %}
{% include elements/button.html link="https://www.testcontainers.org/" text="Testcontainers" %}
{% include elements/button.html link="https://www.mockito.org/" text="Mockito" %}
{% include elements/button.html link="https://www.spring.io/projects/spring-boot" text="Spring Boot" %}
{% include elements/button.html link="https://kafka.apache.org/" text="Apache Kafka" %}
</p>