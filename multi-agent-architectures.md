# Multi-Agent Architectures: Pipeline, Fan-Out, Hub-and-Spoke

*A 5-minute summary*

---

## The one question that separates them

> **Who decides the path, and when?**

- **Pipeline:** you decide the path at design time. Every input takes the same route.
- **Fan-out:** you decide the split upfront. Workers run at once, then results are merged.
- **Hub-and-spoke:** a coordinator decides the next step at runtime, based on the last result.

**Rule of thumb:** if you can draw the flowchart *before* the task runs, use a pipeline. If the model has to draw it *while* the task runs, consider hub-and-spoke.

---

## 1. Sequential pipeline

```
input → [ A ] → [ B ] → [ C ] → [ D ] → output
```

Each stage does one job and passes its output to the next. The order never changes.

**Analogy:** a car assembly line.

**Example: invoice processing**

```
PDF → extract fields → validate vs PO → assign GL codes → post / flag for human
```

**Use it when:**
- The steps are the same for every input
- Each step needs the previous step's output
- You want predictable, testable, easy-to-debug behaviour

**Downside:** total time is the sum of every stage, and it can't adapt when an input needs a different route.

---

## 2. Parallel fan-out

Also called **parallelization (sectioning)**, **scatter-gather**, or **map-reduce**.

```
              ┌──→ [ worker 1 ] ──┐
input → split ├──→ [ worker 2 ] ──┤→ merge → output
              └──→ [ worker 3 ] ──┘
```

Work is split upfront, workers run at the same time, results are combined. One round, then done.

**Analogy:** a group project where everyone writes one chapter at the same time.

**Example: due diligence on 30 companies.** One agent per company gathers financials and news; a final step builds the comparison table.

**Example: pull request review**

```
         ┌──→ security reviewer ──┐
PR diff ─┼──→ perf reviewer ──────┤→ dedupe & merge → one review
         └──→ style reviewer ─────┘
```

**Use it when:**
- Subtasks are independent (no worker needs another's result)
- The split is obvious before you start
- You want speed: total time ≈ the slowest worker, not the sum

**Key traits:**
- Workers often do the *same kind of job* on different input, or each covers one fixed concern
- Results have the *same shape*
- The coordinator splits once and merges once, and is often plain code

---

## 3. Hub-and-spoke

```
                [ logs ]
                    │
 [ deploys ] ── [ HUB ] ── [ metrics ]
                    │
              [ database ]
```

A central coordinator (the hub) calls specialist agents (spokes) **one decision at a time**:

```
hub → call spoke → read result → decide next → call spoke → ... → done
```

The hub can call the same spoke twice, skip spokes, or change the plan halfway. Spokes never talk to each other; the hub holds all the state.

**Analogy:** an ER doctor who orders an ECG, then decides on blood work or a cardiology consult based on what it shows.

**Example: incident investigation** ("checkout latency spiked")

```
metrics  → "only payment service is slow"
deploys  → "payment deployed 20 min ago"
logs     → "connection pool timeouts since deploy"
database → "DB is healthy"
hub      → "likely pool misconfig in new deploy, recommend rollback"
```

A different alert would take a completely different route.

**Example: customer support.** "Charged twice and can't log in" → billing spoke, account spoke, billing again (refund), account again (unlock), one reply. "How do I export data?" → docs spoke only.

**Use it when:**
- The path depends on intermediate results
- There are many distinct capabilities (too many tools for one agent)
- You want context isolation: spokes read lots and return short summaries
- You need one control point for policy, approvals, audit, and budgets
- You add new capabilities often (just plug in a new spoke)

**Avoid it when:**
- The workflow is predictable (use a pipeline)
- The work is simply parallel (use fan-out)
- One agent with a few tools would do; start there first
- Spokes need lots of back-and-forth (the hub becomes a slow relay)

**Costs:** extra latency and tokens per hop, a single point of failure, and early routing mistakes that compound.

---

## Does hub-and-spoke always have a coordinator?

Yes, the hub *is* the coordinator. Fan-out also has one, but it doesn't reason between steps.

The real difference is **when dispatch is decided**:

| | Workers are… | Results | Coordinator |
|---|---|---|---|
| **Fan-out** | same job, different input or fixed concern | same shape | splits once, merges once |
| **Hub-and-spoke** | different capabilities (roles + tools) | vary by spoke and request | decides every next step |
| **Peer / handoff** | different capabilities | vary | none; agents route to each other |

Patterns **without** a coordinator:

```
Handoff:     [ A ] ──→ [ B ] ──→ [ C ]     (each agent picks the next)

Blackboard:  [ A ]   [ B ]   [ C ]
               ↘       ↓       ↙
            [ shared workspace ]           (agents pick up work they can do)
```

These avoid a bottleneck, but are harder to control and debug.

**Real systems mix patterns.** A hub can fan out a group of spokes in parallel each round, and a single spoke can be a small pipeline inside.

**Quick test:** what changes between two inputs?
- **Pipeline:** only the data
- **Fan-out:** maybe the number of workers, but the split is clear upfront
- **Hub-and-spoke:** which specialists run, in what order, and how many times

---

## Applied: turning a sequential code review into fan-out

**Before:**

```
diff → style → quality → performance → complexity → output
```

**After** (a pipeline with a fan-out stage):

```
                  ┌──→ style ────────┐
                  ├──→ quality ──────┤
prepare diff ─────┼──→ performance ──┼──→ merge → output
                  └──→ complexity ───┘
```

### File layout (Claude Code style)

```
.claude/
  agents/
    style-reviewer.md
    quality-reviewer.md
    performance-reviewer.md
    complexity-reviewer.md
  commands/
    review.md          ← coordinator: prepare, dispatch, merge
```

### Each reviewer file contains

1. **Frontmatter:** name, description ("used in /review"), read-only tools, model
2. **Scope:** what to check, plus an explicit **out of scope** list
3. **Rules:** review changed lines only; follow project conventions
4. **Output format:** identical across all reviewers, or `NO_FINDINGS`

### Scope boundaries

| Reviewer | Checks | Out of scope |
|---|---|---|
| Style | naming, formatting, lint rules, consistency | bugs, design, speed |
| Quality | readability, error handling, edge cases, duplication, tests | style, speed |
| Performance | I/O in loops, N+1 queries, allocations, blocking calls, caching | Big-O |
| Complexity | how code scales with input; state current → suggested, e.g. O(n²) → O(n) | constant-factor speedups |

### The coordinator (`review.md`) does

1. Compute the diff **once** and save it; stop if empty
2. Launch all four reviewers **in parallel, in a single message**
3. Merge: drop `NO_FINDINGS`, combine duplicates on the same line (keep higher severity), sort by severity then file
4. Output a summary line with counts, then findings grouped by severity

### Design rules

- **Same output format everywhere**, so merging is trivial (use JSON if a script posts to GitHub)
- **Explicit out-of-scope lists**, so reviewers don't all flag the same issue
- **Compute shared input once**; reviewers read it rather than re-running git
- **Read-only reviewers**; they report, they don't edit
- **Clear descriptions** so reviewers aren't invoked randomly outside /review
- **Check hidden dependencies**: if a later stage relied on an earlier one's findings, parallel removes that link
- **Match models to difficulty**: small model for style, stronger model for complexity

**Result:** total time ≈ the slowest reviewer instead of the sum of four, and each reviewer starts with a clean, focused context.
