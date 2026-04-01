# ROSA Protocol Specification

`v0.2 — draft — subject to change`

---

## 0. Scope

This document defines the observable behaviour required of any agent
or system claiming ROSA compliance. It does not prescribe implementation.
Any agent satisfying these constraints participates correctly in a ROSA
swarm regardless of physical form, domain, or environment.

---

## 1. Definitions
```
Agent          — autonomous physical unit: sense → decide → act.
Environment    — shared physical space. Primary data storage of the swarm.
Observation    — immutable record of a sensor reading at a position and time.
Local zone     — region an agent can sense and affect. Hard boundary of
                 agent knowledge.
Swarm          — set of ≥2 ROSA-compliant agents in a shared environment.
Task           — desired transformation of environment state at a position.
Capability     — declared ability of an agent to execute a class of tasks.
Coalition      — temporary group of agents executing a task requiring
                 multiple agents.
Broadcaster    — interface between Operator Layer and Swarm Layer.
                 Not part of the swarm.
Zero point     — physical reference established before swarm activation.
                 All agent positions are relative to this point.
```

---

## 2. Architecture

ROSA defines two strictly separated layers with a single interface.
```
╔══════════════════════════════════════════╗
║           OPERATOR LAYER                 ║
║                                          ║
║  Slicer → Task Queue → Broadcaster(s)    ║
║                                          ║
║  Owns: intent, coordinate system,        ║
║  task graph, capability requirements     ║
╚══════════════════╤═══════════════════════╝
                   │ single interface
                   │ ← task_request  (swarm → operator)
                   │ → task_assign   (operator → swarm)
                   │ ← task_status   (swarm → operator)
╔══════════════════▼═══════════════════════╗
║             SWARM LAYER                  ║
║                                          ║
║  Agents. Observation Store.              ║
║                                          ║
║  Knows: local zone, own capabilities,   ║
║  current task. Nothing more.             ║
╚══════════════════════════════════════════╝
```

The swarm cannot push data upward. The operator cannot reach inside
the swarm. The Broadcaster is the only crossing point.

### 2.1 Broadcaster sectoring

For large environments, Broadcasters are arranged hierarchically by
sector. Each Broadcaster owns one sector and knows its neighbors.
Agents pull from the nearest Broadcaster. Agents crossing sector
boundaries switch Broadcasters autonomously based on their own position.

---

## 3. Core constraints

Non-negotiable. Violating any one means the agent is not ROSA-compliant.

**C1 — No global state**
Every decision is made using only locally observable information.
An agent must never require knowledge outside its local zone.

**C2 — No permanent authority**
No agent may act as a permanent coordinator.
Leadership is temporary and task-scoped only.

**C3 — Environment over messages**
Direct sensor readings always outweigh received messages.
Physical reality is the highest authority.

**C4 — All actions are verifiable**
Every action must be followed by a verification scan.
An agent declares completion only after its own sensors confirm it.

**C5 — Failure is a valid state**
An agent unable to complete a task must emit TASK_FAIL and release
the task. Blocking indefinitely is a protocol violation.

**C6 — Bounded memory**
An agent stores observations only within its local zone.
Memory is fixed-size. Eviction policy: LRU + TTL.
An agent must never accumulate a global map.

---

## 4. Observation Store

The environment is modelled as an append-only log of Observations.
State is never stored directly — it is always derived from observations.
This is the swarm's shared memory.

### 4.1 Observation structure
```
Observation {

  // UNIVERSAL — protocol owns these fields
  position:      Vector3
  timestamp:     unix_ms
  agent_id:      UUID
  confidence:    float [0.0–1.0]
  type:          ObsType

  // DOMAIN — application owns these fields
  measurement:   bytes  // protocol does not interpret this

}
```

### 4.2 Observation types
```
PASSIVE      — agent observed without acting. Full confidence weight.
ACTIVE       — agent observed after completing a task. Full weight.
SIDE_EFFECT  — agent changed environment incidentally (movement, scanning).
               Reduced weight in aggregation. Treated as noise.
```

### 4.3 State derivation
```
state(position) = aggregate(
  all observations within radius R of position,
  weighted by: confidence × freshness × reliability(agent_id)
)

freshness(obs) = e^(−λ × (now − obs.timestamp))

λ is domain-configurable:
  stable environments  → small λ  (slow decay)
  dynamic environments → large λ  (fast decay)
```

### 4.4 Conflict resolution

When two observations at the same position conflict, priority order:

1. Direct sensor reading beats received message
2. Higher freshness beats lower freshness
3. Higher confidence beats lower confidence
4. If tied: lower agent_id defers

### 4.5 Disturbed zones

While an agent executes a task at position P, neighboring cells within
radius D are marked DISTURBED. Observations from DISTURBED cells receive
reduced aggregation weight until the task completes and the zone is
rescanned.

---

## 5. Agent

### 5.1 Agent structure
```
Agent {
  id:              UUID
  capabilities:    Set<CapabilityID>
  position:        Vector3
  status:          AgentState
  reliability:     float [0.0–1.0]    // computed by neighbors
  current_task:    TaskID | null
  coalition:       List<AgentID> | null
  local_cache:     ObservationStore   // bounded, LRU+TTL
}
```

