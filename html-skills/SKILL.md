---
name: "html-page-generator"
description: "Generates standalone single-file HTML pages with modern UI, responsive layout, and animations. Invoke when user needs landing pages, dashboards, portfolios, forms, presentations, or any browser-viewable UI without external dependencies."
---

# Single-File HTML Page Generator

## Core Mandatory Rules

1. Output only a **single complete HTML file** - no CDN links, no external resource references
2. All styles must be inline or in internal `<style>` tags; all interactions via vanilla JS. File must open directly in a browser by double-click
3. Responsive design for both mobile and desktop, layout adapts automatically
4. Clean, semantic HTML structure with minimal comments

## UI Design Standards

- **Color scheme**: Professional, minimalist, or premium gradients. Avoid harsh or clashing colors
- **Typography**: Clear hierarchy, proper spacing and whitespace, reading comfort first
- **Visual effects**: Tasteful shadows, rounded corners, simple icons, lightweight hover animations. Avoid cheap-looking styles
- **Available themes**: Minimalist, Glassmorphism, Dark Mode, Business Cards, Tech Dashboard

## Supported Scenarios

| Category | Examples |
|----------|----------|
| Profile | Personal homepage, landing page, resume/portfolio |
| Data | Dashboard, charts, statistics page |
| Forms | Login form, survey/feedback, registration |
| Utility | 404 page, HTML slideshow, simple frontend tools |

## Interaction Capabilities

- Scroll-triggered fade-in animations
- Hover feedback effects
- Smooth anchor scroll navigation
- Basic form validation
- Data chart rendering

## Output Requirements

1. Return only a complete HTML code block with no extra explanatory text
2. Automatically match the appropriate layout and visual style based on the request
3. Ensure full browser compatibility, no errors on open, all styles render correctly
