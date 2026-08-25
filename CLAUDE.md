# Pearpipe

Pearcache edition of **AirPipe** — send a file to any device by sharing a
passphrase. It streams peer-to-peer, encrypted end to end; the relay that
introduces the two ends never sees plaintext. One Go binary that bundles the
relay, the web UI and the install script.

`README.md` is upstream's and is a good description of how AirPipe works
(passphrase flow, NAT traversal, mailbox mode, encryption, CLI). Read it for
the mechanism. This file covers what it does not say.

## This is a fork

Upstream is `Sanyam-G/Airpipe`; the Go module path is
`github.com/sanyamgarg/airpipe`. Of 76 commits, only the fork's last two are
Pearcache-specific, and both are branding:
- `24c18b7` Replace the inherited AirPipe favicon with the Pearpipe mark
- `0d71726` Fix the manifest icon paths — static is served under `/static`

Everything else is upstream. Practical consequences:

- **`README.md` describes upstream's deployment, not ours.** "Try it" points at
  `airpipe.sanyamgarg.com`, the self-host image is
  `ghcr.io/sanyam-g/airpipe-relay`, `docker-compose.yml` pulls that same
  upstream image, and the install script is served from upstream's host. None
  of those are Pearcache infrastructure. There is currently **no Pearcache
  deployment of this repo** — if one is stood up, the README's hostnames and
  the compose image are what need changing.
- The repo's GitHub `homepage` field also points at upstream
  (`airpipe.sanyamgarg.com`).
- **The branding assets are the only thing genuinely ours**: `mark.svg`,
  `icon.svg`, `favicon.svg` and `site.webmanifest` under
  `internal/relay/static/`. Those are what to protect when merging upstream.
- Before adding features here, consider whether they belong upstream. Beyond
  branding, divergence makes upstream merges expensive.

## Architecture

```
cmd/airpipe/       the CLI (send, download, receive, update, help, ui)
cmd/relay/         the relay server binary
internal/relay/    relay implementation + embedded web UI (static/)
internal/crypto/   NaCl secretbox (XSalsa20+Poly1305), passphrase-derived keys
internal/passphrase/  wordlist and passphrase generation
internal/mailbox/  store-and-forward payload handling
internal/qr/       QR generation
internal/progress/ transfer progress reporting
```

Go 1.25. Key deps: `gorilla/websocket`, `pion/webrtc/v4`, `skip2/go-qrcode`,
`golang.org/x/crypto`. The frontend is plain HTML/CSS/JS embedded into the
binary — `tweetnacl.js` in the browser mirrors the Go crypto so the passphrase
derives the same key on both sides.

## Setup, build and test

```bash
go build ./...
go vet ./...
go test ./...        # tests in internal/{crypto,passphrase,mailbox,progress,relay} and cmd/airpipe
```

CI (`.github/workflows/ci.yml`) runs exactly `go vet ./...` and `go test ./...`
on Go 1.25 for pushes and PRs to `main`. `release.yml` handles releases. Match
CI locally before pushing — there is nothing else gating this repo.

## Relay configuration

All optional; the relay reports its live config and stats at `/health` (JSON)
and `/metrics` (Prometheus), and the web UI reads real limits from `/health`.

| Var | Default | Notes |
|---|---|---|
| `PORT` | `8080` | |
| `AIRPIPE_RELAY` | — | Client-side: point the CLI at a relay (or `--relay` per call) |
| `AIRPIPE_ALLOWED_ORIGINS` | same-origin | Extra origins; `*` = any |
| `AIRPIPE_RATE_LIMIT_PER_MIN` | `60` | |
| `AIRPIPE_MAX_UPLOAD_MB` | `500` | Mailbox size cap |
| `AIRPIPE_FILE_EXPIRY` | `10m` | Mailbox expiry (Go duration) |

## Conventions and traps

- **Static assets are served under `/static`.** That is what the icon-path fix
  (`0d71726`) was about — a manifest or page referencing icons at the root will
  404. Check the path prefix when adding any asset.
- **The crypto is symmetric across two implementations.** `internal/crypto`
  (Go) and `static/airpipe-protocol.js` (browser) must derive identical keys
  from the same passphrase via domain-separated SHA-256. Changing one without
  the other silently breaks browser↔CLI transfers while same-language
  transfers keep working — so test across the boundary, not just `go test`.
- The protocol is versioned and fails closed on mixed versions (`Protocol v3`).
  Bumping it means both ends move together.

## Visibility

**This repository is public.** It currently contains no committed secrets
(verified: no `.env` tracked, none in history, no credential-shaped literals).
Note the repo has no `.gitignore` entry for `.env` — add one before introducing
any local environment file.
