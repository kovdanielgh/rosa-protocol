# ROSA Protocol Specification

`v0.4 — draft — subject to change`

---

## 0. Scope

This document defines the observable behaviour required of any agent
or system claiming ROSA compliance. It does not prescribe implementation.
Any agent satisfying these constraints participates correctly in a ROSA
swarm regardless of physical form, domain, or environment.

---

## 1. Definitions

```
Agent            — autonomous physical unit: sense → decide → act.
Environment      — shared physical space. Primary data storage of the swarm.
Observation      — immutable record of a sensor reading at a position and time.
Local zone       — region an agent can sense and affect. Hard boundary of
                   agent knowledge.
Swarm            — set of ≥2 ROSA-compliant agents in a shared environment.
Task             — desired transformation of environment state at a position.
Job              — named collection of tasks sharing a common intent and
                   lifecycle. Cancelling a Job cancels all its Tasks.
Capability       — declared ability of an agent to execute a class of tasks.
Coalition        — temporary group of agents executing a task requiring
                   multiple agents.
Broadcaster      — software process that holds the Task Queue for a sector
                   and responds to agent pull requests. Not part of the swarm.
                   May run on any hardware: laptop, embedded device,
                   agent-carrier, or cloud node.
Delta Watcher    — component of the Broadcaster that monitors the sector
                   Observation Store for unplanned state deviations and
                   generates reactive tasks or alerts accordingly.
Zero point       — physical reference established before swarm activation.
                   All agent positions are relative to this point.
Bootstrap config — minimal configuration loaded by an agent at startup.
```

---

## 2. Architecture

ROSA defines two strictly separated layers with a single interface.

```
╔══════════════════════════════════════════════════╗
║                 OPERATOR LAYER                   ║
║                                                  ║
║  Slicer ──► Task Queue ◄── Delta Watcher         ║
║                  │               ▲               ║
║                  ▼               │               ║
║             Broadcaster(s) ──────┘               ║
║                                                  ║
║  Owns: intent, coordinate system, task graph,    ║
║  capability requirements, delta rules            ║
╚══════════════════════╤═══════════════════════════╝
                       │ single interface
                       │ ← task_request    (swarm → operator)
                       │ → task_assign     (operator → swarm)
                       │ ← task_status     (swarm → operator)
                       │ → cancel_signal   (operator → swarm)
                       │ ← delta_alert     (operator layer → operator UI)
╔══════════════════════▼═══════════════════════════╗
║                  SWARM LAYER                     ║
║                                                  ║
║  Agents. Local Observation Stores.               ║
║                                                  ║
║  Knows: local zone, own capabilities,            ║
║  current task. Nothing more.                     ║
╚══════════════════════════════════════════════════╝
```

The swarm cannot push data upward. The operator cannot reach inside
the swarm. The Broadcaster is the only crossing point. There is no
global Observation Store — only local stores per agent and per sector.

Agents never create tasks. Task creation is exclusively the
responsibility of the Operator Layer — either via Slicer (planned)
or Delta Watcher (reactive).

### 2.1 Broadcaster sectoring

For large environments, Broadcasters are arranged by sector. Each
Broadcaster owns one sector. Multiple Broadcasters may run on the same
physical machine as separate processes. Agents pull from the nearest
Broadcaster determined by their position and bootstrap config. Agents
crossing sector boundaries switch Broadcasters autonomously.

### 2.2 Broadcaster persistence

A Broadcaster must checkpoint its Task Queue to durable storage at
configurable intervals. On restart it recovers from the last checkpoint.
CLAIMED tasks whose timeout has expired are returned to OPEN on recovery.

### 2.3 Observation Store federation

Observation Stores are local. There is no global store. Federation
between two independent ROSA installations is an explicit opt-in
operation — installations share a defined subset of observations through
a federation endpoint. This is outside the core protocol and governed
by bilateral agreement.

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

**C7 — Cancellation is graceful**
On receiving a cancel signal an agent completes its current atomic
action before stopping. It must not abandon work mid-action.

**C8 — Agents never create tasks**
Task creation is exclusively the responsibility of the Operator Layer.
An agent that attempts to create or modify a task violates the protocol.

---

## 4. Time synchronization

All timestamps in Observations must be derived from a common time
source. Agents with inconsistent clocks produce incorrect freshness
calculations and unreliable conflict resolution.

