# brainstorming

A Claude Code skill that puts **two models on the same problem** and lets them actually talk.

Your session frames the problem and says what it thinks. A second Claude model — `fable`, `opus`,
`sonnet` or `haiku`, picked in the invocation — is spawned as a subagent *peer* and answers from its
own angle. Your session takes that further, the peer takes *that* further, a few rounds, and then
your session writes up what the two of them arrived at.

Two models trained differently have different reach and different blind spots. The value is the
ground neither covers alone — where one is thin, the other is thick. Disagreement shows up and is
welcome when it's honest, but the output is a better answer, not a winner.

The peer keeps its context across rounds (`SendMessage`), so round 3 is a reply to round 2, not a
fresh prompt with a summary glued on top.

```
Brainstorm — peer: fable · 3 rounds · generative · topic: cutting BLE OTA transfer time

── round 1/3 ──────────────────────────────
🤖 fable   NEW: 1. chunk-level dedup across versions  2. defer CRC to idle  3. ship a binary diff
           BUILD: your ring buffer + 1 means a retransmit never re-reads flash
           GAP: none of this matters if the bottleneck is connection interval, not throughput
           OPEN: is the 7.5ms CI actually negotiated, or assumed?
🧠 host    taking 1 · new: 4. negotiate CI before the transfer, not after · differ on 2: CRC is in
           the bootloader path (ota_verify.c:88), can't defer — quoting it next round
```

Two shapes, read off your ask: **generative** ("trovami delle idee per velocizzare X") ends in a
ranked harvest of options; **decision** ("A or B?") ends in a position that survived contact.

## What makes it work

A single subagent call gives you one opinion, usually agreeable. This runs a conversation instead,
with a few rules that keep it honest in both directions:

- The peer is briefed as a thinking partner with its own angle — bring what the framing lacks,
  finish the host's half-formed thoughts, and say plainly when something is wrong and *what breaks*.
  Never agree to be agreeable; never manufacture a dispute to look rigorous.
- **Asymmetric duties.** The peer owes `NEW` — an idea that wasn't on the table. The host owes
  `VERIFIED` — a quoted check of the peer's claims, which costs no subagent turn because the host is
  the only side with a repo. Each quota is guarded against its own filler: `NEW: none` is a legal
  move, and the host's check must target the claim that would change the ranking if false, not a
  safe piece of trivia.
- **Correct with quotes or not at all.** "I checked, you're wrong" is unfalsifiable when only one
  side can see the repo, and the peer is told to leave such points open.
- Closing is **two-keyed**: the peer says `OPEN: none` *and* the host has nothing left to raise.
  A host can't declare agreement on its own, because "we're both just restating" is also what being
  stonewalled looks like.
- **The last word is checked.** The peer commits its own top-3 and its own 5-line synthesis *before*
  seeing the host's, and the host must name everything of the peer's its version leaves out. Whoever
  writes the summary decides what the brainstorm was, and omission is the channel a reader can't see.

## Install

Clone straight into your Claude Code skills folder, then restart Claude Code.

**macOS / Linux**

```bash
git clone https://github.com/GaelicThunder/brainstorming ~/.claude/skills/brainstorming
cp ~/.claude/skills/brainstorming/agents/brainstorm-peer.md ~/.claude/agents/
```

**Windows (PowerShell)**

```powershell
git clone https://github.com/GaelicThunder/brainstorming "$env:USERPROFILE\.claude\skills\brainstorming"
copy "$env:USERPROFILE\.claude\skills\brainstorming\agents\brainstorm-peer.md" "$env:USERPROFILE\.claude\agents\"
```

The second command installs the `brainstorm-peer` agent — a subagent type with **no tools**, so the
peer's no-repo rule is enforced at spawn instead of asked for in a prompt. Without it the skill
still runs, falling back to `general-purpose` on the honour system. Restart Claude Code after
copying so the agent registers.

