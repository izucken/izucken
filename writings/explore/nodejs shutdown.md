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
