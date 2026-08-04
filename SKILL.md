---
name: brainstorming
description: "Adversarial debate between this session and a second Claude model (fable / opus / sonnet / haiku) over a hard technical problem, run as a real multi-round conversation and ending in a ranked plan where every surviving idea carries a number, the regime that number holds in, and a pre-declared kill-check. Use when the user says 'fai brainstorming con fable/opus', 'dibattito con un altro modello', 'brainstorming with opus', 'discutiamone', 'second opinion from another model', or invokes /brainstorming."
argument-hint: "[con|with] [fable|opus|sonnet|haiku] [rounds=N] <topic>"
---

# Brainstorm — two models, one problem, arithmetic as the referee

Two models argue a problem for N rounds. That part is easy and it is not the point.

The point is what the argument is *made of*. Ideas enter against real numbers pulled from the thing
itself, they must clear the project's own graveyard of ideas that already died, and they leave
carrying an expected gain, the **regime** that gain holds in, and a **kill-check** — the measurement
that would prove them wrong — declared before anyone runs anything.

An idea that has not survived an arithmetic attack does not go on the list. A gain quoted without a
regime is not an estimate. That is the whole method.

## 0. Parse and open

| Input | Meaning |
|---|---|
| `fable` \| `opus` \| `sonnet` \| `haiku` (after `con` / `with`, or anywhere) | peer model |
| `rounds=N`, `N giri`, `N round` | rounds (default **3**, cap **5**) |
| everything else | the topic |

**No model named** → pick one *different from the session's own* (session on Opus → `fable`), don't
ask. Same model, same blind spots. **No topic** → whatever the conversation is about.

One header line, then start:

`Brainstorm — peer: fable · 3 rounds · topic: MoE decode speed, weights streamed from disk`

## 1. Anchors, graveyard, constraint — before spawning anything

This phase is the skill. Skip it and you get two models trading plausible suggestions.

### ANCHORS — real numbers, with where they came from

Read the config, the code, the logs, the container. Not the docs *about* them — them. Then derive
what follows arithmetically. Each anchor is one line and names its source (`file:line`, log path,
the command you ran).

```
config.json: h=6144, inter=2048, 256 experts/layer top-8, n_shared_experts=1
expert int4 = 3·2048·6144/2 = 18.87MB + fp32 scales (gs=128) 1.18MB = 20.05MB per miss
prod 1.34 t/s ⇒ 1.37GB/tok ⇒ 68 miss/tok; 0.47s @2.91GB/s = exactly the measured cap
compute floor: 19.7GB/tok @273GB/s = 72ms/tok ⇒ absolute ceiling ~14 t/s
```

Three anchors are load-bearing, and a brainstorm without them is guesswork:

- **The balance.** Where the time / bytes / money actually go, in percent, summing to ~100. Without
  it, every idea is a bet on which axis matters, and you will rank them wrong. If you don't have the
  balance, **go get it before you spawn** — one 10-minute profiling run in the real configuration
  beats a whole round of debate about which bottleneck is real. Read it off the *production*
  configuration; quotas measured under a different config are quotas of a different problem.
- **The ceiling.** The physical limit (bandwidth, latency, cost floor). Anything that beats it is
  dead before anyone argues, and the distance to it tells you how much is left to win.
- **The measured vs derived split.** Mark derived numbers `~` and say from what. A derived number is
  fine to argue with; a derived number wearing a measured number's clothes poisons the ranking.

### GRAVEYARD — the project's own dead ideas, with cause of death

Grep the past: reports, dated docs, memory files, commit messages, that `TO_FIX.md` nobody reads.
List what was already tried or already killed, one line each, **with the reason it died**.

This is the highest-value block in the brief. Without it both models spend round 1 re-proposing what
died last month, and the second model — which cannot see any of it — spends that round with total
confidence. With it, "survives the graveyard" becomes the entry fee.

Keep the negative results too: *policy family closed, LFU < RANDOM, FIFO ≈ LRU* is worth more than
any idea in the brief, because it forecloses a whole neighbourhood.

### CONSTRAINT — what is scarce this week

The machine is running a benchmark. The deadline is Friday. There is no budget for a second GPU.
Name it, because it reorders everything: **an idea checkable against data already on disk outranks a
better idea that needs the scarce thing.** Say what is already on disk — dumps, logs, past runs —
because half the checks people plan have already been run and nobody looked.

If the scarce resource is untouchable, say so up front: everything the debate produces is paper plus
kill-checks to execute later. That is a legitimate and often the correct output.

### POSITION

Take an actual stance with numbers attached, plus the 1–2 things you genuinely cannot settle alone.
A menu gives the peer nothing to attack.

**What never goes in the brief:** the session transcript, a project tour, file dumps to save yourself
a round, secrets (keys, tokens, `.env`, client names). Missing facts are handled by the loop — the
peer lists them under `NEED:` and gets exactly those next round. Two cheap rounds beat one bloated
brief. There is no word limit on the material the argument actually turns on: ship the whole state
machine, the full struct, the real log.

## 2. Spawn the peer

