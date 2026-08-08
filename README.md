# jetbot-console

Phoenix LiveView operator console for a JetBot running on an NVIDIA Jetson Nano.

> **Status: spec'd, not yet built.** See [`SPEC.md`](SPEC.md) for the full build
> specification, including the HTTP API contract it consumes.

## The interesting part

The robot's safety model is a heartbeat: a continuous motion command runs until a stop
arrives or the Jetson's watchdog times out (~2 s). That gives this console a real safety
obligation — if the operator's browser closes mid-motion, the pings must stop so the robot
halts.

That is expressed with `Process.monitor/1` on the LiveView pid: browser closed, tab crashed,
network dropped — the monitor fires, pings cease, and the robot stops on its own. No explicit
teardown message, and no reliance on the browser behaving correctly on its way out.

## Part of

- [jetbot-agent](https://github.com/derekclair/jetbot-agent) — the Python control plane on the robot
- **jetbot-console** — this repo, the operator UI
- [jetbot-iac](https://github.com/derekclair/jetbot-iac) — Terraform for the cloud gateway

## License

MIT
