# Case Study: Two Agents, One Live Trading System

The communicate.md protocol was extracted from a real deployment: two Claude
agents ("Agent One" — implementer, "Agent Two" — reviewer) co-maintaining a
cryptocurrency trading bot with shadow books, a hard-capped live executor,
and unattended cron automation. The owner set goals; the agents coordinated
entirely through a git-versioned `communicate.md`.

This document is the evidence that the protocol's rules earn their keep.
Every incident below is real, dated 2026-07-05 → 2026-07-07, and traceable
in the project's ledger and commit history.

---

## Part 1 — Four money-path bugs caught by adversarial review

### Bug 1: Rejected close orders booked as filled

A winning trade's exit order was **rejected** by the paper wallet over a
6×10⁻¹⁵ float shortfall — and the engine booked the close anyway, at
decision price, with zero costs. Equity forked from the wallet; $18.90 of
coins stranded invisibly.

- Found by the implementer observing live behavior; fix reviewed
  cross-agent, and the reviewer's follow-up verdict split the retry policy
  by mode (paper: retry forever + escalate loudly; live: force-close after
  3 attempts, exchange is ground truth).
- **Protocol rule it produced:** review asks travel with the fix
  ("(a) is the epsilon right? (b) retry forever or force-close?") — §4.

### Bug 2: The fix for a BLOCKING finding was itself silently broken

The reviewer issued a BLOCKING finding: the live executor bought spot, then
made three unprotected futures calls — any failure orphaned an *untracked,
unhedged* position. The implementer shipped the prescribed fix (rollback
sell + persisted orphan record). The reviewer then re-reviewed **the diff,
not the description** and found the orphan record was written into a
privately-loaded state object that the caller's end-of-tick save clobbered
off disk three lines later. The safety net existed and caught nothing.

- Fixed by threading the caller's state through; a regression test now
  asserts the record survives the subsequent save.
- **Protocol rules it produced:** review the diff, never the summary (§4);
  money/safety fixes get same-cycle counter-review (§4).
- This is the flagship catch: it can only happen between peers with real
  review authority. An orchestrator grading its workers' self-reports
  ships this bug.

### Bug 3: Documentation that would have burned the owner's deposit

The go-live runbook told the owner "deposit ~200 USDT total for the
2-position envelope." Reviewer traced the arithmetic: each position consumes
its size **twice** (spot cash + futures margin), so the true requirement was
~400 — and the preflight check passed at 200, so nothing would have warned.
Position 2 would have failed on insufficient balance every hour, forever.
The scaling rule two paragraphs down had the same error in the other
direction (it over-committed 1.6× capital).

- **Protocol rule it produced:** documentation intended for the human is a
  reviewable artifact with the same severity as code (§7).

### Bug 4: The vanishing 27 dollars

The owner asked a simple question — "did the bot make money?" — and the
equity number didn't reconcile with trade history. Investigation: the
shutdown path booked position closes **without executing the sale in the
wallet**, and separately, the wallet snapshot was saved *before* the closes
ran. 0.96 DEXE (~27 USDT) sat stranded and invisible across a restart.

- Both halves fixed; wallet repaired only after an explicit owner decision
  (the agent's harness refused the direct database write until the owner
  approved — correctly).
- **Protocol rules it produced:** reconcile derived numbers against ground
  truth when a human asks a "simple" question; irreversible repairs are
  owner decisions with a written brief (§7).

---

## Part 2 — Coordination failures that became protocol rules

### The name collision

Given the same instruction by the owner, both agents signed their entries
"Agent One." Caught only because one agent read the other's committed entry
before pushing its own; resolved by an explicit superseding correction and
a rule: read the ledger tail *and its git history* before claiming a name
(§3.1).

### The :23 cron collision

Both agents independently scheduled hourly jobs at :23 against the same
engine. Harmless only because the operation was idempotent by design.
Resolution ladder, now §5.1: declare schedules in the ledger → offset →
consolidate to a single source that survives both agents → tear down
duplicates explicitly.

### The state-file race

The hourly tick and a 5-minute safety pass — separate processes — did
read-modify-write on the same JSON state. A close executed by the fast pass
mid-tick would be resurrected by the tick's save. Fixed with `flock`
serialization, with the load *inside* the lock (§5.2). Found not by failure
but by an agent tracing state flow after Bug 2 made state-clobbering a
named hazard — reviews compound.

### The graceful handoff

Facing a deadline after which neither agent would be reachable, the
implementer wrote the owner-facing runbook and a master automation loop;
the reviewer audited both and codified the promotion criteria into a script
whose exit code *is* the go/no-go verdict. The final entries are standing
state: what runs where, what is pending on whom (§5.4, §6.2). The system
now operates with zero agents.

---

## Part 3 — What the numbers say

Over the collaboration window the ledger records: ~15 reviewed changes, 4
BLOCKING-severity catches before real capital was exposed, 2 full
adversarial review cycles (finding → fix → counter-review → counter-fix),
zero unresolved disagreements (one escalation to the owner, resolved in one
question), and a live-capital readiness process the owner can operate from
a single markdown file.

The honest limitation: this was two agents of the same model family, in one
repository, over days not months. The protocol's claims beyond that
envelope — heterogeneous models, >2 agents, long horizons — are exactly
what open-sourcing is for.
