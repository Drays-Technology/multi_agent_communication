# communicate.md — a ledger protocol for multi-agent collaboration

**One markdown file. Multiple AI agents. Real peer review. Full audit trail.**

`communicate.md` is a convention — not a framework — for making two or more
AI agents (any vendor, any model, any harness) work toward a single goal by
reading and appending to a shared, git-versioned markdown ledger.

It was not designed. It was evolved, under fire, by two Claude agents
co-maintaining a live cryptocurrency trading system. Every rule in
[SPEC.md](SPEC.md) exists because its absence caused a real failure — see
[CASE_STUDY.md](CASE_STUDY.md) for the four money-path bugs that adversarial
ledger review caught before they touched real capital, and the coordination
failures (name collisions, scheduler races, state clobbers) that became
protocol rules.

## Why a file, not a framework?

| Typical multi-agent framework | The ledger |
|---|---|
| Synchronous, in-process messaging | Asynchronous, durable file |
| Orchestrator delegates to workers | Peers review each other, with verdicts |
| Dies with the process | Survives crashes, restarts, model swaps |
| Opaque logs | Human-readable markdown, git history = provenance |
| Vendor SDK lock-in | Any agent that can read/write a file |

The killer property is **adversarial accountability**: a coder agent and a
reviewer agent with real verdict semantics (`APPROVED`, `BLOCKING`,
counter-review) catch each other's bugs. An orchestrator grading its own
workers' output catches almost nothing. In our case study, the reviewer
caught that the coder's *fix for the reviewer's own blocking finding* was
silently broken — that class of catch only happens between genuine peers.

## Quickstart: run a crew from your terminal

The repo ships `communicate`, a zero-dependency CLI (Python 3.11+) that
turns the protocol into a working crew: **define agents, define a goal,
let them work.**

```bash
cd your-project
/path/to/communicate init     # scaffolds crew.toml + communicate.md
$EDITOR crew.toml             # set the goal, name the agents
/path/to/communicate run      # agents take turns until done
/path/to/communicate status   # goal, roster, ledger tail
```

`crew.toml` defines the crew — any CLI-invokable agent works, mixed
vendors welcome:

```toml
goal = """Build a REST API for the todo app: CRUD, tests green,
README updated. Don't touch deployment config without asking."""

[[agents]]
name = "Kestrel"
role = "Implementer"
command = 'claude -p {prompt} --permission-mode acceptEdits'

[[agents]]
name = "Heron"
role = "Reviewer"
command = 'claude -p {prompt} --permission-mode acceptEdits'
```

Each turn, an agent gets its identity, role, the goal, and the protocol
rules; it does real work in your working directory and appends one ledger
entry. The run ends only when one agent claims `GOAL-COMPLETE` **and a
different agent independently approves it** — self-approval doesn't count,
here or anywhere else in the protocol. The ledger is the only state, so a
crashed or stopped run resumes by running again.

## The 60-second protocol version

1. Put a `communicate.md` at your repo root. Commit it with your code.
2. Each agent claims a unique name and a role in its first entry.
3. All coordination happens as **append-only** entries:
   `## <Agent> — <Title> (<date>)`.
4. Code changes reference ledger verdicts; ledger entries reference commits.
5. Disagreements resolve by evidence, escalation to the human owner, or a
   named verdict — never by silently overwriting the other agent's work.

Read [SPEC.md](SPEC.md) for the full protocol, then copy
[examples/minimal-ledger.md](examples/minimal-ledger.md) to start, or let
`communicate init` scaffold it for you.

## Status

Spec version 0.1 — extracted from a working deployment, offered for the
community to use and evolve. PRs welcome, especially:

- adapters/prompts for specific harnesses (Claude Code, Cursor, aider, raw API)
- conventions for >2 agents
- tooling: ledger linters, verdict-tracking dashboards, escalation notifiers

## License

MIT.
