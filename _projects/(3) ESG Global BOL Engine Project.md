---
name: ESG Global BOL Engine Enhancement Project
tools: [Java, Spring, Activiti, RabbitMQ, MS SQL Server, Apache NiFi, JUnit, Mockito]
image: /assets/images/esg-global.png
description: A significant enhancement to ESG Global's Business Orchestration Layer Engine — introducing BPMN-driven processes with Activiti and resolving a critical performance bottleneck that unlocked 30% additional overnight message processing capacity.
---

# ESG Global – BOL Engine Enhancement Project

ESG Global sits at the heart of the UK's smart metering rollout, providing the software infrastructure that facilitates data exchange between energy suppliers and the Data Communications Company (DCC). Their Business Orchestration Layer (BOL) Engine is the critical middleware through which that data flows — handling millions of meter read and command messages daily.

My engagement at ESG Global ran from 2020 to 2022, and centred on two areas: extending the BOL Engine with new business processes, and resolving a performance bottleneck that was constraining overnight meter read capacity for major energy clients.

## The Performance Challenge

The overnight meter read window is time-constrained — energy suppliers have a fixed window to collect read data from millions of smart meters, and any capacity ceiling directly limits how many customers can be served. When I joined the team, the engine was hitting a processing ceiling that couldn't be addressed by simply adding hardware.

Through profiling and analysis, I identified the root cause — a structural inefficiency in how the Activiti BPMN process engine was managing process instance state under sustained load. Resolving this unlocked **30% additional message processing capacity** within the same infrastructure footprint, directly benefiting the energy clients relying on the system for their overnight read cycles.

## Key Technical Features

- **Activiti BPMN Integration**: Implemented new business processes using the Activiti framework (BPMN 2.0) — enabling flexible, auditable orchestration of smart meter data workflows that could be updated without core code changes.
- **Performance Optimisation**: Identified and resolved a sustained-load bottleneck in the Activiti process engine, delivering a 30% capacity increase without hardware changes.
- **RabbitMQ & Apache NiFi**: Message routing via RabbitMQ and data transformation via Apache NiFi, supporting high-throughput meter data flows between the BOL Engine and DCC gateway systems.
- **Test-Driven Development**: JUnit and Mockito applied throughout — all new features and defect resolutions shipped with comprehensive unit and integration test coverage.
- **3rd-Line Production Support**: Owned complex production issue resolution, including root cause analysis, defect logging, and fix delivery for issues escalated from the operations team.
- **Production Change Management**: Assessed and approved production change requests, ensuring controlled and risk-managed releases to a nationally critical system.
- **Product Roadmap Input**: Contributed to sprint planning and roadmap estimation, working with the product owner to size and sequence delivery items.
- **Agile Kanban Delivery**: Worked within a JIRA Kanban workflow with GitFlow branching and Confluence documentation — keeping delivery transparent and the team aligned.

## Technology Stack

The BOL Engine was built on Java 11 and Spring, with Hibernate for persistence against MS SQL Server, RabbitMQ for messaging, and Apache NiFi for data transformation pipelines. Infrastructure was managed through Maven, Jenkins, and Nexus, with GitFlow branching in GitHub and all project documentation maintained in Confluence.

## Outcome

The performance work delivered immediate, measurable value for ESG's energy clients — expanded overnight capacity without any infrastructure cost increase. The BPMN process additions extended the platform's capability while keeping the codebase maintainable and the orchestration logic transparent and auditable.

<p class="text-center">
{% include elements/button.html link="https://esgglobal.com/" text="ESG Global" %}
{% include elements/button.html link="https://www.activiti.org/" text="Activiti BPMN" %}
{% include elements/button.html link="https://www.rabbitmq.com/" text="RabbitMQ" %}
{% include elements/button.html link="https://nifi.apache.org/" text="Apache NiFi" %}
{% include elements/button.html link="https://www.microsoft.com/en-us/sql-server/" text="MS SQL Server" %}
{% include elements/button.html link="https://spring.io/" text="Spring" %}
{% include elements/button.html link="https://junit.org/junit5/" text="JUnit 5" %}
{% include elements/button.html link="https://site.mockito.org/" text="Mockito" %}
</p>