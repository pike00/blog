---
title: "Preserving Digital Handwriting: Building an Offline Vector Archive for Microsoft OneNote"
description: "How I built an open-source pipeline to extract iPad Apple Pencil slide notes from Microsoft Graph into self-contained HTML canvas pages and continuous tall PDFs, protecting years of medical school notes against cloud lock-in."
date: "2026-07-24"
tags: ["Python", "OneNote", "Data Ownership", "PDF", "Playwright", "Medical School"]
draft: false
---

The code is at [github.com/pike00/onenote-archive](https://github.com/pike00/onenote-archive). It is MIT licensed, requires Python 3.14+, syncs through Microsoft Graph, overlays InkML on HTML canvases, and exports single-page tall PDFs.

In medical school I imported slide decks, anatomy atlases, and lecture printouts into Microsoft OneNote on an iPad. I used an Apple Pencil to annotate thousands of pages: drawing pathway diagrams, marking diagnostic criteria, annotating histological images, and writing margin notes over the imported slides. The collection kept growing as training continued.

![OneNote Archive Web Dashboard](/blog/onenote-archive/onenote_web_ui.png)

| Medical School Slide Notes (Example 1) | Clinical Annotations (Example 2) | Pathway Diagrams (Example 3) |
| :---: | :---: | :---: |
| ![Medical School Slide Notes Example 1](/blog/onenote-archive/example1_annotated_slide.png) | ![Clinical Annotations Example 2](/blog/onenote-archive/example2_annotated_slide.png) | ![Pathway Diagrams Example 3](/blog/onenote-archive/example3_annotated_slide.png) |
| *Annotated lecture slides with Apple Pencil ink* | *Margin notes and colored markup on slide content* | *Pathology & anatomy diagram markups* |

Keeping years of notes in one proprietary cloud service tied to an institutional account created two risks:

- My Georgetown email access could be decommissioned, revoked, or restricted after graduation, taking access to the notebooks with it.
- OneNote stores page layouts as absolutely positioned HTML elements while handwriting lives in separate vector stroke streams (InkML XML or proprietary binary structures). Built-in exports can truncate long pages, misalign vector ink relative to background graphics, or fail to preserve stroke fidelity.

I built [OneNote Archive](https://github.com/pike00/onenote-archive) to reduce that dependency. It syncs notebooks from Microsoft Graph and compiles them into self-contained HTML pages that render the ink on a canvas, along with matching continuous single-page tall vector PDFs.

## Pipeline architecture

![OneNote Ink Rendering Pipeline](/blog/onenote-archive/onenote_ink_rendering.jpg)

### 1. Graph sync (`onenote_dl`)
Downloads notebook structures, page metadata, embedded slide images, and raw InkML XML streams via Microsoft Graph (`?includeinkML=true`), caching UUID assets locally in `out/`.

### 2. Ink and DOM preprocessing (`onenote_archive`)
Parses element bounding boxes across the page HTML, calculates spatial extents, and normalizes InkML coordinate channels into 96 DPI CSS pixel offsets.

### 3. Two outputs
The HTML output injects an HTML5 `<canvas>` layer at `z-index: 50` directly over the slide graphics. When the page loads, `ink_render.js` parses the stroke streams and renders vector paths with the original pen colors and widths.

The PDF output uses headless Playwright Chromium to compute the total page height and export an untruncated single-page tall PDF without artificial page breaks.
