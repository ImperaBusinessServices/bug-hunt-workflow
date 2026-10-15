---
name: bug-hunt-workflow
description: Find bugs in a large code base the reliable way - split the code among many fresh agents (hunters), then give every reported bug to a separate skeptic agent whose only job is to prove it wrong. Use when Keith says "check this code for bugs", "audit this code base", "bug hunt", "find everything wrong with this", "use a workflow to review", or asks for a thorough review of a big project. A single agent reviewing a big code base misses most bugs; this pattern found 66 of 70 planted bugs versus 14-27 for one agent.
---

# Bug-hunt workflow (hunters + skeptics)

Source: TikTok by @thehypekevin (2026-10), summarising Anthropic's "dynamic workflows" test: 70 bugs hidden in 116,000 lines of code. One agent found 14, 15, then 27 on a good day. A workflow found 66, three runs out of three.

## Why one agent fails
- It carries the whole job in one context window; every file it reads piles in.
- When the window fills it summarises, each summary drops detail, so by file 400 it has lost the job.
- It trusts its own findings because it wrote them.
- Around bug 15 the remaining pile "looks finished enough" and it stops.
- A bigger window or ordinary helpers do not fix it: helpers still report into the same head, and the lead still decides each step one turn at a time.

## The pattern
1. **Phase 1 - Hunt.** Split the code into pieces (by folder or file group, small enough to read fully). Give each piece its own agent with an empty context. Each returns a list of suspected bugs: file, line, what is wrong, how it fails.
2. **Phase 2 - Disprove.** Take every reported bug and hand it to a *fresh* agent whose only job is to prove it wrong. It did not write the claim, so it has no reason to defend it. Output: real / not real, with evidence.
3. **Program passes results between phases** (the script holds the lists, not an agent's memory), so the plan is never summarised away.
4. Keep only bugs that survive. Report them ranked by severity.

The lead stays free to talk to Keith while the workers run.

## How to run it in Claude Code
1. This uses many agents and lots of tokens, so only run it when Keith asks for a thorough review or explicitly says "use a workflow". Otherwise describe it in one line and offer.
2. Load the `workflow-authoring` skill, then call the `Workflow` tool with a script shaped like this:

```js
export const meta = {
  name: 'bug-hunt',
  description: 'Hunters per code slice, then a skeptic per finding',
  phases: [{ title: 'Hunt' }, { title: 'Disprove' }],
}
// 1. list slices first (Glob/Bash), pass in via args or build here
const hunts = await parallel(slices.map(s => () =>
  agent(`Read every file in ${s}. List real bugs only (file, line, why, how it fails).`,
        { label: `hunt:${s}`, phase: 'Hunt', schema: FINDINGS })))
const claims = hunts.flatMap(h => h.findings)
const judged = await parallel(claims.map(c => () =>
  agent(`Try hard to PROVE this bug claim wrong by reading the code. Claim: ${JSON.stringify(c)}`,
        { label: `skeptic:${c.file}`, phase: 'Disprove', schema: VERDICT })
    .then(v => ({ ...c, verdict: v }))))
return judged.filter(j => j.verdict?.isReal)
```

3. Size the slices so hunters finish a slice fully (rough guide: 5-15 files each). Cap total agents to what Keith agreed to; a run can start up to 1,000 agents and each one spends tokens.
4. In the Anthropic API (Managed Agents) it is one field on the agent to switch dynamic workflows on, and the session needs a token budget so the run pauses when it hits it.

## Reporting back to Keith
Plain English, short: how many suspects the hunters raised, how many the skeptics knocked down, the survivors as a short list (what is wrong, where, why it matters). Mention rough cost and time.

## Reference result (from the video, for calibration)
A real run on a small publishing tool: 5 hunters raised 10 bugs, 10 skeptics, 9 survived. One bug: every follow-up post went out in the same minute. Took 3.5 minutes and about 1.3 million tokens. Expect it to cost far more than a single-agent review - that is the trade.
