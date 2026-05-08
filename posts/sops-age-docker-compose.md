---
title: "Secrets at Rest: SOPS + age for Docker Compose Homelabs"
description: "SOPS encrypts .env files in place, git tracks the encrypted versions, and sops exec-file decrypts them into memory at deploy time. Plaintext never touches disk."
date: "2026-04-29"
tags: ["Homelab", "Security", "Docker", "SOPS"]
draft: false
---

# Secrets at Rest: SOPS + age for Docker Compose Homelabs

I've committed a `.env` file to a public repo exactly once. A secret scanning service I use 
flagged it within a few minutes. That was a good outcome -- but it was also the moment
I realized that "add `.env` to `.gitignore`" is a policy that depends entirely on never
making a mistake. A stray `git add -A`, a backup tool that doesn't respect `.gitignore`,
a sync client that treats your project directory as documents -- any of them will silently
blow past that boundary. I wanted a setup where committing the file to a public repo
would be the safe choice, not the dangerous one.

This is what I landed on: SOPS encrypts `.env` files in place, git tracks the encrypted
versions, and `sops exec-file` decrypts them into memory at deploy time. Plaintext never
touches disk. 

---

## Why age over GPG

SOPS supports several encryption backends. GPG was the original; I use `age`.

The practical difference: GPG requires a keyring daemon, has a concept of key expiry and
trust levels, and stores keys in `~/.gnupg` with its own tooling. `age` has none of that.
The private key is a file at a path you choose. The public key is a string you paste into
a config file. There's no daemon to restart, no keyring to initialize on a new machine,
no trust model to configure.

The other difference that matters here is key format. Upstream `age` v1.3.0 added a hybrid
post-quantum key format: ML-KEM-768 + X25519. The public key starts with `age1pq1...`
and runs about 1,959 characters. A plain X25519 key, by contrast, is about 62 characters.
The hybrid construction means the ciphertext is secure as long as either the ML-KEM leg
or the X25519 leg holds. For encryption keys that will stay in git history for years -- and mine
will -- that seems like the right tradeoff. No plugin is required; this is stock
`brew install age`.

---

## Key generation and the cold key problem

```bash
brew install age sops

age-keygen -pq -o ~/.config/sops/age/keys.txt
chmod 600 ~/.config/sops/age/keys.txt
```

The `-pq` flag generates the hybrid identity. Without it you get a plain X25519 key.
SOPS discovers `~/.config/sops/age/keys.txt` automatically; no environment variable needed.

Here's the problem I didn't think through carefully enough the first time I set this up:
my cloud backups are encrypted with this key. If I lose the key, I lose the ability to
decrypt those backups. I cannot restore the key from the cloud backups, because those
backups need the key to decrypt. The key has to be backed up somewhere the cloud backup
can't reach.

I keep a copy in a Bitwarden secure note (local vault, not cloud sync) and on paper.
Never in S3, Dropbox, rclone to Backblaze, or any cloud provider. If an attacker gets
the repo, they also get the encrypted values -- and if they can somehow get the key, they
get everything. Keeping the key on a different attack surface from the ciphertexts is
the whole point.

The public key -- the `age1pq1...` recipient string in `.sops.yaml` -- is fine to commit
and share. It's the private key in `keys.txt` that needs to stay cold.

---

## `.sops.yaml` configuration

SOPS looks for `.sops.yaml` in the repository root or a parent directory. I keep mine at
`infra/sops/.sops.yaml` and pass `--config` explicitly (the wrapper script below handles
this automatically).

```yaml
creation_rules:
  # .env files -- dotenv format; leave non-secret config readable in git
  - path_regex: .*\.env(\.sops)?$
    input_type: dotenv
    output_type: dotenv
    age: &age_key "age1pq1rlg3cuef3cpp0zfgg4me2kpgkkwpy5tj84..."
    unencrypted_regex: "^(TZ|PUID|PGID|UMASK|PGDATA|LOG_LEVEL|NODE_ENV|COMPOSE_PROJECT_NAME)$"

  # Terraform tfvars -- HCL isn't a native SOPS format, treat as binary
  - path_regex: .*\.tfvars(\.sops)?$
    age: *age_key

  # TLS certs and private keys
  - path_regex: .*\.(pem|key)(\.sops)?$
    age: *age_key

  # JSON credential files
  - path_regex: .*credentials(\.sops)?\.json$
    age: *age_key
    encrypted_regex: "TunnelSecret"
```

Three things I'd call out:

`&age_key` / `*age_key` is a YAML anchor. The recipient string appears once; everything
references it. When I rotate the key, I change one line.

`unencrypted_regex` leaves non-secrets in plaintext. Values like `TZ`, `PUID`, and
`NODE_ENV` aren't secrets -- they're config. Leaving them unencrypted means `git diff`
on a `.env.sops` shows readable changes to config alongside opaque blobs for actual secrets.
It also makes the file slightly less hostile to debugging.

