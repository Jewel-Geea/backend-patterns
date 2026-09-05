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
| [Idempotency Keys](patterns/idempotency_keys.md) | Safely retrying a POST/payment request without double-processing it |
| [Token Bucket Rate Limiter](patterns/token_bucket_rate_limiter.md) | Limiting request rate per client while allowing short bursts |
| [Cursor-Based Pagination](patterns/cursor_pagination.md) | Paginating large/changing datasets without the "shifting page" bug of offset pagination |
| [Circuit Breaker](patterns/circuit_breaker.md) | Stopping a failing downstream dependency from taking the whole system down with it |
| [Sharding](patterns/sharding.md) | Splitting a workload too big for one worker into parallel, independent slices |

## License
MIT — see [LICENSE](LICENSE)
