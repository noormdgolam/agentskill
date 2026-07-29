---
name: responsive_web_mastery
description: Expert guidelines and design patterns for building responsive HTML/CSS web layouts, flexbox/grid alignments, mobile navigation drawers, and preventing layout overlaps.
---

# Responsive Web Mastery Skill

## Core Principles
1. **Fluid Grid Layouts**: Always use CSS Grid (`grid-template-columns: repeat(auto-fill, minmax(280px, 1fr))`) for product catalogs and card listings.
2. **Container Isolation**: Ensure every grid container (`.page-with-sidebar`, `.container`) closes all nested child elements cleanly before starting subsequent sections (`<section>`).
3. **No Overflow Spills**: Prevent layout overlap by setting `clear: both; width: 100%; display: block;` on structural sections below sidebars.
4. **Mobile First & Media Queries**: Ensure navigation drawers (`.mobile-drawer`) and hamburger toggles adapt seamlessly across mobile (<=768px), tablet (<=1024px), and desktop breakpoints.
5. **Image Ratio & Fallbacks**: Image wrappers (`.property-img-wrap`) must enforce fixed aspect ratios (`aspect-ratio: 1/1` or `height: 220px; object-fit: cover`) to avoid layout jumps or broken text overflows.
