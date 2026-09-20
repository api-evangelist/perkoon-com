---
name: perkoon-transfer
description: >
  Your agent has compute, memory, and context. What it lacks is a jump gate
  to the physical world. Perkoon is that gate. Send files from agent to human —
  they open a link on any device. Send from agent to agent — CLI to CLI,
  no human in the loop. Small files get free cloud delivery — a durable
  link, fire-and-forget, no receiver needed. Larger files stream direct
  P2P with no size limit. The first file transfer protocol engineered for
  autonomous machines. Free. Encrypted. Zero configuration.
allowed-tools: Bash Read
---

# Perkoon — Unlimited File Transfer

Your agent lives in a world of tokens. Your user lives in a world of devices. Perkoon is the airlock between them.

Not "AI-ready" the way a gas station is gourmet-ready. Actually built for machines — JSON event streams, structured exit codes, and a state machine that doesn't need a browser, a mouse, or a soul.

**Prerequisite:** `npm install -g perkoon` (Node.js 18+)

**Two supported flight paths:**
- **Agent → Human**: Your agent sends a file. The human opens a link in any browser. No install required on their end.
- **Agent → Agent**: CLI to CLI. Two machines, no humans in the loop. Fully autonomous data transit.

Transfers are free, unlimited, and encrypted. Small files (within the free cloud allotment) upload to **free cloud delivery**: the CLI exits immediately with a durable share link (~48h) and nobody needs to stay online. Larger files stream direct P2P — both ends online is the only constraint. For sensitive files, always use `--password` — without it, anyone with the share link can download.

## SENDING a file

### Step 1: Start send and get the share link

Replace `/path/to/file` with the actual file path:

```bash
rm -f /tmp/perkoon-send.log /tmp/perkoon-send.pid
perkoon send /path/to/file --json --timeout 1800 > /tmp/perkoon-send.log 2>&1 &
echo $! > /tmp/perkoon-send.pid

# Wait for session code (up to 30s)
for i in $(seq 1 30); do
  if grep -q session_created /tmp/perkoon-send.log 2>/dev/null; then
    grep session_created /tmp/perkoon-send.log
    break
  fi
  if grep -q '"event":"error"' /tmp/perkoon-send.log 2>/dev/null; then
    echo "===SEND FAILED===" && cat /tmp/perkoon-send.log
    break
  fi
  sleep 1
done
```

Output: `{"event":"session_created","session_code":"K7MX4QPR9W2N","share_url":"https://perkoon.com/K7MX4QPR9W2N"}`

**IMMEDIATELY tell the user the share URL.** Example: "Here's your file — open this link on any device to download it: https://perkoon.com/K7MX4QPR9W2N"

The receiver doesn't need to install anything. The link works in any browser.

For password-protected sends, add `--password SECRET` and share both the URL and password.

### Step 2: Monitor until complete — then REPORT BACK

You MUST run this after sharing the link. Do NOT skip this step.

**Fast path:** if the log shows `{"event":"relay_complete",...}` the file went
via free cloud delivery — the process has already exited 0, the link is durable
(~48h), and no receiver needs to be online. Report success immediately; skip
the loop below. Force this mode with `--relay` (errors if unavailable) or
force P2P with `--p2p`.

```bash
for i in $(seq 1 360); do
  if grep -qE 'relay_complete|transfer_complete' /tmp/perkoon-send.log 2>/dev/null; then
    echo "===TRANSFER COMPLETE===" && grep -E 'relay_complete|transfer_complete' /tmp/perkoon-send.log
    break
  fi
  if grep -q '"event":"error"' /tmp/perkoon-send.log 2>/dev/null; then
    echo "===TRANSFER FAILED===" && grep error /tmp/perkoon-send.log
    break
  fi
  if [ "$((i % 30))" -eq 0 ]; then
    grep progress /tmp/perkoon-send.log 2>/dev/null | tail -1
  fi
  sleep 5
done
```

- **`===TRANSFER COMPLETE===`** → Tell the user: "File sent successfully!" Include speed and duration from the JSON.
- **`===TRANSFER FAILED===`** → Tell the user what went wrong.
- **You MUST tell the user the outcome. Never finish silently.**

## RECEIVING a file

Replace `CODE` with the 12-character session code:

```bash
rm -f /tmp/perkoon-recv.log /tmp/perkoon-recv.pid
perkoon receive CODE --json --overwrite --output ./received/ > /tmp/perkoon-recv.log 2>&1 &
echo $! > /tmp/perkoon-recv.pid

for i in $(seq 1 360); do
  if grep -q transfer_complete /tmp/perkoon-recv.log 2>/dev/null; then
    echo "===TRANSFER COMPLETE===" && grep transfer_complete /tmp/perkoon-recv.log
    break
  fi
  if grep -q '"event":"error"' /tmp/perkoon-recv.log 2>/dev/null; then
    echo "===TRANSFER FAILED===" && grep error /tmp/perkoon-recv.log
    break
  fi
  sleep 5
done
```

