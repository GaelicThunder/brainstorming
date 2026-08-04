# Known issues

- **The agent listing misreports the tool ban.** `brainstorm-peer` ships `tools: []` and registers
  in Claude Code as "(Tools: All tools)" — alarming, but the listing is rendering an empty list as
  unrestricted. Two probes (2026-08-04), the second explicitly ordering the agent to issue Read and
  Bash calls and warning that declining counted as a failed diagnostic, both came back with no tools
  exposed and `tool_uses: 0`. Residual caveat: that is self-report plus an absent tool call, not a
  read of the harness config. Re-check if a Claude Code release changes agent frontmatter handling.

- **Peer names are not guaranteed unique.** `SendMessage` targets a teammate by name and the latest
  spawn wins. If another agent takes the name mid-brainstorm, the send lands in the wrong context.
  *Workaround:* the skill records the `agentId` from the spawn result and falls back to it; if the
  peer is gone entirely it respawns with the transcript appended.

- **Peers drift back into assistant mode** on long rounds — praise, both-sides summaries, offers to
  help. The brief forbids it and the skill sends one corrective, but smaller models relapse.

- **The host's `VERIFIED` quota can still be gamed.** It must target the peer's `CHECK:` nomination
  or whatever claim would change the ranking if false, and the "nothing checkable this round" opt-out
  is available — a host that leans on the opt-out, or checks something safe, satisfies the letter of
  the rule and grounds nothing. Same shape as the filler problem that `NEW: none` solved, one layer
  over.

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
