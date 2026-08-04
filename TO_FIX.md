# Known issues

- **`brainstorm-peer` tool ban is unverified.** The agent ships with `tools: []`, intended to mean
  "no tools at all". Not yet confirmed empirically — agent definitions only register on a Claude
  Code restart, so the first run after install should check that the peer really cannot Read/Grep.
  If an empty list is parsed as "inherit everything", the ban silently becomes honour-system again
  and the fallback path is what's actually running.

- **Peer names are not guaranteed unique.** `SendMessage` targets a teammate by name and the latest
  spawn wins. If another agent takes the name mid-brainstorm, the send lands in the wrong context.
  *Workaround:* the skill records the `agentId` from the spawn result and falls back to it; if the
  peer is gone entirely it respawns with the transcript appended.

- **Peers drift back into assistant mode** on long rounds — praise, both-sides summaries, offers to
  help. The brief forbids it and the skill sends one corrective, but smaller models relapse.

- **"At least one new idea per turn" can manufacture filler.** The rule stops a peer from spending
  its turn only judging, but a model with nothing left will pad rather than say so. The generative
  stop condition (a round with no new ideas) is meant to catch it; padding defeats that.

- **Brief discipline is a judgement call, not a filter.** The rule is relevance ("would the answer
  change without this?"), deliberately not a word count — so a host that misjudges relevance can
  still over- or under-brief, and nothing catches it. The secret-stripping pass is an instruction
  too, not a scrubber.

- **`SendMessage` resumes asynchronously.** Only the first spawn is synchronous; every later round
  returns immediately and the reply lands as a task notification. Documented, but it makes round
  latency uneven and the host has to resist filling the gap.

- **Panel mode has no shared channel.** Peers cannot see each other; the host relays their turns
  verbatim by hand, so a panel round costs more and can lag one turn behind.

- **Model availability is not probed** before spawning. An unavailable model surfaces as a failed
  `Agent` call, and the skill has to fall back mid-run rather than up front.

- **`rounds=` is a cap, not a promise.** Early closing is intentional, but the two-keyed stop trusts
  the peer's `OPEN: none` — a peer that writes it to be agreeable ends the brainstorm early, and the
  transcript looks identical to real agreement.