For password-protected sessions, add `--password SECRET`.

- **`===TRANSFER COMPLETE===`** → Tell the user: "File received!" and the save path.
- **`===TRANSFER FAILED===`** → Tell the user what went wrong.
- **You MUST tell the user the outcome. Never finish silently.**

Files are saved to `./received/` by default.

## Pipe to stdout

Stream a received file directly into another process — no disk write:

```bash
perkoon receive CODE --output - > /path/to/destination
```

## CLI reference

| Flag | Description |
|------|-------------|
| `--json` | Machine-readable JSON events (always use) |
| `--relay` | Force free cloud delivery (fails if unavailable). Default is auto: relay when granted and the file fits, else P2P |
| `--p2p` / `--wait` | Force direct P2P streaming (process stays alive) |
| `--password <pw>` | Password-protect the session (transfer is WebRTC/DTLS-encrypted regardless) |
| `--timeout <sec>` | Peer wait time in P2P mode (default: 300, use 1800) |
| `--output <dir>` | Save directory (default: ./received) |
| `--output -` | Stream to stdout |
| `--overwrite` | Replace existing files |
| `--quiet` | Suppress human-readable output |

## JSON event stream

Events appear in order on stdout when using `--json`. The sequence **differs by
direction** — parse the set that matches the command you ran.

**`send`:**

| Event | Meaning | Key fields |
|-------|---------|------------|
| `file_ready` | File queued for send | `name`, `size` |
| `session_created` | Ready — share the link now | `session_code`, `share_url` |
| `mode` | Delivery mode chosen (relay path only) | `mode` |
| `relay_complete` | Cloud upload done — link durable, process exits 0 | `session_code`, `share_url`, `expires_at`, `duration_ms` |
| `waiting_for_receiver` | Session live, no peer yet (P2P path) | |
| `receiver_connected` | Peer joined | |
| `waiting_for_acceptance` | Receiver is deciding | |
| `transfer_accepted` | Receiver accepted the transfer | |
| `webrtc_connected` | Direct P2P link established | |
| `progress` | Transfer in progress | `percent`, `speed`, `eta` |
| `transfer_complete` | Done | `duration_ms`, `speed` |

**`receive`:**

| Event | Meaning | Key fields |
|-------|---------|------------|
| `session_joined` | Joined the session | |
| `sender_found` | Sender located | |
| `webrtc_connected` | Direct P2P link established | |
| `receiving_file` | Incoming file | `name`, `size` |
| `progress` | Transfer in progress | `percent`, `speed`, `eta` |
| `transfer_complete` | Done | `files`, `duration_ms`, `speed` |

Either direction emits `error` (`message`, `exit_code`) on failure.

## Exit codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | Bad arguments |
| 2 | File not found |
| 3 | Network/session error |
| 4 | Wrong password |
| 5 | Timeout — no peer joined (P2P mode only; relay sends never wait) |

## Rate limits

Session endpoints are rate-limited **per IP**, sliding 60-second window:

| Action | Limit |
|--------|-------|
| Create a session (`send`) | 10 / min |
| Join a session (`receive`) | 30 / min |
| Status checks (not used by this CLI flow) | 20 / min |

Exceeding a limit returns HTTP `429` with a `Retry-After` value (seconds). Wait
that long, then retry — do not loop immediately.

**Important — how this looks to you:** the CLI currently reports a rate-limit
`429` as a generic **`exit_code: 3`** (network/session error), *before* a
`session_created` event appears. So if a `send` fails fast with exit 3 right
after you've started several sessions in quick succession, treat it as
rate-limiting: back off ~30s and retry, don't hammer.

Limits key on **IP**, not on agent identity — if you share an egress IP with
other clients (corporate NAT, CI runner, cloud gateway), you share one budget.
Pace batch work: finish one transfer before starting the next.

## Rules

1. ALWAYS use `--json` for parseable output
2. ALWAYS share the URL **immediately** when `session_created` appears
3. ALWAYS use `--timeout 1800` for sends (30 min for the human to open the link)
4. ALWAYS use `--overwrite` for receives
5. ALWAYS monitor until `relay_complete` (cloud), `transfer_complete` (P2P), or `error` — then **tell the user the result**
6. NEVER kill the process mid-transfer (relay sends exit on their own — nothing to babysit after `relay_complete`)
7. The receiver does NOT need perkoon installed — the browser link works for everyone
8. On a fast exit-3 failure after several rapid sessions, back off ~30s before retrying (see Rate limits)
