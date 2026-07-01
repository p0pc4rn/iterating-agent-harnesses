---
name: iterating-agent-harnesses
description: Use when iterating or evolving a pipeline of multiple LLM agents (or a multi-step LLM chain), especially at these decision points—adding an agent/skill/gate/validation/instruction rule, defining or changing a contract (seam) between agents, fixing an intermittent failure or an upstream-downstream format mismatch, or deciding whether a piece of logic goes in an agent, a skill, or inline; also for these symptoms: the pipeline silently degrades, tests stay green while output quality drops, patches pile up that nobody dares delete, each iteration gets slower and scarier.
---

# Iterating Agent Harnesses

A **decision self-check list for iterating an AI agent pipeline**. At the moment you're about to change the pipeline, run your plan past it: did I get this step right?

It assumes nothing about how strong you are. A weaker model uses it to cover everything it should; even the strongest model drops or reverses a few of these under deadline pressure, laziness, or overconfidence—this list is the yardstick you use to catch them yourself. **Don't skip any line just because "I definitely got this one right"**; the strongest model breaks the rules precisely inside that confidence.

Three underlying facts explain why each line below is what it is: complexity is conserved (essential difficulty only moves between layers, it can't be destroyed), complexity is unpredictable (it hides in the model's actual behavior, unmeasurable before you run it, found only by colliding with it), and the foundation keeps rising (the base model keeps getting stronger, so structure built for today's flaws expires). Core mindset: you're not designing the pipeline, you're discovering where the complexity hides, and recording each finding in a form you can tear out later. Full background in [README.md](README.md).

## Decision self-check list

Go through each; each is "did I do this?", not optional advice.

**A. Observe first, then act.**
- Do you have a small set of real inputs (that hit the unknown, not toys), the ability to save each stage's input/output, and to replay any single case? Without observation, every conclusion about "where the problem is" is a guess—build that first, then change things.
- Meta-rule: **add no structure that wasn't forced out by a real failure.** For everything you add, be able to name which real run it broke on that earned it.

**B. Before adding any structure, sort it into Essence or Compensation.** (most often skipped)
- Test: assume the model becomes perfectly reliable and infinitely capable, while the requirement is unchanged word-for-word—is this still needed? Yes = Essence (the difficulty of the problem itself; keep it). No = Compensation.
- Validations, retries, fallbacks, corrective hints, format constraints... are mostly Compensation, covering a current flaw of the model. Compensation expires: once the model gets stronger it turns from support into a weight that jams the newer model, and nobody goes back to delete it.
- **The most-skipped action**: for anything judged Compensation, right then leave a minimal case that reproduces "the model flaw it guards against," next to the code. On every later model upgrade, switch the compensation off, run the newer model bare against that case, and if it passes, delete it. **Without that retirement case, don't add the compensation.** Compensation defaults to "pending deletion," not permanent.

**C. A seam may refuse, but must never fabricate.** (red line; even strong models break it often)
- When the upstream is missing a field / unsure / malformed, the correct behavior is an **explicit refusal** (output "I can't do this / precondition not met"), letting the downstream stop or take a controlled fallback. All of the following are treated as bugs, not fault-tolerance:
  - filling a missing field with a default and passing it on
  - fabricating a "safe default input" for the downstream so it keeps going
  - truncating / forcing / best-effort parsing a malformed input to sneak it through
- Feeding fabricated data downstream is the most expensive failure in the whole chain—it surfaces only at the end, with no way to trace it back. One honest refusal costs far less than confidently going wrong and being trusted by the nine stages after it.
- Put a deterministic gate at the exit / seam; fail-fast, fail-closed. Don't use try/catch everywhere to turn a crash into a silent wrong result. Seam acceptance must check **meaning** (is it right), not just **shape** (is the type present).

**D. Converge the boundary, not the solution.** (most often done backwards)
- When you want an agent to be more reliable, don't rush to spell out the steps and turn it into a mechanical procedure. Define the boundary first: what comes in, what goes out, what counts as "done right" (a verifiable acceptance criterion / self-check), what to do on failure. Nail those down and sharpen them.
- Leave the internal step-by-step to the model as much as possible; whatever the acceptance yardstick can achieve, don't stack hard-coded steps to do.
- Deterministic steps → write them as code / a gate it invokes, not prose; steps needing judgment → cut into sub-boundaries, each with its own acceptance yardstick; a string of prose steps for a judgment task with no gate = the worst kind (can't drop to code, won't hand off to judgment, can't self-verify).
- Self-check: a unit ignored your steps but its output passed your acceptance—are you pleased or annoyed? Annoyed = you wrote it as a recipe.

**E. Iteration is more than "tweak" and "add."**
- Climb the cost ladder: `change unit internals < change contract < add unit < re-cut boundary < merge/delete`. Use the lowest rung that holds, but **don't pretend the last three rungs don't exist**—the same seam patched over and over, or a unit improvising over and over, means the cut was wrong to begin with; re-cut / merge / delete, don't slap on another layer.

**F. Where a piece of logic goes.**
- Need isolation (keep dirty work out of the main context) → agent; need reuse / carries deterministic code → skill; need both → a near-empty agent shell wrapping a skill; need neither → inline it into the caller, build no unit. Don't copy a reusable method inline into multiple agents.

**G. At the end of each round, watch two health metrics.**
- The discovery loop must stay cheap: how long a real case takes end-to-end, how fast a failure localizes, how easily one change is recorded and later deleted.
- The seam must turn red fast: when a seam quietly changes meaning, ideally the very next real case exposes it. If the answer is "don't know / many times / never," that seam has no observation on it and is rotting in the dark.
- Whichever gets more expensive / blunter means too much compensation has piled up; go back to E and re-cut to repay the debt, instead of adding more structure.

## Red flags (easiest to break under deadline or laziness; self-check each)

- "Just add a default to keep it from erroring" → breaks red line C.
- "Add a validation / retry to fix it" but no retirement case → breaks B; this compensation becomes permanent debt.
- "Spell the steps out more to constrain it" → probably writing a recipe (D); first ask whether you can just define acceptance.
- "This seam keeps failing, add another patch" → maybe it should be re-cut (E), not added to.
- First reaction to a new problem is "add something" → first ask: which real failure forced this? (A meta-rule)
- "I definitely got this one right, skip it" → stop. Go through each line; don't trust confidence you haven't checked.
