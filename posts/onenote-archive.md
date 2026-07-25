---
title: "Preserving Digital Handwriting: Building an Offline Vector Archive for Microsoft OneNote"
description: "How I built an open-source pipeline to extract iPad Apple Pencil slide notes from Microsoft Graph into self-contained HTML canvas pages and continuous tall PDFs, protecting years of medical school notes against cloud lock-in."
date: "2026-07-24"
tags: ["Python", "OneNote", "Data Ownership", "PDF", "Playwright", "Medical School"]
draft: false
---

> **GitHub Repository:** [https://github.com/pike00/onenote-archive](https://github.com/pike00/onenote-archive)  
> *MIT Licensed · Python 3.11+ · Microsoft Graph Sync · Canvas InkML Overlay · Single-Page PDF Exporter*

Throughout medical school, my primary workflow for absorbing dense clinical material involved importing slide decks, anatomy atlases, and lecture printouts into Microsoft OneNote on an iPad. Using an Apple Pencil, I annotated thousands of pages: drawing pathway diagrams, highlighting diagnostic criteria, marking up histological images, and jotting margin notes directly over imported slide printouts. As medical training progresses, this library of annotated clinical knowledge continues to grow.

![OneNote Archive Web Dashboard](/blog/onenote-archive/onenote_web_ui.png)

| Medical School Slide Notes (Example 1) | Clinical Annotations (Example 2) | Pathway Diagrams (Example 3) |
| :---: | :---: | :---: |
| ![Medical School Slide Notes Example 1](/blog/onenote-archive/example1_annotated_slide.png) | ![Clinical Annotations Example 2](/blog/onenote-archive/example2_annotated_slide.png) | ![Pathway Diagrams Example 3](/blog/onenote-archive/example3_annotated_slide.png) |
| *Annotated lecture slides with Apple Pencil ink* | *Margin notes and highlighted slide content* | *Pathology & anatomy diagram markups* |

However, relying entirely on a single proprietary cloud service to store years of intensive medical education notes introduces severe vulnerabilities:

- **Loss of Institutional Email Access:** Access tied to institutional or university accounts (such as a Georgetown email address) can be decommissioned, revoked, or restricted post-graduation, threatening total loss of access to years of work.
- **Cloud Storage & Sync Corruption:** Silent sync failures, section file corruption, or quota policy shifts in cloud infrastructure can render notebooks unreadable without notice.
- **Proprietary Vendor Lock-In:** Microsoft OneNote stores page layouts as absolutely positioned HTML elements while encoding handwriting in separate vector stroke streams (InkML XML or proprietary binary structures). Built-in export features routinely truncate long pages, misalign vector ink relative to background graphics, or fail to preserve stroke fidelity entirely.

To solve this lock-in problem and permanently protect these notes, I built [OneNote Archive](https://github.com/pike00/onenote-archive): an automated open-source pipeline that syncs notebooks from Microsoft Graph and compiles them into self-contained HTML pages with dynamic canvas ink rendering, alongside matching continuous single-page tall vector PDFs.

---

## Pipeline Architecture

![OneNote Ink Rendering Pipeline](/blog/onenote-archive/onenote_ink_rendering.jpg)

The pipeline operates in three distinct phases:

1. **Graph Sync (`onenote_dl`):** Downloads notebook structures, page metadata, embedded slide images, and raw InkML XML streams via Microsoft Graph (`?includeinkML=true`), caching UUID assets locally in `out/`.
2. **Ink & DOM Preprocessing (`onenote_archive`):** Parses element bounding boxes across the page HTML, calculates spatial extents, and Normalizes InkML coordinate channels into 96 DPI CSS pixel offsets.
3. **Dual Output Generation:**
   - **Path A (HTML Canvas Overlay):** Injects an HTML5 `<canvas>` layer positioned at `z-index: 50` directly over slide graphics. A client-side script (`ink_render.js`) parses stroke streams at load time, rendering smooth vector paths with preserved pen colors and widths.
   - **Path B (Continuous Single-Page PDF):** Uses headless Playwright Chromium to compute total page height and export un-truncated single-page tall PDFs without artificial page breaks.

---

## Technical Challenges: Re-aligning Ink and Canvas

Exporting OneNote pages accurately requires solving spatial misalignment between two distinct layers:
- **Background Graphics:** Absolutely positioned image blocks (`<img>` tags with `position: absolute; left: Xpx; top: Ypx;`).
- **Handwriting Ink Streams:** XML `<inkml:ink>` streams containing raw coordinate channels.

### 1. Spatial Extents and Coordinate Normalization
InkML stores stroke channels in HiMetric units (1/100th of a millimeter), whereas HTML document elements are positioned using CSS pixels. To overlay handwriting accurately, the preprocessor parses element bounds across the DOM, scales HiMetric coordinates to 96 DPI CSS pixels, and expands the container bounds so margin notes are never clipped.

### 2. Canvas Vector Overlay
Rather than flattening handwriting into low-resolution raster images, the rendering pipeline injects an HTML5 `<canvas>` layer. A lightweight client-side renderer parses the stroke streams at load time, drawing smooth paths with preserved pen colors, highlighters, and stroke widths.

### 3. Continuous Single-Page PDF Export
Traditional PDF printers enforce arbitrary page boundaries (such as Letter or A4), which slice through slide graphics and handwritten annotations. Using headless Playwright Chromium, the archiver calculates the exact full-content height of each page and prints a single, continuous, un-truncated PDF document per note.

---

## Configuration and Management

The application is configured using `pydantic-settings` (`BaseSettings`), allowing seamless overrides via environment variables (`ONENOTE_*`) or YAML configuration files (`keys/config.yaml`).

It also includes a status dashboard (`onenote_web`) built with FastAPI and HTMX, alongside a background worker queue (`onenote_worker`) for serial library compilation and status monitoring.

```bash
# Clone repository
git clone https://github.com/pike00/onenote-archive.git
cd onenote-archive

# Install dependencies via uv
uv sync

# Sync notebooks from Microsoft Graph API
uv run python -m onenote_dl.download_onenote

# Compile offline HTML & PDF archive
uv run onenote_archive/run_archive.py
```

By decoupling handwriting data from proprietary platforms and rendering it with standard web technologies, years of medical notes remain readable, searchable, and permanently accessible in open formats independent of institutional account access or cloud platform changes.
