# Pattern S — Single Writer Governance

**Core idea:**  
Each critical data source or subsystem has exactly **one authoritative writer** at any given time.  
All other parts of the system **read from it** or send **intents**, but do not mutate it directly.

This removes “dueling writers” that cause race conditions, state drift, and corruption.

---

## Why It Matters

When multiple modules/classes/threads mutate shared state, you get:

- freezes and UI jank  
- desynchronized views  
- hard-to-reproduce bugs  
- corrupted payloads  
- “works on my machine” ghosts  

A single writer turns that chaos into a predictable flow:

- One place to inspect  
- One place to instrument  
- One place to refactor  
- One place to enforce contracts  

---

## What It Looks Like in Practice

### Ownership contract

For each shared resource, declare:

> “This module owns all writes. Everything else is read-only or sends intents.”

Examples:

- `WebGLBackground` is the **only** writer to geometry/materials.  
- `TheaterDirector` owns scene graph mutations; `BeatBus` only emits events.  
- A `SessionStore` owns session state; components dispatch actions but never mutate state directly.

### Write pathway

All writes go through a governed API:

- a dispatcher (`dispatch(action)`)  
- a reducer function  
- a command queue  
- a dedicated “writer” method on the owner class  

No side-door setters. No `someGlobal.foo = …` from random modules.

### Read-only everywhere else

Consumers get:

- snapshots  
- selectors  
- query methods  

…but never a direct mutable reference.

---

## Guardrails

To keep Pattern S from eroding over time:

- **Runtime assertions** – assert on the caller when a write happens from a non-owner module.  
- **CI gates** – tests or lint rules that fail if certain files touch a governed resource.  
- **Ownership manifest** – a simple doc mapping “Resource → Single writer”.

---

## How To Implement Pattern S (Step-by-Step)

1. **Map shared resources**  
   - scene graph  
   - global state stores  
   - cache layers  
   - session/auth state  
   - event buses  
   - render buffers  

2. **Assign a single writer per resource**  
   Name the owner module explicitly. Example:

   > “`WebGLBackground` is the only writer of BufferGeometry and ShaderMaterial uniforms.”

3. **Build write funnels**  
   - Use one dispatch function or method per resource.  
   - Validate payloads (types, ranges, invariants).  
   - Log every mutation (at least in dev).

4. **Instrument mutations**  
   - Add probes around the write funnel.  
   - Include: timestamp, caller, payload summary.  
   - This becomes your “black box flight recorder.”

5. **Enforce the pattern**  
   - Add runtime checks: if caller !== owner, throw or warn.  
   - Add simple tests / lint rules that forbid direct mutation outside owner files.

6. **Document ownership**  
   - Create `docs/ownership.md` listing each resource and its single writer.  
   - Make it the first place engineers check before adding new code.

---

## How It Reduces Incidents

- **Removes conflicting writes** → fewer freezes, desyncs, “ghost bugs.”  
- **Predictable state pipeline** → debugging becomes “look at the funnel,” not “grep the whole repo.”  
- **Safer changes** → new behavior is added via the governed funnel, not by poking shared state ad hoc.  

---

## Talking Points for Stakeholders

When explaining Pattern S to CTOs / VPs / leads:

- **Reliability:**  
  “We remove an entire *class* of race conditions by having one canonical writer per critical state.”

- **Speed:**  
  “Debug cycles shrink because there’s only one place to inspect and instrument.”

- **Compliance / auditability:**  
  “Every mutation goes through a logged funnel; we can show an evidence trail.”

- **Reusability:**  
  “Once we establish this pattern for one subsystem, we can reuse it across others, boosting velocity without sacrificing stability.”

---

## Pattern S in MetaCurtis

In the MetaCurtis engine:

- `WebGLBackground` is the **single writer** for WebGL geometry, materials, and draw ranges.  
- `TheaterDirector` / `ConsciousnessEngine` emit high-level `RENDER_DIRECTIVE` & `MORPH_PROGRESS` via `BeatBus`.  
- No other module directly mutates buffers.

This is one of the reasons the engine can handle 15k+ particles at 60 FPS without collapsing under AI-generated code.
