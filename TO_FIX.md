# Known issues

Honest list. Most of these are places where a rule is an *instruction* to the host rather than
something the harness enforces.

- **The anchors phase can be faked.** Its whole value is that the numbers are real, but nothing
  checks that. A host that writes plausible-looking anchors from memory instead of reading the config
  produces a debate that looks rigorous and ranks on fiction. This is the skill's single biggest
  failure mode, and it is silent — the output is indistinguishable from a good run.

- **"Get the balance before you spawn" is expensive advice to follow.** The rule says to go run a
  10-minute profile in the production configuration first. In practice the machine is often the
  scarce resource named in `CONSTRAINT`, so the host argues from quotas measured under some other
  configuration and knows it. Currently handled by honesty (declare which config the quotas came
  from), not by anything stronger.

- **The graveyard is only as good as the grep.** It comes from whatever the host finds in old docs,
  memory files and commit messages. An idea that died in a conversation and was never written down
  isn't in it, and its re-proposal will look novel to both models.

- **`P(survives)` is a made-up number.** The ranking formula (gain × P / effort) is sound in shape
  and gives the host a place to hide a preference: nudge one probability from 0.5 to 0.7 and the
  order changes with no argument attached. Writing all three factors down is the only guard.

- **The verify pass has no quota.** It is mandatory in the text, and a host that runs it lazily —
  checking two easy claims and marking the load-bearing one `UNVERIFIED` — satisfies the letter of
  it. Same shape as the problem `VERIFIED` was introduced to solve, one layer up.

- **The agent listing misreports the tool ban.** `brainstorm-peer` ships `tools: []` and registers in
  Claude Code as "(Tools: All tools)" — alarming, but the listing renders an empty list as
  unrestricted. Two probes (2026-08-04), the second explicitly ordering the agent to issue Read and
  Bash calls and warning that declining counted as a failed diagnostic, both came back with no tools
  exposed and `tool_uses: 0`. Residual caveat: that is self-report plus an absent tool call, not a
  read of the harness config. Re-check if a release changes agent frontmatter handling.

- **`SendMessage` resumes asynchronously.** Only the first spawn is synchronous; every later round
  returns immediately and the reply arrives as a task notification. Documented in the skill, but it
  makes round latency uneven and tempts the host into filling the gap.

- **Peer names are not guaranteed unique.** `SendMessage` targets a teammate by name and the latest
  spawn wins. The skill records the `agentId` from the spawn result and sends to that; if the peer is
  gone entirely it respawns with the transcript appended.

- **Peers drift back into assistant mode** on long rounds — praise, both-sides summaries, offers to
  help. The brief forbids it and the skill sends one corrective, but smaller models relapse.

- **Model availability is not probed** before spawning. An unavailable model surfaces as a failed
  `Agent` call and the skill falls back mid-run instead of up front.

- **Secret-stripping is an instruction, not a scrubber.** The brief is assembled by hand and reviewed
  by the same host that assembled it.
