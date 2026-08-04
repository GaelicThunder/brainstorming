# brainstorming

A Claude Code skill that turns "let me think about this" into an actual argument between **two
models**.

Your session frames the problem, a second Claude model — `fable`, `opus`, `sonnet` or `haiku`,
picked in the invocation — is spawned as a subagent *peer*, and the two go back and forth for a few
rounds: position, objection, counter, alternative. Then your session writes the synthesis.

The peer keeps its context across rounds (`SendMessage`), so round 3 is a reply to round 2, not a
fresh prompt with a summary glued on top.

```
Brainstorm — peer: fable · 3 rounds · topic: BLE OTA rollback strategy

── round 1/3 ──────────────────────────────
🤖 fable   POSITION: dual-bank is the wrong default at 512K flash
           OBJECTION: your rollback assumes the bootloader survives a brownout mid-swap
           ALTERNATIVE: A/B with a validity byte written last, no swap at all
🧠 host    concede the brownout window · reject "no swap": OTA server can't address bank B
           new: validity byte + watchdog-armed first boot
```

## Why not just ask a subagent once

A single subagent call gives you one opinion, usually agreeable. The loop here is built to
disagree: the peer is briefed as a colleague who has been handed a proposal and doesn't buy all of
it, must name the strongest objection every turn, must propose something your framing didn't
contain, and is banned from praise and both-sides hedging. Your session is required to push back at
least once per round instead of nodding along — and to check the peer's factual claims against the
actual repo, because a bigger model is not evidence.

## Install

Clone straight into your Claude Code skills folder, then restart Claude Code.

**macOS / Linux**

```bash
git clone https://github.com/GaelicThunder/brainstorming ~/.claude/skills/brainstorming
```

**Windows (PowerShell)**

```powershell
git clone https://github.com/GaelicThunder/brainstorming "$env:USERPROFILE\.claude\skills\brainstorming"
```

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
| `fable` / `opus` / `sonnet` / `haiku` | which model you argue with |
| `rounds=N`, `N giri`, `N round` | exchanges — default 3, cap 6 |
| two or more model names | panel mode: every peer also argues against the others |
| anything else | the topic (omit it and the current conversation is the topic) |

No model named? The skill picks one *different from your session model* — you want a second
perspective, not an echo.

## What you get back

Every round is printed as it happens, then a closing block:

- **AGREED** — decisions both sides back
- **CONTESTED** — open disagreement, both positions stated
- **DECISION** — what your session would do, and why the open points don't block it
- **NEXT STEPS** — concrete and ordered
- **DROPPED** — ideas killed, with the reason, so they don't come back next week

Ask for it to be saved and the transcript lands in `brainstorm-<slug>-<date>.md`.

## Cost

One round ≈ one subagent turn on the peer model. Three rounds with `opus` is not free; `rounds=1`
is a perfectly good quick second opinion, and panel mode multiplies by the number of peers (it
defaults to 2 rounds for that reason). Under a token-saving mode the skill drops to a single round
on the cheapest peer that still argues back.

## Limits

- The peer sees **only** what your session puts in the brief — no repo, no shell, no session
  history. Gather facts first; the skill answers the peer's `NEED:` list on the next turn.
- The peer thinks, it doesn't touch files. Every change goes through your session.
- Brainstorming stops at the plan. Implementation is a separate ask.

## License

MIT — see [LICENSE](LICENSE).
