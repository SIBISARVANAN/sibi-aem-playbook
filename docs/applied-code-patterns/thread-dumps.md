# Thread Dumps — Reading & Analysis

## What it is

A snapshot of every thread in the JVM at one instant — its state and full stack trace. Used to diagnose a hung or slow AEM instance.

## How to take one

```bash
kill -3 <pid>          # writes to stdout/log, JVM keeps running
# or via Felix console: /system/console/threads
```

Take **3 dumps, 10 seconds apart** — a single dump can't distinguish "genuinely stuck" from "just happened to be here."

## Thread states

| State | Meaning |
|---|---|
| `RUNNABLE` | Actively executing or ready to — not automatically a problem |
| `BLOCKED` | Waiting to acquire a `synchronized` lock held by another thread |
| `WAITING` / `TIMED_WAITING` | Parked, waiting on a condition, I/O, or a timed sleep |

## Four patterns to look for

1. **Same stack trace across all 3 dumps, in `BLOCKED` or `WAITING` on the same lock** → genuinely stuck, not just slow.
2. **Many threads `BLOCKED` on the same lock object** → contention bottleneck — find the thread holding the lock, that's your root cause.
3. **Threads `WAITING` on a `ResourceResolver`/JCR session operation for a long time** → likely an unclosed session leak or a slow repository query.
4. **Threads stuck in third-party HTTP client code** → an external API call with no timeout configured — the classic AEM culprit.

The first two or three frames of the stack trace (top of the dump) tell you what the thread is actually doing right now — start reading there, not from the bottom.

## Interview Analysis Checklist

```
1. How many http-nio-*-exec-* threads? What state are most in?
2. If BLOCKED — what lock hex address? Search for "- locked <that hex>" to find the holder.
3. What is the lock holder doing? Read its stack bottom-to-top.
4. Any "Found one Java-level deadlock" at the bottom of the dump?
5. Same RUNNABLE stack in all 3 dumps = runaway loop. Cross-reference with `top -H -p <PID>`.
```

## Common Interview Q&A

**Q: AEM is unresponsive. The thread dump shows 47 request threads `BLOCKED` on the same hex address, held by a scheduler thread inside `fetchProductData()`. What happened and how do you fix it?**
The scheduler is holding a JVM monitor lock while blocked on a slow external HTTP call (or an exhausted connection pool). All request threads queue behind it. Fix: remove `synchronized` from the method, or add a `connectionRequestTimeout` to the HTTP client so it fails fast instead of blocking indefinitely — releasing the lock.

**Q: How do you find which thread is causing 100% CPU?**
`top -H -p <PID>` → note the OS thread ID using the most CPU → convert it to hex → search for `nid=<hex>` in the thread dump → read that thread's call stack.

**Q: What's the difference between `BLOCKED` and `WAITING`?**
`BLOCKED` means the thread wants a monitor lock currently held by another thread — classic lock contention. `WAITING` means the thread voluntarily released the CPU and is waiting for a signal or notification (e.g. `Object.wait()`, `LockSupport.park()`) — often normal for idle threads or threads waiting on async I/O.
