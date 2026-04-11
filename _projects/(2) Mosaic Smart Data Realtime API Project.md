---
name: Mosaic Smart Data Realtime API Project
tools: [Java, Python, AWS Lambda, API Gateway, SNS, SQS, S3, CloudFormation, Ansible, Apache Druid]
image: /assets/images/mosaic.png
description: A fault-tolerant, serverless real-time API service ingesting complex transactional data from major financial institutions across FX, fixed income, credit, and repo markets — with cross-region disaster recovery built in from day one.
---

# Mosaic Smart Data – Real-Time API Project

Mosaic Smart Data provides sophisticated transaction cost analysis (TCA) and market analytics to institutional clients across global financial markets. My engagement there, from 2022 to 2024, centred on building and extending the core data ingestion platform — the system responsible for receiving, validating, enriching, and persisting transactional data from major banks and asset managers in real time.

The platform needed to be zero-maintenance, fault-tolerant, and capable of recovering completely from any failure without data loss. It also needed to handle the operational complexity of ingesting data from institutions with differing API protocols, data formats, and delivery cadences.

## What I Built

The core of my work was a serverless AWS ingestion pipeline handling high-volume, time-sensitive transactional data across foreign exchange, fixed income, credit, and repo markets. The architecture was designed so that no component failure — whether a Lambda timeout, an SQS message delay, or a regional AWS outage — could result in lost or corrupted data.

I also extended the existing backend with a Java and Python data pipeline to improve the enrichment and ingestion of data into Apache Druid, the analytical data store powering Mosaic's client-facing analytics. This significantly reduced ingestion latency and improved the accuracy of the data visible to end users.

## Key Technical Features

- **Serverless Ingestion Pipeline**: AWS API Gateway receiving inbound data from financial institutions, routing via SNS fan-out to SQS queues consumed by Lambda functions for processing and persistence to S3.
- **Cross-Region Disaster Recovery**: A dedicated DR fan-out architecture replicating all inbound data to a secondary AWS region in real time, ensuring full recoverability without manual intervention.
- **Infrastructure as Code**: All infrastructure defined and deployed via Ansible-templated CloudFormation stacks — fully reproducible, version-controlled, and environment-agnostic.
- **Multi-Asset Class Support**: Ingestion pipelines handling FX spot, forward, and swap transactions; fixed income; credit derivatives; and repo market data — each with asset-specific validation and normalisation logic.
- **Fault-Tolerant Design**: Dead-letter queues, idempotent Lambda handlers, and S3-backed audit trails ensuring no message is ever silently dropped and every failure is recoverable.
- **Apache Druid Integration**: Java and Python pipeline enriching and loading data into Druid with configurable batch sizes and retry logic, replacing a fragile predecessor with a stable, observable process.
- **Monitoring & Alerting**: AWS CloudWatch dashboards and alarms providing real-time visibility into ingestion throughput, error rates, and pipeline health.
- **CI/CD Automation**: Jenkins pipelines for infrastructure deployment and application release, with environment-specific configuration managed through Ansible.

## Outcome

The platform delivered reliable, zero-data-loss ingestion for Mosaic's institutional client base throughout my engagement. The DR architecture — tested under both planned and unplanned failure scenarios — performed as designed. The Apache Druid pipeline improvements measurably reduced data latency for end-user analytics, directly improving the client experience Mosaic's sales team could demonstrate.

<p class="text-center">
{% include elements/button.html link="https://www.mosaicsmartdata.com/" text="Mosaic Smart Data" %}
{% include elements/button.html link="https://aws.amazon.com/serverless/" text="AWS Serverless" %}
{% include elements/button.html link="https://aws.amazon.com/lambda/" text="AWS Lambda" %}
{% include elements/button.html link="https://aws.amazon.com/sns/" text="AWS SNS" %}
{% include elements/button.html link="https://aws.amazon.com/sqs/" text="AWS SQS" %}
{% include elements/button.html link="https://aws.amazon.com/cloudformation/" text="AWS CloudFormation" %}
{% include elements/button.html link="https://www.ansible.com/" text="Ansible" %}
{% include elements/button.html link="https://druid.apache.org/" text="Apache Druid" %}
{% include elements/button.html link="https://www.python.org/" text="Python" %}
</p>