### 5.2 Agent state machine
```
         ┌──────────┐
         │   IDLE   │◄──────────────────────────┐
         └────┬─────┘                           │
              │ pull task from Broadcaster       │
              ▼                                  │
         ┌──────────┐                           │
         │ SCANNING │  read local zone           │
         └────┬─────┘  write PASSIVE observations│
              │                                  │
              ▼                                  │
         ┌──────────┐                           │
         │ PLANNING │  select atomic action      │
         └────┬─────┘  check depends_on          │
              │                                  │
              ▼                                  │
         ┌──────────┐                           │
         │  ACTING  │  execute                  │
         └────┬─────┘  emit TASK_CLAIM           │
              │                                  │
              ▼                                  │
         ┌──────────────┐                       │
         │  VERIFYING   │  scan result           │
         └──────┬───────┘  write ACTIVE obs      │
                │                                │
        ┌───────┴────────┐                      │
        ▼                ▼                      │
   [pass]            [fail]                     │
   emit ENV_DELTA    emit TASK_FAIL             │
   task → DONE       task → OPEN               │
        │                │                      │
        └────────────────┴──────────────────────┘
```

### 5.3 Reliability score

Each agent maintains a reliability score computed by neighbors based
on match between emitted observations and subsequent physical verification.
```
On neighbor verification of position P:
  if verify(P) confirms agent_A observation → reliability(A) += δ
  if verify(P) contradicts agent_A          → reliability(A) -= δ

if reliability(agent) < threshold:
  agent's observations excluded from local consensus
  agent flagged to Broadcaster via AGENT_STATUS
```

### 5.4 Offline behaviour

An agent that loses contact with its Broadcaster continues operating
using its local cache. When contact is restored, the agent syncs all
observations produced during isolation. The swarm does not stop when
a Broadcaster is unreachable.

---

## 6. Tasks

### 6.1 Task structure
```
Task {
  id:              UUID
  position:        Vector3
  required_caps:   Set<CapabilityID>
  coalition_size:  int              // default: 1
  target_state:    bytes            // domain-defined
  tolerance:       bytes            // acceptable deviation
  depends_on:      List<TaskID>
  priority:        int              // higher = more urgent
  timeout:         seconds
  deadline:        unix_ms | null
  status:          TaskState
}
```

### 6.2 Task states
```
PENDING  — waiting for depends_on to complete
OPEN     — available for claiming
CLAIMED  — taken by an agent or coalition
DONE     — verified complete
FAILED   — returned to OPEN after timeout or TASK_FAIL
```

### 6.3 Task assignment — pull model

Agents pull tasks. Tasks are never pushed to agents.
```
Agent requests task:
  required: agent.capabilities ⊇ task.required_caps
  required: all task.depends_on are DONE
  preferred: nearest task to agent.position

Broadcaster assigns:
  marks task CLAIMED
  records claimed_by = agent.id
  starts timeout countdown

If timeout expires without DONE:
  task returns to OPEN automatically
```

### 6.4 Coalition tasks

Tasks requiring coalition_size > 1 are executed by a temporary
coalition with a Lead agent.
```
First qualifying agent → takes LEAD role
LEAD broadcasts coalition_request to local zone
Required agents respond → coalition formed
LEAD coordinates execution
On completion → LEAD role dissolves
All agents return to IDLE
```

### 6.5 Capability deficit

If no agent with required capabilities is available, the task remains
OPEN with escalating priority. Broadcaster reports the gap to Operator
Layer. The swarm continues all other work. It does not stop.

---

## 7. Message types

Agents emit only these message types. All others are ignored.
```
ENV_DELTA      — emitted after successful verification
                 contains: position, what changed, confidence
                 scope: local zone only

TASK_CLAIM     — emitted when agent begins a task
                 scope: local zone only

TASK_FAIL      — emitted when agent cannot complete task
                 releases task back to OPEN
                 scope: local zone only

AGENT_STATUS   — periodic heartbeat, minimum 1Hz
                 contains: position, state, resources, reliability
                 scope: local zone only
```

---

## 8. Environment stability

Environmental changes — including those caused by external events or
agent side effects — are handled uniformly. There is no special mode
for unstable environments.
```
External change (storm, erosion):
  New observations overwrite old via freshness weighting
  DONE tasks with new delta automatically reopen
  Swarm resumes work without operator intervention

Agent-caused change:
  Recorded as SIDE_EFFECT observation (reduced weight)
  Neighboring cells marked DISTURBED during execution
  Rescanned after task completion
```

---

## 9. Failure handling
```
Silent failure (agent stops responding):
  Detected via missing AGENT_STATUS heartbeat
  Tasks return to OPEN via timeout
  Neighbors remove agent from local zone table

Byzantine failure (agent sends false data):
  Detected via physical verification by neighbors
  reliability(agent) decreases toward exclusion threshold
  No explicit voting — physics decides

Broadcaster failure:
  Agents enter offline mode (see 5.4)
  Work continues on cached tasks
  Sync on reconnect
```

---

## 10. What this spec does not define

Intentionally left to implementation:

- Physical form of agents
- Communication medium (radio, optical, acoustic, other)
- Zero point establishment method
- Internal planning and path-finding algorithms
- Observation measurement schema (domain-owned)
- Slicer implementation and task decomposition logic
- Any domain-specific behaviour

---

*ROSA Protocol · open specification · implementation proprietary*
*Budapest, 2025 · feedback via GitHub issues*