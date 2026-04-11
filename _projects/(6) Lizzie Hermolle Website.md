---
name: Lizzie Hermolle Website Project
tools: [Jekyll, HTML, SCSS, YAML, GitHub Pages, Figma, JavaScript, Responsive Design]
image: /assets/images/lizzie.png
description: A modern, athlete-focused website built with Jekyll to showcase professional cyclist Lizzie Hermolle's racing achievements, media collaborations, and sponsorship portfolio.
---

# Lizzie Hermolle Website Project

This project involved the design and development of a modern, performance-oriented website for **Lizzie Hermolle**, a professional cyclist competing at UCI and gravel world-level events. The goal was to create a fast, elegant, and sponsor-ready platform inspired by the visual style of leading endurance athletes' sites — tailored to reflect Lizzie's personal brand, athletic achievements, and 2026 racing ambitions.

The website serves as a digital hub for race updates, sponsorship engagement, and media collaborations, balancing visual appeal with technical efficiency using the Jekyll static site generator and GitHub Pages for hosting and version control.

## Features

- **Custom Jekyll Theme** — bespoke site structure with modular layouts for Plans, Content & Media, Sponsorship, and Results.
- **Dynamic Data Rendering** — race results sourced from YAML data files and grouped by year, including podium highlights and alternating row colours.
- **Responsive Design** — mobile-first layout ensuring seamless viewing across all screen sizes.
- **Palette & Typography Redesign** — refreshed SCSS variables for a cleaner, modern aesthetic aligned with Lizzie's personal brand.
- **Sponsor-Focused Sections** — clearly structured Sponsorship Opportunities and Why Me pages designed for commercial engagement.
- **Content Automation** — reusable data blocks and Markdown front matter for straightforward content updates without touching templates.
- **Version-Controlled Deployment** — continuous publishing pipeline using GitHub Pages.
- **Accessibility Enhancements** — improved font scaling, colour contrast, and spacing for optimal readability across devices.
- **Zero-Maintenance Architecture** — static build with minimal dependencies for long-term stability.

## Technical Highlights

**Custom SCSS Palette System**
Implemented `_vars.scss` with a palette map and `_mixins.scss` wrappers, allowing light/dark modes and accent styling to be switched per section with a single variable change.

**Auto Text Colour Logic**
Added a Sass `auto-text-color()` function to dynamically select black or white text depending on background brightness — ensuring readability regardless of section colour.

**Data-Driven Results Page**
A YAML-based results file powers the race results layout, auto-rendering yearly groups with alternating row colours and podium indicators — updated by editing a data file rather than HTML.

**Modular Template Structure**
Jekyll collections and includes used for reusable content components — buttons, headers, CTAs — improving maintainability and making future section additions straightforward.

**Performance-Optimised Build**
Static assets compressed and redundant scripts eliminated, resulting in fast load times across both desktop and mobile connections.

## Design Process

1. **Inspiration & Benchmarking** — Analysed leading athlete sites to capture the tone of modern endurance sports branding: minimal, clean, and human-focused.
2. **Content Mapping** — Worked with Lizzie's biography and 2026 plans to create a coherent narrative flow: *About → Race Plans → Content & Media → Sponsorship → Contact*.
3. **Prototyping in Figma** — Wireframed and tested layout concepts, balancing sponsor presence with personal storytelling and ensuring visual hierarchy throughout.
4. **Implementation** — Custom Jekyll templates with clean SCSS separation, light/dark section wrappers, and dynamic data rendering.
5. **Testing & Refinement** — Validated responsiveness, contrast, and readability across devices, iterating on section spacing and palette tones.
6. **Launch & Versioning** — Deployed to GitHub Pages for continuous publishing with version control.

## Outcome

The finished site delivers a professional yet personal online presence for Lizzie Hermolle — positioning her for sponsorship growth and fan engagement throughout the 2026 gravel racing season. It combines lightweight engineering with thoughtful design, and provides a long-term platform for media collaboration and storytelling that Lizzie can update herself.

<p class="text-center">
{% include elements/button.html link="https://www.lizziehermolle.co.uk/" text="Visit the Site" %}
{% include elements/button.html link="https://www.jekyllrb.com/" text="Jekyll" %}
{% include elements/button.html link="https://pages.github.com/" text="GitHub Pages" %}
{% include elements/button.html link="https://sass-lang.com/" text="SCSS" %}
{% include elements/button.html link="https://www.figma.com/" text="Figma" %}
</p>