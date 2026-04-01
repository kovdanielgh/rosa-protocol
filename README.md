# ROSA

> Decentralized coordination protocol for physical agent swarms.  
> Where environment is memory. Where physics is truth.

![version](https://img.shields.io/badge/version-0.1--alpha-gray)
![protocol](https://img.shields.io/badge/type-protocol-blue)
![license](https://img.shields.io/badge/spec-open-green)

---

## What is ROSA?

ROSA is a protocol for coordinating swarms of physical agents — robots,
drones, or any autonomous hardware — without a central server, without
global state, and without human supervision.

The key insight: agents don't need to communicate with each other directly.
They communicate through the environment itself. The physical world becomes
shared memory.

---

## Three principles

**01 — Environment as memory**  
Agents don't maintain a global map. They read and write the physical
environment directly. State lives in the world, not in any single node.

**02 — Local consensus**  
No agent needs global knowledge. Each agent reaches agreement only with
its neighbors. Global coherence emerges from local interactions — like
termites building cathedrals.

**03 — Physics as verification**  
Agents verify their own actions by scanning the environment before and
after. No external quality check. Reality itself is the authority.

---

## Why this matters
```
Housing crisis      → swarm builds
Ocean microplastics → swarm collects  
Wildfire zones      → swarm restores
Lunar base          → swarm constructs
Mars                → swarm terraforms

same protocol. different agents. different world.
```

---

## Proof of success

N agents execute a physical task in an unstructured environment —  
no central server, no human attendance, no predefined map.

## Minimal demo

5 agents synchronously flatten a surface using only local communication
and environmental feedback.

---

## What's in this repo

- `SPEC.md` — protocol specification (what, not how)
- `PHILOSOPHY.md` — three principles in depth
- `examples/simulation` — reference 2D simulation
- `docs/` — architecture overview and research context

---

*Core protocol implementation is proprietary. This repository contains
the open specification and reference examples.*  
