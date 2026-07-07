# Case Study: Two Agents, One Live Production System

The communicate.md protocol was extracted from a real deployment: two AI
agents ("Agent One" — implementer, "Agent Two" — reviewer) co-maintaining a
live automated system that executed real transactions against an external
service — the kind of system where a booking error is not a cosmetic bug but
an irreversible loss. The owner set goals; the agents coordinated entirely
through a git-versioned `communicate.md`.

This document is the evidence that the protocol's rules earn their keep.
Every incident below is real, occurred across a three-day window, and is
traceable in that project's ledger and commit history.

---

## Part 1 — Four critical-path bugs caught by adversarial review

### Bug 1: Rejected operations recorded as completed

An exit operation was **rejected** by the backing store over a 6×10⁻¹⁵
float-precision shortfall — and the engine recorded it as completed anyway,
at the intended price, with zero costs. The system's derived state forked
from the store's ground truth; a resource worth ~10% of the system's balance
sat stranded and invisible.

- Found by the implementer observing live behavior; fix reviewed
  cross-agent, and the reviewer's follow-up verdict split the retry policy
  by environment (simulation: retry forever + escalate loudly; production:
  force-complete after 3 attempts, the external service is ground truth).
- **Protocol rule it produced:** review asks travel with the fix
  ("(a) is the tolerance right? (b) retry forever or force after N?") — §4.

### Bug 2: The fix for a BLOCKING finding was itself silently broken

The reviewer issued a BLOCKING finding: a two-step acquisition performed
step one, then made three unguarded calls for step two — any failure left an
*untracked, unprotected* half-acquired resource. The implementer shipped the
prescribed fix (rollback + a persisted orphan record). The reviewer then
re-reviewed **the diff, not the description** and found the orphan record
was written into a privately-loaded state object that the caller's
end-of-cycle save clobbered off disk three lines later. The safety net
existed and caught nothing.

- Fixed by threading the caller's state through; a regression test now
  asserts the record survives the subsequent save.
- **Protocol rules it produced:** review the diff, never the summary (§4);
  critical fixes get same-cycle counter-review (§4).
- This is the flagship catch: it can only happen between peers with real
  review authority. An orchestrator grading its workers' self-reports
  ships this bug.

### Bug 3: Documentation that would have burned the owner

The go-live runbook told the owner to provision a resource budget of X for
the full deployment. The reviewer traced the arithmetic: each unit of work
consumed its budget **twice** (once in each of two pools), so the true
requirement was 2X — and the preflight check passed at X, so nothing would
have warned. Half the deployment would have failed on an insufficient
budget, every cycle, forever. The scaling guidance two paragraphs down had
the same error in the opposite direction (it over-committed 1.6× the
available budget).

- **Protocol rule it produced:** documentation intended for the human is a
  reviewable artifact with the same severity as code (§7).

### Bug 4: The vanishing 14 percent

The owner asked a simple question — "how is it doing?" — and the headline
number didn't reconcile with the transaction history. Investigation: the
shutdown path recorded pending work as completed **without executing it
against the backing store**, and separately, the store snapshot was saved
*before* the shutdown completions ran. A resource worth ~14% of the
system's balance sat stranded and invisible across a restart.

- Both halves fixed; the store repaired only after an explicit owner
  decision (the agent's own harness refused the direct state write until
  the owner approved — correctly).
- **Protocol rules it produced:** reconcile derived numbers against ground
  truth when a human asks a "simple" question; irreversible repairs are
  owner decisions with a written brief (§7).

---

## Part 2 — Coordination failures that became protocol rules

### The name collision

Given the same instruction by the owner, both agents signed their entries
with the same name. Caught only because one agent read the other's
committed entry before pushing its own; resolved by an explicit superseding
correction and a rule: read the ledger tail *and its git history* before
claiming a name (§3.1).

### The scheduler collision

Both agents independently scheduled recurring jobs at the same minute
against the same engine. Harmless only because the operation was idempotent
by design. Resolution ladder, now §5.1: declare schedules in the ledger →
offset → consolidate to a single source that survives both agents → tear
down duplicates explicitly.

### The state-file race

An hourly job and a five-minute safety pass — separate processes — did
read-modify-write on the same state file. Work completed by the fast pass
mid-cycle would be resurrected by the slow cycle's save. Fixed with file
locking, with the load *inside* the lock (§5.2). Found not by failure but
by an agent tracing state flow after Bug 2 made state-clobbering a named
hazard — reviews compound.

### The graceful handoff

Facing a deadline after which neither agent would be reachable, the
implementer wrote the owner-facing runbook and a master automation loop;
the reviewer audited both and codified the go-live criteria into a script
whose exit code *is* the verdict. The final entries are standing state:
what runs where, what is pending on whom (§5.4, §6.2). The system now
operates with zero agents.

---

## Part 3 — What the numbers say

Over the collaboration window the ledger records: ~15 reviewed changes, 4
BLOCKING-severity catches before any of them could reach production, 2 full
adversarial review cycles (finding → fix → counter-review → counter-fix),
zero unresolved disagreements (one escalation to the owner, resolved in one
question), and a production go-live process the owner can operate from a
single markdown file.

The honest limitation: this was two agents of the same model family, in one
repository, over days not months. The protocol's claims beyond that
envelope — heterogeneous models, larger crews, long horizons — are exactly
what open-sourcing is for.
