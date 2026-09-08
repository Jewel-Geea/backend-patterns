# Sharding

**TL;DR:** one librarian re-shelving the entire library alone takes all day. Split the
shelves into four sections and give each librarian one section — now it takes roughly a
quarter of the time, as long as no section is a lot bigger than the others. Sharding is
that idea applied to any workload too big for one worker: split it into N independent
slices and run them in parallel.

**Where I ran into this:** an Appium mobile test suite running serially on one emulator
started failing intermittently as the suite grew — not real bugs, just the single runner
buckling under load and producing flaky, inconsistent results. Splitting the same suite
into shards running on separate runners in parallel didn't just make it faster; each
shard carried a lighter, more predictable load, and the flakiness disappeared. A good
reminder that sharding isn't only a throughput fix — an overloaded single worker can
degrade in ways that look like bugs, and spreading the load out fixes both problems at once.

**Problem:** a single worker — a database, a CI runner, a batch job — is handling
everything serially, and the work keeps growing. Eventually one worker isn't enough:
either it can't hold all the data, or it takes too long to get through all the work in
the time you have.

**Fix:** partition the work into N shards using a rule that's cheap to compute and
deterministic (the same input always maps to the same shard), and run each shard
independently. Two common flavors:

- **Data sharding** (databases): each row is routed to a shard by a key, usually
  `hash(key) % shard_count`, so reads/writes for that key always go to the same shard.
- **Workload sharding** (batch jobs, CI test suites): a fixed list of work items is split
  into N contiguous chunks, and each chunk runs on its own worker.

```mermaid
graph LR
    W["100 items of work"] --> S1["Shard 1: items 1-25"]
    W --> S2["Shard 2: items 26-50"]
    W --> S3["Shard 3: items 51-75"]
    W --> S4["Shard 4: items 76-100"]
    S1 --> R["Results merged"]
    S2 --> R
    S3 --> R
    S4 --> R
```

Data sharding, keyed by hash:

```python
import hashlib

def shard_for_key(key: str, shard_count: int) -> int:
    digest = hashlib.sha256(key.encode()).hexdigest()
    return int(digest, 16) % shard_count

# order "user_42" always resolves to the same shard, so all of that user's
# rows land on one node and can be queried without a cross-shard fan-out.
shard_for_key("user_42", shard_count=4)
```

Workload sharding, splitting a fixed list into contiguous blocks (the same arithmetic
test runners and batch frameworks use to split N items across T workers):

```python
def split_into_shards(items: list, shard_count: int) -> list[list]:
    n = len(items)
    per_shard = max(round(n / shard_count), 1)
    shards = [items[i : i + per_shard] for i in range(0, n, per_shard)]
    # rounding can produce more chunks than shard_count when n is small
    # relative to shard_count -- merge any overflow into the last real shard
    while len(shards) > shard_count:
        shards[-2].extend(shards[-1])
        shards.pop()
    return shards
```

## Hazards worth knowing before you ship this
- **Rounding can create an empty shard.** `round(n / t)` rounding up means a shard can end
  up with zero items even though `shard_count <= item_count` looks safe on paper. Guard
  for "did every shard get at least one item" explicitly rather than trusting the input
  sizes to make it impossible.
- **The split must be deterministic and stable.** If the underlying list isn't sorted the
  same way every time, two independent workers slicing "the same" list can disagree on
  which items belong to which shard.
- **Shared external state needs to be pinned once, upstream.** If every shard
  independently resolves something that can change between shard starts (the current
  build to test against, a "latest" config value), different shards can end up testing
  against different versions and the overall result becomes meaningless. Resolve it once
  before the shards start, then hand every shard the same pinned value.
- **Uneven shards don't fully parallelize.** If one shard's slice takes much longer than
  the others (data isn't uniformly distributed, or work items vary a lot in cost), your
  wall-clock time is bounded by the slowest shard, not the average — 4x the workers rarely
  means a clean 4x speedup in practice.

## When to use it
- A single node is close to (or past) its capacity for data volume, throughput, or wall-clock
  time, and the work is naturally divisible with little or no cross-shard coordination needed.

## When not to
- If most operations need to touch data across many shards anyway (heavy joins/aggregations
  spanning keys), sharding adds coordination cost without removing the bottleneck — look at
  read replicas, caching, or a bigger single node first.

## Case study: duration-based dynamic sharding
A real run-in with the "uneven shards" hazard above, on a UI/E2E regression suite's CI.
Specs were split across a **fixed** shard count — a hand-set constant, re-tuned manually
every time someone noticed the suite had grown (count the new spec files, bump the
constant, re-derive the timeout by hand).

That breaks because splitting by file count assumes every spec costs about the same to
run. In a regression suite, spec duration is wildly uneven — a smoke check that just
asserts a screen renders takes a few seconds; a full checkout-and-payment flow that seeds
data through the backend and waits out async state can take minutes. Bin specs by count
instead of duration and shards end up with wildly different actual runtimes. Wall-clock
for the whole job is bounded by the slowest shard, not the average, so a handful of
expensive specs landing on the same shard by bad luck can double the job's runtime while
every other shard finishes early and sits idle. There's no fixed number that's
simultaneously fast and cheap, because the workload isn't uniform and a constant can't see
that — over-provision shards "just in case" and you burn CI cost on idle time;
under-provision and the slowest shard becomes the bottleneck, so the timeout has to be
padded to survive it.

The fix replaced the constant with a plan computed at runtime:
1. **Measure real cost, not file count.** A reporter records each spec file's wall-clock
   duration on every run, persisted as an artifact — ground truth for what each spec
   actually costs, instead of assuming they're interchangeable.
2. **Plan shards from that history.** A planning step reads the duration history and
   computes the shard count at runtime (clamped between a configured floor and ceiling),
   then bin-packs specs across shards by duration — so no shard ends up disproportionately
   loaded just because it drew the slowest specs.
3. **Derive the timeout from the plan actually chosen**, instead of a number a human
   re-derives by hand every time the suite's shape changes.
4. **Skip work a change couldn't affect.** On pull requests, map changed files to the
   specs whose import graph actually reaches them and run only those, falling back to the
   full suite for anything that could affect every spec (CI config, shared tooling,
   lockfile changes) — so "run everything on every PR" stops being the default cost for a
   change that only touched one screen's worth of tests.
5. **Degrade gracefully on a cold start.** With no duration history yet, fall back to an
   even split rather than failing on day one or on a fresh branch with no history.

Net effect: the pipeline moved from "a human periodically retunes a magic number and hopes
it's still right" to "the system measures itself and rebalances every run" — because the
workload is non-uniform and keeps changing shape, no single static number is ever optimal
for long, so the fix is to stop trying to pick one and compute it fresh each run instead.