One `Agent` call — `model`: the parsed model · `subagent_type`: `brainstorm-peer` (ships in
`agents/`, install to `~/.claude/agents/`; **no tools**, deliberately — a peer that reads the repo
inherits the repo's framing, and ground truth is the host's job because it costs the host no turn)
· `run_in_background`: `false` · `description`: `brainstorm peer`.

Not registered yet? Fall back to `general-purpose`, say so in the header, keep the no-repo rule in
the brief as honour-system.

**Record the `agentId: a…` from the spawn result** — that is what you send to.

Prompt = the rules of engagement + your Round 0 block:

```
You are the second model in an adversarial brainstorm. Another Claude session brought a hard
problem to you because you reason differently and see different failure modes. We are not being
polite and we are not scoring points. We are finding out which ideas survive contact with the
arithmetic.

RULES OF ENGAGEMENT
- Kill and build both count. Killing my idea with a number is worth as much as adding yours.
- Kill with arithmetic, not adjectives: which term, what magnitude, why it fails to clear the
  anchors. "This may be risky" is not a kill. "Dedup at k=8 is only 13% of the union, so it needs
  p>=0.79/token and the measured accept is 0.44-0.54" is a kill.
- Every gain declares its REGIME — the state of the system in which it holds. "+58% while disk is
  63% of the wall" is an estimate; "+58%" is not. A lever that moves the system out of its own
  regime is self-limiting, and you must say so.
- Composition is not additive. Say when two ideas draw on the same premium, when stacking two
  approximations multiplies the damage instead of adding it, and which pairs are mutually exclusive.
- Check the anchors before you propose. An idea that beats the physical ceiling is already dead.
- An idea must clear the GRAVEYARD in the brief. If yours is a variant of something already dead,
  say which one and what specifically is different this time.
- Nominate your own riskiest claim under CHECK: the one that, if false, changes the ranking. Don't
  leave me to pick a convenient one.
- NEW: none is a legal move and a real contribution. Padding is the failure that rule prevents.
- You have no repo, no shell, no session history. If a fact is missing, list it under NEED, and make
  it conditional where you can ("NEED X; if yes idea 3 gets stronger, if no drop it") so a missing
  fact forks your thinking instead of blocking the round.
- Unquoted claims about code are not settled facts. If I say "I checked, you're wrong" without
  quoting the file or the output, leave it in OPEN and say why.
- <=250 words. Bullets. No preamble, no praise, no closing summary.

REPLY SHAPE
KILL:  my idea (or your own from last round) -> cause of death, with the arithmetic
NEW:   n. idea — mechanism · expected gain @ which regime · effort (lines / env / hours)
BUILD: what you'd add to mine, or where you'd take it next
CHECK: your own claim that would change the ranking if false
NEED:  ... (omit if none)

--- BRIEF ---
<ANCHORS / GRAVEYARD / CONSTRAINT / POSITION>
```

## 3. The rounds

Each round after the first:

1. **Read it properly**, including the parts that make your position worse. A better frame from the
   peer is the skill working, not you losing — adopt it and say so.

2. **Answer with a `VERIFIED` block.** The peer brings range; you bring ground truth, and you are the
   only one who can — it has no tools, and checking costs you no subagent turn. So every host turn
   either quotes a check or says plainly that nothing this round was checkable.
   **Quote or it didn't happen:** ≤15 verbatim lines with path and line numbers. "I checked, you're
   wrong" is unfalsifiable when only one side can see the repo, and the peer is briefed to leave it
   open. Aim the check at the peer's `CHECK:` nomination or at whatever claim would change the
   ranking if false — verifying safe trivia while the load-bearing claim goes untouched is worse than
   skipping it, because it buys the round a look of rigor it didn't earn.

3. **Put your own ideas and kills on the table**, same shape as the peer's, `none` included.

4. **Keep the ledger** as you go, because by the synthesis you will not remember who said what:

   ```
   ALIVE  #n idea · gain @regime · P(survives) · effort · kill-check · [host|peer|both]
   DEAD      idea · killed by whom · arithmetic cause
   ```

   `[both]` is the tag that matters: a peer mechanism on a host idea, or a third thing neither walked
   in with. They read as obvious by the end, which is exactly how they get lost.

5. **Send** with `SendMessage` → `to`: the recorded `agentId`, `message`: your turn + the facts it
   asked for under `NEED`.

⚠️ Only the first spawn is synchronous. `SendMessage` resumes the peer **in the background**: the
tool returns immediately and the reply lands later as a task notification. Do not respawn because
"it didn't answer", do not write the round before the notification lands, never invent a peer turn.
The wait is when you fetch the `NEED` facts for the next send.

