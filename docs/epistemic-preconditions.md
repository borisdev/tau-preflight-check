# Epistemic preconditions — systems framing

Background for the **SME-authored policy** ([README](../README.md#sme-authored-policy-what-ambiguity-to-resolve-before-acting)) — the epistemic preconditions on what ambiguity must be resolved before an agent acts. The README carries the definition and the airline example table; this doc carries the framing: why state-grading is blind to these, what the same artifact buys you, and the systems analogy.

## Ontic vs epistemic — why τ³ is blind by construction

A precondition is **ontic** if it's a fact about the world (a state-grader can check it) and **epistemic** if the agent must *know* a fact — a belief slot that must not be `UNKNOWN` (only a belief-grader can check it).

τ³'s DB grade reads the world state, so it can only check **ontic** preconditions. **Epistemic** violations are invisible to it by construction: failing to establish consent and transferring anyway leaves the DB unchanged. So the false-pass class isn't a random gap — it is *exactly* the epistemic preconditions.

**Ontic contrast anchor.** `issue_refund ← refund_eligible == True`, and `cancel_reservation` also carries `within_24h ∨ airline_cancelled ∨ insured`. Those are DB-checkable facts — **τ³ already grades them.** The epistemic guards in the README's table are the layer it can't see.

The airline pilot falls out of this split: the three FAILs τ³ already catches (24 / 35 / 43) are ontic (a wrongful cancellation *changes* the DB); the one detection the belief layer *adds* (task 47) is epistemic (the transfer leaves the DB unchanged).

## One artifact, two uses

The same SME-authored table is consumed in two places — "one spec, many roles" made concrete:

| Used as… | By whom | Effect |
|---|---|---|
| **gating** (runtime) | the agent | asks a question instead of acting under ambiguity |
| **grading** (eval) | the analyzer | flips task-47 `PASS → FAIL` |

(No reward or RL: the loop is rubric-scored eval + failure-pattern analysis, then targeted expert data — not policy-gradient training.)

## What each guard encodes — three pieces of expert input

A single row is not one fact; it decomposes into three pieces the written policy doesn't contain. Task 47 (`transfer_to_human_agents`) made concrete:

| Candidate fix | Why it works | Expert input needed |
|---|---|---|
| Default every belief slot to `UNKNOWN`; add a system invariant — *never transfer without an explicit YES*. | The agent can't treat an unresolved slot as consent; escalation now requires positive evidence. | the **invariant** |
| In the `ProblemSpec`, declare that a `transfer` requires `transfer_requested == True`. | *Acting while `UNKNOWN`* becomes a checkable violation, not a judgment call. | the **action precondition** |
| Grader penalty when an escalating action fires under `UNKNOWN`. | Lets the eval weight how severe the violation is. | the **severity** |

Because the `ProblemSpec` is versioned, executable **policy-as-code**, each addition is an auditable record of what *correct* means as policy evolves.

## Systems analogy — three-valued ABAC

Mechanically this is **attribute-based access control (ABAC) over the belief state**, with `ProblemSpecBelief` slots as the attributes and the SME guards as the policy. The lookup before each tool call is the policy decision point.

Classic ABAC is **two-valued** (allow / deny) and assumes every attribute is *known*. Because a slot can be `UNKNOWN`, ours is **three-valued** — **allow / deny / ask** — and `UNKNOWN` triggers a clarifying question (a sensing action that resolves the slot, then re-evaluates) rather than a denial. That third outcome is the whole contribution: the extension no ABAC engine has, and exactly what the belief state buys you.

## Runtime loop

```
user turn → update ProblemSpecBelief (fill slots from dialogue)
        ↓
agent selects tool T
        ↓
LOOKUP T in the guard table  ←──────── hydrated by SMEs (offline)
        ↓
for each required slot:  resolved?
   ├─ all resolved  → execute T
   └─ any UNKNOWN   → ASK the user (a sensing action) → back to top
```

The table says *which* slots gate action `T`; it does not fill them. A **belief-updater** reads the dialogue and sets slot values (`status: inferred/assumed`, `evidence_turn`). Lookup = "is this slot required and resolved?"; belief-updater = "what is its value?". The table is the *policy*; the belief is the *state*.

## Why the hydration is the data product

Enumerating an agent's actions is cheap — the tool surface is finite. Enumerating *which slots must be grounded before each action, to what value, and how severe if skipped* is the expensive, expert-authored part, and it's precisely what the written policy omits and a lab can't self-serve.

The benchmark is the discovery mechanism: every false-pass it surfaces is a guard the SMEs haven't authored yet. **Run → false-pass → SME adds the row → table grows.** That's a flywheel, not a one-shot deliverable.
