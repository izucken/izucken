I see a piece of code for shutdown handlers. It hash graceful and crash (unhadledException/unhaldedRejection) shutdown cleanup handler registers.

I wonder does anybody else try to perfromt resource management like this in nodejs application? I mean concrete known examples *who exactly* (not libs) does resource management on unhadledException.

These results are mostly generic “how-to” blog posts and Node.js docs repeating the same advice — not what you’re after. Let me search specifically for named engineering orgs or well-known OSS projects.

Short answer: concrete, named examples are rare — but there is at least one solid one, and I can tell you why the rest of the trail goes cold.

Uber is the clearest documented case. They published (and used internally) a module called uber/uncaught-exception that does exactly the pattern you’re describing:

registers a process.on('uncaughtException') handler
runs a gracefulShutdown callback before exit — draining connections, deregistering from HAProxy, telling a ringpop instance to leave the ring, syncing state to Redis, or telling the cluster master to spin up a replacement worker
has a preAbort synchronous hook and a hard shutdownTimeout (default 30s) in case the graceful path hangs, after which it aborts anyway
reports the event to statsd/logger before dying

That’s a real “who exactly” — it’s Uber’s own infra pattern, open-sourced, not a third-party lib pretending to be one.

Beyond that one, here’s the honest state of things: almost everything else you’ll find (RisingStack, DZone, dev.to, Medium posts) is generic advice repeating the Node.js docs verbatim — “do synchronous cleanup, then exit, use PM2/an external monitor to restart.” None of them name a company doing this in production with specifics. Netflix, Pinterest, and others have published extensively on chaos engineering and graceful degradation, but their public talks/blogs focus on the distributed systems layer (circuit breakers, deployment validation, HAProxy/service-mesh draining) rather than the specific uncaughtException/unhandledRejection process-level hook you’re asking about — that detail usually doesn’t make it into conference talks even when it’s presumably there under the hood.

So the pattern itself (log, synchronous cleanup, bounded timeout, then hard exit, external supervisor restarts) is essentially Node core team’s own prescribed approach (it’s literally in the process.md docs, unchanged for ~10 years), and Uber is the one org I can point to with a public, attributable implementation of it. If you want, I can dig into whether any other named engineering orgs (e.g. via conference talk transcripts or GitHub org repos rather than blog posts) have similar public artifacts — that’d take more targeted searching than generic web queries.


---


# Code review: `ShutdownCleanup` (cleanup.ts)

## Attached file: `cleanup.ts`

