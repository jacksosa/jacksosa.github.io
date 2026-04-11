---
name: Ribby Hall Village Data Warehouse Project
tools: [Java, Spring Boot, Docker, Kubernetes, Google Cloud, GitLab, REST, SOAP, GraphQL]
image: /assets/images/ribbyhall.png
description: A bespoke Spring Boot data warehouse solution centralising fragmented business data for Ribby Hall Village — integrating third-party systems via REST, SOAP, and GraphQL APIs and deployed on Google Cloud's Kubernetes platform.
---

# Ribby Hall Village – Data Warehouse Project

Ribby Hall Village is a premium leisure resort in Lancashire, running hotel, spa, holiday lodge, and restaurant operations across multiple systems. The business had valuable data distributed across a fragmented set of third-party platforms — accountancy, marketing, reservations, and more — with no single place to see it together. Management reporting was largely manual, time-consuming, and prone to inconsistency.

I was brought in to design and deliver a solution that would centralise that data, automate reporting, and give the business a single source of truth for their operational and marketing intelligence.

## What I Built

The solution is a Spring Boot microservice acting as the core of a lightweight data warehouse — pulling data from third-party systems on schedule, normalising and enriching it, and persisting it in a structured store ready for reporting. The application integrates with Xledger (accountancy ledger) and Campaign Monitor (marketing platform) among others, using whichever API protocol each provider offered — REST, SOAP, or GraphQL.

The application is containerised with Docker, managed in GitHub, and deployed to a Kubernetes cluster on Google Cloud Platform, with a GitLab CI/CD pipeline handling automated testing, image builds, and deployment.

## Key Technical Features

- **Multi-Protocol API Integration**: Consumers built for REST, SOAP, and GraphQL APIs — allowing integration with third-party providers regardless of the protocol they offered.
- **Xledger Integration**: Automated ingestion of accountancy ledger data from Xledger, providing the business with up-to-date financial reporting without manual data extraction.
- **Campaign Monitor Integration**: Marketing platform data pulled and centralised, enabling cross-referencing of campaign activity with bookings and revenue.
- **Spring Boot Microservice**: Clean, modular application design with scheduled data pull jobs, a normalisation layer, and structured persistence — all independently testable.
- **Test-Driven Development**: JUnit and Mockito throughout — integration adapters, data transformation logic, and scheduling all covered by automated tests.
- **Docker Containerisation**: The application and all its dependencies packaged into a Docker image, ensuring consistent behaviour across development, staging, and production environments.
- **Google Cloud Kubernetes Deployment**: Deployed to a GKE cluster, providing high availability, rolling deployments, and resource scaling as data volumes grow.
- **GitLab CI/CD Pipeline**: Automated pipeline running tests, building Docker images, and deploying to Kubernetes on every merge — eliminating manual deployment steps and reducing release risk.
- **Centralised Reporting**: Consolidated data store enabling management reporting across all operational areas — bookings, revenue, marketing performance, and more — from a single platform.

## Outcome

The solution replaced a labour-intensive manual reporting process with an automated, reliable, and consistent data pipeline. The Ribby Hall team gained access to reporting they previously couldn't produce at all, and the platform was designed to accommodate new data sources and reports as the business's needs evolved.

<p class="text-center">
{% include elements/button.html link="https://www.ribbyhall.co.uk/" text="Ribby Hall Village" %}
{% include elements/button.html link="https://spring.io/projects/spring-boot" text="Spring Boot" %}
{% include elements/button.html link="https://xledger.com/" text="Xledger" %}
{% include elements/button.html link="https://www.campaignmonitor.com/" text="Campaign Monitor" %}
{% include elements/button.html link="https://www.docker.com/" text="Docker" %}
{% include elements/button.html link="https://kubernetes.io/" text="Kubernetes" %}
{% include elements/button.html link="https://cloud.google.com/" text="Google Cloud" %}
{% include elements/button.html link="https://about.gitlab.com/topics/ci-cd/" text="GitLab CI/CD" %}
</p>