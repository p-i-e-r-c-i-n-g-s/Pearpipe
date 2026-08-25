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
- **Divergence from upstream is now deliberate, not accidental.** An earlier
  version of this file said to consider whether a feature belongs upstream
  before adding it here. That was right when the fork was branding-only; it is
  no longer the plan. This repo is being personalised for Pearcache, so build
  what Pearcache needs and expect the gap to widen.

## What diverges from upstream, and what a merge must preserve

Anything upstream does not have. Re-check this list before any `git merge
upstream/main`, because a merge that silently reverts one of these looks like
nothing happened until something breaks in production.

1. **Branding assets** — `mark.svg`, `icon.svg`, `favicon.svg`,
   `site.webmanifest` and the PNG icons under `internal/relay/static/`, plus
   the `/static` path prefix they are served under.
2. **The GHCR image name.** `.github/workflows/release.yml` publishes to
   `ghcr.io/p-i-e-r-c-i-n-g-s/pearpipe-relay`. Upstream's copy says
   `ghcr.io/sanyam-g/airpipe-relay`, which this fork's `GITHUB_TOKEN` cannot
   write to — that failed the Release workflow on every push to `main` until
   it was retargeted. A merge that takes upstream's version reintroduces it.
3. **Seven bug fixes that were never sent upstream** (deliberate call, Aug
   2026). These live only here, so an upstream merge that touches the same
   functions can silently undo them. Each is worth re-verifying after a merge:
   - `internal/transfer/protocol.go` — `DecodeMessage` length check must widen
     to `uint64` before adding, or a crafted header panics the receiver.
   - `internal/p2p/peer.go` — `deliver()` must hold `incomingMu` across the
     send, or a concurrent `Close()` panics pion's read goroutine.
   - `internal/relay/rooms.go` / `server.go` — empty-room delete must run in
     the same defer as `RemoveClient`, via `deleteRoomIfEmpty(token, room)`.
   - `internal/relay/store.go` — `FileStore.Store` must re-check the token
     under the write lock.
   - `internal/transfer/signaling.go` — `stopTrickle()` on the post-trickle
     error returns, or ordinary NAT-traversal failure races the websocket.
   - `cmd/airpipe/send.go` — `os.Stdin.Stat()`'s error must be handled.
   - `cmd/airpipe/update.go` — paths passed to `sudo sh -c` must be
     positional, not interpolated.

   The branch `upstream-fixes/panic-races-relay-leaks` holds these as seven
   atomic commits on top of `upstream/main`, ready to contribute if that call
   is ever revisited. It is not merged anywhere and nothing depends on it.

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
