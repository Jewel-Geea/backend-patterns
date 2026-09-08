# Backend Patterns

![Python](https://img.shields.io/badge/python-3670A0?style=flat-square&logo=python&logoColor=ffdd54)
![License](https://img.shields.io/badge/license-MIT-blue?style=flat-square)

Short write-ups of backend engineering patterns, each with a plain-English analogy, a
diagram, and a working Python implementation — plus a note on when to actually reach for
it (and when not to). Added incrementally as I study or apply them — not a tutorial
series, a running notebook.

## Patterns

| Pattern | What it solves |
|---|---|
| [Sharding](patterns/sharding.md) | Splitting a workload too big for one worker into parallel, independent slices |
| [Vertical vs. Horizontal Scaling](patterns/scaling.md) | Adding capacity by upgrading one server vs. adding more of them |
| [Flaky Test Root Causes](patterns/flaky-test-root-causes.md) | Diagnosing intermittent E2E/UI test failures instead of retrying past them |

## Log
A short diary of each session — the point isn't the content, it's keeping the streak of
showing up going.

- **2026-09-06** — Added Vertical vs. Horizontal Scaling, from Frank Kane's *Mastering the
  System Design Interview* (Udemy). First log entry — starting the habit of noting a line
  here every time a pattern gets added.
- **2026-09-07** — Worked on Appium authentication code; the device got too heavy and got
  stuck, so testing it stalled. Picking it back up tomorrow.
- **2026-09-08** — Added a real-world case study to Sharding: replacing a fixed shard
  count with duration-based dynamic shard planning in a CI regression suite. Also added a
  new pattern, Flaky Test Root Causes, distilled from a day of review fixes across six
  PRs — race conditions, unstable locators, false-confidence assertions, silent retries,
  and shared-state pollution.

## License
MIT — see [LICENSE](LICENSE)
