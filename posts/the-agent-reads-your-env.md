---
title: "The agent reads your .env. So does the prompt cache."
description: "I built drape because the obvious fixes — don't have .env in the repo, use a vault, gitignore harder — don't help when the agent is the thing doing the reading."
date: "2026-05-07"
tags: ["Security", "Claude Code", "Python", "drape"]
draft: false
---

# The agent reads your .env. So does the prompt cache.

Claude Code asks to read `.env` so it can wire up your local dev server. You hit accept. The file goes into the conversation. The conversation goes into the prompt cache. The cache lives on someone else's hardware for some retention window you don't fully control.

Now multiply that by every agent you've pointed at a repo with credentials checked out next to the code.

I built [drape](https://github.com/pike00/drape) because the obvious fixes (don't have `.env` in the repo; use a vault; gitignore harder) don't help when the agent is the thing doing the reading.

## What it does

```
$ cat .env
DATABASE_URL=postgres://user:hunter2@localhost/db
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
GITHUB_TOKEN=ghp_1234567890abcdef1234567890abcdef1234
APP_PASSWORD=correct-horse-battery-staple
RANDOM_HEX=a8f3c9e7b2d1f4a6c8e0b9d7f2a1c4e6

$ drape .env
DATABASE_URL=<basic-auth>
AWS_ACCESS_KEY_ID=<aws-access-key>
GITHUB_TOKEN=<github-token>
APP_PASSWORD=<low-entropy-secret>
RANDOM_HEX=a8f...
```

The agent still gets a useful file. It sees the shape of the config, the variable names, and a hint at what kind of credential each holds. The actual bytes never enter the model context.

## Three strategies, most-protective wins

drape applies three masking strategies in order. Whichever hides more, wins.

**Pattern recognition.** Known credential shapes are replaced with a type label. AWS keys, GitHub tokens, Slack tokens, JWTs, Stripe keys, private keys, basic-auth URIs, and about fifteen others. Powered by Yelp's `detect-secrets`. Zero characters leak; the agent only learns the type.

**Entropy gate.** Values whose Shannon entropy falls below 3.0 bits per character render as `<low-entropy-secret>`. This catches `hunter2`, `correct-horse-battery-staple`, and the like. A three-character prefix of a passphrase is too informative to share with anything that logs.

**Length-bounded prefix.** For high-entropy values that don't match a pattern, drape reveals the first three characters, capped at 25% of the value's length. A 22-char hex blob shows three chars (16^19 brute force space remaining). A four-char value shows one char.

Order matters. A short AWS key would otherwise leak a meaningful prefix; the pattern detector catches it first and replaces it with a label. The 25% cap prevents the prefix strategy from doing the wrong thing on tiny values. The defaults are conservative because the cost of getting them wrong is asymmetric.

## Read isn't the only leak path

The first version of drape only intercepted `Read` calls. That's not enough. An agent with `Bash` access can `cat .env` directly. Or `grep PASSWORD .env`. Or pipe through `awk -F=` to skip the masking layer entirely. The hook now covers three Claude Code tools:

- `Read` on a `.env`-shaped path returns the masked rendering.
- `Grep` on a `.env`-shaped path re-runs the search against the masked rendering, so matches on key names still surface but values stay hidden.
- `Bash` commands that try to read a `.env`-shaped path with `cat`, `head`, `tail`, `less`, `grep`, `rg`, `ag`, `sed`, `awk`, or `cut` are denied with a redirect telling the agent to use `drape` instead.

The agent doesn't lose the ability to look at config. It loses the ability to look at config the unsafe way.

## SOPS without round-tripping plaintext

For encrypted `.env.sops` files, drape detects the format, shells out to `sops -d`, masks the in-memory plaintext, and prints. The decrypted file is never written to disk and never returned to the caller. Same logic whether you invoke `drape` from the shell or the hook fires it.

## Audit without disclosure

Set `DRAPE_AUDIT_LOG=~/.drape-audit.jsonl` and every masking operation appends a line:

```
{"ts":"2026-05-06T19:30:00+00:00","event":"cli_mask","file":".env",
 "format":"env","key_count":12,"prefix_chars":3}
```

Filename, format, key count. No values, ever. You get a forensic trail of when an agent looked at credentials without writing the credentials anywhere new. The audit writer is the only path in drape that swallows exceptions: if the log destination is unwritable, masking still succeeds. Failing closed on the audit log would be worse than failing open.

## Threat model, briefly

drape protects against:
- secrets appearing verbatim in agent transcripts and prompt-cache snapshots
- accidental copy-paste of a credential into chat
- third-party access to exported or shared sessions

It does not protect against:
- a compromised local machine (the agent can read what your shell can read)
- a compromised LLM account (the attacker can request the unmasked file directly)
- multiline secrets in `.env` (use a structured format and `--format yaml|json|toml`)

It's a defense-in-depth tool. It removes the easiest, dumbest leak path. The rest of your security still has to do its job.

## Install

```
pip install drape                # core: .env, .env.sops
pip install 'drape[all]'         # + YAML, TOML
bash scripts/install-claude-hook.sh --project-dir .
```

64 tests, 84% line coverage, MIT license. Source at https://github.com/pike00/drape.
