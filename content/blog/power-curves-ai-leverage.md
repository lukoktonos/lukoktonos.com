+++
title = "AI Gives You Gears, Not Experience"
date = 2026-02-22
+++

Software knowledge and software productivity both look like power curves to me.

At a high level, we name part of that curve with career labels: junior, mid-level, senior, staff. Those labels are useful, but they hide a lot of spread inside each band. Two engineers with the same title can differ a lot in how they reason about systems, how quickly they find the real problem, and how consistently they make good tradeoffs.

Some of that spread is experience. Some of it is curiosity. Some of it is how often someone goes deep instead of stopping at the first working answer. Talent probably plays a role too, but in practice I notice who learns quickly and who consistently makes high-quality engineering choices.

Most software work was never about raw typing speed, although curating an efficient development environment and learning your tools (IDE, editor, CLI commands, etc.) are themselves productivity multipliers that many people never properly take the time to curate. The hard part is choosing the right problem, reducing state and complexity, deciding what can fail and how, knowing what to measure, and being right about tradeoffs under real constraints.

These skills compound over time, and they stack. You rarely jump straight to strong distributed systems judgment without first getting solid at debugging, API design, and operational hygiene.

That is most of the explanation for the curve: progression between levels and wide variation within levels.

AI did not remove that shape. If anything, it made the slope steeper.

## What AI changed

AI is very good at turning intent into output. It lets you explore more options before committing, build throwaway prototypes quickly, delegate routine implementation and transformation tasks, and keep momentum with less context-switching.

The mechanical analogy that fits best for me is gearing. A higher gear lets you cover more ground per rotation, but it does not teach balance, line choice, or when to brake. In practice, AI acts more like amplification than equalization. Engineers with strong judgment can steer models, discard weak options quickly, and combine drafts into something coherent. Engineers earlier in the curve still get speed, but often spend more of that speed recovering from wrong turns.

## Why the gap can widen

A strong engineer with AI often behaves like a small team, running one thread on architecture options, another on concrete implementation, another on validation and guardrails, and another on documentation and operationalization. The bottleneck shifts toward synthesis, and the better your synthesis loop is, the more useful parallel work you can absorb.

That is why the top of the curve can pull away faster now: they can translate judgment into parallel execution. Give two engineers the same gear ratio and the one with better handling still exits the corner faster.

## Can you move up the curve?

I think so, but mostly through the same path as before.

The path still runs through fundamentals:

1. Learn to debug systems you did not write.
2. Learn to predict failure modes before production teaches you.
3. Learn to make tradeoffs explicit: latency, cost, consistency, complexity, team cognition.
4. Learn to close loops with measurement instead of opinions.

Then layer AI on top as leverage:

1. Use it to generate alternatives, not just answers.
2. Make it explain assumptions and edge cases.
3. Treat every output as a draft that must survive tests, instrumentation, and operational reality.
4. Build personal workflows where AI handles bulk work and you spend time on decisions.

The sequence matters. People keep looking for shortcuts around fundamentals; I have not seen one yet. Better gearing helps once you can already transfer power effectively.

AI has changed the amount of leverage available at each level. It has not changed the fact that levels exist, or that climbing them still takes deliberate practice and exposure to real production consequences. You still have to build the legs first.
