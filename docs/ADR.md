# Architecture — Agentic Platform

> **Document type:** Architecture Decision Record (ADR-style). This is **not** the README.
> The README says *what this is and how to run it*; this doc says *how it's built and why*.
> Revise it the moment a decision changes — a stale architecture doc is worse than none.

A user-scoped AI support agent with authenticated, authorized, identity-propagating tool
access. Monorepo of separately-deployed services — Streamlit UI, MCP client, MCP server,
backend, user-management service (UMS) — fronting an LLM agent that can read device state
and issue control commands to field hardware.

---

## Thesis

**The LLM must never carry authority.** Identity lives on the transport, never in tool
arguments the model can read or write. Authorization is enforced at the backend — the one
layer assumed to be behind no attacker. Every other check is a fail-fast courtesy.

> _Structure enforces what prompts only request._

---

## 1. Component view

```
   ┌────────────────┐        ┌────────────────┐        ┌────────────────┐
   │  Streamlit UI  │        │   MCP Client   │        │   MCP Server   │
   │  (untrusted    │  chat  │  (agent brain, │  MCP /  │  (OAuth        │
   │   edge)        │───────▶│   Gemini ReAct)│  HTTP   │   resource     │
   │  login/logout  │        │                │────────▶│   server)      │
   │  role-aware UI │        │  LLM never     │ Bearer  │  verifies aud, │
   └───────┬────────┘        │  sees token    │ on hdr  │  sig, roles    │
           │                 └────────────────┘         └───────┬────────┘
           │ user JWT (aud=mcp)                                  │ OBO exchange
           ▼                                                     │ (per session,
   ┌────────────────┐                                            │  cached 10m)
   │      UMS        │◀───────────────────────────────────────────┘
   │  authN + authZ  │   mint aud=mcp @login · exchange→aud=backend
   │  ┌───────────┐  │   live freshness check on writes
   │  │  UMS DB   │  │                                            │ token
   │  │ creds,    │  │                                    aud=backend,
   │  │ roles     │  │                                    sub + roles
   │  └───────────┘  │                                            ▼
   └────────────────┘                                   ┌────────────────┐
                                                         │    Backend     │
                                                         │  ENFORCEMENT   │
                                                         │  validates     │
                                                         │  token, role,  │
                                                         │  writes audit  │
                                                         │  ┌──────────┐  │
                                                         │  │Domain DB │  │
                                                         │  │ devices, │  │
                                                         │  │ downlinks│  │
                                                         │  │ audit    │  │
                                                         │  └──────────┘  │
                                                         └───────┬────────┘
                                                                 ▼
                                                           physical devices
```

No cross-DB joins. UMS DB = credentials + roles. Domain DB = devices, downlink queue, audit.
Linking "who owns what" is application-layer, not a foreign key — the trade accepted for a
clean credential/domain boundary.

---

## 2. The auth spine — a downlink request end to end

```
User     UI        MCP Client    MCP Server        UMS           Backend
 │ login  │            │             │              │              │
 │───────▶│──────────────── authenticate ──────────▶│              │
 │        │◀──────── user JWT (aud=mcp, sub, roles) ─│              │
 │"reset  │            │             │              │              │
 │ dev-3" │── chat ───▶│             │              │              │
 │        │            │ JWT on Authorization header │              │
 │        │            │ (LLM never sees it) ───────▶│              │
 │        │            │             │ verify sig + aud=mcp + role  │
 │        │            │             │── OBO exchange ▶│            │
 │        │            │             │  (1st call/session,         │
 │        │            │             │   cached 10m)   │            │
 │        │            │             │◀── aud=backend token ───────│
 │        │            │             │── WRITE? live freshness ───▶│
 │        │            │             │◀── role still valid? ───────│
 │        │◀── confirm? ─────────────│  propose (no send)          │
 │ click  │            │             │              │              │
 │Confirm │── send ───▶│── send_downlink ──────────────────────────▶│
 │        │            │             │              │ validate aud,│
 │        │            │             │              │ role, enqueue│
 │        │            │             │              │ write audit  │
 │        │◀─────────────────── queued ───────────────────────────│
```

