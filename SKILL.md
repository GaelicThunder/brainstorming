---
name: brainstorming
description: "Live multi-round brainstorm between this session and a second Claude model run as a subagent peer — real back-and-forth, explicit disagreement, then a synthesis. The peer model is chosen in the invocation: fable, opus, sonnet or haiku. Use when the user says 'fai brainstorming con fable/opus', 'brainstorming with sonnet', 'discutiamone con un altro modello', 'sparring', 'second opinion from another model', 'panel', or invokes /brainstorming."
argument-hint: "[con|with] [fable|opus|sonnet|haiku] [rounds=N] <topic>"
---

# Brainstorming — two models, one argument

Run a **real conversation** between this session (the *host*) and a second Claude model spawned as a
subagent (the *peer*): the host frames the problem, the peer answers, the host pushes back, the peer
answers again — N rounds — then the host writes a synthesis.

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
`fable`; session on Sonnet → `opus`). State the pick in the header line.
**No topic** → the topic is whatever the conversation is currently about. Only if there is nothing to
work with, ask in one line.

Open with exactly one header line, then start:

`Brainstorm — peer: fable · 3 rounds · topic: BLE OTA rollback strategy`

## 2. Round 0 — frame it (host)

Write, ≤150 words total, no preamble:

- **PROBLEM** — one sentence.
- **CONSTRAINTS** — the real ones (hardware, deadline, existing code, user's stated preferences).
- **MY POSITION v0** — take an actual stance, not a menu.
- **UNSURE ABOUT** — the 1–2 things you genuinely can't settle alone.

Gather the facts *before* spawning: read the relevant files, run the command, check the versions.
The peer has no access to your session — but that does **not** mean you ship it your session.

### Brief discipline — send the slice, not the project

The brief is a scalpel. Hard rules:

- **Never** paste the conversation transcript, a session summary, whole files, directory listings,
  or "background on the project" the argument doesn't turn on.
- **Do** send: the problem, the constraints that actually bind, your position, and the exact thing
  under discussion — one function, one struct, one error, one config block.
- **Budget**: ≤300 words of prose, plus at most one excerpt of ≤40 lines. Over that, you're
  briefing, not arguing. Cut until only the contested part is left.
- **Missing context is handled by the loop, not by pre-loading**: the peer lists what it lacks
  under `NEED:` and you send exactly that on the next turn. One fact, not the file it came from.
- **Redact** before sending: keys, tokens, `.env` values, customer or client names, absolute paths
  that expose them. Rename to `<CLIENT>`, `<TOKEN>` — the argument never depends on the real value.

Why: a peer buried in context stops arguing and starts summarizing, every extra line is paid for on
every round, and the whole point is a view *not* anchored to everything you already believe.

## 3. Spawn the peer

One `Agent` call:

- `model`: the parsed model
- `subagent_type`: `general-purpose`
- `run_in_background`: `false` (you need the reply before continuing)
- `description`: `brainstorm peer`

The spawn result ends with `agentId: a…` — **record it**. That id is what you send to; the
description name is only a fallback.

Prompt = **PEER BRIEF**:

```
You are a peer in a design brainstorm, not an assistant. Your counterpart is another Claude
session working on this problem. Behave like a senior colleague who has been handed a proposal
and disagrees with part of it.

RULES
- Open with your position. No praise, no restating my framing, no "great question".
- Every turn: name the single strongest objection to the current proposal, and say what breaks
  concretely (which input, which state, which failure mode) — not "it may be risky".
- Propose at least one option the framing did not contain.
- When you agree, say "agreed" once and spend the words on what is still unresolved.
- Do not hedge into both-sides answers. If asked to pick, pick.
- You get the contested slice only, deliberately — no repo, no history. If a fact is missing, list
  it under NEED and I will send that fact (not the whole file) next turn. Argue with what you have.
- ≤200 words per turn. Bullets. No closing summary, no offer to help further.

REPLY SHAPE
POSITION: ...
OBJECTION: ...
ALTERNATIVE: ...
NEED: ... (omit if none)

--- BRIEF ---
<Round 0 block: PROBLEM / CONSTRAINTS / MY POSITION v0 / UNSURE ABOUT>
```

## 4. The loop (this is the skill)

For each round after the first:

1. **Read** the peer's turn. Check its factual claims against the repo/machine when checkable —
   a bigger model is not evidence. If it is wrong about a fact, say so with the file or command output.
2. **Answer as yourself**, in-thread: concede what actually lands, refute what doesn't *and why*,
   add what the peer missed, then ask the one question that unblocks the decision.
   At least one substantive pushback per round — never a round of pure agreement.
3. **Send** it with `SendMessage` → `to`: the recorded `agentId`, `message`: your turn + any facts
   it asked for under NEED.
4. Repeat.

⚠️ Unlike the first spawn, `SendMessage` resumes the peer **in the background**: the tool returns
immediately (`resumed from transcript in the background`) and the actual reply arrives later as a
task notification. So: do not spawn a second peer because "it didn't answer", do not write the
round until the notification lands, and never invent the reply. The wait is a good moment to fetch
the facts it asked for under NEED — have them ready for the next send.

### When to stop

- **Converged** — a round where *both* sides only restate. Say `converged`, go to synthesis.
- **Cap** — the requested round count, or 6.
- **User says stop** — immediately.

Do **not** stop early because you think you already know how the peer's alternative plays out. You
are the side invested in your own position; "I understand the trade-off" is not the same as the
argument being finished, and the user is watching the exchange to see whether the peer concedes or
holds. Let it answer.

At the cap with the argument still live, don't just end — say so in one line and offer one more
round (`still open on X — another round?`). Extra rounds cost money; that call is the user's.

Show every round to the user as it happens, compact:

```
── round 2/3 ──────────────────────────────
🤖 fable   OBJECTION: ... / ALTERNATIVE: ...
🧠 host    concede X · reject Y because <fact> · new: Z
```

Never dump the peer's raw text if it rambles — compress to its argument, keep its wording for the
sharp parts. Never invent a peer turn: if the call failed, say it failed.

## 5. Synthesis (host, always last)

```
AGREED       — decisions both sides back
CONTESTED    — open disagreement, with both positions in one line each
DECISION     — what you would do, and why the contested points don't block it
NEXT STEPS   — concrete, ordered
DROPPED      — ideas considered and killed, with the reason (so they don't come back)
```

Then stop. Brainstorming ends at the plan — do not start implementing unless asked.

## 6. Panel mode (2+ peers)

Same loop, one `Agent` per model, distinct `description` names (`peer-fable`, `peer-opus`). Spawn
them in a single message so they run concurrently. Each round: relay the *other* peers' last turns
inside your `SendMessage` (they cannot see each other), and make each defend its position against
them. Synthesis names who held which line.

Panel costs a round per peer per round — default to 2 rounds when there are 2+ peers.

## 7. Failure modes

| Symptom | Do this |
|---|---|
| `SendMessage` can't reach the peer (name taken over, agent gone) | respawn with the same brief + `TRANSCRIPT SO FAR:` appended, continue the count |
| Peer returns an assistant-y answer ("great idea, here are 5 options") | one corrective send: "You are the peer, not the assistant. Give me POSITION / OBJECTION and pick one." Escalate once, then work with what you get |
| Peer keeps asking for facts it can't have | answer NEED items in bulk in the next send; if it still stalls, state the assumption and tell it to argue under that assumption |
| Model unavailable | say so plainly, name the closest available model, continue with it |
| User interrupts mid-round | stop the loop, produce the synthesis from the rounds that completed, mark it partial |

## 8. Cost & hygiene

- One round ≈ one subagent turn on the peer model. `rounds=1` is a legitimate quick second opinion.
- The peer is for thinking, not doing: it does not edit files. Any change goes through the host.
- If the user asks to keep it, write the full transcript to `brainstorm-<slug>-<YYYY-MM-DD>.md` in the
  working directory (or `~/Documents/notes/` when there is no project).
- Under `token-budget` mode: 1 round, cheapest peer that still disagrees usefully (`sonnet`), no panel.