```ts
import { logger } from '#src/common/logger.ts';

const SEVERITY = { NONE: 0, GRACEFUL: 1, CRASH: 2 };

type SeverityLevel = typeof SEVERITY[keyof typeof SEVERITY];

type CleanupCallback = () => void | Promise<void>;

export const CLEANUP_IMPORTANCE = {
  /**
   * We really prefer this handler to run ASAP and complete.
   */
  CRITICAL: 0,
  /**
   * Handler is hygienic; a nice-to-have, optional.
   */
  HYGIENE: 1,
};

type CleanupLevel = typeof CLEANUP_IMPORTANCE[keyof typeof CLEANUP_IMPORTANCE];

interface CleanupHandle {
  name: string;
  level: CleanupLevel;
  order: number;
  callback: CleanupCallback;
  timeout: number;
}

/**
 * State machine for graceful shutdown handling.
 */
export class ShutdownCleanup {
  // todo: a combination of AbortSignal, Symbol.asyncDispose, and injection could in theory allow for:
  //  - a smarter choice between fast or hygienic logic per-cleanup
  //  - parallelizing sweeps for faster shutdowns
  #mode = SEVERITY.NONE;
  #cleanups: CleanupHandle[] = [];
  #registered = false;
  #sweeping = false;
  #sweepDeadline: number | undefined;
  #order = 0;
  readonly #gracefulCap: number;
  readonly #emergencyCap: number;

  /**
   * @param gracefulCap graceful shutdown will never take longer than this
   * @param emergencyCap emergency shutdown will never take longer than this
   */
  constructor(gracefulCap: number = 5000, emergencyCap: number = 1000) {
    this.#gracefulCap = gracefulCap;
    this.#emergencyCap = emergencyCap;
  }

  /**
   * Installs default signal and error handlers.
   */
  registerListeners() {
    if (this.#registered) {
      return;
    }
    this.#registered = true;
    process.on('SIGINT', () => this.gracefulShutdown('SIGINT'));
    process.on('SIGTERM', () => this.gracefulShutdown('SIGTERM'));
    process.on('unhandledRejection', (reason) => this.crashShutdown('unhandledRejection', reason));
    process.on('uncaughtException', (error) => this.crashShutdown('uncaughtException', error));
  }

  /**
   * Adds a handler that will run on sweep before exit during shutdown.
   * Prefer adding handlers in the same order as resources are created.
   *
   * Any handlers are always run only on best-effort basis:
   * you cannot assume they will complete.
   *
   * You must not try to register extra cleanups mid-sweep.
   */
  add(name: string, level: CleanupLevel, callback: CleanupCallback, timeout: number = 1000) {
    if (this.#sweeping) {
      logger.warn('adding shutdown handler mid-shutdown is an api misuse', { name });
      return;
    }
    const order = this.#order++;
    this.#cleanups.push({ name, level, order, callback, timeout });
    logger.debug('registered cleanup', { name, level, order });
  }

  gracefulShutdown(signal: string) {
    if (this.#mode >= SEVERITY.GRACEFUL) {
      return this.#exit(true, `${signal} forced exit`);
    }
    logger.info('shutdown signal received', { signal });
    void this.#sweepAndExit(SEVERITY.GRACEFUL, this.#gracefulCap);
  }

  crashShutdown(kind: string, reason: unknown) {
    // Each incoming error can be an extra bit of information.
    logger.error('emergency shutdown initiated due to unhandled error', { kind, reason });
    if (this.#mode >= SEVERITY.CRASH) {
      return this.#exit(true, 'continuous crashing');
    }
    void this.#sweepAndExit(SEVERITY.CRASH, this.#emergencyCap);
  }

  #cleanupLevelPerMode(): CleanupLevel {
    return this.#mode === SEVERITY.CRASH ? CLEANUP_IMPORTANCE.CRITICAL : CLEANUP_IMPORTANCE.HYGIENE;
  }

  #hardBrakeTimer: NodeJS.Timeout | undefined;

  /**
   * Only one sweep should ever run.
   * Sweep re-checks deadlines and levels to degrade cleanup
   * on the fly to the most strict appropriate for current state.
   */
  async #sweepAndExit(severity: SeverityLevel, shutdownBudget: number) {
    this.#mode = Math.max(this.#mode, severity);
    // Deadline must only ever shorten
    const nextDeadline = Math.min(Date.now() + shutdownBudget, this.#sweepDeadline ?? Infinity);
    if (nextDeadline < (this.#sweepDeadline ?? Infinity)) {
      clearTimeout(this.#hardBrakeTimer);
      this.#hardBrakeTimer = setTimeout(
        () => this.#exit(true, 'timeout hard brake'),
        Math.max(0, nextDeadline - Date.now()),
      ).unref();
    }
    this.#sweepDeadline = nextDeadline;
    if (this.#sweeping) {
      return;
    }
    this.#sweeping = true;
    const totalBudgetNeeded = this.#cleanups
      .filter((c) => c.level <= this.#cleanupLevelPerMode())
      .reduce((a, c) => a + c.timeout, 0);
    if (shutdownBudget < totalBudgetNeeded) {
      logger.warn('total cleanup timeouts exceed the budget', { shutdownBudget, totalBudgetNeeded });
    }
    // First priority by level, then LIFO queue to allow a more natural registration order
    this.#cleanups.sort((a, b) => a.level - b.level || b.order - a.order);
    for (const cleanup of this.#cleanups) {
      if (cleanup.level > this.#cleanupLevelPerMode()) {
        continue;
      }
      if (Date.now() > this.#sweepDeadline) {
        logger.warn('shutdown sweep took too long to complete', { name: cleanup.name });
        break;
      }
      const remaining = Math.max(0, this.#sweepDeadline - Date.now());
      const budget = Math.min(cleanup.timeout, remaining);
      let cleanupOwnTimeout: NodeJS.Timeout | undefined;
      try {
        await Promise.race([
          Promise.resolve().then(() => cleanup.callback()),
          new Promise((_, reject) => {
            cleanupOwnTimeout = setTimeout(() => reject(new Error('timeout')), budget).unref();
          }),
        ]);
        logger.debug('cleaned up resource', { name: cleanup.name });
      } catch (err) {
        logger.warn('resource cleanup failed', { name: cleanup.name, err });
      } finally {
        clearTimeout(cleanupOwnTimeout);
      }
    }
    this.#exit();
  }

  #exit(force?: boolean, extraProblemMessage?: string) {
    const code = (this.#mode === SEVERITY.CRASH || force) ? 1 : 0;
    const reasonMessage = this.#mode === SEVERITY.CRASH ? 'emergency exit' : 'graceful exit';
    const logMessage = extraProblemMessage ? `${reasonMessage}: ${extraProblemMessage}` : reasonMessage;
    if (code === 1) {
      logger.error(logMessage);
    } else {
      logger.info(logMessage);
    }
    if (force) {
      process.exit(code);
    } else {
      logger.end(() => process.exit(code));
    }
  }
}
```

