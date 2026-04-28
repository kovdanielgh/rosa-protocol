# ROSA

> Coordination protocol for physical agent swarms.
> Where environment is memory. Where physics is truth.

![spec](https://img.shields.io/badge/spec-v0.5--draft-gray)
![protocol](https://img.shields.io/badge/type-protocol-blue)
![license](https://img.shields.io/badge/spec-open-green)

---

## What is ROSA?

ROSA is a protocol for coordinating swarms of physical agents — robots,
drones, or any autonomous hardware — without a central server, without
global state, and without human supervision during execution.

The key insight: agents share information through the environment itself.
The physical world becomes shared memory.

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

## Reference simulation

8 agents flatten an unstructured surface in 60 seconds.
No global map. No central commander during execution.

[link to demo video](https://github.com/user-attachments/assets/2ec9ca81-a773-4ddf-9784-3e518cfffb63)

---

## What's in this repo

- [SPEC.md](SPEC.md) — protocol specification (v0.5 draft)
- [PHILOSOPHY.md](PHILOSOPHY.md) — three principles in depth

---

*This is a blueprint. The specification describes how a ROSA-compliant
system should behave — not the limits of any current implementation.
Core protocol implementation is proprietary; specification and reference
examples are open.*

*Feedback via GitHub issues.*
