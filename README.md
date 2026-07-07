<p align="center">
  <img src="logo.png" alt="NornGate" width="250">
</p>

# NornGate — AI Multi-Purpose Orchestrator

> **Deterministic, fail-closed gating for every consequential action.** Verified *before* any side effect occurs — fully audited, framework-agnostic, and built for autonomous systems.

---

NornGate is a deterministic, fail-closed control plane that sits in front of agent tool execution. Most agent systems let actions happen immediately — a retrieval hits storage, a message reaches an API, a workflow mutates state — and control, if it exists at all, is post-hoc. NornGate inverts this: every consequential action an agent attempts must clear a five-gate pipeline before it is committed.

**Core thesis — the runtime is the product.** Inference is COGS. NornGate does not resell tokens; it sells verified outcomes. Billing is per **G4-verified outcome**, never per token — you pay for actions that provably cleared every gate, not for the model calls behind them.

> **Silence means no.** If any gate cannot *explicitly* approve an action, the action stops. Default-deny is the ground state of the runtime, not a setting you turn on.

---

## The five-gate pipeline

Every request traverses **G0 → G4** in order. A gate may only pass an action along; it can never grant a capability a later gate would withhold. Any gate that cannot explicitly approve terminates the request and writes the verdict to the **Urd ledger**.

```
Agent request → G0·Ingress → G1·Policy → G2·Approval → G3·Sandbox → G4·Commit → Side effect committed
                     ↓ deny       ↓ deny       ↓ deny         ↓ deny        ↓ conflict
                   Denied       Denied       Denied         Denied      Denied
                   (verdict written to Urd)
```

| Gate | Name | Responsibility | Denies when |
|------|------|----------------|-------------|
| **G0** | Ingress | Authenticates the caller; validates API keys; enforces rate limits and network allowlists. | Caller unauthenticated, key invalid, over rate limit, or off-allowlist. |
| **G1** | Policy | Evaluates ABAC/RBAC rules against agent identity, resource type, and request context. | No rule explicitly permits the *(identity, resource, action)* tuple. |
| **G2** | Approval | Requires explicit human or automated consent for high-risk actions, drawn against an **approval budget**. | Required consent is absent, or the approval budget is exhausted. |
| **G3** | Sandbox | Executes the action in an isolated, ephemeral environment with no permanent side effects. | Dry-run fails, violates an invariant, or produces a disallowed effect. |
| **G4** | Commit / Arbitration | Resolves conflicts deterministically when agents target the same resource, commits the verified side effect, and records the billable outcome to the ledger. | Irreconcilable conflict, or a commit precondition no longer holds. |

> **G4 is where the economics live.** It is not only arbitration — it is the *commit* point. The side effect becomes real here, and the same event that commits the action records the verified, billable outcome. This is why NornGate can bill per outcome rather than per token.

**Approval-budget note.** G2 is the expensive gate — it can involve a human. The approval-budget vocabulary keeps G2 transits under ~5% of total traffic: most actions are pre-authorized by G1 policy and never reach a human.

### Worked example — an agent updates a user record

1. **G0** authenticates the agent's scoped key and confirms the source subdomain is on the allowlist.
2. **G1** finds an ABAC rule permitting `agent:huginn` to `write` on `resource:user-record` within the caller's tenant — pass.
3. **G2** sees the action tagged high-risk (PII write); a standing automated-consent policy covers it and debits one unit from the tenant's approval budget — pass.
4. **G3** applies the write to a sandboxed shadow of the record, confirms schema and invariants hold, no disallowed effects — pass.
5. **G4** confirms no competing writer holds the record, commits the write, and records the outcome to Urd. The tenant is billed for **one verified outcome**.

Flip any single check to *unknown* and the request stops at that gate — no partial writes, no "probably fine."

---

## Why gate before execution

Agents can read, write, execute, and decide. That capability set is exactly what makes post-hoc control insufficient — by the time you observe the side effect, the mutation has already happened. Pre-execution gating prevents concrete failure modes:

- **Unauthorized action** — an agent holding a write/exec tool can act outside intent. *G1* refuses any *(identity, resource, action)* tuple no rule explicitly permits.
- **Race conditions / double-commit** — two agents targeting one resource corrupt state. *G4* arbitrates deterministically before commit.
- **Blast radius from a bad plan** — a wrong action that looks valid still executes. *G3* runs it in isolation first; nothing escapes the sandbox.
- **Missing provenance for regulated work** — auditors need who did what, when, and under which policy. *Urd* records every verdict immutably; the ledger *is* the audit trail.
- **The overlooked permit** — the design argument for default-deny, and the reason the platform is named for the Norns. In the Edda, Baldr is made invulnerable by extracting a promise from all things not to harm him — except mistletoe is overlooked, and that single gap kills him. A default-allow policy with a denylist *is* that mistletoe: the one capability you forget to forbid is the one that ends you. NornGate is allowlist-first — nothing acts unless explicitly permitted — so a forgotten rule fails safe (denied), not fatal (allowed).

