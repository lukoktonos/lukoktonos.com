+++
title = "Interruptions Get More Expensive with Agentic Work"
date = 2026-02-26
+++

There is nothing new under the sun here: interruptions have always been expensive, and the tension between management cadence and developer focus has always been real. If you are deep in a problem, a ping from your manager is usually not a thirty-second detour. It is context eviction followed by a slow rebuild.

The new twist is not that interruptions hurt. The new twist is what gets interrupted.

AI helps with recovery. You can externalize state into prompts, notes, diffs, and transcripts, then reload that state later. That part is genuinely better than it used to be.

At the same time, many of us now run several active streams: implementation in one pane, exploration in another, tests or cleanup in a third. These streams are related. They share assumptions, converge on the same interfaces, and invalidate each other when decisions change. So an interruption often preempts a coordinated set of streams, not a single linear task.

The OS analogy is useful here. A context switch is the system pausing one unit of work, saving enough state to resume it later, selecting another runnable unit, restoring state, and continuing execution.

Switching between threads in one process is often cheaper because much of the memory context is shared. Switching between separate processes is typically heavier because process-level memory context changes, which is rougher on locality and cache warmth. Forked process trees are heavier still: parent and workers can be paused at different points, and resume now includes re-coordination, not just restart.

That is close to what multi-stream agentic work feels like in practice. The immediate interruption might be one ping, but the recovery cost includes rehydrating each stream and then reconciling them.

You can model it roughly as:

```text
interruption_cost
  ~= base_restore
   + stream_count * per_stream_restore
   + interaction_points * reconciliation_cost
   + (stream_count * interaction_points) * coordination_penalty
```

The last term is the important one: it captures coupling overhead. More streams and more interaction points multiply each other, which is why interruption cost can grow faster than linearly.

For developers, the play is to reduce interaction points and offload state aggressively: short handoff notes, explicit decision records, cleaner boundaries between active streams.

For managers, this is a reminder that existing best practices matter even more now. If engineers are running multiple agentic streams, ad hoc preemption is expensive. Protected focus blocks, batched asks, and a culture where pings are not assumed to require immediate response all pay off.

AI did not change the basic physics of attention. It increased how much parallel work we can run, which makes interruption discipline more important than ever.
