---
title: "Preserving Digital Handwriting: Building an Offline Vector Archive for Microsoft OneNote"
description: "How I built an open-source pipeline to extract iPad Apple Pencil slide notes from Microsoft Graph into self-contained HTML canvas pages and continuous tall PDFs, protecting years of medical school notes against cloud lock-in."
date: "2026-07-24"
tags: ["Python", "OneNote", "Data Ownership", "PDF", "Playwright", "Medical School"]
draft: false
---

**GitHub Repository:** [https://github.com/pike00/onenote-archive](https://github.com/pike00/onenote-archive)  
*MIT Licensed · Python 3.11+ · Microsoft Graph Sync · Canvas InkML Overlay · Single-Page PDF Exporter*

Throughout medical school, my primary workflow for absorbing dense clinical material involved importing slide decks, anatomy atlases, and lecture printouts into Microsoft OneNote on an iPad. Using an Apple Pencil, I annotated thousands of pages: drawing pathway diagrams, highlighting diagnostic criteria, marking up histological images, and jotting margin notes directly over imported slide printouts. As medical training progresses, this library of annotated clinical knowledge continues to grow.

![OneNote Archive Web Dashboard](/blog/onenote-archive/onenote_web_ui.png)

| Medical School Slide Notes (Example 1) | Clinical Annotations (Example 2) | Pathway Diagrams (Example 3) |
| :---: | :---: | :---: |
| ![Medical School Slide Notes Example 1](/blog/onenote-archive/example1_annotated_slide.png) | ![Clinical Annotations Example 2](/blog/onenote-archive/example2_annotated_slide.png) | ![Pathway Diagrams Example 3](/blog/onenote-archive/example3_annotated_slide.png) |
| *Annotated lecture slides with Apple Pencil ink* | *Margin notes and highlighted slide content* | *Pathology & anatomy diagram markups* |

However, relying entirely on a single proprietary cloud service to store years of intensive medical education notes introduces two major vulnerabilities:

- Loss of Institutional Email Access: Access tied to institutional or university accounts (such as my Georgetown email address) can be decommissioned, revoked, or restricted post-graduation, threatening total loss of access to years of work.
- Proprietary Vendor Lock-In: Microsoft OneNote stores page layouts as absolutely positioned HTML elements while encoding handwriting in separate vector stroke streams (InkML XML or proprietary binary structures). Built-in export features routinely truncate long pages, misalign vector ink relative to background graphics, or fail to preserve stroke fidelity entirely.

To solve this lock-in problem and permanently protect these notes, I built [OneNote Archive](https://github.com/pike00/onenote-archive): an automated open-source pipeline that syncs notebooks from Microsoft Graph and compiles them into self-contained HTML pages with dynamic canvas ink rendering, alongside matching continuous single-page tall vector PDFs.

---

## Pipeline Architecture

![OneNote Ink Rendering Pipeline](/blog/onenote-archive/onenote_ink_rendering.jpg)

The pipeline operates in three distinct phases:

1. Graph Sync (`onenote_dl`): Downloads notebook structures, page metadata, embedded slide images, and raw InkML XML streams via Microsoft Graph (`?includeinkML=true`), caching UUID assets locally in `out/`.
2. Ink & DOM Preprocessing (`onenote_archive`): Parses element bounding boxes across the page HTML, calculates spatial extents, and Normalizes InkML coordinate channels into 96 DPI CSS pixel offsets.
3. Dual Output Generation:
   - Path A (HTML Canvas Overlay): Injects an HTML5 `<canvas>` layer positioned at `z-index: 50` directly over slide graphics. A client-side script (`ink_render.js`) parses stroke streams at load time, rendering smooth vector paths with preserved pen colors and widths.
   - Path B (Continuous Single-Page PDF): Uses headless Playwright Chromium to compute total page height and export un-truncated single-page tall PDFs without artificial page breaks.

---
