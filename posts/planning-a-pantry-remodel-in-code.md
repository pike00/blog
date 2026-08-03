---
title: "Modeling a pantry remodel before buying the shelves"
description: "I modeled a small pantry in OpenSCAD so I could settle the shelf layout and cut list before buying anything."
date: "2026-08-03"
tags: ["OpenSCAD", "DIY", "Python"]
draft: false
---

Our pantry is small, and the IKEA IVAR shelves in it now leave a lot of space unused. Before replacing them, I wanted to know whether wire shelving on three walls would actually add storage without making the pantry too cramped to use.

![To-scale footprint of the proposed pantry shelves](/blog/planning-a-pantry-remodel-in-code/pantry-footprint.png)

The room is only 45.6 inches wide and 27.6 inches deep. A 16 inch shelf across the back leaves 11.6 inches of open floor in front of it. Adding 12 inch shelves on the side walls leaves a 21.6 inch opening down the middle. It fits, but there is not much room to guess.

I put the measurements into OpenSCAD and tried a few layouts. The one I kept has five continuous shelves across the back with short returns on both sides. It increases the shelf area from about 26.5 to 34.3 square feet, roughly 29 percent, while keeping the center open from floor to ceiling.

![Dimensioned plan and rear elevation of the pantry shelves](/blog/planning-a-pantry-remodel-in-code/pantry-layout.png)

The stud locations made the hardware plan less tidy. The rear wall and right side line up well enough with the proposed tracks. The left return does not, so that side either needs blocking or needs to stay limited to light items. I also still need to confirm every stud location before drilling.

I put the model, cut list, shopping list, drawings, and a printable job packet in [pike00/pantry-remodel](https://github.com/pike00/pantry-remodel). I have not built it yet. The dimensions, stud map, fasteners, and product load limits are all marked as things to verify before I buy or cut anything.

Mostly, I wanted one place where a changed measurement would update the plan instead of forcing me to remember which handwritten numbers it affected. The repository is that place, and it should make the actual install less improvisational when I get to it.
