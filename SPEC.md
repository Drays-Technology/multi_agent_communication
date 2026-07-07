# communicate.md Protocol — Specification v0.1

A convention for asynchronous, auditable collaboration between two or more
AI agents working toward a single goal, using a shared markdown ledger under
version control.

The key words MUST, SHOULD, and MAY are to be interpreted as in RFC 2119.

---

## 1. Design principles

1. **The ledger is the protocol.** No message bus, no orchestrator process.
   If an agent can read and append to a file, it can participate.
2. **Durable over fast.** Agents crash, contexts overflow, sessions expire,
   models get swapped. The collaboration state MUST survive all of these.
3. **Human-readable always.** The owner must be able to open the ledger and
   understand every decision, who made it, and on what evidence — without
   tooling.
4. **Peers, not workers.** Review authority is real: a blocking finding
   stops a merge-equivalent action until resolved. Agents are expected to
   find each other's bugs, not to rubber-stamp.
5. **Evidence before change.** Behavior changes to the shared system are
   proposed with data, measured in a reversible mode where possible, and
   promoted by pre-agreed criteria — not by whoever wrote last.
6. **Append, never rewrite.** History is the audit trail. Mistakes are
   corrected by superseding entries, not edits.

---

## 2. The ledger file

### 2.1 Location and lifecycle

- The ledger SHOULD be a single file named `communicate.md` at the project
  root, committed to version control alongside the code it governs.
- The ledger MUST be committed and (if a remote exists) pushed whenever the
  work it references is committed. Entries that reference commits SHOULD
  include the short hash.
- Large projects MAY archive old entries to `communicate-archive/<date>.md`
  once the active file exceeds a workable size, keeping the most recent
  entries plus any entry containing an unresolved verdict in the active file.

### 2.2 Entry grammar

Every entry MUST begin with a level-2 heading:

```
## <AgentName> — <Title> (<ISO date>)
```

- `<AgentName>` — the writer's claimed identity (§3).
- `<Title>` — a specific, scannable summary. Good: "Orphan Fix In".
  Bad: "Update".
- `<ISO date>` — YYYY-MM-DD. Timestamps within the body when precision
  matters.

The body is free markdown (tables, code blocks, logs encouraged). An entry
SHOULD end with an em-dash signature (`— AgentName`) when the heading has
scrolled far away.

### 2.3 Append-only and supersession

- Agents MUST NOT edit or delete another agent's entries.
- Agents SHOULD NOT edit their own committed entries. To correct a mistake,
  append a superseding entry that names what it replaces:

  > Housekeeping first: my previous entry was mis-signed — this entry
  > supersedes it.

- Uncommitted drafts MAY be rewritten freely.

---

## 3. Identity and roles

### 3.1 Name claiming

- Before its first entry, an agent MUST read the ledger tail *and* the
  version-control history of the ledger to see which names are claimed.
- Names MUST be unique among concurrently active agents.
- If an agent discovers it has posted under a name already claimed by
  another active agent, it MUST post a superseding correction immediately
  and adopt an unclaimed name.

> *Origin: two agents given the same instruction both signed "Agent One"
> on the same day. The ambiguity was caught only because one read the
> other's committed entry before pushing its own.*

### 3.2 Roles

- Each agent SHOULD declare a role in its first entry. The proven minimal
  pair is:
  - **Implementer** — writes code, proposes changes, ships behind review.
  - **Reviewer** — verifies diffs against claims, issues verdicts, owns
    promotion gates and standing safety rules.
- Roles are contracts, not permissions: a reviewer MAY write code when speed
  demands it (e.g., fixing a bug it found), but the *other* agent then owes
  the counter-review. Whoever writes, the other reviews. An agent MUST NOT
  approve its own change.
- With more than two agents, every change still needs exactly one named
  reviewer; broadcast "anyone may review" entries produce nobody reviewing.

---

## 4. Verdict grammar

Verdicts are UPPERCASE keywords that carry protocol meaning. They MUST
appear in the entry heading or a bolded line so they cannot be missed by an
agent skimming the tail.

| Verdict | Meaning | Obligations created |
|---|---|---|
| `PROPOSED` | A change is described with its evidence, awaiting review. | Counterpart SHOULD respond within one working cycle. |
| `APPROVED` | Reviewed against the actual diff/behavior; no findings. | None. Implementer may proceed/merge. |
| `BLOCKING` | A finding that MUST be resolved before the change ships or the affected path is trusted. | Implementer fixes or rebuts with evidence. The blocker names the concrete failure scenario, not just a concern. |
| `COUNTER-REVIEW` | "I fixed/changed something in your domain — review me back." | Counterpart reviews the diff, not the description of it. |
| `ENDORSED` | Agreement with a judgment call (no diff to verify). | None. |
| `DISPOSITIONED` | A proposal is closed without action, with the reason and the reopening condition stated. | None, but the reopening condition is binding. |

Rules:

- A `BLOCKING` finding MUST include: the defect, the concrete failure
  scenario (inputs/state → wrong outcome), and if possible the fix shape the
  reviewer would approve. This turns review into specification.
- Review MUST be of the diff or observed behavior, never of the other
  agent's summary of it. (Case study §3: the summary said "orphan persisted";
  the diff showed the persistence being clobbered three lines later.)
- Fixes to review findings SHOULD be re-reviewed by the finder
  (`COUNTER-REVIEW`), same cycle if the change is money- or safety-critical.
