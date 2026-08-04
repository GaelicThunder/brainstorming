# TODO

- [ ] **Anchor provenance as a required field.** Every anchor line carries its source today by
      convention; make it structural (`value ← source`) so an anchor without one is visibly missing
      rather than merely unsourced. Closest thing to a fix for the fabricated-anchors failure in
      `TO_FIX.md`.
- [ ] **Ship the regime taxonomy as data** instead of improvising it per run — I/O-bound,
      compute-bound, latency-bound, cost-bound, contention-bound — each with the quantity that
      defines it and the observation that says you're in it. Would make "declare your regime" a lookup
      instead of a judgement call.
- [ ] **Graveyard extractor.** A helper that greps dated docs, memory files and commit messages for
      dead-idea markers (`⛔`, "morta", "killed", "smentito", "does not compose") and drafts the block
      for the host to prune. Today it's hand-assembled every time.
- [ ] **Carry the kill-checks forward.** They are declared in the plan and then live only in a
      markdown file. Emit them as a checklist an execution session can tick off, so a lever that
      shipped without its check being run is visible.
- [ ] **`resume`** — reopen a saved brainstorm and continue it in a later session, with the ledger
      and the graveyard reloaded (the graveyard should have grown since).
- [ ] **Measure the verify pass.** The "roughly one plumbing claim in four is wrong" figure comes from
      a handful of sessions. Count it properly across runs; if it's much lower, the pass can be
      narrowed to the load-bearing claims only.
- [ ] **Second peer, one round.** Not the full panel apparatus of an earlier draft — just a single
      round where a third model gets the final ranked list and only has to find what both missed.
      Cheap, and it targets the correlated blind spot two models share.
- [ ] **Decide whether taste-question brainstorms belong here at all** (naming, tone, visual
      direction). Currently declared out of scope in the README, since with no anchors the method
      reduces to two models being agreeable. If they do belong, they need a different referee, not a
      relaxed version of this one.
