---
title: "Coldkey: Paper Backup for Post-Quantum Age Keys"
description: "A Go CLI that generates age encryption keys and produces a single-page printable HTML backup with QR codes. Print it, laminate it, store it in a fireproof safe."
date: "2026-07-22"
tags: ["Security", "Cryptography", "Go", "CLI"]
draft: false
---

I use [age](https://age-encryption.org/) and [SOPS](https://github.com/getsops/sops) for all my secrets management. Every `.env` file in my homelab is encrypted in git, decrypted in memory at deploy time. It works well -- but it creates a single point of failure: the age private key. Lose that key and every encrypted file becomes unrecoverable.

I needed a paper backup. Not a digital backup (that's the same failure domain), not a password manager export (tied to a cloud account), but something I could print, laminate, and put in a fireproof safe. Something that would survive a house fire, a stolen laptop, and a cloud provider going under.

I built [coldkey](https://github.com/pike00/coldkey).

## What it does

`coldkey generate` creates a post-quantum age key pair (ML-KEM-768 + X25519 hybrid) and produces a single-page HTML document with:

- The raw key text in monospace for manual transcription
- QR code(s) encoding the full key file
- A SHA-256 checksum for verification
- Step-by-step recovery instructions

The QR encoding is the interesting part. A post-quantum age key stores only the 32-byte seed (not the expanded ML-KEM-768 private key), so the full `keys.txt` is about 2,089 bytes -- fitting in a single version-40 QR code. If a key file exceeds that, coldkey splits across multiple QR codes with a framing protocol (`COLDKEY:part/total:data`) and a checksum.

## Security model

The key generation itself is hardened:

- `mlockall(MCL_CURRENT | MCL_FUTURE)` prevents key material from being swapped to disk
- Files written with mode `0600` and fsynced; temporaries shredded with 3-pass overwrite
- Secrets passed via stdin or files only, never in process arguments

There's also a Docker hardened mode using `distroless/static:nonroot` with `--network none --read-only --cap-drop ALL --security-opt no-new-privileges:true`. The container has no shell, no network, and a read-only root filesystem. Key material exists only in RAM and the host-mounted output directory.

## The paper backup problem

Most key management advice says "back up your keys" but doesn't specify how. A digital backup to S3 or Backblaze is in the same trust domain as the original -- if an attacker gets your cloud credentials, they get both the ciphertexts and the key. A paper backup breaks that chain. The key lives in a different physical and digital domain from the secrets it protects.

The recovery procedure is straightforward: scan the QR codes, strip the framing prefix, concatenate, verify the SHA-256 checksum, and you have your `keys.txt` back. I tested it by generating a key, printing the backup, shredding the digital copy, and recovering from the printout. It works.

## Install

```bash
brew install --cask pike00/tap/coldkey
# or
go install github.com/pike00/coldkey/cmd/coldkey@latest
```

The repo is at [github.com/pike00/coldkey](https://github.com/pike00/coldkey) with a Homebrew tap, Docker image, and CI. 64 stars so far -- more people than I expected have the same failure mode.
