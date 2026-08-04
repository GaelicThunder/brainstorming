# Known issues

- **Peer names are not guaranteed unique.** `SendMessage` targets a teammate by name and the latest
  spawn wins. If another agent takes the name mid-brainstorm, the send lands in the wrong context.
  *Workaround:* the skill records the `agentId` from the spawn result and falls back to it; if the
  peer is gone entirely it respawns with the transcript appended.

- **Peers drift back into assistant mode** on long rounds — praise, both-sides summaries, offers to
  help. The brief forbids it and the skill sends one corrective, but smaller models relapse.

- **Panel mode has no shared channel.** Peers cannot see each other; the host relays their turns by
  hand, so a panel round costs more and can lag one turn behind.

- **Model availability is not probed** before spawning. An unavailable model surfaces as a failed
  `Agent` call, and the skill has to fall back mid-run rather than up front.

- **Brief discipline is a judgement call, not a filter.** The rule is relevance ("would the answer
  change without this?"), deliberately not a word count — so a host that misjudges relevance can
  still over- or under-brief, and nothing catches it. The secret-stripping pass is an instruction
  too, not a scrubber.

- **`SendMessage` resumes asynchronously.** Only the first spawn is synchronous; every later round
  returns immediately and the reply lands as a task notification. Documented, but it makes round
  latency uneven and the host has to resist filling the gap.

- **`rounds=` is a cap, not a promise.** Early convergence is intentional, but a lazy peer that
  restates its position twice looks identical to genuine agreement.
