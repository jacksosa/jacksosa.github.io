---
name: ESG Global BOL Engine Enhancement Project
tools: [ Java, Spring, Activiti, RabbitMQ, MS SQL Server, Nifi ]
image: /assets/images/esg-global.png
description: A project enhancing the Business Orchestration Layer (BOL) Engine for smart metering, introducing BPMN processes with Activiti and optimizing performance to unlock 30% additional capacity.
---

# ESG Global BOL Engine Enhancement Project

This project, undertaken at ESG Global, involved significant enhancements to the Business Orchestration Layer (BOL)
Engine, a critical component in the smart metering technology stack. Built using Java 11, Spring, and the Activiti BPMN
framework, the project introduced new business processes and resolved a performance bottleneck that unlocked 30%
additional message processing capacity for overnight meter reads, ensuring scalability for major clients in the energy
sector.

## Features

- **BPMN Process Integration**: Implemented new business processes using the Activiti framework (BPMN 2.0), enabling
  flexible and scalable orchestration of smart meter data flows.
- **Performance Optimization**: Identified and resolved a critical performance bottleneck, increasing message processing
  capacity by 30% without hardware changes, supporting high-volume overnight meter reads.
- **Seamless Data Orchestration**: Enhanced the BOL Engine to serve as a robust interface between energy suppliers and
  the Data Communications Company, ensuring reliable data communication.
- **Test-Driven Development (TDD)**: Utilized JUnit and Mockito to implement TDD, ensuring high code quality,
  reliability, and maintainability of new features and defect fixes.
- **Scalable Architecture**: Leveraged Spring, Hibernate, and RabbitMQ for a modular and scalable backend, handling
  large-scale smart meter data transactions.
- **Message Queue Integration**: Used RabbitMQ and Apache Nifi for efficient message routing and data transformation,
  supporting high-throughput meter data processing.
- **Database Integration**: Integrated with MS SQL Server for robust data storage and retrieval, optimized for smart
  metering use cases.
- **CI/CD Pipeline**: Employed Maven, Jenkins, and Nexus for continuous integration and deployment, enabling rapid and
  reliable feature rollouts.
- **Version Control with GitFlow**: Adhered to GitFlow branching patterns in Github, ensuring organized and
  collaborative development workflows.
- **Kanban Workflow**: Managed tasks using JIRA Kanban boards, streamlining project progress and enhancing team
  collaboration.
- **Comprehensive Documentation**: Maintained detailed project documentation in Confluence, ensuring transparency and
  knowledge continuity.
- **3rd Line Support**: Provided expert support for complex production issues, performing root cause analysis and
  raising defects for prompt resolution.
- **Production Change Management**: Assessed and approved production change requests, ensuring system stability and
  reliability.
- **Product Roadmap Planning**: Estimated size and complexity of roadmap items, aligning development efforts with
  project goals and timelines.
- **High Reliability**: Ensured minimal disruption to operations through rigorous testing and proactive issue
  resolution, supporting major energy clients.
- **Smart Meter Focus**: Tailored enhancements to meet the needs of the smart metering industry, enabling efficient and
  accurate meter data processing.

<p class="text-center">
{% include elements/button.html link="https://esgglobal.com/" text="ESG Global" %}
{% include elements/button.html link="https://www.activiti.org/" text="Activiti" %}
{% include elements/button.html link="https://www.rabbitmq.com/" text="RabbitMQ" %}
{% include elements/button.html link="https://www.microsoft.com/en-us/sql-server/" text="MS SQL Server" %}
{% include elements/button.html link="https://nifi.apache.org/" text="Apache Nifi" %}
{% include elements/button.html link="https://www.java.com/en/" text="Java" %}
{% include elements/button.html link="https://spring.io/" text="Spring" %}
{% include elements/button.html link="https://www.junit.org/junit5/" text="JUnit 5" %}
{% include elements/button.html link="https://site.mockito.org/" text="Mockito" %}
</p>