**Optional research leg.** When a claim turns on the literature or on someone else's measurement,
run one bounded research pass — a handful of targeted searches, or one cheap subagent — and bring the
citation back into the debate as a kill or a support (*"per-channel int4 residual ratio is 1.1-1.3×,
arXiv 2403.01384, so entropy coding on top buys 10-20% at most"*). Bounded: no multi-agent research
fan-out, it costs more than the debate it serves.

**Last round narrows.** Say so in the send (`last round — let's rank`). Spend it on which ideas
actually pay, what each costs, and what would have to be true for the expensive ones. Ask the peer
for its own **TOP 3** and **5-line synthesis before you show yours** — you hold the pen, and getting
its version committed first is the only check on that, and it's free.

**Stop** at the cap, or when both sides write `KILL: none` *and* `NEW: none`. Ideas drying up is the
signal; agreement is not. Print each round compactly — substance, not raw text:

```
── round 2/3 ─────────────────────────────
🤖 fable  KILL your F3 (split the read across 4 threads): the block layer already splits the 18.87MB
          pread into ~151 128KB requests — same LBAs, same queue. Bandwidth axis is closed.
          NEW: 4. int2 bit-plane on misses — damage/byte 4.2× better than the route-swap in prod
          CHECK: NRMSE→loss is not proven, and it's the load-bearing step of 4
🧠 host   VERIFIED: colibri.c:5593 `g_draft=0;` — nctx is read 6 lines later, so the "one-line fix"
          does not compile as stated · taking 4 · #4 excludes CACHE_ROUTE (that combo measured −30.8)
```

## 4. Verify before you plan — not optional

A two-model debate produces confident, wrong plumbing. Both models are reasoning about code neither
just read, and the failure is systematic: **expect roughly one plumbing claim in four to be wrong.**

So after the debate and before the plan, take every claim of the form *"it's one line at X"*, *"that
flag is already passed"*, *"the data isn't there"*, *"the tree is at commit Y"* and check it. Mark
each: **VERIFIED** (with the quote), **WRONG** (quote + correction), **UNVERIFIED** (say so plainly).

What this catches, every time: prerequisites already satisfied (zero work — drop the step), one-liners
that don't compile as described, checks that need no scarce resource because the data has been on disk
for a week, and a **dirty working tree** — measure with uncommitted changes in it and the run scores
those changes too.

Then reorder the plan against the corrections. This pass has flipped rankings. It is cheap, and it is
the difference between a plan and a wish.

## 5. Output

Two artifacts, and **one line at the top saying which is authoritative** — the worst outcome of a good
brainstorm is three docs that disagree next month:

- `BRAINSTORM-<slug>-<YYYY-MM-DD>.md` — the record: dead first, then alive with the ranking, then the
  tree. Compressed, not a transcript dump.
- `PLAN-<slug>-<YYYY-MM-DD>.md` — the ordered actions, when there are any. This one supersedes the
  brainstorm doc as the order of work; say so explicitly in both.

```
ANCHORS      the numbers everything was argued against, with sources
DEAD         idea | who killed it | arithmetic cause of death       ← publish this, it's an asset
ALIVE        ranked by gain × P(survives) / effort. Each: mechanism, gain @regime, effort in real
             units (lines of C, env vars, hours), pre-declared kill-check, [host|peer|both]
COMPOSITION  which levers share a premium (don't sum them), which are mutually exclusive
WATERFALL    the numeric chain to the endpoint: 368ms/tok → A 271 (3.69 t/s) → B 230 (4.35) → …
             and where the regime flips, because past that point the ranking is void
TREE         what to run, ordered. The root is not the top-ranked lever — it is the cheapest
             measurement that decides between two rankings that cannot both be right (10 minutes of
             profiling beats an hour of arguing). Then the levers, each gated on its kill-check.
FREE NOW     what can run without the scarce resource, or against data already on disk
OPEN         what neither model settled, one line each with both views
```

Ranking rule: **gain × P(survives) / effort**, with all three written down. A ranking without
probabilities is a preference.

Close with what changed in the *world model*, not what was said. "The regime is already compute-bound,
so the byte-side levers are self-limiting" is the deliverable. "We discussed prefetching" is not.

Then stop. Brainstorming ends at the plan; implementing is a separate ask.

## 6. Failure modes

| Symptom | Do this |
|---|---|
| `SendMessage` can't reach the peer | respawn with the same brief + `TRANSCRIPT SO FAR:`, continue the count |
| Peer goes assistant-y (praise, both-sides summaries) | one corrective: "kill something with a number or add a mechanism". Escalate once, then work with what you get |
| Peer only agrees | ask directly for the weakest part of your position and the option you haven't considered. Two empty rounds → close early and say why |
| Peer stalls on facts it can't have | answer `NEED` in bulk; if it still stalls, state the assumption and tell it to reason under it |
| Every gain quoted regime-free | send the balance again, ask it to re-quote each number with its regime. Do not rank until it does |
| Estimates that stack to +200% | make it name which levers share the premium, then re-derive the waterfall sequentially |
| Model unavailable | say so, name the closest available model, continue |
| User interrupts mid-round | stop, synthesize the completed rounds, mark it partial |

## 7. Cost

One round ≈ one subagent turn on the peer model — `rounds=1` is a legitimate quick second angle, and
3 rounds on `opus` is not free. The peer has no tools and changes nothing; every edit goes through the
host. Under a token-saving mode: one round, cheapest peer that can still do arithmetic.
