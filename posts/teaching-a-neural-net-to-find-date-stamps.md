---
title: "Teaching a Neural Net to Find Date Stamps on Scanned Photos"
description: "Fine-tuning YOLO to locate burned-in LED date stamps on 7,500 scanned family photos so EXIF can be rewritten from the film itself."
date: "2026-04-17"
tags: ["YOLO", "Computer Vision", "Python", "Machine Learning"]
draft: false
---


## The problem

I have roughly 77,000 family photos spread across an old HDD. About 7,500 of them are scans from a service called [ScanMyPhotos](https://www.scanmyphotos.com/) [^1] -- four DVDs, shoeboxes of 4x6 prints from the 90s and early 2000s. The scans are fine. The problem is the metadata: every JPG file has a `DateCreated` of whenever the scanner operator fed it into the machine, which is useless. A photo of my kindergarten graduation shows up in the library as "2018".

The good news: a lot of these photos have date stamps burned right onto the film. If you remember disposable cameras and early point-and-shoots, you remember the little orange LED digits in the corner — like the `4 23 '95` glowing in the bottom-right of this scan:

![Scan of the Golden Gate Bridge from the Marin Headlands with a glowing orange '4 23 '95' LED date stamp in the bottom-right corner](/blog/teaching-neural-net/golden-gate-stamp.jpg)

That is the real date. If I can read those stamps, I can write them back into EXIF and the whole library snaps into chronological order. So the question reduces to: given a scan, find the date stamp, then read it.

This post is about the "find it" half. The reading (OCR) is a different story, to be followed up in a part 2. 

## Attempt 1: OpenCV heuristics

The first instinct was "this is a computer vision toy problem, not a machine learning problem." The stamps are always orange-ish, always small, always near an edge. Surely I can just threshold on HSV and find them.

Here is roughly what I tried:

1. Convert to HSV.
2. Mask pixels in the orange/amber range (hue 10-25, saturation > 100, value > 100).
3. Find contours, filter by aspect ratio (wider than tall, reasonable size).
4. Assume the largest orange blob near the bottom edge is the stamp.

For whatever reason, I couldn't get this to work reliable - what should be a trivial OpenCV problem ended up with unreliable reading:

The failure modes kept accumulating:

- A picture of a sunset over the Grand Canyon. The entire bottom half of the image was orange. The "date stamp" the detector found was a cliff.
- Orange flowers, orange shirts, orange sunsets, orange Christmas lights.
- Stamps on bright backgrounds (overexposed sky) that washed out and didn't cross the saturation threshold.
- Photos rotated 90 degrees where the stamp was on the side, not the bottom.
- Photos with *no* stamp at all, which the heuristic would confidently produce a garbage box for anyway.
- Faded scans where the stamp was more brown than orange.

I could have kept bolting on exceptions. Edge case handling in CV pipelines tends to metastasize, and I could already see the shape of it: per-photo tuning, manual overrides, an ever-growing list of `if sunset: skip`. Not fun.

## Attempt 2: Fine-tune YOLO

Object detection is the textbook solution for "find a bounding box around a known thing in a photo." [YOLO](https://github.com/ultralytics/ultralytics) (You Only Look Once) is a family of single-pass detectors from Ultralytics that are fast, well-tooled, and come with pretrained weights on [COCO](https://cocodataset.org/) — Microsoft's ~330k-image object detection dataset covering 80 everyday categories, which gives the model a strong generic visual prior (edges, textures, "thing-ness") before you fine-tune it on your own data. That pretraining is the whole reason 2,600 hand-labeled photos is even enough to train a working detector. Fine-tuning one of these on a few thousand labeled stamps should crush the heuristic approach, and I wouldn't have to think about sunsets at all.

I picked YOLOv8-nano to start, because (a) I'm on a desktop with no discrete GPU (AMD Ryzen 5 5600G, integrated Vega graphics that torch can't talk to) and (b) the task is simple from a detector's perspective: one class, distinctive color, consistent shape. The nano variant is 3 million parameters, which is small by 2026 standards. It should be overkill.

## Building an annotation UI

The first real problem: there are no labels. YOLO wants a `.txt` file per image with normalized `class cx cy w h` lines. I need to draw bounding boxes on a few thousand scans.

I built a tiny browser-based annotator (`scripts/annotate/annotate.py` + `ui/index.html`). It's a Python HTTP server exposing a REST API and serving a vanilla-JS Canvas frontend:

![Screenshot of the browser-based annotation UI: a scanned photo of a bridge over a canyon fills the viewport, a green bounding box has just been dragged in the bottom-right corner, and the header shows the image filename, a 101/7456 progress counter, and labeled/skipped tallies. A button bar at the bottom lists Reset Box, Confirm, Skip, and Back, and a row of keyboard hints sits underneath.](/blog/teaching-neural-net/annotator.jpg)

Features I ended up caring about, in order of how much they mattered:

1. **Keyboard-first workflow.** Arrow keys to move between images, `s` to skip, `enter` to save, `z` to undo. Mouse only for drawing the box. Labeling thousands of photos gets painful fast if you have to click buttons.
2. **Skip as signal.** When I marked a photo "no stamp here," the stem got written to `state/skipped.txt`. These become *negative* training examples -- photos with no label file but present in the dataset. YOLO handles this correctly (it treats them as "nothing to detect here") and they matter a lot for precision.
3. **Auto-advance.** When I saved a box, the next image loaded immediately. No confirmation dialogs, no "are you sure." I can afford to mislabel one out of a thousand; I would prefer not to have a two-second stall per photo.
4. **Persistent state.** The server tracked my position, so when I got bored and closed the tab, the next session picked up where I left off.

Edge case that took an hour to diagnose: my first annotator session produced labels named `00000080.txt`, matching the original scan filenames. I later reorganized source photos with disc prefixes (`d1_00000080.jpg`) so I could tell Disc 1 from Disc 3, and suddenly every label was orphaned. I wrote a migration block in `setup_dataset()` that walks old-style label filenames and rewrites them to match disc-prefixed image stems. Keeping it there is ugly but it runs every train and means I never have to think about it again.

## Labels and the 80/20 split

After a few evenings of mindless clicking while watching TV, I had around 2,600 labeled images and another ~300 "skipped" negatives. Split 80/20 into train and val with `random.seed(42)` so splits are reproducible across training runs. Symlink the images into `dataset/images/{train,val}/` and copy the labels into `dataset/labels/{train,val}/`.

Note on the copy-vs-symlink choice: I symlink images (they're big, I don't want two copies) and *copy* labels (they're tiny, and symlinking them caused weird permissions issues when the dataset got mounted into a Docker container). Small detail; mattered a lot when training refused to start with `PermissionError: [Errno 13]` on day one.

## The first real training run

With ~2,900 examples, I kicked off a training run:

```python
model.train(
    data="dataset/data.yaml",
    epochs=100,
    patience=10,
    imgsz=640,
    batch=4,
    device="cpu",
    ...
)
```

Epoch 1 took about 14 minutes. Epoch 2 was faster. By epoch 15 I was at mAP50 = 0.89. By epoch 27 I had:

| Metric        | Value |
| ------------- | ----- |
| Precision     | 95.3% |
| Recall        | 95.8% |
| mAP@50        | 95.0% |
| mAP@50-95     | 73.8% |
| F1 (peak)     | 0.96 at conf 0.37 |

Training early-stopped at epoch 37. That 73.8% mAP@50-95 was the only result I wasn't too excited about -- it means the model finds the stamps but its bounding boxes are a little loose. For my purposes that is fine: the downstream OCR only needs a rough crop that contains the stamp, and I'd rather have a loose box that includes the whole stamp than a tight one that clips half of a digit.

## Batch inference and the first reality check

Running the trained model across ~7,500 scans took about 40 minutes on CPU with `imgsz=384` (smaller than training, deliberately -- inference doesn't need full resolution). I got 6,458 detections at `conf >= 0.01`.

I built a corrections dashboard (`scripts/annotate/corrections_dashboard.py`, `ui/dashboard.html`) to review the results. Same keyboard-first vibe as the annotator, but now the model proposes a box and I either confirm it, nudge it, or mark it as wrong.

![Screenshot of the corrections dashboard: a prioritized review queue of predictions runs down the left (each item tagged with its confidence and 'pending' status), the center pane shows the current scan with its proposed bounding box (blurred here for privacy), and the right pane has Confirm / No Stamp / Skip / Rotate / Next action buttons plus box coordinates, source, confidence, and status fields. A 'Train + Re-infer' button sits at the bottom-right.](/blog/teaching-neural-net/dashboard.jpg)

The confidence histogram was bimodal: a big peak near 0.8 (obvious correct detections) and a smaller cluster around 0.3-0.4 (borderline cases worth reviewing). Anything above 0.7 I could bulk-approve with a glance. Anything below 0.5 I actually looked at.

![Histogram of detection confidences for the CPU model across all 6,458 scans, showing a dominant peak between 0.7 and 0.9 and a clearly separated secondary bump in the 0.3–0.4 range. An overlay of the later GPU run shifts the peak tighter and higher around 0.85–0.9 (previewed here for comparison).](/blog/teaching-neural-net/confidence-distribution.jpg)

The failure modes I saw:

- **Bright skies washing out the stamp.** The model just missed these. The stamp was there, the colors were right, but the contrast was gone. Challenging to see even by my human eye. 
- **Rotated photos.** A scan flipped 90 degrees has the stamp on the side, and the model -- which saw 99% of its training data in a consistent orientation -- would either miss it or confidently predict a region of empty sky.
- **Orange content near an edge.** Orange curtains, a child's orange sweater right at the bottom of the frame. The model got better than the HSV heuristic at ignoring these, but not perfect.
- **No-stamp photos with spurious detections.** This is what negative examples are for, and it was *much* worse before I added the skipped-photo pipeline. Zero false positives on background images after I added them -- the confusion matrix came back completely clean.

## Hard-case augmentation: the bright-background fix

I built `scripts/data/augment_hard_cases.py` specifically to fight the bright-background failure mode. The idea was to take the photos I already labeled and synthesize augmented copies with the characteristics the model was missing. This included: 

- Brightness up 1.6x and 2.0x
- Contrast down to 0.5x
- Gamma corrections
- Warm and cool color-temperature shifts. 

Since all the transforms are global (no rotation, no crop), the bounding box labels are valid on the augmented copies without any remapping.

Run the full augmentation over ~2,600 labeled images producing 6 variants each, and you get about 15,000 augmented images. Add them to the training split (never to val -- validation must remain independent). The augmented data went under `dataset/augmented/` with a separate labels dir so I could nuke them with `just augment-clean` whenever I wanted.

And this is where things started to get slow.

## The 58-day training run

I kicked off `just train` on the augmented set:

```
Epoch 1/100:   3%|▎         | 134/4456 [25:21<14:03:22, 11.4s/it]
```

14 hours/epoch × 100 epochs ≈ 58 days. I had an overnight. Thirty seconds of arithmetic before hitting go would have caught this.

## Root causing the slowdown

Four real problems, one red herring:

1. **Augmentation inflated the dataset 7x** (2,651 → 17,824). Meant to fix precision, not pad data. Fix: `just augment-clean`, resubsample to 2-3x later.
2. **Model was `yolo26s`, not `yolo26n`.** ~3x slowdown for no reason on a single-class high-contrast detector. Fix: switch back, expose as a CLI flag.
3. **`imgsz=640`.** CPU cost is ~quadratic: (640/416)² ≈ 2.4x. Inference works at 384. Fix: train at 416.
4. **`epochs=100`, `patience=10`.** First epoch hit mAP50 = 0.951. Fix: cap at 40.
5. **`workers=0` (red herring).** Ultralytics force-sets this on CPU; GIL + cache locality mean more workers hurt. I burned an afternoon before reading their comment.

Stacked: 7 × 3 × 2.4 × 3 ≈ 150x. 58 days / 150 ≈ 9 hours. Overnight. Done.

Yes, I could have rented a T4 and been done in half an hour. I didn't want to. Part of the appeal of this project was doing it on my own hardware -- seeing what a 5600G with no discrete GPU could actually pull off if I bothered to tune it. Renting someone else's machine would have gotten me the model, but it would have skipped the part I was interested in.

## What it looks like working

Pipeline end-to-end now:

1. `just annotate` -- label 50 more photos in 20 minutes.
2. `just train --no-aug --model yolo26n.pt --epochs 40` -- train overnight on CPU.
3. `just infer` -- batch inference across all 7,500 scans in about 40 minutes.
4. `just dashboard` -- review predictions, correct the borderline cases, confirm the easy ones. A few hours of focused clicking.
5. Feed corrections back into `dataset/corrections/`, re-train.

The detection part is basically solved. The remaining uncertainty lives in the OCR step, where "4 23 '95" and "4 23 '85" look very similar on a faded scan, and is a different blog post entirely.

## Things I'd do differently

- **Sanity-check training runtime with a 1-epoch dry run before committing to the full config.** 30 seconds of arithmetic would have saved me from the 58-day fiasco.
- **Build the negative-example pipeline before training, not after.** The first model had enough false positives that I almost gave up on YOLO. Adding "skipped" photos as background examples fixed it in one training run.
- **Augment surgically, not generously.** The 7x augmentation explosion was me pattern-matching "more data is better" when what I actually needed was targeted bright-background samples. A 2x augmentation focused on the actual failure mode would have been more effective *and* cheaper to train.
- **Just rent the GPU.** Seriously. $0.50.

## Things I'd keep

- **Keyboard-first annotation UI.** The single best return on investment in the whole project. Labeling is the bottleneck; ergonomics of labeling is the bottleneck of the bottleneck.
- **Corrections dashboard that feeds back into training.** Active learning in its simplest form. The model gets better every week I use it, not because I retrained it on a bigger dataset but because I retrained it on the images where it was *wrong*.
- **Ultralytics' defaults.** Every time I went to override one, I either ended up putting it back or finding a comment in their source explaining why they were right. The project's opinions are load-bearing and mostly correct.
- **The apprise notifications.** Getting a Mattermost ping every 10 epochs meant I could kick off a run, walk away, and find out if it crashed without babysitting TensorBoard. Tiny feature, disproportionate quality-of-life.

## Final numbers

| Thing | Value |
| ----- | ----- |
| Photos to process | ~7,500 |
| Labeled by hand | ~2,600 |
| Negative examples | ~300 |
| Base model | YOLOv8-nano (~3M params) |
| Training time (CPU, no aug, 40 epochs) | ~9 hours |
| Inference time (all photos) | ~40 minutes |
| Precision / Recall / mAP50 | 95.3% / 95.8% / 95.0% |
| Cost to run on cloud GPU instead | roughly $0.25 |
| Sunsets incorrectly identified as dates | 0 |

The sunset count is the metric I'm proudest of.

## Epilogue: I finally rented the GPU

A couple of weeks after writing everything above, I stopped saying "I should just rent a GPU" and actually did it. The result was pretty embarrassing for the CPU side of this story.

I wrote `scripts/train/gpu_bench_one_epoch.py` as a throwaway "one epoch and bail" benchmarking harness -- stage a clean copy of the dataset (resolving every symlink, skipping the broken ones from my reorg), upload via presigned S3 URLs, launch a `g4dn.xlarge` spot instance with a bash cost monitor and a hard safety-net shutdown, run N epochs, pull the weights back, and terminate the instance on every exit path. I was nervous enough about leaving an EC2 instance running that I also put a second process on the host polling EC2 every 3 minutes and set to issue `terminate-instances` if `wall_hours * $0.5342/hr` ever exceeded `$4.50`. Belt, suspenders, and a third belt.

Passing `--epochs 40` instead of the default `--epochs 1` quietly repurposed the benchmark into a full training run. The script's output header still says "GPU BENCH RESULTS" and "validation metrics (after 1 epoch)" even when it's doing 40. I did not fix this. The lie is the feature.

### The numbers

| Thing | CPU run (earlier) | GPU run (g4dn.xlarge, 40 epochs) |
| --- | --- | --- |
| Per-epoch time | ~14 min (and climbing with augmentation) | 50.3 s |
| Total training wall time | ~9 hours | 33.55 min |
| mAP@50 | 0.950 | 0.962 |
| mAP@50-95 | 0.738 | 0.754 |
| Precision | 0.953 | 0.959 |
| Recall | 0.958 | 0.961 |
| Total $ cost | "power bill" | **$0.35** |

One catch: the first spot launch failed because 5 of 6 us-east-1 AZs had no g4dn.xlarge spot capacity and the 6th was us-east-1e, which doesn't offer g4dn.xlarge at all. The error handler correctly retried past `InsufficientInstanceCapacity` but didn't know to skip `Unsupported`. Added the `Unsupported` case, switched to `--pricing on-demand` for reliability, re-launched, and the whole run came in under $0.35. The spot price I was paranoid about saving would have bought me maybe ten cents.

The other thing that surprised me: the 1-epoch benchmark projected ~117 s/epoch, and the real run averaged 50.3. The first epoch on this model carries a big compile/warmup tax that doesn't amortize if you only measure one epoch. Future bench runs on this harness should discount the first-epoch cost or just run two epochs and measure the delta.

### Did the new weights actually get better, or just different?

Validation metrics moving from 0.95 to 0.962 mAP@50 is real but not dramatic. I was more worried about the failure modes shifting silently -- i.e. the new model scoring better on the *val split* while quietly breaking detections on photos the val split doesn't exercise. So before promoting the new weights I wrote `scripts/infer/compare_predictions.py` to diff the two models' predictions across all 6,458 scans the old CPU model had already handled.

For each stem, compute the IoU between the old bounding box and the new one. Flag each as:

- **stable** -- IoU >= 0.5 (same stamp, roughly same box)
- **drift** -- IoU < 0.5 (new box is somewhere meaningfully different)
- **gone** -- new model produces no detection where the old one did

The split came back:

```
stable   6,229   96.5%
drift      197    3.0%
gone        32    0.5%
```

and, importantly, the mean detection confidence jumped from **0.70 to 0.85**. The number of predictions clearing `conf >= 0.7` went from 4,583 to 6,075. Roughly 1,500 photos that the CPU model had flagged as "borderline, go look at this" got promoted to "obviously correct" by the GPU model -- so the practical payoff wasn't the +1.2 points of mAP, it was a review queue that shrank by about 25%.

I manually spot-checked maybe two dozen drifted and gone photos. Almost every one followed one of two patterns:

- **Drift**: the old model found something orange-and-rectangular that wasn't actually the stamp (a toy train, a price tag, a corner of a children's book cover) and the new model ignored the distractor in favor of the real stamp elsewhere in the frame. The README has a good example of a child-with-toy-train photo where the CPU model boxed the toy at 0.72 confidence and the new model found the corner stamp at 0.89.
- **Gone**: the CPU model got fooled by *drawings of postage stamps*. My single favorite example is a "The Old Post Office" illustration on a mall directory banner, where the cartoon has a zigzag stamp border and a building drawing inside it, and the CPU model scored this as a date stamp at 0.42 confidence. The GPU model correctly produces no detection. Somewhere in the training data there is not a single mall directory.

Both categories represent the new model being *more right*, not drifting away from a correct answer. Exactly what you want from a retraining delta.

### Lessons from a $0.35 experiment

- **Just rent the GPU.** I know. I keep writing this in every postscript.
- **Drift comparison is the promotion gate, not val metrics.** The val split is 663 images the model *trained against*. The 6,458-image drift comparison is the actual production set, and it caught qualitative improvements (the FP stamps-on-stamp-graphics case) that val-metric deltas alone don't surface.
- **Hard-cap the cost before you launch.** The EC2 cost monitor fired 13 times during the run and was prepared to kill the instance if it ran past $4.50. It never had to. But knowing it existed meant I could leave the run unattended without nervously refreshing the billing page.
- **Resume logic is still a landmine.** `gpu_bench_one_epoch.py` staged a fresh run from whatever `runs/` directory was on disk, which was a prior yolo26m checkpoint. I thought I was running yolo26n based on the CLI default and only noticed when the val log came back with "YOLO26m summary: 20,350,223 parameters." Not what I planned. Got lucky that medium + GPU was basically free at this scale and the model is objectively better than nano would have been. Same fix as last time, and I still haven't done it: make resume respect the `--model` flag and error out if architectures don't match.

So the new story is: CPU training is a fine development loop -- you can iterate on annotation, augmentation, and training config without touching a credit card -- but the moment you have a config worth committing to, spend 35 cents and promote from the GPU run. The two workflows don't compete; they stack.

And the review queue got shorter, which is what I actually cared about.


[^1]: If you're looking for a review, here is a short winded one: Easy process, expensive but worth it, good quality scans, easy to review images. Highly recommended to digitize old photos if you're in the same boat. 