---
title: "Oban.py - deep dive"
date: 2026-01-27T00:00:00Z
description: Oban, the job processing framework from Elixir, has finally come to Python. I spent some time exploring it, and here is how it works.
tags:
  - general
  - oban
  - python
  - elixir
categories:
  - General
draft: false
---

## Setting the Stage

I've used [Oban in Elixir](https://github.com/oban-bg/oban) for almost as long as I've been writing software in Elixir, and it has always been an essential tool for processing jobs. I always knew Oban was cool, but I never dug deeper. This article is a collection of my notes and observations on how the Python implementation of Oban works and what I've learned while exploring its codebase. I'll also try to compare it with the Elixir version and talk about concurrency in general.

## Surface Level

[Oban](https://oban.pro/docs/py/index.html) allows you to **insert and process jobs using only your database**. You can insert the job to send a confirmation email in the same database transaction where you create the user. If one thing fails, everything is rolled back.

Additionally, like most job processing frameworks, Oban has [queues](https://oban.pro/docs/py/0.5.0/defining_queues.html) with local and global queue limits. But unlike others, it **stores your completed jobs** and can [even keep their results if needed](https://oban.pro/docs/py/0.5.0/writing_jobs.html#recording-results). It has built-in [cron scheduling](https://oban.pro/docs/py/0.5.0/periodic_jobs.html#periodic-jobs) and [many more features](https://oban.pro/docs/py/0.5.0/managing_queues.html) to control how your jobs are processed.

Oban comes in two versions - [Open Source Oban-py](https://oban.pro/docs/py/index.html) and [commercial Oban-py-pro](https://oban.pro/docs/py_pro/adoption.html).

OSS Oban has a few limitations, which are automatically lifted in the Pro version:

- **Single-threaded asyncio execution** - concurrent but not truly parallel, so CPU-bound jobs block the event loop.
- **No bulk inserts** - each job is inserted individually.
- **No bulk acknowledgements** - each job completion is persisted individually.
- **Inaccurate rescues** - jobs that are long-running might get rescued even if the producer is still alive. Pro version uses smarter heartbeats to track producer liveness.

In addition, Oban-py-pro comes with a few extra features you'd configure separately, like [workflows](https://oban.pro/docs/py_pro/0.5.0/workflow.html), [relay](https://oban.pro/docs/py_pro/0.5.0/relay.html), [unique jobs](https://oban.pro/docs/py_pro/0.5.0/unique_jobs.html), and [smart concurrency](https://oban.pro/docs/py_pro/0.5.0/smart_concurrency.html).

OSS Oban-py is a great start for your hobby project, or if you'd want to evaluate Oban philosophy itself, but for any bigger scale - I'd go with [Oban Pro](https://oban.pro/docs/py_pro/index.html). The pricing seems very compelling, considering the amount of work put into making the above features work.

I obviously can't walk you through the Pro version features, but let's start with the basics. How Oban Py works under the hood, from the job insertion until the job execution. Stay tuned.

## Going Deeper - Job Processing Path

Let's get straight to it. You insert your job:

```python
from oban import job

@job(queue="default")
async def send_email(to: str, subject: str, body: str):
    # Simple and clean, but no access to job context
    await smtp.send(to, subject, body)

await send_email.enqueue("user@example.com", "Hello", "World")
```

After the insertion, the job lands in the `oban_jobs` database table with `state = 'available'`. Oban fires off a PostgreSQL `NOTIFY` on the `insert` channel:

```python
# oban.py:414-419
# Single inserts go through bulk insert path
result = await self._query.insert_jobs(jobs)
queues = {job.queue for job in result if job.state == "available"}
await self._notifier.notify("insert", [{"queue": queue} for queue in queues])
```

Every Oban node listening on that channel receives the notification. **The Stager** on each node gets woken up, but each **Stager** only cares about queues it's actually running. Be aware that each node decides which queues it runs, so if the current node runs this queue, the producer is notified:

```python
# _stager.py:95-99
async def _on_notification(self, channel: str, payload: dict) -> None:
    queue = payload["queue"]

    if queue in self._producers:
        self._producers[queue].notify()
```

That `notify()` call sets an `asyncio.Event`, breaking **the Producer** out of its wait loop, so it can dispatch the jobs to the workers:

```python
# _producer.py:244-262
async def _loop(self) -> None:
    while True:
        try:
            # <--- This is where the event is received --->
            await asyncio.wait_for(self._notified.wait(), timeout=1.0)
        except asyncio.TimeoutError:
            continue
        except asyncio.CancelledError:
            break

        # <--- Reset the event so it can be triggered for the next batch --->
        self._notified.clear()

        try:
            # <--- A little debounce to potentially process multiple jobs at once --->
            await self._debounce()
            # <--- Dispatch (Produce) the jobs from the database to the workers --->
            await self._produce()
        except asyncio.CancelledError:
            break
        except Exception:
            logger.exception("Error in producer for queue %s", self._queue)
```

Before fetching the jobs, **the producer** persists all pre-existing job completions (acks) to the database to make sure queue limits are respected. Next, it fetches new jobs, transitioning their state to executing at the same time. A slightly more complex version of this SQL is used:

```sql
-- fetch_jobs.sql (simplified)
WITH locked_jobs AS (
  SELECT priority, scheduled_at, id
  FROM
  oban_jobs
  WHERE state = 'available' AND queue = %(queue)s
  ORDER BY priority ASC, scheduled_at ASC, id ASC
  LIMIT %(demand)s
  FOR UPDATE SKIP LOCKED
)
UPDATE oban_jobs oj
SET
  attempt = oj.attempt + 1,
  attempted_at = timezone('UTC', now()),
  attempted_by = %(attempted_by)s,
  state = 'executing'
FROM locked_jobs
WHERE oj.id = locked_jobs.id
```

And this is **the first really cool part**.

**Segue to _FOR UPDATE SKIP LOCKED_.**

- `FOR UPDATE` - Locks the selected rows so no other transaction can modify them until this transaction completes. This prevents two producers from grabbing the same job.

- `SKIP LOCKED` - If a row is already locked by another transaction, skip it instead of waiting. This is crucial for concurrency.

Why this matters for job queues:
Imagine two producer instances (A and B) trying to fetch jobs simultaneously:

| Without SKIP LOCKED              | With SKIP LOCKED                 |
| -------------------------------- | -------------------------------- |
| A locks job #1                   | A locks job #1                   |
| B **waits** for job #1 to unlock | B **skips** job #1, takes job #2 |
| Slow, sequential processing      | Fast, parallel processing        |

Back in Python, we know that the jobs we just fetched should be processed immediately. When we fetched the job, we already transitioned its state and respected the queue demand.

Each job gets dispatched as an **async task**:

```python
jobs = await self._get_jobs()
for job in jobs:
    task = self._dispatcher.dispatch(self, job)
    task.add_done_callback(
        lambda _, job_id=job.id: self._on_job_complete(job_id)
    )

    self._running_jobs[job.id] = (job, task)
```

`add_done_callback` ensures that independent of success or failure, we can attach a callback to handle job completion.

**The dispatcher** controls how exactly the job is run. For the non-pro Oban version, it just uses `asyncio.create_task` to run the job in the event loop:

```python
# _producer.py:69-71
class LocalDispatcher:
    def dispatch(self, producer: Producer, job: Job) -> asyncio.Task:
        return asyncio.create_task(producer._execute(job))
```

[For pro version](https://oban.pro/releases/py_pro), local asyncio dispatcher is automatically replaced with a pool of processes, so you don't need to do anything to have true parallelism across multiple cores.

After the job is dispatched, **the Executor** takes over. It resolves your worker class from the string name, runs it, and pattern-matches the result:

```python
# _executor.py:73-83
async def _process(self) -> None:
  self.worker = resolve_worker(self.job.worker)()
  self.result = await self.worker.process(self.job)
```

```python
# _executor.py:95-133
match result:
    case Exception() as error:
        # Retry or discard based on attempt count
    case Cancel(reason=reason):
        # Mark cancelled
    case Snooze(seconds=seconds):
        # Reschedule with decremented attempt
    case _:
        # Completed successfully
```

And that's **the second cool part**! You see how similar it is to [Elixir's pattern matching](https://hexdocs.pm/elixir/pattern-matching.html)? I love how it's implemented!

When execution finishes, the result gets queued for acknowledgement:

```python
# _producer.py:315
self._pending_acks.append(executor.action)
```

The completion callback notifies **the Producer** to wake up again-fetch more jobs, and batch-ack the finished ones in a single query.

That's the hot path: `Insert → Notify → Fetch (with locking) → Execute → Ack.` Five hops from your code to completion. What about the background processes? What about errors and retries? What about periodic jobs, cron, and all these other pieces? Stay tuned.

## The Undercurrents - Background Processes

Oban runs several background loops that keep the system healthy.

### Leader Election

In a cluster, you don't want every node pruning jobs or rescuing orphans. Oban elects a single leader:

```python
# _leader.py:107-113
async def _election(self) -> None:
    self._is_leader = await self._query.attempt_leadership(
        self._name, self._node, int(self._interval), self._is_leader
    )
```

```sql
-- Cleanup expired leaders first
DELETE FROM
  oban_leaders
WHERE
  expires_at < timezone('UTC', now())
```

```sql
-- If current node is a leader, it re-elects itself
INSERT INTO oban_leaders (name, node, elected_at, expires_at)
VALUES (
  %(name)s,
  %(node)s,
  timezone('UTC', now()),
  timezone('UTC', now()) + interval '%(ttl)s seconds'
)
ON CONFLICT (name) DO UPDATE SET
  -- Only update if we're the same node (i.e. current leader re-electing itself).
  -- Other nodes can't overwrite an active leader's lease.
  expires_at = EXCLUDED.expires_at
WHERE
  oban_leaders.node = EXCLUDED.node
RETURNING node
```

```sql
-- Try to insert as a new leader if no leader exists
INSERT INTO oban_leaders (
  name, node, elected_at, expires_at
) VALUES (
  %(name)s,
  %(node)s,
  timezone('UTC', now()),
  timezone('UTC', now()) + interval '%(ttl)s seconds'
)
ON CONFLICT (name) DO NOTHING
RETURNING node
```

The leader refreshes twice as often to hold onto the role:

```python
# _leader.py:101-105
# Sleep for half interval if leader (to boost their refresh interval and allow them to
# retain leadership), full interval otherwise
sleep_duration = self._interval / 2 if self._is_leader else self._interval
```

When a node shuts down cleanly, it resigns and notifies the cluster:

```python
# _leader.py:83-87
if self._is_leader:
    payload = {"action": "resign", "node": self._node, "name": self._name}

    await self._notifier.notify("leader", payload)
    await self._query.resign_leader(self._name, self._node)
```

And that's **the third cool part**! Leader election is delegated entirely to PostgreSQL. Oban uses `INSERT ... ON CONFLICT` with a TTL-based lease - no Raft, no consensus protocol, no external coordination service. If the leader dies, its lease expires and the next node to run the election query takes over. Simple, effective, and zero additional infrastructure.

### Lifeline: Rescuing Orphaned Jobs

Workers crash. Containers get killed. When that happens, jobs can get stuck executing indefinitely. **The Lifeline** process (leader-only) rescues them:

```python
# _lifeline.py:73-77
async def _rescue(self) -> None:
    if not self._leader.is_leader:
        return

    await use_ext("lifeline.rescue", _rescue, self._query, self._rescue_after)
```

Oban-py rescue mechanics are purely time-based - any job in `executing` state longer than `rescue_after` (default: 5 minutes) gets moved back. Unlike the Oban Pro version, it doesn't check whether the producer that owns the job is still alive. This means legitimately long-running jobs could be rescued and executed a second time.

The takeaway is that you should set `rescue_after` higher than your longest expected job duration, and design workers to be idempotent.

The SQL itself is straightforward - jobs stuck executing get moved back to available or discarded if they've exhausted retries:

```sql
-- rescue_jobs.sql (simplified)
UPDATE oban_jobs
SET
  state = CASE
    WHEN attempt >= max_attempts THEN 'discarded'
    ELSE 'available'
  END,
  meta = CASE
    WHEN attempt >= max_attempts THEN meta
    ELSE meta || jsonb_build_object('rescued', coalesce((meta->>'rescued')::int, 0) + 1)
  END
WHERE
  state = 'executing'
  AND attempted_at < timezone('UTC', now()) - make_interval(secs => %(rescue_after)s)
```

The rescued counter in meta lets you track how often jobs needed saving.

### Pruner: Cleaning Up Old Jobs

Without pruning, your oban_jobs table grows forever. **The Pruner** (also leader-only) deletes terminal jobs older than max_age (default: 1 day):

```sql
-- prune_jobs.sql
WITH jobs_to_delete AS (
SELECT id FROM oban_jobs
WHERE
(state = 'completed' AND completed_at <= timezone('UTC', now()) - make_interval(secs => %(max_age)s)) OR
(state = 'cancelled' AND cancelled_at <= timezone('UTC', now()) - make_interval(secs => %(max_age)s)) OR
(state = 'discarded' AND discarded_at <= timezone('UTC', now()) - make_interval(secs => %(max_age)s))
ORDER BY id ASC
LIMIT %(limit)s
)
DELETE FROM oban_jobs WHERE id IN (SELECT id FROM jobs_to_delete)
```

The LIMIT prevents long-running deletes from blocking other operations.

### Retry & Backoff Mechanics

When a job raises an exception, **the Executor** decides its fate:

```python
# _executor.py:96-109
match result:
    case Exception() as error:
        if self.job.attempt >= self.job.max_attempts:
            self.action = AckAction(
                job=self.job,
                state="discarded",
                error=self._format_error(error),
            )
        else:
            self.action = AckAction(
                job=self.job,
                state="retryable",
                error=self._format_error(error),
                schedule_in=self._retry_backoff(),
            )
```

Simple rule: under `max_attempts` - retry, otherwise - discard.

The default backoff uses jittery-clamped exponential growth with randomness to prevent thundering herds:

```python
# _backoff.py:66-87
def jittery_clamped(attempt: int, max_attempts: int, *, clamped_max: int = 20) -> int:
    if max_attempts <= clamped_max:
        clamped_attempt = attempt
    else:
        clamped_attempt = round(attempt / max_attempts * clamped_max)

    time = exponential(clamped_attempt, mult=1, max_pow=100, min_pad=15)

    return jitter(time, mode="inc")
```

And that's **the fourth cool thing**! Backoff includes jitter to prevent [thundering herds](https://en.wikipedia.org/wiki/Thundering_herd_problem) - without it, all failed jobs from the same batch would retry at the exact same moment, spiking load all over again.

The formula: 15 + 2^attempt seconds, with up to 10% added jitter. Attempt 1 waits ~17s. Attempt 5 waits ~47s. Attempt 10 waits ~1039s (~17 minutes).

The clamping handles jobs with high `max_attempts` - if you set `max_attempts=100`, it scales the attempt number down proportionally so you don't wait years between retries.

Workers can override this with custom backoff:

```python
@worker(queue="default")
class MyWorker:
    async def process(self, job: Job):
        ...

    def backoff(self, job: Job) -> int:
        # Linear backoff: 60s, 120s, 180s...
        return job.attempt * 60
```

## Surfacing - Takeaways

- **PostgreSQL does the heavy lifting.** `FOR UPDATE SKIP LOCKED` for concurrent job fetching, `LISTEN/NOTIFY` for real-time signaling, `ON CONFLICT` for leader election - the database isn't just storage, it's the coordination layer. There's no Redis, no ZooKeeper, no external broker. One less thing to operate.
- **Oban-py is concurrent, but not parallel**. Async IO allows multiple jobs to be in-flight, but the event loop is single-threaded. For I/O-bound workloads, this is fine. For CPU-bound tasks, consider using the Pro version with a process pool.
- **Leader election is simple and effective.** No consensus protocol, no Raft - just an `INSERT ... ON CONFLICT` with a TTL. The leader refreshes at 2x the normal rate to hold the lease. If it dies, the lease expires and another node takes over. Good enough for pruning and rescuing.
- **The codebase is a pleasure to read.** Clear naming, consistent patterns, and well-separated concerns - exploring it felt more like reading a well-written book than understanding a library.
- **OSS gets you far, Pro fills the gaps.** Bulk operations, smarter rescues, and true parallelism are all Pro-only - but for what you get, Pro license feels like a great deal.

Overall, [Oban.py](https://oban.pro/docs/py/index.html) is a clean and well-structured port. If you're coming from Elixir and miss Oban, or if you're in Python and want a database-backed job queue that doesn't require external infrastructure beyond PostgreSQL - it's worth looking at.
