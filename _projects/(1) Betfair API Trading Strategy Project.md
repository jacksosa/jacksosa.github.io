---
name: Betfair Exchange Automated Trading Framework
tools: [Java, Spring Boot, AWS, EC2, S3, GitHub Actions, Python]
image: /assets/images/betfair.png
description: Production-grade Java automated trading framework using Betfair Exchange API and Betfair Streaming API. Spring Boot event-driven architecture with proprietary signal engine covering Weight of Money, LTP dynamics, price velocity, and order flow imbalance.
---

# Betfair Exchange Automated Trading Framework

This is the project I'm most proud of — and the one that most distinctly defines my specialism. Built from scratch as a personal venture through SJ Ltd, this production-grade automated trading framework integrates directly with Betfair's Exchange API and Streaming API to execute data-driven betting strategies in real time across pre-race and in-play markets.

The goal was to build a system that could identify and act on genuine market signals faster and more consistently than any manual approach could achieve, while keeping risk controlled and the execution architecture robust enough to run continuously in a live trading environment.

## What It Does

The framework connects to Betfair's Streaming API to consume a continuous, low-latency feed of live market data — prices, traded volumes, order books, and matched amounts — across selected horse racing and sports markets. It processes this data through a configurable strategy engine that evaluates proprietary signals and places orders directly on the exchange via the Exchange API when execution criteria are met.

The system is designed to run headlessly on AWS, handling its own reconnection, state recovery, and error management.

## Key Technical Features

- **Betfair Streaming API Integration**: Persistent WebSocket connection consuming live market data with sub-second latency, with automatic reconnection and subscription management.
- **Betfair Exchange API Integration**: Full order lifecycle management — place, update, cancel — with position tracking and real-time P&L monitoring per market.
- **Proprietary Signal Engine**: Strategies driven by configurable metrics including Weight of Money (WoM) trends, Last Traded Price (LTP) dynamics, price velocity, traded volume acceleration, and order flow imbalance.
- **Event-Driven Architecture**: Spring Boot application with an internal event bus decoupling market data ingestion, signal evaluation, and order execution — allowing strategies to be swapped or updated without touching the core engine.
- **Pre-race & In-play Support**: Configurable strategy profiles for both pre-race liquidity markets and in-play execution, with automated market state detection and trading window enforcement.
- **Risk Controls**: Per-market and per-strategy stake limits, maximum liability caps, and automatic trading halt triggers built into the execution layer.
- **AWS Deployment**: EC2-hosted application with S3-backed persistent state storage, Route 53 for DNS management, and CloudWatch for operational alerting.
- **CI/CD Pipeline**: GitHub Actions workflow for automated build, test, and deployment to AWS EC2, enabling rapid iteration and rollback capability.
- **Strategy Backtesting**: Python-based backtesting suite consuming historical Betfair data to evaluate signal performance prior to live deployment.
- **Operational Observability**: Structured JSON logging, market-level performance summaries, and alerting on anomalous market conditions or execution failures.

## Architecture Overview

The application is structured around three core concerns:

1. **Market Data Layer** — Streaming API consumer normalising raw market data into domain events (price changes, volume updates, market status transitions).
2. **Strategy Layer** — Pluggable strategy implementations consuming domain events, maintaining per-market state, and emitting execution instructions when signal thresholds are met.
3. **Execution Layer** — Exchange API client managing order placement, modification, and cancellation with idempotent retry logic and position reconciliation.

This clean separation means strategies can be developed and tested in isolation, and the execution engine can be improved independently of strategy logic.

## Outcome

The framework has run in live production across Betfair horse racing markets. It demonstrated the viability of systematic, data-driven trading on the Betfair Exchange using a purely Java-based stack — and gave me deep, hands-on experience of the operational challenges that distinguish production betting systems from prototypes: latency management, state recovery, API rate limits, and market liquidity dynamics.

This project underpins my claim as a specialist Java developer for betting exchange work. If your project involves Betfair, Betdaq, Smarkets, or Matchbook integration, I'd welcome the conversation.

<p class="text-center">
{% include elements/button.html link="https://developer.betfair.com/exchange-api/" text="Betfair Exchange API" %}
{% include elements/button.html link="https://developer.betfair.com/exchange-streaming-api/" text="Betfair Streaming API" %}
{% include elements/button.html link="https://spring.io/projects/spring-boot" text="Spring Boot" %}
{% include elements/button.html link="https://aws.amazon.com/ec2/" text="AWS EC2" %}
{% include elements/button.html link="https://aws.amazon.com/s3/" text="AWS S3" %}
{% include elements/button.html link="https://github.com/features/actions" text="GitHub Actions" %}
{% include elements/button.html link="https://www.python.org/" text="Python" %}
</p>