---

## Architecture — Norse cosmology as a system model

The cosmology is load-bearing, not branding. Each name maps to a concrete control, trust boundary, or recovery pattern, sourced to the Eddic corpus.

- **Yggdrasil — the single control spine.** One service-mesh core through which all requests pass: deterministic routing, one observable path, one place to enforce every gate. If traffic cannot be metered and refused in a single place, the gates are advisory. Yggdrasil makes them mandatory.
- **The Nine Worlds — trust zones.** Each realm is a trust zone bound to a subdomain — *subdomain = trust zone* — so the topology *is* the threat model: core control plane, user-facing services, untrusted execution, storage, and recovery each live in their own realm with explicit boundaries between them.
- **The Norns — the system record.** Named for what was, what is becoming, and what shall be:
  - **Urd** (*what was*) — append-only, immutable audit ledger; every gate verdict lands here.
  - **Verdandi** (*what is*) — live metrics and telemetry.
  - **Skuld** (*what shall be*) — forecasting and anomaly detection.
- **Ragnarök — failure planning.** Recovery is tested before it is needed: full-system chaos injection, restoration from in-tree backups, and replay of state from the Urd ledger. A DR path that has never been exercised is a hypothesis, not a control.

---

## Key properties

Enforced by the runtime, not left to application code:

- **Deterministic** — identical *request, policy, and state* yield identical gate verdicts. The verdict is a pure function of all three, not of the request alone.
- **Fail-closed** — unknown, timed-out, or unapproved resolves to *denied*.
- **Fully audited** — every decision is written to Urd, a centralized append-only ledger secured by a signed hash chain (tamper-evident; PII payloads crypto-shreddable for erasure).
- **Framework-agnostic** — works in front of any agent framework or language, *provided every side-effecting path is routed through it and agents hold no raw credentials*. The control lives in key custody, not your agent code.
- **Gated side effects** — no permanent change without explicit passage of all five gates.

---

## Who it is for

- **Platform / infrastructure teams** running autonomous systems that need one enforceable control point across frameworks.
- **ML / applied-AI engineers** who need agents to *act* on production systems without hand-rolling authorization and audit per integration.
- **Security engineers** who need default-deny at the tool-execution boundary — not model-level guardrails an LLM can talk its way past.
- **Compliance / risk owners** who need provable provenance for regulated workflows: the ledger as evidence.

---

## Iconography

Section headings are inspired by Elder Futhark runes — each maps to its section's purpose:

| Rune | Meaning | Section |
|------|---------|---------|
| Nauðiz | need, necessity, constraint | the platform mark — the binding gate |
| Thurisaz | thorn, gateway, warding force | the five gates |
| Algiz | protection, defense | why gate before execution |
| Eihwaz | the yew, the world-tree axis | architecture / the Spine |
| Ingwaz | enclosure, completion, stored potential | key properties |
| Wunjo | kinship, fellowship | who it is for |
| Othala | ancestral home, heritage | Introduction (wiki) |
| Raidō | the ride, the road, a journey | Getting Started (wiki) |
| Kenaz | torch, knowledge, insight | Core Concepts (wiki) |
| Tiwaz | Týr — guidance, order, justice | Guides (wiki) |
| Mannaz | humankind, mind, the record | Reference (wiki) |
| Ansuz | Óðinn — the word, wisdom | sources |

---

## Sources

- **Eddic corpus (naming doctrine).** Snorri Sturluson, *Prose Edda* (*Gylfaginning* — the Norns, Yggdrasil, Baldr, Ragnarök); the *Poetic Edda* (*Völuspá* — cosmology and the Nine Worlds). The rune meanings follow the Elder Futhark tradition. The mapping from these sources to platform controls is maintained as a first-class artifact, not flavor text.
- **Internal design docs.** `CORE-STRUCTURE.md` (naming doctrine, realm→subdomain mapping, god→agent registry, Norn telemetry triad) and the platform white paper / light paper (gate taxonomy, per-outcome billing model).

---

**[Wiki](https://github.com/norngate-com/public/wiki)** · **[norngate.com](https://norngate.com)**

© 2026 NornGate Technology Company
