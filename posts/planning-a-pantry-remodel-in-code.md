---
title: "Planning a pantry remodel in code"
description: "Turning pantry measurements, shelf stock, and an awkward stud map into an OpenSCAD model, cut schedule, and print-ready job packet."
date: "2026-08-03"
tags: ["OpenSCAD", "DIY", "Python"]
draft: false
---

I expected this pantry remodel to be a shopping-list problem. Measure the walls, count the shelves, buy some brackets. It turned into a small constraint-solving project instead.

![Three concept views of the planned wire-shelf pantry layout](/blog/planning-a-pantry-remodel-in-code/pantry-concept.png)

The interior is 1,158 mm wide, 701 mm deep, and 2,112 mm high. The old IKEA IVAR setup has six 808 by 508 mm shelves, or about 26.5 square feet of shelf surface. I wanted more capacity without making the center of the pantry unpleasant to stand in.

## The layout

The selected layout uses five continuous 406 mm deep shelves across the rear wall, plus short 305 mm deep returns at the front of both side walls. Each level leaves a 548 mm wide standing bay at the entrance.

That produces about 34.3 square feet of shelf surface, a 29.4 percent increase over the IVAR shelves. More importantly, the rear shelf has no center seam, and the stock plan avoids buying unwieldy 144 inch wire shelving.

The stud map is the part that kept this from being simple. The reported rear and right-wall locations support the main shelves, but the left return lands outside the reported stud zone. The plan therefore treats that return as light-goods-only unless I add blocking, and every reported support location still needs a pilot-hole check before I buy or drill anything.

## The repository

I put the whole planning packet in [pike00/pantry-remodel](https://github.com/pike00/pantry-remodel):

- A parametric OpenSCAD model of the pantry and five shelf levels.
- Exact metric cuts with inch conversions and stock allocations.
- A provisional Menards cart with model numbers and stop conditions.
- Technical drawings and a three-page PDF job packet generated with ReportLab.
- A demolition, wall-repair, installation, and load-check sequence.

The useful artifact is not just the rendering. It is the chain from measured envelope, to geometry, to stock lengths, to support constraints, to a packet I can carry into the pantry. If a field measurement changes, the source makes the consequences visible instead of leaving them buried in a handwritten cut list.

This is still a planning record, not a completed installation or a generic shelving prescription. The repository is explicit about the unverified measurements, product instructions, and load limits. I published it now because that provisional state is part of the project, and because the model and packet may be useful to anyone working through a similarly awkward small space.
