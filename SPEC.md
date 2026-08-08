# jetbot-console — Build Spec

**Target:** a Phoenix LiveView operator console for a physical robot (JetBot on a Jetson Nano).
**Purpose:** portfolio piece for Elixir-shop applications (Podium is the #1 target). Idiomatic
OTP matters more than feature count. This is a weekend, ~300-500 lines of application code.

**Author identity:** `Derek Clair <derek@derekclair.com>` · **License:** MIT · **Remote:**
`git@github.com:derekclair/jetbot-console.git` (already configured)

---

## 1. What this is

The JetBot runs a Tornado HTTP server (`jetbot-agent`) that exposes motors, camera, odometry
and on-device GPU inference. Today the only clients are an LLM agent and `curl`. This is the
human operator's console: live telemetry, manual driving, behavior control, and an E-STOP.

**Why it's worth building beyond the demo:** the robot's safety model is a *heartbeat*. A
continuous motion command keeps running until either a stop arrives or the server's watchdog
times out. That means the console holds a genuine safety obligation — if the operator's browser
closes, the pings must stop so the robot halts. Expressing that with `Process.monitor` and a
supervision tree is exactly what OTP is for, and it is the thing to talk about in an interview.

---

## 2. The API you are consuming

Base URL `http://<jetson-host>:5100`. Every response is JSON with an `ok` field:

```json
{"ok": true,  "...": "..."}          // success
{"ok": false, "error": "message"}    // failure
```

| method | path | request | response |
|---|---|---|---|
| GET | `/health` | — | `{ok, service, uptime_s}` |
| GET | `/probe` | — | hardware inventory (i2c bus scan, camera presence) |
| POST | `/robot/move` | `{direction, speed, duration?, left?, right?}` | `{ok, ...}` |
| POST | `/robot/stop` | — | `{ok}` |
| GET | `/robot/status` | — | see below |
| POST | `/robot/reset_odometry` | — | `{ok, yaw_deg: 0.0}` |
| POST | `/heartbeat/ping` | — | `{ok}` |
| GET | `/camera/snapshot` | query: `w`, `h`, `fmt` (`json`\|`raw`) | `{ok, image_b64, width, height, bytes_b64}` |
| POST | `/gpu/classify` | `{source, model, topk, image_b64?}` | top-K ImageNet predictions |
| GET | `/behavior/list` | — | `{ok, behaviors: [...]}` |
| POST | `/behavior/start` | `{name, params}` | `{ok, job_id, name, params}` · **409** if one is already active or the name is unknown |
| POST | `/behavior/stop` | `{job_id?}` | `{ok, stopped: [...], name}` |
| GET | `/behavior/status` | — | `{ok, active, history, registered}` |

`GET /robot/status` returns:
```json
{"ok": true,
 "left": 0.0, "right": 0.0,
 "max_speed": 0.5,
 "timer_pending": false,
 "last_command_age_s": 12.4,
 "odom": {"yaw_deg": -193.2, "heading_deg": 166.8},
 "camera": {"width": 640, "height": 480}}
```

### Contract details that will bite you if ignored

- **`speed` is clamped server-side to `[0, MAX_SPEED]` where `MAX_SPEED` defaults to `0.5`** —
  not 1.0. A UI slider running 0–1 will silently cap at half. Read `max_speed` from
  `/robot/status` and scale the slider to it rather than hardcoding.
- **`direction`** ∈ `forward | backward | left | right | custom`. `custom` uses `left`/`right`
  motor values in `[-MAX_SPEED, MAX_SPEED]`. An unknown direction is a 400.
- **Omitting `duration` starts a *continuous* command.** This is the dangerous path and the
  reason `JetBot.Heartbeat` exists. Supplying `duration` makes the server schedule its own stop,
  and no heartbeat is needed.
- **`HEARTBEAT_TIMEOUT_S` defaults to 2.0 s.** Ping at ~750 ms to leave margin for jitter. The
  watchdog only enforces the heartbeat for *continuous* commands — if a duration timer is
  pending it skips the check.
- **`/behavior/start` returns 409**, not 400, when a behavior is already running. Only one
  behavior may be active at a time. Surface this as a real UI state, not a generic error toast.
- **Snapshot sizes:** ~5 KB at 224×224, ~80 KB at 640×480, ~200 KB at 1280×720. Base64 inflates
  by ~33%. Poll at ~1 fps; do **not** attempt to stream over the LiveView socket.
- **`/gpu/classify` first call costs 30–40 s** (cold model load), then <200 ms. The UI must not
  appear hung — show a distinct "loading model" state on first invocation.

---

## 3. What to build

### 3.1 `JetBot.Client` — HTTP client behind a behaviour

Define a behaviour with the calls above. Two implementations:

- `JetBot.Client.HTTP` — real, via `Req` or `Finch`.
- `JetBot.Client.Mock` — returns realistic fixtures.

**The mock is not optional and is not a test-only concern.** Ship it wired up so
`mix phx.server` runs the entire app with no robot attached. A reviewer cloning this repo has no
Jetson; an app that boots and animates against a mock is dramatically more likely to be looked
at than one that crashes on connection refused. Make the mock drift yaw slowly and return a
placeholder image so the dashboard visibly lives.

Select implementation via config (`config :jetbot_console, :client, JetBot.Client.Mock`).

### 3.2 `JetBot.Robot` — GenServer

One per robot. Holds connection state, last known status, yaw, camera dims. Polls
`GET /robot/status` on an interval (~500 ms). Broadcasts changes over PubSub.

Must survive the Jetson being unreachable — a connection refusal is an expected state
(`:disconnected`), not a crash. Do **not** let a network blip take down the supervision tree;
that is the difference between a toy and something an SRE wrote.

### 3.3 `JetBot.Heartbeat` — GenServer · **the centerpiece**

While a continuous motion command is active, `POST /heartbeat/ping` every ~750 ms.

The important part: **when the LiveView process that requested motion dies, the heartbeat must
stop.** Wire it with `Process.monitor/1` on the LiveView pid. Browser closed, tab crashed,
network dropped, node partitioned — the monitor fires, pings cease, and ~2 s later the Jetson's
watchdog halts the motors. No explicit stop message required, and no reliance on the browser
doing anything correct on its way out.

That is the OTP expression of *the operator went away, so stop the wheels*, and it mirrors the
Python-side safety model rather than duplicating it. Write this up in the README — it is the
single most interesting thing in the repo.

Also stop the heartbeat on: explicit stop, a timed command completing, and client errors
exceeding a small threshold (if the robot is unreachable, pinging is pointless).

### 3.4 `JetBot.Supervisor`

Supervise `Robot` and `Heartbeat`. **Pick a strategy deliberately and justify it in a comment
and the README.** The heartbeat is meaningless without a live robot connection, which argues for
`:rest_for_one` with `Robot` started first — if `Robot` dies, `Heartbeat` should go with it
rather than keep pinging into the void. Interviewers ask exactly this; a defensible answer with
stated reasoning beats a "correct" one you can't explain.

### 3.5 LiveView dashboard

Single page. Phoenix PubSub so multiple browsers show identical state.

- Connection state (connected / disconnected / degraded) — prominent
- Yaw and heading, live
- Active behavior: name, `job_id`, iteration count, elapsed, `last_msg`
- Latest camera snapshot (~1 fps poll)
- Directional controls + speed slider scaled to `max_speed`
- Behavior list with start/stop; 409 rendered as "another behavior is active", not an error
- **E-STOP** — large, always visible, calls `/robot/stop` and kills the heartbeat
- Heartbeat indicator: visible pulse, plus a miss counter

Keep the styling minimal and clean. Default Phoenix + a little CSS is fine; do not spend the
weekend on design.

### 3.6 Telemetry

Attach `:telemetry` handlers for command latency and heartbeat misses. Surface both in the UI.
Cheap to add, and it demonstrates instrumentation instinct.

### 3.7 Tests

ExUnit with `Mox` against the `JetBot.Client` behaviour.

**The one test that must exist:** *the heartbeat stops when the monitored process dies.* Spawn a
process, start a heartbeat monitoring it, assert pings are happening, kill the process, assert
pings stop. That test is the portfolio piece — name it so a reader understands the safety
property it protects.

Also cover: `Robot` survives a client error and reports `:disconnected`; 409 from
`/behavior/start` maps to a distinct return value; speed is clamped to `max_speed` before send.

---

## 4. Constraints

- **Build on the Mac.** Do not attempt Elixir on the Jetson (aarch64, JetPack 4.5) — it is a
  wasted day for zero gain.
- Robot host/port from config/env. **No LAN IPs, no `.local` hostnames committed.** Provide
  `.env.example` / `runtime.exs` reading `JETBOT_HOST` and `JETBOT_PORT`.
- `mix format` clean; Credo clean.
- `.github/workflows/ci.yml` running `mix test` and `mix format --check-formatted`. CI has no
  robot — the mock client makes this trivially green.
- Conventional Commits, authored as `Derek Clair <derek@derekclair.com>` (already set in this
  repo's local config).
- MIT LICENSE (already present).

## 5. Scope discipline

**Do not build:** authentication, a database, user accounts, multi-robot fleet management,
historical time-series storage, a custom design system.

Depth in the supervision and heartbeat story beats breadth every single time. If you find
yourself adding Ecto, stop.

---

## 6. Acceptance criteria

- [ ] `mix phx.server` boots and the dashboard animates **with no hardware**, via the mock
- [ ] Pointed at a real Jetson, manual driving and behavior start/stop work
- [ ] Heartbeat verifiably stops when the LiveView dies — proven by the test, and observable by
      closing the browser mid-motion and watching the robot halt ~2 s later
- [ ] Supervision strategy chosen with reasoning written down
- [ ] `mix test` green; `mix format --check-formatted` clean; CI badge green
- [ ] README: what it is, why OTP suits robot supervision, architecture diagram, screenshot/GIF,
      how to run against the mock, and the `## Part of` cross-links
- [ ] No secrets, no LAN IPs, no hostnames in the tree or history

## 7. Handoff

When done, push to `origin main`. I will pull, review against this spec, and fold revisions in.
Flag anything you deliberately deviated from — deviations with reasons are fine, silent ones
cost review time.