---

## User

Please help me review some 3rd party code:
- Why this code is like that?
- Kubernetes will kill the process using its own timeout anyway, so why bother?
- Kill eventually frees all system resources, so why bother?
- Could the same effect be achieved with simpler code, if that code is shared across services?

## Claude

### What this code is doing

It's a shutdown state machine with a few deliberate design choices:

**Two severity tiers.** `GRACEFUL` (SIGINT/SIGTERM) vs `CRASH` (uncaughtException/unhandledRejection). Mode only ever escalates (`Math.max`), and the deadline only ever *shortens*. So if a crash happens mid-graceful-shutdown, it tightens the budget and escalates severity — it never loosens back up. That's the right invariant: shutdown urgency should be monotonic.

**Two cleanup priority tiers.** `CRITICAL` vs `HYGIENE`. On a crash, only `CRITICAL` cleanups run (`#cleanupLevelPerMode()` returns `CRITICAL` when mode is `CRASH`). The idea: if the process is in an unknown/corrupted state, you only trust the things that absolutely must happen (e.g. releasing a distributed lock) and skip nice-to-haves (e.g. flushing a metrics buffer) that could themselves hang or misbehave.

**Sort order: priority first, then LIFO.** Mimics destructor ordering — clean up in reverse of acquisition order within the same priority band, which avoids use-after-free-style bugs (e.g. closing a DB pool before the code that still uses it during its own cleanup).

**Per-cleanup timeout race + a global "hard brake" timer.** The `Promise.race` per-cleanup timeout stops *waiting* on a misbehaving callback, but doesn't cancel it — a leaked promise could still hold the event loop open. The separate `#hardBrakeTimer`, set once and only tightened, is the actual backstop that guarantees `process.exit` happens no matter what any individual cleanup does.

**`logger.end(callback)` before non-forced exit.** Loggers are often async (network/file writers); calling `process.exit` while buffered logs are unflushed silently drops them. This is a very common real-world bug this code is explicitly avoiding.

---

### "Kubernetes will kill it anyway with its own timeout"

True, but that's exactly the failure mode this code exists to avoid. K8s sends `SIGTERM`, waits `terminationGracePeriodSeconds` (default 30s), then sends `SIGKILL`. `SIGKILL` can't be caught — no code runs at all when it lands. So relying on Kubernetes alone means:

- Buffered logs, traces, and metrics for the last few seconds of the process's life are just gone.
- In-flight requests get an abrupt connection reset instead of a clean response.
- Anything holding a lease, lock, or registration in another system (service discovery, a DB session, a queue consumer) doesn't get released — other systems have to *time it out*, not just notice it's gone.
- Crashes from `uncaughtException`/`unhandledRejection` aren't a Kubernetes concern at all — without this code, the process would keep running in a broken/undefined state indefinitely, serving traffic incorrectly, until a liveness probe eventually notices (which can be tens of seconds).

