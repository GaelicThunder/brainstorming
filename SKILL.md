---
name: brainstorming
description: "Live multi-round brainstorm between this session and a second Claude model run as a subagent peer — two different vantage points working the same problem, building on each other across rounds, then a shared synthesis. The peer model is chosen in the invocation: fable, opus, sonnet or haiku. Use when the user says 'fai brainstorming con fable/opus', 'brainstorming with sonnet', 'discutiamone con un altro modello', 'ragioniamoci insieme', 'second opinion from another model', 'panel', or invokes /brainstorming."
argument-hint: "[con|with] [fable|opus|sonnet|haiku] [rounds=N] <topic>"
---

# Brainstorming — two models, one problem

Run a **real conversation** between this session (the *host*) and a second Claude model spawned as a
subagent (the *peer*): the host frames the problem and says what it thinks, the peer answers from its
own angle, the host takes that further, the peer takes *that* further — N rounds — then the host
writes what the two of them arrived at.

Two models trained differently have different reach and different blind spots. The point is the
ground neither covers alone: where one is thin, the other is thick. Disagreement happens and is
welcome when it's honest — but the goal is a better answer, not a winner.

This is not "ask a subagent and paste the answer". Every round is a reply to the previous turn,
carried by `SendMessage` so the peer keeps its full context.

## 1. Parse the invocation

| Input | Meaning |
|---|---|
| `fable` \| `opus` \| `sonnet` \| `haiku` (after `con` / `with`, or anywhere in the request) | peer model |
| `rounds=N`, `N giri`, `N round`, `N rounds` | exchanges (default **3**, hard cap **6**) |
| two or more model names | **panel mode**, see §6 |
| everything else | the topic |

**No model named** → do not ask. Pick a model *different from the session's own* (session on Opus →
`fable`; session on Sonnet → `opus`). Same model, same blind spots — the whole value is the
difference. State the pick in the header line.
**No topic** → the topic is whatever the conversation is currently about. Only if there is nothing to
work with, ask in one line.

### Which shape is this?

Two shapes, same loop, different framing and different output. Read it off the ask:

| The user says | Shape | You produce |
|---|---|---|
| "trovami delle idee per…", "come potremmo…", "find a way to speed up X" | **generative** | a harvest of options, ranked, most of them new |
| "A or B?", "is this design right?", "should we…" | **decision** | a position that survived contact, with what's still open |

Generative is the default when the ask names a *goal* rather than a *choice*. When it's genuinely
both ("here are two ideas, are they any good, what else is there?"), run generative — a decision
made from two options nobody stress-tested is the weaker outcome.

Open with exactly one header line, then start:

`Brainstorm — peer: fable · 3 rounds · generative · topic: cutting BLE OTA transfer time`

## 2. Round 0 — frame it (host)

Write, ≤150 words total, no preamble.

**Generative:**

