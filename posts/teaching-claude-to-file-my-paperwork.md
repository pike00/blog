---
title: "Teaching Claude to file my paperwork"
description: "A Claude Code skill that turns junk filenames in scattered inbox directories into properly-named, deduped, OCR'd documents in the right folder — walked through end-to-end with a Valvoline oil change receipt."
date: "2026-05-24"
tags: ["Homelab", "Claude", "Automation", "Documents"]
draft: false
---

I have several "inbox" directories scattered across my filesystem — `~/Documents/Finance/inbox`, `~/Documents/Car/inbox`, a generic `~/inbox`, and so on. Things land there from browser downloads, scanner output, forwarded email attachments, and a pipeline that fetches PDFs from links inside forwarded emails. They have names like `attachment (3).pdf`, `Screenshot 2026-04-12 at 10.15.32.png`, or `statement.pdf`. Useless.

For years I'd let them pile up, then once a quarter sit down and rename, dedupe, and file them by hand. It was the kind of chore I'd put off until I couldn't find a thing I needed.

So I wrote a Claude Code skill called `ingest-inbox`. It's about 1,100 lines of `SKILL.md` — mostly a destination catalog ("paystubs go here, vet bills go there"), filename conventions, dedupe rules, and a handful of mode-switches. The interesting part isn't the rules. It's that I no longer have to know the rules.

## How it works, mechanically

The skill resolves the inbox path to one of five **modes**:

| Inbox | Mode | What changes |
|---|---|---|
| `~/Documents/Finance/inbox` | `finance` | Auto-files per account map, runs beancount extraction |
| `~/Documents/<Subtree>/inbox` | `subtree` | Files within that subtree, proposes per group |
| `~/Documents/inbox` | `documents` | Heuristic destinations, asks per group |
| `~/inbox` | `home` | Routes real docs into `~/Documents/inbox/`, flags trash |
| anything else | `generic` | Identifies, proposes per file |

For every PDF, it runs a local docling container at `http://localhost:5001` and writes a `.md` sidecar — the canonical text-extraction step. It dedupes against the destination (existing OCR'd copies have sidecars; raw inbox downloads don't, and I never want the raw one overwriting the OCR'd one). Filenames get normalized to `YYYYMMDD <descriptor>.ext`. Duplicates go to `_duplicates/` for audit, never deleted outright.

In Finance mode it goes further: extracts every transaction and balance, categorizes against the existing accounts in my beancount ledger (never inventing new ones), and writes a self-contained ingestion note in `cleanup/` that I can paste into the ledger. (Follow up blog post to come about how I use plaintextaccounting)

## Worked example: an oil change

Earlier today, I got the car serviced. Here's what happened end-to-end without me touching anything until the very last step:

1. Valvoline emails the receipt to my Gmail. I forward it to `<inboxname>@<domain-i-own.com>`.
2. **Email-ingest pipeline** (Gmail → SES → S3 → Lambda → a FastAPI receiver on my homelab) classifies the route as `car`, appends the body to `~/Documents/Car/vault/2026-05.md` with a tag, and notices the email contains a link to a PDF invoice.
3. It follows the link and saves that page if appropriate (e.g. this email from Valvoline had a direct link to the PDF to download). If it's a PDF, drop it into `~/Documents/Car/inbox/` with three sidecars: `.meta.yml` (sender, subject, classifier route), `.links.fetched.yml` (source URL, sha256, http status), and a `.docling.md` it already extracted.
4. I tell Claude: *"ingest the Car inbox."* It invokes `ingest-inbox`, resolves the path to **`subtree` mode** with `SUBTREE=Car`, reads the sidecars (so it skips re-running docling), and identifies the file as a Valvoline invoice for the 2023 Forester.
5. It proposes filing into `~/Documents/Car/Service/20260524 Valvoline Oil Change/`, with the PDF renamed to `20260524 Valvoline Oil Change.pdf` and the sidecar travelling with it. I confirm.
6. `mv -n`. Done.

What used to be a ten-minute task — open the PDF, figure out what it is, rename it, find the right folder, check for a duplicate, move it — is now me typing one sentence.

## Why I bother

The skill itself isn't impressive — it's a destination map plus some bash plus a polite invocation of docling. What's impressive (to me) is the second-order effect: I now forward things to my inbox without dread. The Valvoline email used to be an item on a future chore list. Now it's a 30-second round-trip through the homelab and it ends up in exactly the folder I'd have put it in, with a markdown sidecar I can grep.

The next time I'm at the mechanic and they ask "when was the last oil change", I can answer in five seconds with `ls ~/Documents/Car/Service/ | tail`. That's the whole point.