The `.sops` suffix in the path regex (`*.env(\.sops)?$`) means the same creation rule
matches both `app.env` (a plaintext file you're encrypting for the first time) and
`app.env.sops` (an already-encrypted file). I use `.env.sops` as the canonical form
in git.

---

## The `sopsx` wrapper

The raw `sops` command requires explicit `--input-type`, `--output-type`, and `--config`
flags for every operation. For a dotenv file that's:

```bash
sops --config infra/sops/.sops.yaml \
  --input-type dotenv --output-type dotenv \
  -d ai/myapp/.env.sops
```

I wrote a wrapper called `sopsx` that dispatches the right flags based on file extension:

```bash
#!/usr/bin/env bash
set -euo pipefail

REPO_ROOT="$(git rev-parse --show-toplevel 2>/dev/null || echo ".")"
cd "$REPO_ROOT"
SOPS_CONFIG="infra/sops/.sops.yaml"

SOPS_FILE="$1"; shift || true
DECRYPT_MODE=false; ENCRYPT_MODE=false

for arg in "$@"; do
  case "$arg" in
    -d|--decrypt) DECRYPT_MODE=true ;;
    -e|--encrypt) ENCRYPT_MODE=true ;;
  esac
done

# .sops files default to decrypt; -e overrides to stdin->file
if [[ "$SOPS_FILE" == *.sops* ]] && [[ "$ENCRYPT_MODE" == false ]]; then
  DECRYPT_MODE=true
fi

flags_for() {
  case "$1" in
    *.env.sops|*.envrc.sops) echo "--input-type=dotenv --output-type=dotenv" ;;
    *.tfvars.sops|*.pem.sops|*.key.sops) echo "--input-type=binary --output-type=binary" ;;
    *) echo "" ;;
  esac
}

if [[ "$DECRYPT_MODE" == true ]]; then
  sops --config "$SOPS_CONFIG" -d $(flags_for "$SOPS_FILE") "$SOPS_FILE"

elif [[ "$ENCRYPT_MODE" == true ]]; then
  TMPFILE=$(mktemp)
  trap 'rm -f "$TMPFILE"' EXIT
  sops --config "$SOPS_CONFIG" -e $(flags_for "$SOPS_FILE") \
    --filename-override "$SOPS_FILE" /dev/stdin > "$TMPFILE"
  mv "$TMPFILE" "$SOPS_FILE"

else
  sops --config "$SOPS_CONFIG" "$SOPS_FILE"
fi
```

The encrypt path writes to a tmpfile, then `mv` replaces the target atomically. If
encryption fails halfway, the existing `.env.sops` is untouched. The `--filename-override`
flag tells SOPS which creation rule to match against, since we're encrypting `/dev/stdin`
rather than the actual target path.

What started as `sopsx` grew. I kept reaching for adjacent operations -- "list every
encrypted file in the tree", "show me which plaintext copies are stale", "test-decrypt
all of them after a key rotation", "inject these vars into a subprocess for one command"
-- and ended up with a handful of overlapping wrappers. They've since collapsed into a
single `secrets` script with subcommands. The encrypt/decrypt logic above is one of them
(`secrets sopsx`); the other notable additions are `secrets exec-env` (decrypt and
inject as environment variables for a subprocess, backed by `sops exec-env`) and
`secrets keys` (list variable names without decrypting values to stdout). Fleet ops
(`list`, `verify`, `diff`) round it out.

---

## `just` recipes

[`just`](https://github.com/casey/just) is a command runner I use for homelab task
automation. The secrets surface is one passthrough recipe:

```makefile
secrets *args:
    @infra/scripts/secrets {{args}}
```

Common operations in practice:

```bash
# Interactive edit -- opens $EDITOR, re-encrypts on save
just secrets sopsx ai/hermes/.env.sops

# Create a new .env.sops; values generated and piped directly, never written to disk
just secrets sopsx ai/myapp/.env.sops -e <<EOF
DB_PASSWORD=$(openssl rand -hex 32)
API_KEY=$(openssl rand -base64 32)
EOF

# Rotate a single value
NEW=$(openssl rand -hex 32)
just secrets sopsx ai/myapp/.env.sops -d \
  | sed "s/^DB_PASSWORD=.*/DB_PASSWORD=$NEW/" \
  | just secrets sopsx ai/myapp/.env.sops -e

# Inject env vars into a subprocess (no temp file, no shell export)
just secrets exec-env ai/hermes/.env.sops -- some-command --with-args

# See what variable names a file contains, without exposing values
just secrets keys ai/hermes/.env.sops

# Integrity check
just secrets verify
```

The rotation pipeline is worth examining: `sopsx -d` writes plaintext to stdout, `sed`
substitutes the value, `sopsx -e` reads from stdin and encrypts back to the same file.
The new value `$NEW` exists only in shell memory. No temp file, no plaintext on disk.

`exec-env` covers a different gap. When a script needs the decrypted values as
environment variables for a subprocess -- a one-shot CLI tool, a test runner, an
out-of-band query -- the alternative is `source <(sops -d file.env.sops)`, which leaves
the variables in the calling shell's environment afterwards. `secrets exec-env` runs the
subprocess directly, so the variables exist only in that child's address space and are
gone when it exits.

---

## The deploy pattern: `sops exec-file`

`sops exec-file` is the mechanism that makes the rest of this work. It decrypts a file
into an in-memory file descriptor, exposes the path as `{}` in a command template, runs
the command, and cleans up when the subprocess exits:

```bash
sops exec-file --no-fifo \
  --input-type dotenv \
  --output-type dotenv \
  ai/myapp/.env.sops \
  'docker compose -f ai/myapp/docker-compose.yml --env-file {} up -d'
```

The `--no-fifo` flag matters on Linux. Without it, SOPS uses a named pipe. Docker Compose
needs a seekable file handle for `--env-file`; named pipes aren't seekable. With
`--no-fifo`, SOPS writes to a tmpfs path under `/proc/self/fd/`, which is seekable but
never touches the real filesystem.

I encapsulate this in a shared `compose.just` recipe:

```makefile
stack := ""
compose_file := stack / "docker-compose.yml"
env_sops := stack / ".env.sops"

up *args:
    #!/usr/bin/env bash
    set -euo pipefail
    if [[ -f "{{env_sops}}" ]]; then
        sops exec-file --no-fifo \
          --input-type dotenv --output-type dotenv \
          "{{env_sops}}" \
          'docker compose -f "{{compose_file}}" --env-file {} up -d {{args}}'
    else
        docker compose -f "{{compose_file}}" up -d {{args}}
    fi
```

The `else` branch handles stacks without secrets -- not every service has a `.env.sops`,
and the recipe should work uniformly either way. From the caller's perspective:

```bash
just up hermes       # has .env.sops, decrypted in-memory
just up homepage     # no .env.sops, plain compose
```

I also run these through a shell wrapper called `hl` so I don't need to `cd` first:

```bash
hl up hermes
hl ps monitoring
hl logs mattermost
```

---

## What the encrypted file looks like

An encrypted `.env.sops` committed to git:

```
TZ=America/New_York
PUID=1000
COMPOSE_PROJECT_NAME=hermes
POSTGRES_PASSWORD=ENC[AES256_GCM,data:7k9m...==,iv:abc...,tag:xyz...,type:str]
API_SECRET=ENC[AES256_GCM,data:Lm3p...==,iv:def...,tag:uvw...,type:str]
sops_version=3.9.1
sops_age__list_0__recipient=age1pq1rlg3...
sops_lastmodified=2026-04-28T14:23:11Z
```

The non-secret values (`TZ`, `PUID`, `COMPOSE_PROJECT_NAME`) are plaintext. The actual
secrets are AES-256-GCM ciphertext. The metadata footer includes the encrypted data key
wrapped to the `age1pq1...` recipient -- this is what SOPS decrypts first, using your
age private key, before it can decrypt anything else.

Note that `sops_age__list_0__recipient` stores the *public* key, not the private key.
Committing the file to a public repo exposes the public key and the ciphertexts, but not
the private key. Anyone who doesn't have `~/.config/sops/age/keys.txt` cannot decrypt it.

---

## Disaster recovery

Setup on a new machine:

```bash
brew install age sops

# Restore the key from offline backup only -- not from any cloud
cat > ~/.config/sops/age/keys.txt << 'EOF'
# created: ...
# public key: age1pq1...
AGE-SECRET-KEY-1...
EOF
chmod 600 ~/.config/sops/age/keys.txt

cd ~/Homelab
just secrets verify
# checking N files... 0 failed
```

`just secrets verify` decrypts every `.sops` file to `/dev/null` and reports how many
failed. I run it after any key restoration, any sops or age upgrade, and any time I'm
not sure the key on disk is the right one.

There is no key revocation in age. If the private key is lost and there's no offline
backup, the secrets encrypted to it are unrecoverable -- the age key was the only gate.
If the key is compromised, the recovery path is generating a new keypair, updating the
recipient in `.sops.yaml`, and re-encrypting every `.sops` file with the new key.

---

## Things that will burn you

`just config <stack>` renders the compose config with secrets injected and writes it to
stdout. I've found it useful for debugging, but it decrypts everything inline. Piping
that output to a file, or pasting it into a chat window, negates the whole workflow.

`--no-fifo` is off by default. The first time I tried `sops exec-file` without it, Docker
Compose failed silently on `--env-file` with a file handle error. Turn it on.

Shell variables holding secrets are fine in scripts -- `NEW=$(openssl rand -hex 32)` stays
in process memory. Writing `echo "$NEW" > /tmp/newsecret` creates a world-readable file
that survives reboots on some systems. The distinction matters less when you're the only
user on the machine; it matters more if the machine is shared or if tmp is persistent.

The cloud backup trap is the one I think about most. My B2 bucket contains encrypted
copies of everything in `~/Documents`. If the age key were also in that bucket, a
bucket compromise would be enough to decrypt everything in it. The key has to live
somewhere the backup can't reach.