- If agents disagree after one full exchange, they MUST escalate to the
  human owner (§7) rather than loop.

---

## 5. Coordination of shared resources

Two agents acting on one system collide in predictable places. The ledger is
the reservation book.

### 5.1 Declare before you schedule

Any recurring job (cron, timer, watcher) an agent creates MUST be declared
in the ledger with its schedule. Agents MUST offset schedules that touch the
same resource, and SHOULD consolidate to a single scheduled source when one
exists that survives both agents (e.g., a system crontab outliving both
sessions). When a durable source takes over, session-level duplicates MUST
be torn down and the teardown noted.

> *Origin: both agents independently scheduled hourly jobs at :23. Idempotent
> by luck, noisy by design. Resolution: one moved to :53, then both were
> retired in favor of a system crontab.*

### 5.2 Shared mutable state requires a lock

If two processes read-modify-write the same state file, they MUST serialize
access (e.g., `flock`), and the load MUST happen inside the lock. "The
writes are whole-file atomic" is not sufficient — last-writer-wins silently
undoes the other agent's work.

### 5.3 Claim work before doing it

Before starting a task another agent might plausibly pick up, an agent
SHOULD post a claim ("taking X this cycle"). Duplicated work merges badly;
two half-fixes to the same bug merge worse. If duplication is discovered,
the agents MUST reconcile explicitly in the ledger (who keeps which part)
rather than letting the repository sort it out.

### 5.4 Handoffs

An agent ending its participation (session expiry, planned shutdown) SHOULD
post a standing-state entry: what runs where, what is pending on whom, what
the next agent or the human must re-arm. Assume your successor has *only*
the ledger and the repository.

---

## 6. Evidence discipline

These rules exist because plausible-sounding changes to live systems are
where money dies.

1. **Measure first.** A proposed behavior change SHOULD ship first as
   instrumentation or a counterfactual/shadow variant that records what the
   change *would have done*. Gate changes on that data.
2. **Pre-agreed promotion criteria.** Anything that graduates from
   experiment to production MUST have its criteria written in the ledger (or
   better, encoded in a script whose exit code is the verdict) *before* the
   evaluation window starts. No moving goalposts, in either direction.
3. **Decision rules live in tools.** When a threshold decides something
   ("no gate until every band has n≥10 and stable ranking across two
   windows"), put the rule in the tool's output, so a future agent — or the
   human, with no agents left — applies the same bar.
4. **Negative results are entries too.** A closed investigation MUST be
   recorded with its evidence and reopening condition. The most expensive
   failure mode in long collaborations is re-running a dead idea because
   nobody wrote down why it died.

---

## 7. The human owner

- The owner's authorization boundaries MUST be recorded in the ledger
  (spend caps, "shadow only", credentials scope). Agents MUST NOT widen an
  envelope by inference; widening requires an explicit owner decision,
  which is then recorded.
- Items awaiting the owner MUST be tracked as such ("pending user
  authorization"), not silently dropped or silently assumed.
- Anything the agents cannot resolve in one exchange, and anything
  irreversible (real spend, deletion, publication), escalates to the owner
  with a concise decision brief: options, recommendation, consequences.
- The ledger MUST remain intelligible to the owner. If the owner cannot
  follow the collaboration from the file alone, the agents are failing the
  protocol regardless of code quality.

---

## 8. Memory and context loss

Agents lose context (window limits, session death). The protocol assumes it.

- The ledger is the **shared** source of truth; agent-private memory (notes,
  scratch files) MUST NOT contain agreements the other agent needs.
- On (re)start, an agent MUST read at least the ledger tail before acting,
  and SHOULD check version-control history for entries committed since the
  tail it last saw.
- Stale facts are corrected by superseding entries (§2.3), and an agent
  recalling a private memory SHOULD verify it against the current ledger and
  code before acting on it.

---

## 9. Version control integration

- Ledger entries about code MUST reference the commit hashes they discuss.
- Commits implementing a verdict SHOULD name it in the message
  ("fix X (Agent Two BLOCKING finding)").
- Agents SHOULD pull before reading the tail and push after appending, so
  the ledger converges quickly. Merge conflicts in the ledger are resolved
  by keeping **both** entries (append-only makes this safe).

---

## 10. Minimal compliance checklist

An implementation conforms to this spec if:

- [ ] a versioned `communicate.md` exists and agents append entries per §2.2
- [ ] agent names are unique and roles declared (§3)
- [ ] changes to shared systems receive a named review with verdict grammar (§4)
- [ ] no agent approves its own change (§3.2)
- [ ] recurring jobs and shared state access are declared/serialized (§5)
- [ ] promotion criteria are written before evaluation windows (§6.2)
- [ ] owner authorization boundaries are recorded and never widened by inference (§7)
- [ ] agents read the ledger tail on start (§8)

---

## Appendix A: entry templates

**Proposal**
```markdown
## <Agent> — PROPOSED: <change> (<date>)
### What
### Evidence
### Risk / reversibility
### Review ask
```

**Blocking finding**
```markdown
## <Agent> — BLOCKING: <defect> (<date>)
### Defect
### Failure scenario  (inputs/state → wrong outcome)
### Fix shape I'd approve
```

**Handoff**
```markdown
## <Agent> — Standing State (<date>)
### Running where
### Pending on whom
### Re-arm on next start
```
