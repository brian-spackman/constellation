# Constellation Architecture Specification

**Author:** Brian Spackman  
**Version:** 0.1  
**Date:** 2026-10-28

> For the motivation, problem statement, and overview of why Constellation exists, see **[`README.md`](./README.md)**. This document specifies the core primitives, authoritative records, rules, and building blocks of a conformant Constellation domain.

---

## 1. Two Primitives (`Entity` & `Contract`) and Two Authoritative Records

Constellation's graph model is built from **two primitives**, each carrying a **`class`** field:

1. **`Entity` (`class: domain | agent | service | shim | projection | human`)** — Every node in the graph (where a `shim` is a protocol/format adapter between mismatched interfaces, and a `projection` is a derived, read-only view computed from the Enrolment Ledger and Contract Journal for discovery or human comprehensibility, stood up either as shared domain substrate or ad-hoc for a specific entity's needs).
2. **`Contract` (`class: stewardship | workflow`)** — Every accountability link (edge) between entities.

Because the two contract classes operate on different timescales, Constellation records them in **two mandatory authoritative records**:

- **The Enrolment Ledger (Slow-Changing Record — Document, Relational, or Graph Database):** Records **`Entity` Declarations** and standing **`stewardship` Contracts** (vertical links). Optimized for compatibility checks, decision-rights lookup, and human-timescale updates.
- **The Contract Journal (Fast-Changing Record — MVCC State-Revision Log / KV + History):** Records ephemeral **`workflow` Contracts** (horizontal links) that may form, hand off, and discharge in milliseconds. Each write appends a full state revision (`revision: N`) keyed by `contract_id` (`parent_id`, `holder`, `accountable_to`, `offered_to`, `terms`, `status`, `outcome`, and `transition_reason`), providing instant latest-state lookup, native compare-and-set (`expected_revision`), and full revision history per contract without maintaining a separate event-verb schema.
  - **Unforgeable provenance for free:** Because the Journal retains every state revision (and `parent_id` chain) starting from `revision: 1`, any downstream entity can inspect a contract's revision history directly in the Journal to verify which surface or entity originally initiated the chain (for example, distinguishing an unauthenticated room microphone from an authenticated mobile client) without relying on mutable provenance headers in payloads.

Beyond the Enrolment Ledger and Contract Journal, all optional domain infrastructure (such as routing recommenders, proactive watchdogs, or compliance monitors) is implemented as **ordinary Constellation entities** that use these exact same primitives and are flagged as substrate.

---

## 2. Accountability Chains and Broken-Link Roll-Up

In Constellation, **accountability**—whether for governing a domain, maintaining a long-lived service, or completing a single piece of work—is represented as **a directed chain of contracts, never as a single assignee field.**

In practical deployments, the **horizontal workflow view** forms a directed graph of nodes with work moving left-to-right across capabilities, while the **vertical stewardship view** over those exact same nodes forms an organizational hierarchy rolling up through accountable agents and managers to a human at the top:

![Two Projections of the Same Graph — Horizontal Workflow vs. Vertical Stewardship](./two-axes-of-accountability.svg)

1. **Every chain exits Constellation at a named `human` Entity.** An AI model or software service cannot hold the terminal end of an accountability chain because it cannot bear external legal, professional, or organizational consequence. In the real world, that chain may continue beyond the designated human through organizational, corporate, or legal structures (`Manager → CIO → CEO → Board`); tracking it past the human boundary is outside Constellation's scope, though optional two-way integrations may connect with external human governance or asset-inventory systems.
2. **Both contract classes intersect orthogonally at every service.** At any moment, a service has a vertical **`stewardship` chain** answering for _whether the service behaves as declared over time_, and a horizontal **`workflow` chain** answering for _whether a specific invocation meets the requester's need_.
3. **Each link in a chain is a contract recording:**
   - `contract_id`, `class` (`stewardship | workflow`), and optional `parent_id` (linking to the upstream contract of the same `class`)
   - `holder` (the entity currently accountable) and `accountable_to` (the upstream entity)
   - `terms` (what the holder commits to do and what counts as completion), plus delegated `decision_rights` on `stewardship` contracts
   - `on_break` (what happens if this link—whether a service or a human—drops out or can no longer fulfill its role)
4. **Broken-link behavior (`on_break`) is executed by the downstream entity and defaults to roll-up and notify.**
   Every entity verifies its own stewardship chain before acting (or is notified by an optional watchdog that an upstream link broke). Whether the break occurs at an intermediate software agent or at a human, **the downstream entity executes its configured `on_break` action**:
   - **`roll_up_and_notify` (Default):** When an intermediate link breaks, `accountable_to` automatically resolves one link upstream (analogous to responsibility defaulting from a departing IT manager to the CIO until the role is filled), emits an immediate notification to that upstream party, and permits downstream services to continue operating uninterrupted until a replacement steward is assigned.
   - **`suspend_and_notify` (Configurable per domain or contract):** For high-consequence links, the downstream entity pauses new work and notifies the upstream party until custody is explicitly re-established.

---

## 3. The Four Structural Building Blocks

### Block 1: `domain` Entities & Subdomains (Governance Boundaries)
A **Domain** is an `Entity` of `class: domain` stood up under a standing `stewardship` chain that exits to a human (e.g., `ops.example → Steward-Agent → Lead-Engineer`). 
- **Subdomains by composition:** Because a domain is an `Entity` governed by a chain of contracts, a **Subdomain** is a `domain` entity whose upstream parent is another `domain` entity—inheriting its parent domain's house rules by default and overriding locally where permitted.
- **What a Domain Declaration specifies:**
  - **Stewardship chain & default `on_break` policy** (`roll_up_and_notify`).
  - **Mandatory record locations:** Where the authoritative **Enrolment Ledger** and **Contract Journal** live.
  - **Optional substrate & domain conventions:** Which optional substrate entities or policies exist in this domain (such as a preferred transport bus, identity/access management like IAM/LDAP, cryptographic contract signing, routing recommenders, or watchdogs) and whether using each is mandatory or optional.

### Block 2: `agent`, `service`, `shim`, `projection`, and `human` Entities & Enrolment
Every actor or view in the domain is enrolled in the **Enrolment Ledger** as an `Entity` (`class: agent | service | shim | projection | human`) with a **Declaration**:
- **Standing `stewardship` contract:** Who owns and tends this entity (`accountable_to`), chaining to a human, plus any delegated **decision rights** granted down that link.
- **Capabilities & Requirements (`provides` / `requires`):** What the entity does, explicit **declared non-scope** (what it will refuse), expected inputs and outputs, completion criteria, and what it requires from the domain. This enables mechanical compatibility checks when importing externally built services that may not natively match a domain's house rules.

**Enrolment and Negotiated Shims (`class: shim`):**
Enrolling an entity checks whether its declaration satisfies the domain's declaration:
- **Compatible:** Direct structural and semantic match (`provides` satisfies `requires`). Recorded in the Enrolment Ledger.
- **Translatable:** Mismatched interfaces or conventions can be bridged by a **Translator Shim (`class: shim`)**. For example, if a domain requires structured logs on a message bus while an imported third-party service only writes local text files, a small log-forwarding shim can bridge the gap. A shim must never exist as unowned glue code: the domain's steward and the service's steward **negotiate who builds and holds the standing `stewardship` contract for the shim**, enrolling it as a first-class `shim` entity in the Enrolment Ledger.
- **Incompatible:** Unbridged conflict. Constellation surfaces the exact mismatch directly to **both accountable stewards** (the **Service Steward** and the **Domain Steward**), **never arbitrates**, and halts enrolment until they establish common ground.

### Block 3: Contracts (`class: stewardship | workflow`)
A **Contract** is an explicit, recorded commitment held by a single accountable entity (`holder`) toward an upstream entity (`accountable_to`).

#### Shared Mechanics Across Both Contract Classes
- **Single holder per contract:** Exactly one entity holds accountability for a given contract at any moment.
- **Two moves—Transfer vs. Subcontract:**
  - **Transfer ("Passing the Torch"):** The receiver accepts accountability for the contract and the sender is **released**—used both for runtime work handoffs and for permanently transferring operational stewardship of a service to a new owner without leaving stale asset records.
  - **Subcontract:** The current holder **retains** the parent contract and spawns a linked child contract (`parent_id`)—used both for runtime fan-out and for temporary stewardship delegation (such as on-call or vacation coverage).
- **Two-phase handoff (Enforced by the Enrolment Ledger & Contract Journal):** Every move is negotiated (`propose → accept / refuse / counter-propose`) and mediated through the Enrolment Ledger (for `stewardship` contracts) or the Contract Journal (for `workflow` contracts): the holder records intent (`offered_to: B`), the receiver records acceptance (`holder: B`), and only then is the sender released. **The Enrolment Ledger and Contract Journal themselves must enforce conditional append (`CAS`) and transition validity**—rejecting any stale revision write and rejecting any acceptance where `offered_to` did not name that entity.
- **Delegated Decision Rights:** When a human or upstream steward delegates to an `agent` or `service`, the `stewardship` contract records the **decision-authority scope** that entity is permitted to approve on its own versus what must escalate up the chain.

#### `class: stewardship` (Standing / Slow-Changing, in Enrolment Ledger)
- Governs the ongoing operation, maintenance, and delegated decision authority of a `domain`, `agent`, `service`, or `shim` over time. Remains active until transferred, subcontracted, or discharged on retirement.

#### `class: workflow` (Ephemeral / Fast-Changing, in Contract Journal)
- Governs a specific runtime invocation accepted against an entity's declaration, living for milliseconds to hours.
- **Discharge & reporting to parent:** Ends only by recording an explicit outcome against its completion criteria in the Contract Journal: `succeeded`, `failed`, `refused`, `cancelled`, or **`unknown_effect`** (when an external effect was attempted but its result cannot be verified, preventing silent retries or false success claims). Because the Journal supports state watches per key, recording the terminal outcome on a child contract automatically notifies the `parent_id` holder watching that child contract.
- **Decision Requests:** When a workflow requires approval beyond the current holder's delegated decision rights, it spawns a Decision Request contract up the chain to an entity holding the required decision authority, with an optional response window and explicit expiry disposition (**silence is never approval**).
- **Null move (Queries):** Pure reads and catalog lookups move no accountability and form no contract.

### Block 4: Projections (`class: projection` — For Comprehensibility)
To ask a human (or steward) to be accountable for an outcome, the system owes them an account they can comprehend. **Projections (`class: projection`) add zero new authoritative data**; they are derived, read-only entities computed from the Enrolment Ledger and the Contract Journal.

**All projections are optional**—they can be deployed as shared domain substrate available to everyone, or created ad-hoc as custom `projection` entities tailored to a specific human's or service's needs:
- **Workflows Are Projections, Not Inputs:** In Constellation, entities discover and select capabilities **at runtime**, ensuring every run automatically takes advantage of the latest enrolled services without rewriting callers. Because the path is discovered dynamically rather than pre-wired, a workflow's full shape is non-deterministic in advance and exists as an **output** reconstructed from `parent_id` links in the Contract Journal once the initiating contract discharges. **In a traditional workflow engine, the workflow is the input; in Constellation, the workflow is the output.**
- **Capability & Decision-Rights Index:** Projected from the Enrolment Ledger so callers can discover who can perform a task or who holds decision authority for a given scope.
- **Stewardship Graph:** Projected from standing `stewardship` contracts to show who owns and tends every entity across domains and subdomains.
- **Comprehension Views:** Re-expressions of journal and observability records tailored to the human at the end of an accountability chain (from a periodic summary digest to a full causal trace).

---

## 4. Non-Negotiable Runtime Rules

1. **Verify before acting:** Before executing work, an entity checks the Contract Journal and Enrolment Ledger to confirm both that it is the active `holder` of the `workflow` contract and that its standing **`stewardship` chain** is intact. If its stewardship chain is broken, it immediately executes its `on_break` rule (`roll_up_and_notify` or `suspend_and_notify`)—meaning governance holds at runtime even with zero background health-check services deployed.
2. **Effect-time authorization via underlying resource permissions:** When an entity performs an external effect, authorization is enforced **at the moment the effect is attempted** by the underlying target system's access controls (API credentials, bus permissions, IAM/LDAP, or optional cryptographic contract signatures)—never assumed merely because an upstream planner proposed the step.

---

## 5. Substrate: Mandatory vs. Optional

### Mandatory Substrate (Required for Any Conformant Domain)

| Mandatory Substrate | Role in the Architecture |
|---|---|
| **1. Enrolment Ledger** | Authoritative slow-changing record of enrolled `Entity` declarations (`domain`, `agent`, `service`, `shim`, `projection`, `human`) and standing `stewardship` contracts (including delegated decision rights). |
| **2. Contract Journal** | Authoritative fast-changing MVCC State-Revision Log (`KV + History + CAS`) of ephemeral `workflow` contracts, enforcing conditional append and two-phase handoff validity. |

### Optional Substrate (Ordinary Entities Flagged as Substrate, Declared by the Domain)

| Optional Substrate | Role in the Architecture |
|---|---|
| **3. Projections (`Capability Index`, `Workflow Tree`, `Stewardship Graph`, `Comprehension Views`)** | Derived, read-only views over the Enrolment Ledger and Contract Journal for runtime discovery and human comprehensibility. |
| **4. Recommender / Router** | Classification service consulted via **subcontract** to rank candidate capabilities for open requests *(requests specifying a required capability rather than a target entity ID)* so callers can discover routes dynamically. |
| **5. Cryptographic Contract Signing & Authenticity (e.g., [`jit-ca`](https://github.com/brian-spackman/jit-ca))** | Allows entities with decision authority to attach short-lived cryptographic signatures/certificates to contracts so multiple agents can safely share unprivileged intermediate services while downstream effect boundaries verify the approver's signature. |
| **6. Enrolment & Compatibility Service** | Evaluates declaration compatibility (`provides` / `requires`) on admission, surfaces conflicts to the Service Steward and Domain Steward, and recommends Translator Shims. |
| **7. Conformance & Audit Monitor** | Watches the Enrolment Ledger and Contract Journal to detect entities acting without a valid contract or with a broken stewardship chain, reporting violations to both the offending entity's steward and the monitor's own steward. |
| **8. Proactive Health & Stall Watchdog** | Monitors for expired leases, unverified discharges, or broken stewardship links to notify stewards *before* the next runtime invocation. Detection only; never holds authority to remedy. |
| **9. Transport Backbone (e.g., Event Bus)** | Shared messaging fabric if a domain prefers unified pub/sub over direct point-to-point API calls (decoupled from the Contract Journal). |
| **10. Secret & Reference Resolver** | Resolves secret and large-content pointers at effect time against the caller's live authority. |
| **11. Infrastructure Failure Sink** | Out-of-band alert channel when an entity cannot reach the Journal or an upstream steward is unreachable. |

---

## License & Attribution

Copyright © 2026 Brian Spackman.

- **Specification text and architectural diagrams:** Licensed under the [Creative Commons Attribution 4.0 International License (CC BY 4.0)](./LICENSE-CC-BY-4.0) — attribution to **Brian Spackman**.
- **Schemas, protocol mechanics, and reference implementations:** Licensed under the [Apache License, Version 2.0 (Apache-2.0)](./LICENSE-APACHE).