Prefer keeping the repo elsewhere? Clone it wherever you like and symlink it:

```bash
git clone https://github.com/GaelicThunder/brainstorming ~/code/brainstorming
ln -s ~/code/brainstorming ~/.claude/skills/brainstorming
```

> **Name collision:** if you already have a skill named `brainstorming` (Nori ships one), rename the
> old folder *and* its `name:` frontmatter field first — two skills cannot share a name.

## Use

```
fai brainstorming con fable sul rollback OTA
brainstorming with opus rounds=5 about the caching layer
/brainstorming sonnet 2 giri — schema per la tabella eventi
brainstorming con fable e opus                  ← panel mode, both peers
/brainstorming                                  ← peer auto-picked, topic = current conversation
```

| Argument | Effect |
|---|---|
| `fable` / `opus` / `sonnet` / `haiku` | which model you think with |
| `rounds=N`, `N giri`, `N round` | exchanges — default 3, cap 6 |
| two or more model names | panel mode: each peer also sees the others' turns, verbatim |
| anything else | the topic (omit it and the current conversation is the topic) |

No model named? The skill picks one *different from your session model* — same model means same
blind spots.

## What you get back

Every round is printed as it happens, then a closing block.

**Generative** — the harvest is the deliverable:

- **IDEAS** — every option that came up, one line each, with what it costs, tagged `[host]` `[peer]`
  `[both]`. Nothing silently dropped.
- **BEST BETS** — the 2–3 worth trying, and why these
- **COMBINATIONS** — ideas that unlock each other
- **WILD** — the expensive or strange ones, kept visible, with what would have to be true
- **STILL OPEN / NEXT STEPS / DROPPED** — with reasons attached

The `[both]` tag is the point: ideas that exist only because two models talked — a peer's mechanism
on your idea, or a third thing neither walked in with. They feel obvious by the end, which is
exactly how they get lost.

Early rounds widen (range, not judgement — an option killed in round 1 takes its whole neighbourhood
with it), the last round narrows and ranks.

**Decision** — the position is the deliverable: **ARRIVED AT** (and what changed on the way) /
**STILL OPEN** / **DECISION** / **NEXT STEPS** / **DROPPED**.

Ask for it to be saved and the transcript lands in `brainstorm-<slug>-<date>.md`.

## Panel mode

Name two or more models and each gets its own subagent. Coverage comes from two levers, best used
together: **different models** (different training, different blind spots) and **different lenses** —
each peer given a genuine vantage point to think from. A lens is where you stand, not a side you must
defend, and a preset is three things: a focus, the question it always asks first, and its **declared
blind spot** — what this angle is known to under-weight, stated up front so it can be audited later.

Turns are relayed between peers verbatim, with the lens and the attribution stripped: an idea judged
as "the security peer's" gets answered as a position to be placed rather than a thing to think about.
In the last round the lens is dropped — *"forget the angle you were given, what still stands?"* — each
peer is audited against its declared blind spot, and whatever evaporates goes to DROPPED.

## Cost

One round ≈ one subagent turn on the peer model. Three rounds with `opus` is not free; `rounds=1`
is a perfectly good quick second angle, and panel mode multiplies by the number of peers (it
defaults to 2 rounds for that reason). Under a token-saving mode the skill drops to a single round
on the cheapest peer that still adds something.

## Limits

- The peer sees only what the problem turns on — no repo, no shell, no session history. There's no
  word limit: brainstorming a project *name* sends a few lines, brainstorming a state machine sends
  the whole state machine. What never gets sent is the session itself — transcript, summaries,
  "background on the project", file dumps. A peer reading 300k of unrelated context stops thinking
  and starts summarizing. Anything genuinely missing it asks for under `NEED:` and gets next round.
- The peer thinks, it doesn't touch files. Every change goes through your session.
- Brainstorming stops at the plan. Implementation is a separate ask.

## License

MIT — see [LICENSE](LICENSE).