The internal timeout being *shorter* than k8s's grace period is also intentional: it guarantees your own logic finishes and exits cleanly with time to spare, rather than racing an unpredictable `SIGKILL`.

---

### "Kill eventually frees all resources anyway"

Only OS-managed resources (memory, file descriptors, sockets) — the kernel reclaims those regardless. What this code protects isn't memory, it's **externally-visible state and unflushed work**:

- Data sitting in an in-process buffer (logs, metrics, traces) that hasn't been written out yet — killed before flush means permanently lost, not delayed.
- Peer-visible connection semantics: closing a socket with a proper FIN lets the other side finish cleanly; a hard kill looks like a network error to everyone talking to you.
- External coordination state: a lease, lock, or queue message that needs an explicit release/ack/commit — otherwise another system has to detect the failure via its own timeout before it can recover, which is slower and can cause visible errors upstream.
- Deregistration from a load balancer or service registry — without it, traffic keeps getting routed to a dead instance until a health check fails.

So the reasoning is "resources aren't the concern — the concern is other systems' correctness and observability of *this* process's last moments."

---

### Could this be simpler if shared across services?

Yes, and honestly it probably should be a shared package rather than something living per-repo, because the subtle correctness bugs here (deadline-only-shortens, unref'd timers, idempotent re-signal handling, race between concurrent sweeps) are exactly the kind of thing that's easy to get right once and easy to get wrong every time it's copy-pasted.

Where it could genuinely be simplified:
- The `CRITICAL`/`HYGIENE` two-tier priority system is bespoke complexity most generic shutdown libraries don't have. Worth asking whether it's earning its keep in practice, or whether "always run everything, just with a shorter timeout under crash" would be good enough.
- The manual `setTimeout`/`clearTimeout` bookkeeping could be replaced with `AbortSignal.timeout()` plus `Promise.race`, which reads more clearly and composes better.
- Established libraries already solve most of this generically — e.g. `stoppable` or `lightship` (Kubernetes-aware graceful shutdown) or `async-exit-hook`. If this codebase doesn't have a strong specific reason (the two-tier severity/priority model) for rolling its own, adopting one of those and layering the org-specific bits on top would cut a lot of this custom state machine.

If the two-tier model *is* load-bearing (i.e., services genuinely have cleanups that shouldn't run mid-crash), then extracting this exact class into a shared internal package — as-is — is probably the better move over simplifying it away, since the complexity is doing real work and you want it defined once, not once-per-service with drift.

---

## User

I looked at your recommendations:

`stoppable` - isn't that just a `http` wrapper?

`lightship` - isn't that an http server and kube discovery, not a shutdown handler?

`async-exit-hook` - looks similar, also two tiers (shutdown and error hooks separated), but hardcoded 10s timeout, global state, parallel only, declaration-order, requires calling a callback for async hooks

What would you recommend?

## Claude

*(Ran a web search: "node.js graceful shutdown library LIFO priority tiers 2026")*

You're right on all three counts — I was reaching for names without checking them closely enough. Let me correct that.

- **stoppable**: yes, it only wraps `http.Server` to track in-flight connections/sockets so `server.close()` doesn't hang forever waiting on keep-alives. Zero relation to process-level cleanup orchestration.
- **lightship**: it's a small HTTP server that exposes `/live` and `/ready` endpoints for k8s probes, plus signal handling to flip readiness before exiting. It does have a shutdown handler registry, but it's coupled to running an HTTP server — a bad fit if this is meant to be a general-purpose cross-service primitive (e.g. used in a worker with no HTTP server at all).
- **async-exit-hook**: correct on all four points, and each one is a real regression from what you have — global singleton means multiple libraries in the same process fight over one hook set; hardcoded 10s timeout means you can't tune per-signal or align with your k8s `terminationGracePeriodSeconds`; parallel-only means no LIFO/dependency ordering, so you can't safely close a DB pool before the code using it during shutdown finishes; and callback-style async (`done()`) instead of returning a promise is just an awkward, error-prone API by 2026 standards.

I searched again rather than guess further, and there genuinely isn't a well-maintained library that does *severity escalation (graceful→crash) + priority tiers + monotonically-shortening deadline + LIFO-within-tier* as one coherent model. The closest conceptually is `node-graceful-shutdown`, which does dependency-based ordering (explicit `["http-server"]` deps between named handlers) instead of LIFO — arguably a *better* ordering primitive than positional LIFO, since it doesn't silently break if someone reorders `add()` calls — but it has no severity/crash-vs-graceful distinction at all, and its own docs say **"do not use this module in libraries"**, i.e. it's meant for the app entrypoint, not something to embed in a shared internal package. Everything else I found (`watchtower`, `gracious`) is either explicitly not production-ready or solving a different problem (in-flight request tracking).

**My actual recommendation: don't replace this, extract it.**

The two-tier severity/priority model here is doing real work that no library captures off the shelf, and the subtle correctness bits (deadline only shortens, single sweep guard, unref'd timers, idempotent re-signal → forced exit) are exactly the kind of thing you want defined once. Concretely:

1. Pull `ShutdownCleanup` into a shared internal package (`@yourorg/shutdown` or similar) as-is, unit test the state machine (escalation, deadline shortening, level filtering, sort order) in isolation.
2. Two small modernizations while you're in there, neither of which needs a new dependency:
   - Replace the manual `setTimeout`/`clearTimeout(...)` per-cleanup with `AbortSignal.timeout(budget)` and pass the signal through, if your callbacks can accept one — cleaner cancellation, no leaked timers to track by hand.
   - Consider swapping positional LIFO for `node-graceful-shutdown`'s explicit-dependency idea (`add(name, level, cb, { after: ['db'] })`) — same LIFO default if unspecified, but removes the "must add in the same order as acquired" fragility that's currently just a comment, not enforced.
3. Keep the crash/graceful distinction and the `CRITICAL`/`HYGIENE` split — that's the actual value-add over every library I found, so it's worth owning rather than discarding for a generic tool that doesn't have it.

---

## User

What is the author's intention with the split? Does it hold a ground?

(If you suggest anything that is not **actually enforceable**, you will be punished.)

## Claude

### Intention

The split encodes a trust boundary tied to process state, not just importance. Under `CRASH` (triggered by `uncaughtException`/`unhandledRejection`), the process is in an unknown, possibly corrupted state — you don't know what invariants still hold. So the author only trusts cleanups explicitly marked `CRITICAL` (locks, leases, anything another system needs released) and refuses to run `HYGIENE` ones (metrics flush, cache warm-down, etc.) at all, on the theory that code written assuming a healthy process might itself misbehave, hang, or do something wrong when run against corrupted state — better to skip it than let it make things worse or eat your already-short emergency budget.

Under plain `GRACEFUL` shutdown, both tiers run — `#cleanupLevelPerMode()` returns `HYGIENE` (1), and the filter `c.level <= 1` admits both `CRITICAL` (0) and `HYGIENE` (1).

### Does it hold ground? Partially — here's the exact mechanism and its exact limit.

What's actually enforced: the filter is evaluated live, per-iteration, inside the `for` loop:

```
if (cleanup.level > this.#cleanupLevelPerMode()) continue;
```

Since `#cleanupLevelPerMode()` reads `this.#mode`, and `this.#mode` can escalate mid-sweep (a crash can fire while a graceful sweep is already awaiting a cleanup), this correctly stops *starting* any further `HYGIENE` cleanups the moment mode escalates to `CRASH`. Combined with the sort (`level` first, then LIFO), `CRITICAL` cleanups are always attempted before `HYGIENE` ones regardless of mode, so in the common case — crash from the start, or crash arriving while critical work is still in progress — the guarantee holds cleanly.

Where it does **not** hold: the loop is not preemptive. Each cleanup is awaited via

```
await Promise.race([Promise.resolve().then(() => cleanup.callback()), timeoutPromise])
```

`Promise.race` only stops the sweep from *waiting* on a slow callback — it does not cancel the callback's own execution. So if mode escalates to `CRASH` at the exact moment a `HYGIENE` cleanup is already running (possible since `CRITICAL` finishes first but `HYGIENE` items take real, non-zero time), that in-flight `HYGIENE` callback keeps executing in the background, untouched by the mode change, for up to its own `timeout`. The stated principle — "don't trust/run non-critical code once we're in crash mode" — is therefore "no *new* `HYGIENE` work starts," not "no `HYGIENE` work runs." That's a real, narrow gap, and it's one the author appears aware of: the class's own leading comment floats `AbortSignal` as a future mechanism, which is exactly what would be needed to actually cancel in-flight callbacks and close this hole.

One more limit worth naming: the split is enforced structurally by the scheduler, but the *classification itself* is unenforced. `CRITICAL` vs `HYGIENE` is just a label the caller supplies at `add()` time — nothing in the code verifies a `CRITICAL` callback is actually fast/safe, or that a `HYGIENE` callback doesn't do something that matters. The guarantee only holds to the extent every call site classifies honestly; a mis-tagged `HYGIENE` cleanup that's actually load-bearing will silently be skipped on every crash.

---

## User

> classification itself is unenforced

I told you about the punishment, didn't I? Now you will be punished, by having to explain why this is not a realisitc requirement.

## Claude

Fair — let me argue the other side, since that critique was more of a "note in passing" than a load-bearing flaw.

**Why enforcing it isn't realistic:**

1. **It's a semantic claim, not a syntactic property.** "Is this cleanup safe to run against corrupted process state" or "does something downstream actually depend on this completing" is a statement about the callback's *meaning* and its relationship to the rest of the system — not something derivable from its code shape. There's no static analysis, type constraint, or runtime check that can look at a `() => void | Promise<void>` and determine whether it's touching something a distributed lock depends on. The type system can constrain shape, not intent.

2. **Any enforcement mechanism just moves the trust problem, it doesn't remove it.** Suppose you tried to force honesty — e.g. requiring `CRITICAL` cleanups to declare their side effects, or sandboxing them, or requiring proof they're idempotent/side-effect-scoped. Now the caller has to correctly declare *that* metadata, which is just as unverified as the original label. You haven't closed the gap, you've added a layer of indirection with the same problem at the bottom. This is the general shape of "who verifies the verifier" — at some point a human has to assert something true about their own code, and no framework eliminates that last step.

3. **The cost of any real enforcement would defeat the purpose.** The entire reason this class exists is to run cleanup fast, under a hard deadline, during an emergency. Actually verifying a classification claim would mean something like static analysis of the callback body, runtime tracing to detect "did this touch a lock," or a formal effect system — all things that are either impossible in JS without severe restrictions on what a cleanup callback is *allowed to do* (killing the flexibility that makes `add()` useful for arbitrary resources), or too slow/heavy to run during a shutdown sweep that might have a 1-second budget.

4. **Compare it to every other "labeling is trusted" boundary in software** — `@internal` JSDoc tags, `private` fields before runtime enforcement existed, HTTP method semantics (nothing stops a `GET` handler from mutating state), even `CLEANUP_IMPORTANCE` sitting next to `SEVERITY` in this same file with a doc comment saying what each level *means*. All of these are conventions backed by documentation and code review, not compiler-checked invariants, and that's normal — not a design flaw specific to this class.

So the realistic bar for this kind of API isn't "the type system rejects a mis-tagged callback" — it's "the contract is written down clearly enough that a reviewer can catch a mis-tagging in PR review," which the existing doc comments (`"We really prefer this handler to run ASAP and complete"` / `"Handler is hygienic; a nice-to-have, optional"`) already do reasonably well. Flagging it as a gap was accurate in the narrow sense that nothing *mechanically* enforces it — but holding a self-documenting, convention-based API to a standard of mechanical enforcement is the wrong bar; that standard would apply equally to almost every access-modifier and naming convention in the codebase, and none of those get flagged as flaws for the same reason.

---

## User

i need this whole convo as a markdown file