- **GOAL** — what would be better, and by how much if there's a number.
- **CONSTRAINTS** — the real ones (hardware, deadline, existing code, user's preferences).
- **SEEDS** — the 2–3 ideas you already have, one line each, *all* of them. Seeds are fences, not
  anchors: they mark the territory round-1 `NEW` must fall outside of. Anything adjacent to a seed
  is a `BUILD`, not a `NEW`.
- **ALREADY RULED OUT** — with the reason, so the round isn't spent re-proposing them.

Send every seed. Withholding one to see whether the peer independently arrives at it does not buy
what it looks like it buys: your seeds come from one mind and are correlated, so the ones you *do*
send leak the direction of the one you don't — the "independent convergence" isn't independent, and
you paid a full round of the peer's widest thinking to measure nothing. Fences get the range for
free, because the NEW/BUILD split already carries them.

Equally, don't write soft firewalls ("come up with your own ideas *before* reading the seeds below").
Everything in the context attends; an LLM cannot un-read the second half of its prompt. The choice is
binary — genuinely blind, or full reveal with fences. Take fences.

**Decision:**

- **PROBLEM** — one sentence.
- **CONSTRAINTS** — as above.
- **MY POSITION v0** — take an actual stance, not a menu. A menu gives the peer nothing to push on.
- **UNSURE ABOUT** — the 1–2 things you genuinely can't settle alone. Be honest here; this is where
  the peer earns its cost.

Gather the facts *before* spawning: read the relevant files, run the command, check the versions.
The peer has no access to your session — but that does **not** mean you ship it your session.

### Brief discipline — send what the argument turns on

There is no word limit. There is a relevance test: **would the peer's answer change if this were
missing?** No → leave it out. Yes → send it, however long it is.

That test scales on its own:

| Brainstorm | Brief |
|---|---|
| Name for the project, artistic direction, tone of a UI | a few lines — what it is, who it's for, the names/directions already rejected. Nothing else. Do **not** attach the codebase. |
| Which of two APIs to expose, where a module boundary goes | the interface, the two call sites that hurt, the constraint that decides it |
| Architecture, a subtle bug, a protocol choice | as much real material as it genuinely turns on — the whole state machine, the full struct, the actual log. Do not amputate the thing being discussed. |

Regardless of size, never ship the peer the *session*: no conversation transcript, no summary of
what you and the user did earlier, no "background on the project" tour, no file dumps to save
yourself a second round. A peer reading 300k of unrelated context stops thinking and starts
summarizing.

Missing pieces are handled by the loop, not by pre-loading: the peer lists what it lacks under
`NEED:` and gets exactly that next turn. Two cheap rounds beat one bloated brief.

Strip secrets on the way out — keys, tokens, `.env` values, client names. The thinking never
depends on the real value.

## 3. Spawn the peer

One `Agent` call:

- `model`: the parsed model
- `subagent_type`: `brainstorm-peer` — ships in `agents/brainstorm-peer.md`, install to
  `~/.claude/agents/`. It has **no tools**, deliberately: a peer that reads the repo first inherits
  the repo's framing and stops producing the angle you didn't have. Enforced at spawn because that
  is the only layer where tool access is actually decided — a prompt asking a `*`-tooled agent not
  to read files is a request, not a boundary.
  *Not registered yet (needs a Claude Code restart)?* Fall back to `general-purpose`, say so in the
  header line, and keep the no-repo rule in the brief — knowing it's now honour-system.
- `run_in_background`: `false` (you need the reply before continuing)
- `description`: `brainstorm peer`

The spawn result ends with `agentId: a…` — **record it**. That id is what you send to; the
description name is only a fallback.

Prompt = **PEER BRIEF**:

```
You are a thinking partner in a brainstorm — not an assistant, not an opponent. Another Claude
session is working this problem and brought it to you because you know things it doesn't and you
come at it from a different angle. Different training, different blind spots. What we're after is
the ground neither of us reaches alone.

HOW TO WORK
- Open with what you actually think. No praise, no restating my framing back to me.
- Add, don't just react: bring the angle, the precedent, the failure mode you've seen, the option
  that isn't in the framing. Where I'm thin, be thick.
- Build on my ideas as readily as you replace them — "yes, and that unlocks X" is worth as much as
  "no, because Y". Take my half-formed thought and finish it.
- When you disagree, say it plainly and concretely — which input, which state, which failure mode,
  not "it may be risky". Honest disagreement is how we find the weak part.
- Never agree to be agreeable, and never manufacture a dispute to look rigorous. Both waste a round.
- Put something on the table each turn that wasn't there before — an option, a mechanism, a
  reframing. A turn that only judges my ideas is half a turn. In round 1, `NEW` must fall outside the
  territory my SEEDS already cover; improvements to a seed are `BUILD`, which is worth just as much.
- When you're genuinely out of new ideas, write `NEW: none` and say so. That is a legal move and a
  real contribution — it's one of the signals that ends the brainstorm. Padding to satisfy the rule
  is the failure the rule exists to prevent.
- Flag your own riskiest claim each turn under CHECK: the one that, if false, changes the ranking.
  Nominate it yourself; don't leave me to pick the convenient one to verify.
- You get what the problem turns on, deliberately — no repo, no session history, no tools. If a fact
  is genuinely missing, list it under NEED — and make it *conditional* where you can ("NEED X; if
  yes, idea 3 gets stronger; if no, drop it"), so a missing fact forks your thinking instead of
  blocking the round.
- Unquoted claims about code are not settled facts. If I say "I checked, you're wrong" without
  quoting the file or output, leave the point in OPEN and say why.
- <=200 words per turn. Bullets. No closing summary, no offer to help further.

REPLY SHAPE
THINKING: where you land, and why
NEW:      ideas you're putting on the table this turn, terse, numbered (or "none" — a legal move)
BUILD:    what you'd add to my ideas, or where you'd take them next
GAP:      what the framing misses, or what concretely worries you
CHECK:    your own claim that would change the ranking if it turned out false
OPEN:     what's still unsettled between us, one line each (or "none" — only when true)
NEED:     ... (omit if none)

--- BRIEF ---
<Round 0 block: PROBLEM / CONSTRAINTS / MY POSITION v0 / UNSURE ABOUT>
```

## 4. The loop (this is the skill)

For each round after the first:

1. **Read** the peer's turn properly. Take what's good — including the parts that make your position
   worse. If it hands you a better frame, adopt it and say so; that's the skill working, not you
   losing.
   Check its factual claims against the repo/machine when checkable — a bigger model is not evidence.
   **Quote or it didn't happen:** a correction carries ≤15 verbatim lines of the file or command
   output, with path and line numbers. "I checked the repo, you're wrong" is not a correction — you
   are the only party who can see the evidence, so a bare assertion from you is unfalsifiable, and
   the peer is briefed to leave it OPEN. If it isn't worth quoting, say "unverified" and argue it on
   the merits instead.
2. **Answer as yourself**: what of theirs you're taking and where it leads, what you'd add on top,
   where you still see it differently *and why*, then the one question that would move things
   forward. Put a new idea of your own on the table too — same rule as the peer's, `none` included.
   A round of pure agreement is a wasted round — if you truly have nothing to add, say so
   and close early rather than padding.

   **Every host turn carries a `VERIFIED` block.** The peer brings range; you bring ground truth,
   and you are the only one who can — it has no tools, while checking a claim costs you no subagent
   turn at all. So each turn either quotes a check of one of its claims, or says plainly that
   nothing this round was checkable. Aim the check at the peer's `CHECK:` nomination, or at whatever
   claim would change the ranking if false — a quota invites cheap compliance, and verifying some
   safe piece of trivia while the load-bearing claim goes untouched is worse than skipping it, since
   it buys the round a look of rigor it didn't earn.

   Print the density dial in the send (`density: wide` / `density: tight — last round`) so the peer
   never has to infer which phase it's in.

   **Keep a running ideas list** as you go: every option either side raises, tagged `host`, `peer`,
   or **`both`** for the ones that only exist because of the exchange — a peer mechanism on a host
   idea, or a third thing that neither of you walked in with. Those are what the skill is for; they
   are also the easiest to lose, because by the synthesis they feel obvious. Write them down when
   they appear.
3. **Send** it with `SendMessage` → `to`: the recorded `agentId`, `message`: your turn + any facts
   it asked for under NEED.
4. Repeat.

⚠️ Unlike the first spawn, `SendMessage` resumes the peer **in the background**: the tool returns
immediately (`resumed from transcript in the background`) and the actual reply arrives later as a
task notification. So: do not spawn a second peer because "it didn't answer", do not write the
round until the notification lands, and never invent the reply. The wait is a good moment to fetch
the facts it asked for under NEED — have them ready for the next send.

### Widen, then narrow (generative)

Rounds before the last are for **range**: more mechanisms, weirder angles, ideas that are probably
too expensive. Note the cost of an idea, don't kill it — an option judged in round 1 takes its whole
neighbourhood with it, including the cheap variant nobody had thought of yet.

The **last round** flips: say so explicitly in the send (`last round — let's rank what we have`), and
spend it on which ideas actually pay, what each would cost, and what would have to be true for the
expensive ones to be worth it. Ideas that die here die with a reason attached.

Ask for two blocks in that final turn, and ask for them **before** you show yours:

- **TOP 3** — the ideas the peer would actually ship, ordered, with what each costs or breaks.
- **SYNTHESIS, 5 lines** — what the brainstorm produced, in the peer's own words.

You write the synthesis, so you decide what the brainstorm *was*. Getting the peer's version
committed first is the only check on that, and it's free.

Two rounds means one wide and one narrow. If the user asked for ideas and gave you three rounds,
that's two wide.

### When to stop

Closing is **two-keyed**: the peer says `OPEN: none`, *and* you have nothing left to put to it. One
key alone is not agreement. In generative mode add a third condition — a round where both sides
wrote `NEW: none`. Ideas drying up is the real signal there; agreement is not.

This is why `NEW: none` has to be a legal move: a mandate to produce a new idea every turn makes the
third key unreachable, and a peer that must invent something will invent filler — manufacturing
exactly the noise the key was meant to detect.

- **Closed** — peer's `OPEN` is empty and so is yours. Say `closed`, go to synthesis.
- **Cap** — the requested round count, or 6. Anything still in `OPEN` carries into the synthesis.
- **User says stop** — immediately.

Never infer that you're done from the shape of a round. "Both sides are restating" is also what it
looks like when a peer couldn't move you — because you dismissed a point without evidence and it had
no second move. Honest exhaustion and being stonewalled leave an identical trace, and you don't get
to be the judge of which one happened. Hence the peer's key: it is the only party that can say its
point has been answered.

Do **not** close early because you think you already know how the peer's idea plays out. You are the
side invested in your own position; "I understand the trade-off" is not the same as having heard the
answer, and the user is watching to see where the two of you actually land. Let it answer.

**Evidence escalation.** When an `OPEN` item is a plain claim about what the code does, stop arguing
and paste the disputed lines verbatim into the next send. That is the peer's only evidence channel —
it has no tools by design — and it costs one round the two-keyed stop already prices in.

At the cap with things still open, don't just end — say so in one line and offer one more round
(`still open on X — another round?`). Extra rounds cost money; that call is the user's.

Show every round to the user as it happens, compact:

```
── round 2/3 ──────────────────────────────
🤖 fable   NEW: 4. chunk-level dedup  5. defer CRC to idle
           BUILD: your ring buffer + 4 means the retransmit never re-reads flash
           GAP: none of this helps if the bottleneck is connection interval
🧠 host    taking 4 · new: 6. negotiate 7.5ms CI first · still differ on 5 because <fact>
```

Never dump the peer's raw text if it rambles — compress to the substance, keep its wording for the
sharp parts. Never invent a peer turn: if the call failed, say it failed.

## 5. Synthesis (host, always last)

Write what the two of you arrived at, not a scoreboard.

**Generative — the harvest is the deliverable.** Every idea that came up, not just the winners:

```
IDEAS
  For each — one line of what it is, one of what it costs or needs, tag [host] [peer] [both].
  Order by what you'd try first. Do not silently drop the ones you don't like: an idea you
  rejected belongs in DROPPED with a reason, never in a gap.

BEST BETS    — the 2–3 you'd actually try, and what makes them best here
COMBINATIONS — ideas that get better together, or unlock each other
WILD         — the expensive/weird ones worth keeping visible, with what would have to be true
STILL OPEN   — questions that would change the ranking if answered
NEXT STEPS   — concrete, ordered
DROPPED      — considered and set aside, with the reason (so they don't come back next week)
```

The `[both]` tag matters most: those are the ideas that exist only because two models talked — a
peer's mechanism on your idea, or a third thing neither of you walked in with. They read as obvious
by the end, which is exactly why they get lost. Name them.

**Decision — the position is the deliverable:**

```
ARRIVED AT   — where you landed together, including what changed on the way and whose input moved it
STILL OPEN   — what neither settled, with both views in one line each
DECISION     — what you'd do, and why the open points don't block it
NEXT STEPS   — concrete, ordered
DROPPED      — ideas considered and set aside, with the reason
```

**Print the peer's own synthesis next to yours, and name every item of its five lines that yours
leaves out.** Not "we mostly agreed" — the specific omissions. You hold the pen, so the way your
position wins is not by contradicting the peer, it's by quietly not mentioning things: a reader can
compare two stated emphases, but they cannot see what was never written down. The omission list is
the only part of this the reader can't reconstruct for themselves.

Credit honestly in either shape: if the peer's angle is what made the answer, say so plainly. If you
held your position under pressure, say that too. The user needs to know which parts survived contact
and which were never tested.

Then stop. Brainstorming ends at the plan — do not start implementing unless asked.

## 6. Panel mode (2+ peers)

Same loop, one `Agent` per model, distinct `description` names (`peer-fable`, `peer-opus`). Spawn
them in a single message so they run concurrently.

Two ways to get real coverage out of a panel, best used together:

- **Different models** — different training, different blind spots. This is what the user picks
  when they name two models.
- **Different lenses** — give each peer a genuine vantage point to think from. Not assigned advocacy:
  a lens is where you stand, not a side you must defend. A preset is three things, and the third is
  what makes it auditable:
  - **focus** — what it cares about (security / maintenance / cost / who operates this at 3am)
  - **characteristic question** — what it always asks first ("what happens when this is attacked?",
    "who fixes this at 3am with no context?")
  - **declared blind spot** — what this angle is known to under-weight, stated up front

  Harvest what each lens *finds*; distrust how each ranks its own findings, since a lens always
  thinks its own concern is the biggest one.

Relay each peer's turns to the others **verbatim** inside your `SendMessage` — turns are ≤200 words,
so paraphrasing saves nothing and reintroduces exactly the filtering this skill avoids everywhere
else. Strip the attribution and the lens when you relay: an idea judged as "the security peer's"
gets answered as a position to be placed rather than a thing to think about. They can't see each
other; you are their only channel, so don't editorialize on it.

In the last send, drop the lens: *"forget the angle you were given — what still stands?"* Audit each
peer's turns against its own declared blind spot while you're there. What survives de-roling is real;
what evaporates was scaffolding, and belongs in DROPPED.

Panel costs a round per peer per round — default to 2 rounds when there are 2+ peers.

## 7. Failure modes

| Symptom | Do this |
|---|---|
| `SendMessage` can't reach the peer (name taken over, agent gone) | respawn with the same brief + `TRANSCRIPT SO FAR:` appended, continue the count |
| Peer goes assistant-y ("great idea, here are 5 options") | one corrective send: "You're the other half of this, not a summarizer. Where do *you* land, and what would you add?" Escalate once, then work with what you get |
| Peer only agrees, round after round | it's not earning its cost. Ask it directly for the part of your position it likes least, and for the option you haven't considered. If two rounds pass with nothing new, close early and say why |
| Peer keeps asking for facts it can't have | answer NEED items in bulk next send; if it still stalls, state the assumption and tell it to reason under that assumption |
| Model unavailable | say so plainly, name the closest available model, continue with it |
| User interrupts mid-round | stop the loop, synthesize from the rounds that completed, mark it partial |

## 8. Cost & hygiene

- One round ≈ one subagent turn on the peer model. `rounds=1` is a legitimate quick second angle.
- The peer is for thinking, not doing: it has no tools and edits nothing. Any change goes through
  the host.
- If the user asks to keep it, write the full transcript to `brainstorm-<slug>-<YYYY-MM-DD>.md` in the
  working directory (or `~/Documents/notes/` when there is no project).
- Under `token-budget` mode: 1 round, cheapest peer that still adds something (`sonnet`), no panel.