```
Required: all agents in a swarm share a time source.

Implementations:
  Open environment (GPS available)  → GPS time (±100ns)
  Closed environment / indoors      → NTP over mesh network
  High-precision requirement        → PTP (IEEE 1588, ±1µs)
  Isolated / no infrastructure      → one agent acts as time master,
                                      others sync to it on join

Time source address is part of bootstrap config.
An agent must not write Observations until time is synchronized.
```

---

## 5. Agent bootstrap

Every agent requires a bootstrap config before it can join a swarm.
This is the only configuration an agent receives externally.

```
BootstrapConfig {
  broadcaster_address:  string
  zero_point:           GeoPoint
  sector_id:            string
  time_source:          string        // NTP/PTP address or "gps"
  agent_id:             UUID
  capabilities:         List<CapabilityID>
}

Bootstrap sequence:
  1. Load config
  2. Sync time
  3. Establish position relative to zero_point
  4. Connect to Broadcaster
  5. Begin SCANNING → IDLE cycle
```

---

## 6. Observation Store

The environment is modelled as an append-only log of Observations.
State is never stored directly — it is always derived from observations.
Each agent maintains a local store bounded to its local zone.
Sector-level stores aggregate agent observations within a sector.

### 6.1 Observation structure

```
Observation {

  // UNIVERSAL — protocol owns these fields
  position:      Vector3
  timestamp:     unix_ms         // from synchronized time source
  agent_id:      UUID
  confidence:    float [0.0–1.0]
  type:          ObsType

  // DOMAIN — application owns these fields
  measurement:   bytes           // protocol does not interpret this
                                 // may encode lidar points, chemical
                                 // concentrations, ML inference results,
                                 // or any domain-specific reading
}
```

### 6.2 Observation types

```
PASSIVE      — agent observed without acting. Full confidence weight.
ACTIVE       — agent observed after completing a task. Full weight.
SIDE_EFFECT  — agent changed environment incidentally (movement, scanning).
               Reduced weight in aggregation. Treated as noise.
```

### 6.3 State derivation

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

### 6.4 Conflict resolution

When two observations at the same position conflict, priority order:

1. Direct sensor reading beats received message
2. Higher freshness beats lower freshness
3. Higher confidence beats lower confidence
4. If tied: lower agent_id defers

### 6.5 Disturbed zones

While an agent executes a task at position P, neighboring cells within
radius D are marked DISTURBED. Observations from DISTURBED cells receive
reduced aggregation weight until the task completes and the zone is
rescanned.

### 6.6 Observation propagation

Agents share observations with neighbors within their local zone.
This is the mechanism by which local consensus spreads across the swarm.

```
Propagation rules:

  Agent broadcasts fresh observations to local zone peers at a
  configurable rate (default: on write, max 1Hz to prevent flooding).

  Receiving agent applies conflict resolution (6.4) before accepting
  a remote observation into its local cache.

  Remote observations retain their original agent_id and timestamp.
  They are never re-attributed to the receiving agent.

  An observation propagates at most one hop beyond its origin zone.
  Agents do not relay observations received from other agents.
  Wider propagation happens via sector-level store aggregation only.
```

---

## 7. Agent

### 7.1 Agent structure

```
Agent {
  id:              UUID
  capabilities:    Set<CapabilityID>
  position:        Vector3
  status:          AgentState
  reliability:     float [0.0–1.0]
  current_task:    TaskID | null
  coalition:       List<AgentID> | null
  local_cache:     ObservationStore     // bounded, LRU+TTL
  neighbor_table:  Map<AgentID, NeighborRecord>
}

NeighborRecord {
  agent_id:      UUID
  last_seen:     unix_ms
  position:      Vector3
  reliability:   float
  ttl:           seconds    // record expires if no heartbeat received
}
```

### 7.2 Neighbor discovery

Neighbors are discovered passively through AGENT_STATUS heartbeats.
No explicit discovery handshake is required.

```
On receiving AGENT_STATUS from agent X:
  if X not in neighbor_table → add entry
  update last_seen, position, reliability
  reset TTL

On TTL expiry for agent X:
  remove X from neighbor_table
  treat pending observations from X as unverifiable
```

### 7.3 Agent state machine

