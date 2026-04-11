---
title: Architecture | Hexagonal vs Clean Architecture in Spring Boot
tags: [ Architecture, Hexagonal Architecture, Clean Architecture, Spring Boot ]
style: fill
color: primary
description: Hexagonal vs Clean Architecture in Java Spring Boot — a practical comparison from a senior developer who has applied both in production across event-driven, microservices, and government-scale systems.
---

Once you move beyond simple CRUD services, architecture choices start to matter. Two patterns that frequently come up in the same conversation are Hexagonal Architecture and Clean Architecture. They share common goals, but they are not the same — and choosing between them (or blending them) has real consequences for Spring Boot systems.

I've applied both across long-lived, high-stakes systems — from real-time financial data pipelines to government benefit platforms — and here's how I think about the choice.

## The Shared Goal

Both architectures aim to:
- Isolate business logic from infrastructure concerns
- Prevent framework lock-in
- Improve testability and long-term maintainability
- Support controlled evolution under changing requirements

Where they differ is in structure and emphasis.

## Hexagonal Architecture

Hexagonal Architecture (Ports and Adapters) focuses on **interaction points**. The domain sits at the centre, and everything outside it — HTTP, Kafka, persistence, third-party APIs — is an adapter connected through a port. The model is interaction-driven and maps naturally to event-driven systems.

**Strengths:**
- Excellent fit for Kafka and messaging-heavy platforms
- Simple mental model that scales well to microservices
- Adapters are easy to swap — test doubles replace production implementations cleanly
- Minimal ceremony for small to medium codebases

**Where I use it:** At DWP Digital and Mosaic Smart Data, Hexagonal Architecture was the natural choice. Multiple entry points (Kafka events, HTTP, scheduled jobs) and the need for domain logic untouched by infrastructure made Ports and Adapters the right foundation.

## Clean Architecture

Clean Architecture introduces concentric dependency layers with strict inward-pointing rules. It emphasises use cases as first-class citizens alongside entities, and provides more prescriptive guidance on layer responsibilities.

**Strengths:**
- Clear separation of concerns with explicit, enforced layer boundaries
- Strong guidance for large teams where consistency matters
- Use cases as explicit objects make complex domain flows easier to reason about
- Well-suited to large, domain-rich systems with multiple bounded contexts

**Where I use it:** When domain complexity grows and teams expand, I selectively introduce Clean Architecture concepts — particularly explicit use case objects — to bring additional structure where Hexagonal Architecture alone starts to feel ambiguous.

## Which I Default To and Why

In Spring Boot systems, I start with Hexagonal Architecture. It gives flexibility without unnecessary ceremony, and the port/adapter model maps directly to the Spring component model. When systems grow large, or when teams expand to the point where boundary ambiguity causes friction, I layer in Clean Architecture concepts where they add genuine clarity — not as a rewrite, but as a targeted evolution.

## The Pragmatic View

The cleanest systems I've worked on borrowed from both. Hexagonal Architecture as a foundation, with explicit use case objects from Clean Architecture where domain complexity demanded them. The goal is always the same: protect your domain from the churn of frameworks and infrastructure, and keep business logic testable without spinning up a database or a Kafka broker.

You don't need to pick sides. Pick what solves your actual problem.

<p class="text-center">
{% include elements/button.html link="https://alistair.cockburn.us/hexagonal-architecture/" text="Hexagonal Architecture" %}
{% include elements/button.html link="https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html" text="Clean Architecture" %}
</p>