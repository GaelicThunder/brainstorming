# brainstorming

A Claude Code skill that runs a **real debate between two models** over a hard technical problem —
and, more importantly, decides what the debate is allowed to be made of.

Your session pulls the actual numbers out of the system, collects the project's own graveyard of
ideas that already died and why, names what's scarce this week, states a position, and hands all of
it to a second Claude model — `fable`, `opus`, `sonnet` or `haiku` — spawned as a subagent peer with
**no tools**. Then they go at it for a few rounds: the peer kills with arithmetic or adds a
mechanism, your session verifies the load-bearing claims against the actual code, and both keep a
ledger of what's alive and what's dead.

What comes out is not a summary of a conversation. It is a ranked list where every surviving idea
carries an expected gain, **the regime that gain holds in**, an effort in real units, and a
**kill-check** — the measurement that would prove it wrong — declared before anything runs.

## The method

Most model-vs-model brainstorms fail the same three ways. Each rule here exists because of one.

**1. Anchors before opinions.** Nobody proposes anything until the real numbers are on the table:
where the time actually goes as percentages summing to 100, the physical ceiling, and which numbers
are measured versus derived. Ideas then die by arithmetic instead of by taste — and an idea that
beats the ceiling is dead before anyone argues.

> If you don't have the balance, get it before you spawn. One 10-minute profiling run in the
> production configuration beats a whole round of debate about which bottleneck is real.

**2. Every gain names its regime.** "+58%" is not an estimate. "+58% while disk is 63% of the wall"
is. This one rule catches the failure that ruins otherwise-good plans: a lever whose gain was quoted
in a regime that the *previous* lever already moved the system out of. Levers that improve one axis
until a different axis dominates are self-limiting, and the skill makes them say so.

**3. The graveyard is part of the brief.** The peer cannot see your repo, your old reports or last
month's dead ends — so without the list it will re-propose them with total confidence, and it will
sound good. Negative results (*"policy family closed: LFU < RANDOM, FIFO ≈ LRU"*) are worth more in
a brief than any idea, because they foreclose a whole neighbourhood.

Then two rules about who is allowed to be sure of what:

- **Quote or it didn't happen.** The peer has no tools, so the host is the only side that can see
  the code — which makes "I checked, you're wrong" unfalsifiable. Corrections carry ≤15 verbatim
  lines with `path:line`, or they don't count, and the peer is briefed to leave the point open.
- **Verify before you plan.** A two-model debate produces confident, wrong plumbing: *"it's one line
  at X"*, *"that flag is already passed"*, *"the data isn't there"*. Expect about one such claim in
  four to be wrong. The skill has a mandatory pass that checks each one and reorders the plan against
  the corrections — it routinely finds prerequisites already satisfied (zero work), one-liners that
  don't compile as described, and checks that need no scarce resource because the data has been on
  disk for a week.

## What a round looks like

```
Brainstorm — peer: fable · 3 rounds · topic: MoE decode speed, weights streamed from disk

── round 2/3 ─────────────────────────────
🤖 fable  KILL your F3 (split the read across 4 threads): the block layer already splits the 18.87MB
          pread into ~151 128KB requests — same LBAs, same queue. Bandwidth axis is closed.
          NEW: 4. int2 bit-plane on misses — damage/byte 4.2× better than the route-swap in prod
          CHECK: NRMSE→loss is not proven, and it's the load-bearing step of 4
🧠 host   VERIFIED: colibri.c:5593 `g_draft=0;` — nctx is read 6 lines later, so the "one-line fix"
          does not compile as stated · taking 4 · #4 excludes CACHE_ROUTE (that combo measured −30.8)
```

## What you get at the end

A dated record (`BRAINSTORM-<slug>-<date>.md`), and a plan when there are ordered actions, with one
line at the top saying which of the two is authoritative:

| Block | Contents |
|---|---|
| **ANCHORS** | the numbers everything was argued against, with sources |
| **DEAD** | idea · who killed it · arithmetic cause of death — publish this, it's an asset |
| **ALIVE** | ranked by **gain × P(survives) / effort**, each with mechanism, gain @regime, effort in real units, pre-declared kill-check, and a `[host]` `[peer]` `[both]` tag |
| **COMPOSITION** | which levers share a premium (don't sum them), which are mutually exclusive |
| **WATERFALL** | the numeric chain to the endpoint, and where the regime flips — past that point the ranking is void |
| **TREE** | what to run, ordered. The root is *not* the top lever: it's the cheapest measurement that decides between two rankings that can't both be right |
| **FREE NOW** | what runs without the scarce resource, or against data already on disk |
| **OPEN** | what neither model settled, both views in one line each |

A ranking without probabilities is a preference, so all three factors get written down.

## Install

```bash
git clone https://github.com/GaelicThunder/brainstorming ~/.claude/skills/brainstorming
mkdir -p ~/.claude/agents && cp ~/.claude/skills/brainstorming/agents/brainstorm-peer.md ~/.claude/agents/
```

**Windows (PowerShell)**

```powershell
git clone https://github.com/GaelicThunder/brainstorming "$env:USERPROFILE\.claude\skills\brainstorming"
mkdir "$env:USERPROFILE\.claude\agents" -Force
copy "$env:USERPROFILE\.claude\skills\brainstorming\agents\brainstorm-peer.md" "$env:USERPROFILE\.claude\agents\"
```

Restart Claude Code so the agent registers. The second command is what enforces the peer's no-tools
rule at spawn instead of asking for it in a prompt; without it the skill still runs, falling back to
`general-purpose` on the honour system.

Prefer the repo elsewhere? Clone anywhere and symlink it into `~/.claude/skills/`.

> **Name collision:** if you already have a skill named `brainstorming`, rename the other folder
> *and* its `name:` frontmatter field — two skills can't share a name.

## Use

```
fai brainstorming con fable sulla velocità di decode
brainstorming with opus rounds=5 about the caching layer
/brainstorming sonnet 2 giri — schema per la tabella eventi
/brainstorming                       ← peer auto-picked, topic = current conversation
```

| Argument | Effect |
|---|---|
| `fable` / `opus` / `sonnet` / `haiku` | which model you argue with |
| `rounds=N`, `N giri`, `N round` | rounds — default 3, cap 5 |
| anything else | the topic (omit it and the current conversation is the topic) |

No model named? The skill picks one *different from your session model* — same model means same blind
spots.

## Scope and cost

One round ≈ one subagent turn on the peer model. `rounds=1` is a perfectly good quick second angle;
3 rounds on `opus` is not free.

This skill is for problems where **arithmetic can referee**: performance, capacity, cost, protocol
and architecture choices with numbers attached. It is a poor fit for taste questions — naming, tone,
visual direction — where there are no anchors to argue against and the whole apparatus of regimes and
kill-checks is dead weight.

The peer thinks and never touches files; every change goes through your session. Brainstorming stops
at the plan — implementing is a separate ask.

## License

MIT — see [LICENSE](LICENSE).