```
         ┌──────────┐
         │   IDLE   │◄──────────────────────────┐
         └────┬─────┘                           │
              │ pull task from Broadcaster       │
              ▼                                  │
         ┌──────────┐                           │
         │ SCANNING │  read local zone           │
         └────┬─────┘  write PASSIVE obs         │
              │         propagate to neighbors   │
              ▼                                  │
         ┌──────────┐                           │
         │ PLANNING │  select atomic action      │
         └────┬─────┘  check not_before          │
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

On CANCEL signal received at any state:
  complete current atomic action
  emit TASK_FAIL for current task
  return to IDLE
  stop pulling new tasks
```

### 7.4 Reliability score

```
On neighbor verification of position P:
  if verify(P) confirms agent_A observation → reliability(A) += δ
  if verify(P) contradicts agent_A          → reliability(A) -= δ

if reliability(agent) < threshold:
  agent's observations excluded from local consensus
  agent flagged to Broadcaster via AGENT_STATUS
```

### 7.5 Offline behaviour

An agent that loses contact with its Broadcaster continues operating
using its local cache. When contact is restored the agent syncs all
observations produced during isolation. The swarm does not stop when
a Broadcaster is unreachable.

---

## 8. Jobs

A Job is a named collection of Tasks sharing a common intent and
lifecycle. Jobs are created by the Operator Layer and are never
visible to agents. Agents only see Tasks.

```
Job {
  id:          UUID
  name:        string
  tasks:       List<TaskID>
  status:      JobState
  created_at:  unix_ms
  created_by:  "slicer" | "delta_watcher" | "operator"
}

JobState:
  ACTIVE    — tasks are running
  DONE      — all tasks DONE
  CANCELLED — operator cancelled. All tasks → CANCELLED.
  FAILED    — all tasks exhausted max_attempts
```

Cancelling a Job sets all non-terminal Tasks to CANCELLED and sends
CANCEL signals to all agents currently holding CLAIMED tasks within
that Job.

---

## 9. Tasks

### 9.1 Task structure

```
Task {
  id:              UUID
  job_id:          UUID
  position:        Vector3
  required_caps:   Set<CapabilityID>
  coalition_size:  int               // default: 1
  target_state:    bytes             // domain-defined
  tolerance:       bytes
  depends_on:      List<TaskID>
  not_before:      unix_ms | null    // earliest execution time
  priority:        int
  priority_cap:    int               // max priority after escalation
  timeout:         seconds
  deadline:        unix_ms | null
  status:          TaskState
  attempt_count:   int
  max_attempts:    int | null        // null = unlimited
}
```

### 9.2 Task states

```
PENDING   — waiting for depends_on or not_before
OPEN      — available for claiming
CLAIMED   — taken by agent or coalition
DONE      — verified complete
FAILED    — returned to OPEN after timeout or TASK_FAIL
CANCELLED — terminal. Never reopened.
EXHAUSTED — attempt_count reached max_attempts. Terminal.
            Broadcaster emits DELTA_ALERT with reason "task_exhausted".
```

### 9.3 Priority escalation

```
On each FAILED → OPEN transition:
  task.priority    = min(task.priority + 1, task.priority_cap)
  task.attempt_count += 1

  if max_attempts set and attempt_count >= max_attempts:
    task.status → EXHAUSTED
    Broadcaster emits DELTA_ALERT
```

### 9.4 Wait tasks

Time-based blocking (curing, drying, cooling) is expressed as a WAIT
task with no required_caps and a not_before timestamp. The Broadcaster
transitions WAIT tasks to DONE automatically when not_before is reached.
No agent action required.

```
Example — concrete curing:

Task {
  id:            "cure-42"
  required_caps: []
  not_before:    T + 28800s       // 8 hours
}

Task {
  id:            "next-layer"
  depends_on:    ["cure-42"]
}
```

### 9.5 Task assignment — pull model

```
Agent requests task:
  required: agent.capabilities ⊇ task.required_caps
  required: all task.depends_on are DONE
  required: now ≥ task.not_before
  preferred: nearest task to agent.position

Broadcaster assigns:
  marks task CLAIMED
  records claimed_by = agent.id
  starts timeout countdown

If timeout expires:
  task → OPEN (priority escalates, see 9.3)
```

### 9.6 Task cancellation

```
Operator cancels task_id or job_id:
  Broadcaster marks task(s) CANCELLED
  Sends CANCEL signal to agents holding CLAIMED tasks
  Agent: complete atomic action → TASK_FAIL → IDLE
  CANCELLED is terminal
```

