# ROSA Protocol Specification

`v0.1 — draft — subject to change`

---

## 0. Scope

This document defines the observable behaviour required of any agent
claiming ROSA compliance. It does not prescribe implementation. Any
agent that satisfies these constraints participates correctly in a
ROSA swarm.

---

## 1. Definitions
```
Agent       — any autonomous physical unit capable of sensing,
              deciding, and acting in the environment.

Environment — the shared physical space all agents operate in.
              Treated as the swarm's primary data storage.

Local zone  — the region an agent can sense and affect directly.
              No agent has knowledge outside its local zone.

Swarm       — a set of ≥2 ROSA-compliant agents operating in a
              shared environment under a common task description.

Task        — a desired transformation of the environment,
              expressed as a target state, not a sequence of steps.
```

---

## 2. Core constraints

These are non-negotiable. Violating any one means the agent
is not ROSA-compliant.

**C1 — No global state**
An agent must not require knowledge of the full environment
or the full swarm to make a decision. Every decision is made
using only locally observable information.

**C2 — No central authority**
No agent may act as a permanent coordinator for others.
Leadership, if it emerges, must be temporary and task-scoped.

**C3 — Environment is source of truth**
Agents must treat their direct sensor readings of the environment
as more authoritative than any message received from another agent.

**C4 — Actions must be verifiable**
Every action an agent takes must be followed by a verification step.
An agent must confirm the outcome through its own sensors before
declaring the action complete.

**C5 — Failure is a valid state**
An agent that cannot complete its current task must announce this
to its local zone and release the task — not block indefinitely.

---

## 3. Agent state machine
```
         ┌─────────┐
         │  IDLE   │ ◄─────────────────────┐
         └────┬────┘                       │
              │  task available            │
              ▼                            │
         ┌─────────┐                       │
         │SCANNING │  read local zone      │
         └────┬────┘                       │
              │  environment parsed        │
              ▼                            │
         ┌─────────┐                       │
         │PLANNING │  select atomic action │
         └────┬────┘                       │
              │  action selected           │
              ▼                            │
         ┌─────────┐                       │
         │ ACTING  │  execute action       │
         └────┬────┘                       │
              │  action complete           │
              ▼                            │
         ┌──────────────┐                  │
         │  VERIFYING   │  scan result     │
         └──────┬───────┘                  │
                │                          │
        ┌───────┴────────┐                 │
        ▼                ▼                 │
   [pass] emit      [fail] emit            │
    ENV_DELTA        TASK_FAIL             │
        │                │                 │
        └────────────────┴─────────────────┘
```

---

## 4. Message types

These are the only message types a ROSA agent emits.
Agents must ignore any message type not listed here.
```
ENV_DELTA
  Emitted after successful action verification.
  Contains: what changed, where, confidence score.
  Scope: local zone broadcast only.

TASK_CLAIM
  Emitted when an agent begins work on an atomic task.
  Prevents redundant parallel work on the same unit.
  Scope: local zone broadcast only.

TASK_FAIL
  Emitted when an agent cannot complete its claimed task.
  Releases the task back to the swarm.
  Scope: local zone broadcast only.

AGENT_STATUS
  Periodic heartbeat.
  Contains: position, current state, battery/resource level.
  Scope: local zone broadcast only.
  Frequency: implementation-defined, minimum 1Hz.
```

---

## 5. Task description format

A task given to a ROSA swarm must describe a *target state*
of the environment — not a sequence of steps.
The swarm derives its own execution plan.
```
Task {
  target_state:  description of desired environment state
  priority:      integer, higher = more urgent
  depends_on:    list of task IDs that must complete first
  tolerance:     acceptable deviation from target state
  deadline:      optional, unix timestamp
}
```

---

## 6. Local consensus rule

When two agents in the same local zone have conflicting
environment models, they resolve conflict by priority:

1. Direct sensor reading beats received message
2. More recent reading beats older reading
3. Higher confidence score beats lower score
4. If tied: the agent with lower ID defers

Global consistency is not required. It emerges as local zones
overlap and propagate updates across the swarm.

---

## 7. Failure handling

A ROSA swarm must remain functional if up to `⌊(N−1)/2⌋` agents
fail simultaneously, where N is swarm size. This is a design
constraint, not a guarantee — implementations must be tested
against this threshold.

---

## 8. What this spec does not define

Intentionally left to implementation:

- Physical form of agents
- Communication medium (radio, optical, acoustic)
- Internal planning algorithm
- Sensing technology
- Task slicing logic
- Any domain-specific behaviour (construction, cleanup, etc.)

---

*ROSA Protocol · open specification · implementation proprietary*  
*Feedback welcome via GitHub issues.*