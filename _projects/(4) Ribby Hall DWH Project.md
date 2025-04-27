---
name: Ribby Hall Village Data Warehouse Project
tools: [ Java, Spring Boot, Docker, Kubernetes, Google Cloud, GitLab ]
image: /assets/images/ribbyhall.png
description: A data warehouse solution centralizing business data for Ribby Hall Village, built with Spring Boot microservices and deployed on Google Cloud's Kubernetes cluster.                                                                                                              
---

# Ribby Hall Village Data Warehouse Project

This project, undertaken at Ribby Hall Village, involved the development and deployment of a robust data warehouse
solution to centralize fragmented business data, enabling streamlined reporting and enhanced marketing capabilities.
Built using a Spring Boot microservice, the solution integrated with third-party providers via REST, SOAP, and GraphQL
APIs, connecting systems like Xledger (accountancy) and Campaign Monitor (marketing). The application was containerized
with Docker, managed via GitHub, and deployed to a Kubernetes cluster on Google Cloud, with a GitLab CI/CD pipeline
ensuring automated testing and deployment.

## Features

- **Centralized Data Warehouse**: Consolidated fragmented business data from multiple sources, providing a unified
  platform for reporting and analytics.
- **Spring Boot Microservice**: Developed a bespoke microservice using Spring Boot, serving as the core of the data
  warehouse for secure and efficient data processing.
- **Third-Party API Integration**: Integrated with diverse systems, including Xledger (accountancy ledger) and Campaign
  Monitor (sales/marketing), using REST, SOAP, and GraphQL APIs.
- **Test-Driven Development (TDD)**: Employed TDD practices with JUnit and Mockito, ensuring high code quality,
  resilience, and maintainability of the microservice.
- **Containerization with Docker**: Containerized the application using Docker, enabling consistent environments and
  seamless deployment across development and production.
- **Kubernetes Deployment**: Deployed the solution to a Kubernetes cluster on Google Cloud, ensuring high availability,
  scalability, and optimal performance.
- **CI/CD Pipeline in GitLab**: Established a robust Continuous Integration/Continuous Deployment pipeline in GitLab,
  automating testing, building, and deployment processes.
- **Source Control with GitHub**: Managed code versioning via GitHub, ensuring integrity and collaboration across
  development efforts.
- **Streamlined Reporting**: Enabled advanced reporting capabilities, providing actionable insights for business
  operations and decision-making.
- **Enhanced Marketing Capabilities**: Integrated marketing platforms to support targeted campaigns, leveraging
  centralized data for improved customer engagement.
- **Scalable Architecture**: Designed the microservice and Kubernetes setup to scale effortlessly with growing data
  volumes and business needs.
- **Secure Data Processing**: Implemented security best practices to protect sensitive business data, ensuring
  compliance with relevant standards.
- **High Availability**: Leveraged Google Cloud’s Kubernetes cluster for fault-tolerant, always-available data warehouse
  operations.
- **Automated Deployments**: Streamlined development cycles with GitLab CI/CD, reducing manual overhead and accelerating
  feature delivery.
- **Cross-Platform Integration**: Supported integration with diverse third-party providers, accommodating various API
  protocols for flexibility.
- **Performance Optimization**: Optimized the microservice for efficient data processing, handling large datasets from
  multiple sources without latency.

<p class="text-center">
{% include elements/button.html link="https://www.ribbyhall.co.uk/" text="Ribby Hall Village" %}
{% include elements/button.html link="https://spring.io/" text="Spring" %}
{% include elements/button.html link="https://www.campaignmonitor.com/" text="Campaign Monitor" %}
{% include elements/button.html link="https://xledger.com/" text="Xledger" %}
{% include elements/button.html link="https://www.docker.com/" text="Docker" %}
{% include elements/button.html link="https://kubernetes.io/" text="Kubernetes" %}
{% include elements/button.html link="https://cloud.google.com/" text="Google Cloud" %}
{% include elements/button.html link="https://gitlab.com/" text="GitLab CI/CD" %}
</p>