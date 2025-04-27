---
name: Mosaic Smart Data Realtime API Project
tools: [ Java, AWS, SQS, SNS, API Gateway, Lambda, S3, Kafka, Python ]
image: /assets/images/mosaic.png
description: A real-time API service for ingesting complex transactional data from financial institutions, built with serverless AWS infrastructure.
---

# Mosaic Smart Data Realtime API Project

This project is a Java-based real-time API service developed at Mosaic Smart Data, designed to ingest and process
complex transactional data from leading financial institutions across asset classes such as foreign exchange, fixed
income, credit, and repo markets. Leveraging serverless AWS infrastructure, the application ensures uninterrupted data
flow, fault tolerance, and complete recoverability, with a cross-region Disaster Recovery (DR) architecture.

## Features

- **Real-time Data Ingestion**: Seamlessly processes high-volume transactional data from financial institutions in real
  time, ensuring low latency and high throughput.
- **Multi-Asset Class Support**: Handles diverse asset classes, including foreign exchange, fixed income, credit, and
  repo markets, enabling broad market coverage.
- **Serverless Architecture**: Utilizes AWS API Gateway, Lambda functions, SNS, SQS, and S3 for a scalable,
  cost-efficient, and maintenance-free infrastructure.
- **Fault-Tolerant Design**: Ensures uninterrupted data flow with robust mechanisms to eliminate data loss, even under
  high-load or failure scenarios.
- **Complete Recoverability**: Implements comprehensive data recovery mechanisms, allowing full restoration of
  transactional data in case of failures.
- **Cross-Region Disaster Recovery (DR)**: Features a DR fanout architecture to replicate data across AWS regions,
  ensuring business continuity.
- **Ansible-Templated CloudFormation**: Deploys infrastructure using Ansible-driven CloudFormation templates for
  consistent, reproducible, and automated setup.
- **High Scalability**: Leverages AWS serverless components to automatically scale with demand, handling large-scale
  financial data streams.
- **Secure Data Handling**: Incorporates security best practices to protect sensitive financial data, complying with
  industry standards.
- **Modular Backend Services**: Built with Java and designed for modularity, enabling easy integration of new features
  and asset classes.
- **Infrastructure as Code (IaC)**: Uses CloudFormation and Ansible for programmatic infrastructure management, reducing
  manual errors.
- **Performance Optimization**: Optimized for high-frequency transactional data, ensuring efficient processing and
  minimal latency.
- **Extensive Monitoring**: Integrates with AWS CloudWatch for real-time monitoring and alerting, enabling proactive
  issue resolution.
- **Cross-Region Resilience**: Ensures data integrity and availability through geographically distributed AWS
  deployments.
- **Streamlined Deployment**: Supports automated deployments via Jenkins CI/CD pipelines, facilitating rapid iteration
  and updates.

<p class="text-center">
{% include elements/button.html link="https://www.mosaicsmartdata.com/" text="Mosaic Smart Data" %}
{% include elements/button.html link="https://aws.amazon.com/serverless/" text="AWS Serverless" %}
{% include elements/button.html link="https://www.ansible.com/" text="Ansible" %}
{% include elements/button.html link="https://aws.amazon.com/cloudformation/" text="AWS CloudFormation" %}
{% include elements/button.html link="https://aws.amazon.com/sqs/" text="AWS SQS" %}
{% include elements/button.html link="https://aws.amazon.com/sns/" text="AWS SNS" %}
{% include elements/button.html link="https://aws.amazon.com/lambda/" text="AWS Lambda" %}
{% include elements/button.html link="https://aws.amazon.com/s3/" text="AWS S3" %}
{% include elements/button.html link="https://kafka.apache.org/" text="Apache Kafka" %}
{% include elements/button.html link="https://www.python.org/" text="Python" %}
{% include elements/button.html link="https://www.java.com/en/" text="Java" %}
{% include elements/button.html link="https://www.jenkins.io/" text="Jenkins CI/CD" %}
</p>