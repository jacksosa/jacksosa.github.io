---

name: Lizzie Hermolle Website Project
tools: [ Jekyll, HTML, SCSS, YAML, GitHub Pages, Figma, JavaScript, Responsive Design ]
image: /assets/images/lizzie.png
description: A modern, athlete-focused website built with Jekyll to showcase professional cyclist Lizzie Hermolle’s racing achievements, media collaborations, and sponsorship portfolio.
---

# Lizzie Hermolle Website Project

This project involved the design and development of a modern, performance-oriented website for **Lizzie Hermolle**, a professional cyclist competing at UCI and gravel world-level events.
The goal was to create a fast, elegant, and sponsor-ready platform inspired by the visual style of Demi Vollering’s site, tailored to reflect Lizzie’s personal brand, athletic achievements, and 2026 ambitions.

The website serves as a digital hub for race updates, sponsorship engagement, and media collaborations — balancing visual appeal with technical efficiency using the Jekyll static site generator and GitHub Pages for hosting and version control.

---

## Features

* **Custom Jekyll Theme** – bespoke site structure with modular layouts for “Plans,” “Content & Media,” “Sponsorship,” and “Results.”
* **Dynamic Data Rendering** – race results sourced from YAML data files and grouped by year, including podium highlights and alternating row colours.
* **Responsive Design** – mobile-first layout ensuring seamless viewing across all screen sizes.
* **Palette & Typography Redesign** – refreshed SCSS variables for a cleaner, modern aesthetic inspired by Demi Vollering’s brand palette.
* **Sponsor-Focused Sections** – clearly structured “Sponsorship Opportunities” and “Why Me” pages designed for commercial engagement.
* **Content Automation** – use of reusable data blocks and Markdown front matter for easy content updates.
* **Version Controlled Deployment** – continuous publishing pipeline using GitHub Pages.
* **Accessibility Enhancements** – improved font scaling, colour contrast, and spacing for optimal readability.
* **Zero-Maintenance Architecture** – static build with minimal dependencies for long-term stability.

---

## Technical Highlights

* **Custom SCSS Palette System**
  Implemented `_vars.scss` with a palette map and `_mixins.scss` wrappers, allowing light/dark modes and accent styling to be switched per section.

* **Auto Text Colour Logic**
  Added a Sass `auto-text-color()` function to dynamically select black or white text depending on background brightness.

* **Data-Driven Results Page**
  Introduced a YAML-based results file mirroring your existing credits system. The layout (`results.html`) auto-renders yearly groups with alternating row colours and podium emojis 🥇🥈🥉.

* **Typography Scaling**
  Refined font settings in `_vars.scss` for consistent proportions, improved hierarchy, and better alignment with modern web readability standards.

* **Modular Template Structure**
  Used Jekyll collections and includes for reusable content components (buttons, headers, CTAs), improving maintainability and scalability.

* **Performance-Optimised Build**
  Static assets are compressed, and redundant scripts eliminated — resulting in sub-second load times even on mobile connections.

---

## Design Process

1. **Inspiration & Benchmarking**
   Analysed leading athlete sites such as Demi Vollering’s to capture the tone of modern endurance sports branding — minimal, clean, human-focused.

2. **Content Mapping & Structure**
   Worked with Lizzie’s biography and 2026 plans to create a narrative flow: *About → Race Plans → Content & Media → Sponsorship → Contact.*

3. **Prototyping in Figma**
   Wireframed and tested layout concepts, ensuring visual hierarchy and readability while balancing sponsor presence with personal storytelling.

4. **Implementation**
   Developed custom Jekyll templates with clean SCSS separation, light/dark wrapper styles, and dynamic section rendering.

5. **Testing & Refinement**
   Validated responsiveness, contrast, and readability across devices. Iteratively refined section spacing and palette tones for visual balance.

6. **Launch & Versioning**
   Deployed to GitHub Pages for continuous publishing with version control, ensuring easy content iteration and transparency.

---

## Outcome

The finished site delivers a professional yet personal online presence for Lizzie Hermolle — positioning her for sponsorship growth and fan engagement throughout the 2026 gravel racing season.
It combines lightweight engineering with thoughtful design, offering a long-term platform for media collaboration and storytelling.

<p class="text-center">
{% include elements/button.html link="https://www.jekyllrb.com/" text="Jekyll" %}
{% include elements/button.html link="https://github.com/" text="GitHub Pages" %}
{% include elements/button.html link="https://sass-lang.com/" text="SCSS" %}
{% include elements/button.html link="https://developer.mozilla.org/en-US/docs/Web/HTML" text="HTML5" %}
{% include elements/button.html link="https://developer.mozilla.org/en-US/docs/Web/JavaScript" text="JavaScript" %}
{% include elements/button.html link="https://www.figma.com/" text="Figma" %}
{% include elements/button.html link="https://www.lizziehermolle.co.uk/" text="Website" %}
{% include elements/button.html link="https://www.demivollering.com/" text="Inspiration" %}
{% include elements/button.html link="https://en.wikipedia.org/wiki/Responsive_web_design" text="Responsive Design" %}
</p>