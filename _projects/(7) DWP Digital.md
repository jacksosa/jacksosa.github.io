---
name: DWP Digital – New Style JSA Modernisation Programme
tools: [Java, Spring Boot, Apache Kafka, MongoDB, AWS, REST, Hexagonal Architecture]
image: /assets/images/dwp-digital.png
description: A major DWP Digital programme to modernise the New Style Jobseeker's Allowance platform — improving resilience, scalability, and policy agility through cloud-native, event-driven microservices architecture.
---

# DWP Digital – New Style JSA Modernisation Programme

DWP Digital is responsible for the technology platforms that support millions of UK citizens accessing the benefits system. The New Style Jobseeker's Allowance (JSA) platform is one of those critical services — managing the full claimant journey from initial application and eligibility assessment through to ongoing bi-monthly payments.

My current engagement, which began in 2025 on a 12-month contract, sits within the programme modernising this platform — replacing legacy constraints with a resilient, event-driven architecture capable of meeting present and future demand at national scale.

## The Challenge

Benefits platforms carry a unique combination of requirements that make them architecturally demanding: they must be completely reliable (a missed payment affects a citizen's livelihood), highly testable (policy changes need to be safely deployable without regressions), and capable of evolving rapidly as legislation changes. The legacy platform made all three of these harder than they needed to be.

The modernisation programme is addressing this by rebuilding the platform's core services on Spring Boot microservices, Apache Kafka, and MongoDB — structured around Hexagonal (Ports and Adapters) Architecture to keep domain logic clean and isolated from the infrastructure concerns that tend to erode it over time.

## My Contributions

- **Legacy Modernisation**: Redesigning and refactoring legacy components into modern, well-tested Spring Boot services — reducing technical debt and improving the platform's long-term maintainability.
- **Event-Driven Architecture**: Implementing Kafka-based asynchronous workflows to decouple core claimant processing steps, improving throughput and system resilience under load.
- **Hexagonal Architecture**: Applying Ports and Adapters patterns to maintain a strict boundary between domain logic and external concerns — messaging, persistence, third-party integration — keeping the domain model clean and independently testable.
- **Idempotency & Fault Tolerance**: Designing processing patterns that ensure safe reprocessing, exactly-once business outcomes, and graceful recovery from partial failures — essential properties for a payment-processing system.
- **MongoDB Data Modelling**: Designing document schemas for flexible persistence of claimant state, balancing query performance with the evolving shape of benefit eligibility data.
- **API-Led Integration**: Building and consuming RESTful APIs to integrate with upstream and downstream DWP services, maintaining clear service ownership and separation of concerns.
- **Quality & Test Discipline**: Automated test coverage throughout — unit, integration, and contract tests — alongside code review and adherence to DWP Digital and GDS engineering standards.
- **Multi-Team Delivery**: Working within a large-scale, multi-supplier delivery environment alongside product managers, architects, testers, and policy stakeholders — maintaining alignment across a complex programme.

## Technology Stack

Java, Spring Boot, Apache Kafka, MongoDB, AWS, REST APIs — all structured around Hexagonal Architecture with TDD applied throughout. CI/CD via GitHub Actions with automated deployment to AWS infrastructure.

## Outcomes and Value

- Improved resilience and reliability for a nationally critical benefit platform
- Greater agility to respond to policy and legislative change without destabilising the wider system
- Reduced operational risk through decoupled, fault-tolerant event-driven design
- A platform foundation capable of future expansion and reform at scale

This programme is a clear example of large-scale, mission-critical digital delivery where engineering quality, operational resilience, and real-world societal impact are equally important.

<p class="text-center">
{% include elements/button.html link="https://www.gov.uk/government/organisations/department-for-work-pensions" text="Department for Work & Pensions" %}
{% include elements/button.html link="https://www.gov.uk/service-manual" text="GDS Service Manual" %}
{% include elements/button.html link="https://spring.io/projects/spring-boot" text="Spring Boot" %}
{% include elements/button.html link="https://kafka.apache.org/" text="Apache Kafka" %}
{% include elements/button.html link="https://www.mongodb.com/" text="MongoDB" %}
{% include elements/button.html link="https://aws.amazon.com/" text="AWS" %}
</p>