---
name: Co-op Competitor Pricing Reports Project
tools: [Java, REST APIs, Teradata, SQL, MicroStrategy, GitHub, Agile, TDD]
image: /assets/images/co-op.png
description: A Java-based data integration solution for Co-op's Food Retail Business Intelligence team — ingesting weekly competitor pricing data for 50,000 products across 30 providers and delivering management reports to drive pricing strategy.
---

# Co-op – Competitor Pricing Reports Project

Co-op's Food Retail Business Intelligence team needed a way to systematically understand how their pricing compared to competitors — not just at a headline level, but product-by-product, store-by-store, accounting for local market demographics and competitor mix. The data was available from Assosia, a retail pricing intelligence provider, but pulling it, processing it, and making it useful for the pricing team required building a reliable, repeatable data pipeline from scratch.

My role was to design and deliver that pipeline: a Java-based integration solution that ingested weekly pricing data, loaded it into the Co-op data warehouse, and produced the management reports the pricing team needed to act on it.

## What I Built

The solution centred on a set of pure Java worker services — scheduled jobs that pulled competitor pricing data from Assosia's REST API on a weekly cadence, processed and validated the inbound data, and loaded it into Co-op's Teradata data warehouse. From there, MicroStrategy-based management reports gave the pricing team visibility into price positioning across the product range, broken down by store and local competitor set.

The pipeline handled pricing data for approximately **50,000 products** across **30 competitor providers** every week, with validation and error handling to ensure that incomplete or malformed data from any one provider didn't corrupt the wider dataset.

## Key Technical Features

- **Assosia REST API Integration**: Weekly batch ingestion of competitor pricing data from Assosia's API — configurable per provider, with per-provider data normalisation to accommodate format differences.
- **Java Worker Services**: Lightweight, independently deployable Java services handling scheduled API calls, data transformation, validation, and warehouse loading — built for reliability and observability.
- **Teradata Data Warehouse**: Competitor pricing data loaded into Co-op's existing Teradata warehouse alongside other business intelligence datasets — weather forecasts, promotional data, and store demographics.
- **MicroStrategy Reporting**: Warehouse data surfaced through MicroStrategy reports giving the pricing team filterable, actionable views of competitor positioning by product, store, and region.
- **Test-Driven Development**: JUnit applied throughout — API integration adapters, data transformation logic, and warehouse loading all tested at unit and integration level.
- **Agile Delivery**: Sprint-based delivery working closely with the pricing team to iteratively refine the data model and report format based on real feedback.
- **GitHub Version Control**: All code managed in GitHub with pull request reviews and branching strategy aligned to team working practices.

## Outcome

The reporting solution gave Co-op's pricing team a consistent, reliable view of competitor pricing at a product and store level — something they'd previously had to piece together manually from inconsistent sources. It fed directly into pricing decisions across the food retail estate, and the pipeline was extended over the engagement to incorporate additional data sources including weather forecast data to support demand-based pricing.

<p class="text-center">
{% include elements/button.html link="https://www.coop.co.uk/" text="Co-op" %}
{% include elements/button.html link="https://www.assosia.com/" text="Assosia" %}
{% include elements/button.html link="https://www.java.com/en/" text="Java" %}
{% include elements/button.html link="https://www.teradata.com/" text="Teradata" %}
{% include elements/button.html link="https://www.microstrategy.com/" text="MicroStrategy" %}
{% include elements/button.html link="https://junit.org/junit5/" text="JUnit" %}
</p>