### 9.7 Coalition tasks

```
Task with coalition_size > 1:
  First qualifying agent → LEAD role
  LEAD emits coalition_request to local zone
  Required agents respond → coalition formed
  LEAD coordinates execution
  On completion → LEAD dissolves → all agents IDLE
```

### 9.8 Capability deficit

Task remains OPEN with escalating priority. Broadcaster emits
DELTA_ALERT with reason CAPABILITY_DEFICIT. Swarm continues other work.

---

## 10. Delta Watcher

The Delta Watcher monitors the sector Observation Store for unplanned
deltas — observed states that differ from expected states with no
corresponding active task.

### 10.1 Delta Rule structure

```
DeltaRule {
  id:        UUID
  name:      string
  priority:  int                    // rule evaluation order

  condition {
    observed_type:   string | null  // match on measurement classification
    position_zone:   BoundingBox | null
    min_confidence:  float
  }

  action: CREATE_TASK {
            required_caps:  Set<CapabilityID>
            priority:       int
            timeout:        seconds
          }
        | ALERT_OPERATOR
        | IGNORE
}
```

Rules are defined by the operator before swarm activation.
They are evaluated in priority order — first match wins.

### 10.2 Detection cycle

```
Delta Watcher runs at configurable interval (default: 1s):

  for each cell in sector Observation Store:
    current  = aggregate(observations for cell)
    expected = target_state from active job plan

    if |current − expected| > tolerance:
      if active task covers this cell → skip
      else:
        evaluate rules in priority order
        first match → execute action
        no match    → emit DELTA_ALERT
```

### 10.3 DELTA_ALERT

```
DELTA_ALERT {
  position:        Vector3
  observed_state:  bytes
  expected_state:  bytes
  confidence:      float
  timestamp:       unix_ms
  reason:          "no_rule_match"
               | "capability_deficit"
               | "task_exhausted"
               | "job_failed"
}
```

DELTA_ALERT is delivered to the operator interface. The swarm does
not pause waiting for operator response.

---

## 11. Message types

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
                 used for neighbor discovery and reliability tracking
```

---

## 12. Environment stability

Environmental changes are handled uniformly. There is no special mode
for unstable environments.

```
External change (storm, erosion):
  New observations dominate via freshness weighting
  DONE tasks with new delta automatically reopen
  Delta Watcher may generate reactive tasks
  Swarm resumes without operator intervention

Agent-caused change:
  Recorded as SIDE_EFFECT (reduced weight)
  Neighboring cells marked DISTURBED during execution
  Rescanned after task completion
```

---

## 13. Failure handling

```
Silent failure (agent stops responding):
  Detected via missing AGENT_STATUS heartbeat
  Neighbor TTL expires → removed from neighbor table
  CLAIMED tasks return to OPEN via timeout

Byzantine failure (agent sends false data):
  Detected via physical verification by neighbors
  reliability(agent) decreases toward exclusion threshold
  No explicit voting — physics decides

Broadcaster failure (temporary):
  Agents enter offline mode (see 7.5)
  Work continues on cached tasks
  Sync on reconnect

Broadcaster failure (permanent):
  New Broadcaster recovers from last checkpoint
  CLAIMED tasks past timeout return to OPEN on recovery
  Delta Watcher resumes from last known sector state
```

---

## 14. Perception integration

ROSA does not define how agents sense the environment. The measurement
field in Observation accepts any byte encoding.

```
SLAM output      → point cloud or pose estimate encoded as bytes
Lidar scan       → processed surface geometry
Camera + ML      → object classification results (YOLO or similar)
Chemical sensors → concentration values at position
IMU              → velocity and orientation data

Integration pattern:
  Perception library produces reading
  Domain adapter encodes reading → bytes
  Agent writes Observation with encoded measurement
  Protocol treats measurement as opaque
```

---

## 15. What this spec does not define

Intentionally left to implementation:

- Physical form of agents
- Communication medium (radio, optical, acoustic, other)
- Zero point establishment method
- Internal planning and path-finding algorithms
- Observation measurement schema (domain-owned)
- Slicer implementation and task decomposition logic
- Federation protocol between installations
- Delta Rule evaluation engine internals
- Any domain-specific behaviour

---

*ROSA Protocol · open specification · implementation proprietary*
*Budapest, 2025 · feedback via GitHub issues*