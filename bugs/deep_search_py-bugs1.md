You are absolutely correct. If `len(pending) == cfg.fetch_concurrency`, the inner `while` loop is skipped. However, **it does not cause a deadlock or skip waiting indefinitely**; the design intentionally relies on the outer loop and `asyncio.wait` to handle this.

Here is the step-by-step reasoning of how the code behaves in your scenario (`fetch_concurrency=3`, `len(pending)=3`, `len(docs)=14`, `max_docs=20`):

### 1. The Inner Loop is Skipped (As Intended)

When `len(pending) == cfg.fetch_concurrency`, the condition `len(pending) < cfg.fetch_concurrency` evaluates to `False`. The inner loop does nothing and is skipped. This is correct because there are no available "concurrency slots" to schedule new tasks.

### 2. The Code Waits at `asyncio.wait`

Execution immediately proceeds to the next lines outside the inner loop:

```python
done, pending = await asyncio.wait(
    pending, return_when=asyncio.FIRST_COMPLETED
)
```

This is **where the code actually waits**. It pauses the main coroutine until at least one of the 3 pending fetch tasks completes. 

### 3. The Slot Frees Up

Once a task finishes:

* It moves from the `pending` set to the `done` set.
* `len(pending)` decreases (e.g., from 3 to 2).
* The outer `while len(docs) < cfg.max_docs:` loop iterates again.
* Now, `len(pending) < cfg.fetch_concurrency` (2 < 3) is `True`, so the inner loop runs, pops a new item from the frontier, and fills the slot back to 3.

---

### ⚠️ The Real Edge Case: The Empty Frontier

While the code handles a full concurrency slot correctly, your intuition reveals a **real flaw** when the `frontier` is temporarily empty but tasks are still running.

Consider this scenario:

1. `len(pending) = 3` (Inner loop skipped).
2. `asyncio.wait` returns one `done` task.
3. Outer loop iterates. `len(pending) = 2`.
4. Inner loop runs: `item = await frontier.pop()`.
5. If the frontier is empty, `pop()` returns `None`, and the inner loop `break`s.
6. `len(pending)` is now stuck at 2, and the frontier is empty.

**What happens next?** 
The code proceeds to `asyncio.wait` again. When the remaining 2 tasks finish, they move to `done`. On the next outer loop iteration, `len(pending) = 0`. The check `if not pending: break` executes, and the crawl terminates. 

**The problem:** If the currently running tasks discover new URLs (e.g., via the Critic's gap analysis or crawling deeper links), they will push new items to the `frontier`. But because the inner loop broke when it saw `None`, and `pending` eventually hit 0, the outer loop exits prematurely. It never waits for the running tasks to populate the frontier.

### Suggested Fix

To make the code robust against a temporarily empty frontier while tasks are still running, you should modify the `if not pending:` check to also consider if there are tasks that might still populate the frontier:

```python
            if not pending:
                # Only break if there are no running tasks to discover new URLs
                break

            done, pending = await asyncio.wait(
                pending, return_when=asyncio.FIRST_COMPLETED
            )
```

*(Note: In the current codebase, `CrawlFrontier` doesn't seem to push new URLs from running tasks directly, but if it did, or if you plan to add concurrent gap-filling, this fix is crucial).*