**Why each link is load-bearing**

- **JWT on the header, not a tool arg** — anything the LLM can read, a prompt injection can
  forge. Identity must live where the model cannot reach it.
- **MCP server verifies `aud=mcp`** — a token minted for another service is rejected
  (RFC 8707 audience binding). The server is an OAuth resource server, not a blind pipe.
- **OBO exchange, not pass-through** — the MCP spec forbids transiting a foreign-audience
  token (confused deputy). The backend is a separate audience → needs its own `aud=backend`
  token that still carries `sub` + roles.
- **Live freshness on writes, at the backend** — a write that touches hardware re-checks the
  user's role at the UMS *before* proceeding. The check lives at the backend (next to the
  action), not at the MCP hop — enforce at the resource, not in the pipe.
- **Human confirm after the role gate** — role ("allowed at all?") precedes HIL ("confirm
  this instance?"). A viewer never sees a confirm button.
- **Backend is final** — if every other check were bypassed, the backend still validates
  `aud`, reads the role from verified claims, refuses, and writes the audit row from the
  token, not from a request field.

---

## 3. Decisions & trade-offs

Each entry carries: options weighed · choice · trade accepted · falsifier (what would make
it wrong). Nothing here is "clean" — every choice costs something, named on purpose.

### 3.1 Repo / deploy / data topology
- **Choice:** Monorepo · separate deployables · two DBs (UMS, Domain).
- **Why:** Monorepo for atomic cross-cutting changes (one auth-contract change moves all
  services in one PR). Separate deployables for independent lifecycles / blast radius. Two
  DBs for different security postures.
- **Trade:** No foreign keys across the DB boundary — referential integrity is app-layer.
- **Falsifier:** Monorepo is wrong the day these need independent release cadences owned by
  separate teams. Not soon, for a solo engineer.

### 3.2 Identity transport
- **Choice:** User JWT as `Authorization: Bearer` on the MCP transport; never a tool argument.
- **Why:** The LLM must never hold a credential. Transport-level auth keeps the token in
  server-side request context, out of the model's reach.
- **Trade:** Ties the MCP layer to an OAuth-capable transport (streamable HTTP), not stdio.
- **Falsifier:** If identity ever rides as a tool arg the model fills in, the agent is back
  in the auth path and the thesis collapses.

### 3.3 No token pass-through _(spec-confirmed)_
- **Choice:** MCP server does **not** forward the user's token downstream; it obtains a
  separate `aud=backend` token via OBO exchange that still carries `sub` + roles.
- **Why:** MCP spec (2026-07-28): a server MUST NOT transit a foreign-audience token
  (confused deputy + audience binding). Backend is a distinct resource → distinct audience.
- **Trade:** An exchange step and a token-minting dependency on the UMS.
- **Falsifier:** If the backend ever accepts an `aud=mcp` token, audience binding is broken
  and any service's token works anywhere.
- **⚠ Recorded wrong turn:** the mentor first recommended plain pass-through on YAGNI/latency
  grounds. That was **wrong** — the spec forbids it. Caught by reading the dated MCP spec
  directly rather than trusting recall. The falsified entry stays: it's proof the process
  works, not just the outcome.

### 3.4 Downstream token minting & caching
- **Choice:** MCP server mints the `aud=backend` token server-side via OBO exchange; cached
  per session; TTL 10 min; cache key = `sub` from the **signature-verified** token.
- **Why:** Per-request exchange is a hot-path UMS dependency and latency cost; both backend
  tokens on the client widens attack surface. Per-session cache pays the UMS hop ~once per
  TTL window; the backend token never touches the client.
- **Trade:** Up to 10 min of staleness — a just-demoted/fired user's cached token still
  grants its old power for the window.
- **Falsifier:** Keying the cache on anything mutable or client-supplied (email, username,
  a request field) lets one user read another's token. Key MUST be the verified `sub`.

### 3.5 Revocation: point-of-use freshness, not event-driven invalidation
- **Choice:** Reads ride the 10-min cache. Writes that affect the physical system do a
  **live UMS freshness check before human confirmation**. No event bus to invalidate caches.
- **Why:** Event-driven invalidation means every role change pushes to every cache — many
  moving parts, fails open if an event is missed. Point-of-use freshness pulls current state
  only on the rare, dangerous action — simple, fails closed, cost paid on <1% of calls.
- **Trade:** A UMS hop on every write. Accepted because writes are rare and safety-critical;
  reads stay cache-fast.
- **Falsifier (closed):** Fired operator at 14:00, cached token valid to 14:02, fires a
  downlink at 14:01 → write path's live freshness check sees role revoked → rejected. The
  stale window never reaches hardware because the write path doesn't trust the cache.

### 3.6 UMS-outage policy — identity is absolute, freshness degrades
- **Choice:** Two independent gates. **Identity** (token valid? — signature, `exp`, `aud`)
  is checked offline against the UMS public key on **every** request; invalid → reject,
  read or write, regardless of UMS state. **Freshness** (role still current?) is checked
  live at the backend on **writes only**.
- **UMS-down behavior:**

  | | token valid | token invalid |
  |---|---|---|
  | **read** | allow (identity only; freshness never needed) | reject |
  | **write, UMS up** | allow after live freshness check | reject |
  | **write, UMS down** | allow, bounded-stale | reject |

- **Why writes degrade open when UMS is down:** not because freshness is "optional" — it is
  load-bearing — but because the **failure costs are asymmetric for a fire-detection system.**
  Blocking writes during an auth outage means operators cannot command wildfire hardware;
  allowing a valid token's write means a user demoted *during* the outage keeps their role
  for ≤ the token TTL. For Dryad, losing hardware control is the worse failure, so we accept
  the bounded staleness. (Reads have no such tension — identity alone suffices, so they just
  allow.)
- **Guardrails (required for this to be defensible):** (1) write-token TTL short, so
  "bounded-stale" is genuinely bounded; (2) every degraded-mode downlink is audited with
  `freshness=degraded, reason=ums_unavailable`, so the window is recoverable when UMS returns.
- **Trade:** During a UMS outage, a mid-outage demotion isn't enforced until the token
  expires. Bounded and audited.
- **Falsifier:** "Identity is checkable offline" — if token validation ever required a live
  UMS call, a UMS outage would black out identity too, and degrade-open would be waving
  through unverifiable (possibly forged) tokens. It must stay offline-verifiable.
- **Note on framing:** avoid the word "priority" here. Writes aren't "higher priority" than
  reads — they have a different, opposite failure cost during an outage. Priority would
  imply more protection; this is about failure-cost asymmetry.

### 3.7 Token TTLs — two tokens, two problems
- **Choice:** User JWT (`aud=mcp`, browser session) = **60 min**. Backend token
  (`aud=backend`, minted server-side, cached) = **10 min**.
- **Why different:** they solve different problems. The user JWT's TTL is mostly a UX knob
  (re-auth frequency). The backend token's TTL is the **staleness window** — how long a
  demoted/fired user keeps power on the sensitive leg — so it's kept short.
- **Trade (named in security terms, not just UX):** a 60-min user JWT is also a **60-min
  theft/revocation window** — a leaked user token is usable for up to an hour and cannot be
  revoked before it expires. Accepted for v1 of an internal tool; the backend token's 10-min
  TTL independently bounds the dangerous-leg blast radius.
- **Falsifier:** if the user JWT were used to authorize sensitive writes *directly* (it
  isn't — writes go through the backend token + live freshness), the 60-min theft window
  would be unacceptable.

### 3.8 Refresh tokens — deferred to end of v1 _(explicit learning goal)_
- **Choice:** No refresh tokens in v1. Revisit at the end, deliberately, as a topic to learn
  properly rather than copy-paste.
- **Why defer:** refresh tokens are the standard way to get *both* rare logins (long refresh
  token) and a small theft window (short access token) — they dissolve the UX-vs-security
  tension in 3.7. But they add machinery (refresh endpoint, rotation, reuse-detection,
  secure storage). YAGNI for v1; the 60-min no-refresh token is the simpler defensible start.
- **Trade:** v1 carries the 60-min theft window above until refresh lands.
- **Note for when it lands:** the real risk in refresh tokens is that the refresh token is a
  *higher-value credential than the access token it mints* (longer-lived, mints many). The
  craft is in **rotation** (each use issues a new refresh token, invalidates the old) and
  **reuse-detection** (a rotated token used again = theft signal). That — not "how to get a
  new access token" — is where the learning is.

### 3.9 Downlink authority = role, not scope
- **Choice:** Downlink permission is an auth-time role granted by an admin in the UMS. No
  self-service step-up. The user re-authenticates to receive a token with the new role.
- **Why:** The spec's `insufficient_scope`/step-up flow is for re-proving authority you
  already have. Downlink is authority you do NOT have until someone changes who you are —
  granting it at runtime via step-up is a privilege-escalation vector.
- **Trade:** Role changes aren't instant for the user — they re-login to pick up the role.
  Acceptable; promotions are infrequent and admin-mediated.
- **Falsifier:** If a user could elevate their own role through a runtime flow without admin
  action, the role gate is meaningless.

---

## 4. Three-layer authorization

| Layer | Role in auth | Trust assumption |
|---|---|---|
| **UI (Streamlit)** | Hide actions the user can't take; login/logout | **Untrusted** — courtesy only, attacker skips it |
| **MCP server** | Verify JWT sig + `aud=mcp`; refuse to pipe forged tokens | **Fail-fast** optimization, not the boundary |
| **Backend** | Validate `aud=backend`, enforce role, live freshness on writes, write audit | **Enforcement** — the one layer safety rests on |

Mental model that keeps this honest: the backend is the only real security boundary; the UI
and MCP checks exist to fail early and cheaply, and the design assumes an attacker bypasses
both.

---

## 5. Open questions / not yet decided

_Resolved: UMS-outage policy → §3.6. Freshness location → backend (§3.3, §3.6).
TTLs → §3.7 (user 60m / backend 10m). Refresh tokens → §3.8 (deferred, learning goal)._

- ~~**Audit schema.**~~ RESOLVED — see §5b (Request table: actor/target snapshots, action,
  result, freshness, append-only).

## 5a. Project structure decisions

Project name: **understory** — the forest layer beneath the canopy that governs what gets
through; successor to the `canopy` capstone. (Lineage goes in the description, not the name.)

### 5a.1 Monorepo, one language (Python/Django)
- **Choice:** single monorepo `understory/`; all services Python/Django.
- **Why:** polyglot solves no present use case here and adds cost (YAGNI). All-Python also
  dissolves the hardest layout problem — one shared token-verifier, no risk of two drifting
  copies of security-critical code.
- **Trade:** gives up best-tool-per-service; accepted, no service needs a different language.
- **Falsifier:** wrong the day a service has a need Python genuinely can't meet.

### 5a.2 Independent dependencies — per-service `pyproject.toml`, one shared dev venv
- **Choice:** each service declares only its own deps in its own `pyproject.toml`; a single
  shared venv at root installs all of them (editable) for dev convenience.
- **Why:** per-service declaration is what "independent deployables" requires — each service's
  Docker image installs only its own `pyproject.toml`, so images stay lean and isolated
  (a CVE in Streamlit is not a CVE in the UMS image). The shared venv is a *dev* convenience
  and does not couple declarations.
- **Trade:** editing `library` needs a reinstall across the venv; minor.
- **Falsifier:** a root-only `pyproject.toml` would couple the thing we decided to keep
  independent — UMS would ship every service's deps. Rejected for that reason.
- **Note:** "shared venv" (dev convenience) and "shared dependency declaration" (a coupling)
  are different things — keep the venv shared, keep declarations per-service.

### 5a.3 Boundary enforcement — import-linter (not per-service venvs)
- **Choice:** `.importlinter` at root enforces "each service may import `library` and
  `contracts`, but no service may import another service's internals." Level 2 (lint in CI),
  not Level 3 (physically separate venvs).
- **Why:** all-Python removes the language barrier that used to prevent cross-service imports,
  so the boundary needs an enforcer or it erodes. Lint is cheap and catches it at PR time.
  Physical isolation (separate environments) arrives for free at containerization; paying its
  dev-friction cost now isn't justified by any present need.
- **Trade:** the dev venv *could* resolve a forbidden import at runtime; the linter is what
  makes it fail. If the linter isn't run, the boundary isn't enforced.
- **Falsifier:** justifying isolation by "switch a service to another language later" is
  requirement-laundering — polyglot was rejected in 5a.1. The real justification is
  present-day boundary protection, not future swappability.

### 5a.4 library vs contracts — two different shared things
- **Choice:** `library/` = shared *callable code* (token verify, gate pattern, logging).
  `contracts/` = language-neutral *enforced agreements* (JWT claim shape, audience values,
  role names, API schemas).
- **Why:** services *implement* a contract; they *call* a library. The contract documents the
  agreement independent of any one service's code, and survives a future non-Python consumer.
- **Trade:** two shared dirs instead of one; the discipline is worth it.

### 5a.5 Base tree (internals deferred)
```
understory/
├── docker-compose.yml
├── .importlinter          # services → library, contracts only; never each other
├── .gitignore             # includes .venv
├── README.md              # root: orients to the whole system
├── docs/                  # architecture.md
├── contracts/             # language-neutral enforced agreements
├── library/               # shared callable code
├── ui/          (pyproject.toml, README.md)  Streamlit + mcp-client (agent brain)
├── ums/         (pyproject.toml, README.md, migrations/, models/)  Django; symmetric w/ backend
├── backend/     (pyproject.toml, README.md, migrations/, models/)  Django; symmetric w/ ums
└── mcp_server/  (pyproject.toml, README.md)  its own shape (tools, gate)
```
- Symmetry is enforced only across *like* services (ums ↔ backend, both Django-DB). ui and
  mcp_server have their own shapes — forcing one skeleton on all would be symmetry for its
  own sake. Migrations live inside their service (never a shared root dir) — enforces the
  no-cross-DB-coupling rule. README per service + a root README.
- The MCP **client** (agent brain, ReAct loop) lives inside `ui/` as a module, not a separate
  service — it runs in the UI process, holds the user's JWT from login, calls `mcp_server`
  over HTTP. LLM-never-holds-token still holds: the token is in the UI's request context, not
  in tool arguments.

### 5a.6 Meta-lesson (for the engineer, not the architecture)
- Failure mode named: reaching for "fancy" techniques heard elsewhere and building for
  imagined futures. Fix is a three-question test before adding any complexity: (1) what breaks
  *today* without it? (2) can it be added later without a rewrite? (3) am I reaching for this
  because the problem needs it, or because I want to use it? If (1) = "nothing," stop.
- Second pattern: "we already decided X" drifted twice this session toward *more* coupling
  (merge DBs; root pyproject.toml), while the real earlier decision was the less-coupled one.
  Rule: "we already decided that" is a claim to verify against this log, not to trust.

## 5b. Data model

Two databases, no cross-DB joins (per §3.1). UMS owns credentials/roles; Domain owns the
request audit. Links across the boundary are **snapshots**, never foreign keys.

### Thinking tool used (reusable)
Every access-control concept answers one of three questions: **membership** ("does this
subject exist in this container?"), **role/permission** ("what can it DO?"), **scope/
assignment** ("WHERE does that capability apply?"). A company's "assignments, subscriptions,
roles, tenancy" are just *their names* for answers to these three. Derive from requirements
by asking which questions the system actually needs — don't copy another system's vocabulary.

Applied to understory: **single-tenant, no site-level scope.** → Role answers Q2 (field on
user). Q3 (scope) has nothing to scope → **no assignment table.** Q1 (membership) has one
implicit org (understory itself) → **no Org/Site/Subscription entities.** All four were
considered and dropped for want of a present-day requirement, not kept because an employer's
system has them.

### UMS DB
```
User
├── id
├── email
├── password_hash
├── role        enum: viewer / operator / admin   (global; single-tenant, so role is a field not a table)
├── status      enum: active / suspended / deleted
│                 active=usable · suspended=reversible-off · deleted=terminal soft-delete
├── created_at
└── updated_at
```
- `status` is ONE enum, not multiple booleans: a user's state is one lifecycle with one
  current position, so illegal combinations (active+deleted) are unrepresentable.
- `deleted` is soft-delete (row stays) because the Request audit references the user — a
  removed user must stay resolvable. `deleted` is terminal (no reactivation); reactivation is
  a forbidden transition out of it.
- Role is an enum, not a roles table: few fixed roles, roles carry no attributes.

### Backend (Domain) DB
```
Request   (append-only audit; one insert per event, terminal on write — never updated)
├── id
├── actor_snapshot    { user_id, email, role-AT-THE-TIME }   frozen at write (§3.6)
├── action            enum: read / downlink
├── target_snapshot   { device_id, name-at-the-time }        device not stored; snapshot, no FK
├── result            enum: served / queued / denied / failed
├── freshness         enum: live / degraded                  (§3.6 UMS-outage flag)
├── created_at        server-set
└── retry_of          nullable FK to original Request — V2 ONLY (added when retry ships)
```
- **Append-only, terminal on write.** The backend does not await the device, so the terminal
  result (`queued`/`denied`/`failed`, or `served` for reads) is known synchronously at
  handling time — the row is born terminal and is never updated. No `pending` state.
- **Device execution is NOT tracked.** `queued` means "backend accepted and enqueued," not
  "hardware acted." If device-outcome tracking is ever wanted, it is a *separate* append from
  the device layer, never an update to this row.
- **actor/target are snapshots, not FKs** — append-only history must record facts as they
  were (a demoted user's past downlink must show the role they *had*), and there is no
  cross-DB FK to the UMS user anyway.
- **No raw request/response blob stored** — structured `action` + `target` + `result`, not
  the payload. Audit records *what happened*, not the content (size + sensitive-data + two-
  jobs reasons). "Store the result, not the response."
- **`denied` attempts ARE logged** — a rejected (wrong-role) attempt is exactly what a
  security audit most wants: the attack-detection signal.
- **Retry (V2):** a retry is a NEW row in the SAME table (same entity → same table; a retry
  is another downlink *attempt*, not a different kind of thing). It is NOT an update of the
  failed row (that would falsify history / break append-only) and NOT a separate table
  (identical shape → same table). Only addition: a nullable `retry_of` FK, added when retry
  ships. No schema change needed now — append-only already accommodates it.

### Modeling principles applied (reusable)
- One concept → one field. Two fields that can never legally contradict are one field.
- Split tables by *entity type*, never by *status/variant* of the same entity.
- Append-only audit: a resolved fact is never overwritten; corrections/retries/outcomes are
  new appends. A record's result reflects what *this layer* knows, not what a downstream
  system eventually does.
- Snapshot (freeze at write) vs reference (point at live) — audit history snapshots; it must
  not change when the referenced thing later changes.

## 6. Next decisions (dependency order)

1. **Data model** — both DBs. UMS: users, roles. Domain: devices, downlinks, audit (the
   parked audit schema gets designed here). First, because tools and agent both depend on
   the shapes.
2. **MCP server internals** — tool contracts; how the canopy gate pattern carries over.
3. **Agent loop** — reuse the canopy ReAct client, or rethink.
4. **Refresh tokens** (§3.8) — end of v1, as a learning stop.
- **Session termination on firing.** Does firing a user also kill their active UI session?
  If yes, the stale-window risk shrinks further.
- **TTL by action class.** 10 min confirmed fine for reads and (via the freshness check) for
  writes — revisit if a write class appears the freshness check can't cover.

---

*Reference architecture derived from the design sessions. Every decision is the engineer's
own, pressure-tested against its trade-off and falsifier.*
