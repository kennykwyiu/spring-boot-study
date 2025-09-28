# SDLC Explained — With Waterfall and Agile Examples

## Purpose

A clear overview of the Software Development Life Cycle (SDLC) with a concrete example, shown in both Waterfall and Agile (Scrum-style) approaches.

---

## What is SDLC?

The end‑to‑end process of turning an idea into working, maintained software. Common phases:

1. Requirements
2. Design
3. Implementation
4. Testing
5. Deployment
6. Maintenance and Operations

These phases exist in any approach; different models (Waterfall, Agile) organize and iterate through them differently.

---

## Concrete Example: "Customer Claims Portal"

Goal: Customers submit insurance claims, upload documents, and track status.

Key constraints: security, auditability, reasonable performance.

### High‑level scope

- Claim submission form with validation
- Document upload with virus scan
- Status tracking (Submitted, In Review, Approved, Rejected)
- Admin config of claim types and reason codes
- Audit log of user and system actions

---

## Waterfall SDLC for the Portal

Characteristics: sequential phases, heavy upfront planning, formal sign‑offs.

1) Requirements (6–8 weeks)

- Detailed BRD and SRS covering all flows, edge cases, non‑functional requirements
- Compliance and audit requirements finalized; change control defined

2) Design (4–6 weeks)

- Architecture diagrams, data model, integration contracts
- Complete UI mockups and accessibility specs

3) Implementation (12–16 weeks)

- Build entire portal according to approved specs
- Periodic internal reviews, but scope changes go through change control board

4) Testing (4–6 weeks)

- System test, security testing, performance testing
- UAT with business stakeholders; defects triaged via formal process

5) Deployment (1–2 weeks)

- Release plan, runbooks, approvals, change window
- One “big bang” release to production

6) Maintenance (ongoing)

- Patches and enhancements scheduled into later releases following change control

Pros

- Predictable scope and documentation
- Easier compliance sign‑offs

Cons

- Risk of late discovery of usability or requirements issues
- Costly to incorporate mid‑project changes

---

## Agile SDLC (Scrum) for the Portal

Characteristics: iterative, incremental delivery in short Sprints, continuous feedback.

Setup

- Sprint length: 2 weeks
- Roles: Product Owner, Scrum Master, Developers
- Artifacts: Product Backlog, Sprint Backlog, Increment, Definition of Done (DoD)

Sample Product Backlog items

- As a policyholder, submit a basic claim with policy number, type, description
- Upload documents up to 20 MB with virus scan and type restrictions
- View claim status; events logged to an audit trail
- Admin config for claim types and reason codes
- NFRs: p95 < 300 ms for submit/status; security headers; CSRF; input validation

Sprint 1

- Sprint Goal: “Submit a basic claim end‑to‑end”
- Deliverables: backend POST /claims, validations, persistence; frontend form; tests; staging deploy

Sprint 2

- Sprint Goal: “Secure document uploads and basic status view”
- Deliverables: upload with AV scan, allowed types; status page; audit entries

Sprint 3

- Sprint Goal: “Hardening and accessibility”
- Deliverables: security headers, CSRF, input validation everywhere; WCAG improvements; performance checks

Feedback loop

- Review each Sprint with stakeholders; reorder backlog as needs emerge
- Retrospective: improve DoD, flows, and team practices continuously

Pros

- Early value and risk reduction through working increments
- Embraces changing requirements

Cons

- Requires strong product ownership and disciplined backlog management
- Forecasting long‑term scope requires empirical data and adaptation

---

## Side‑by‑Side Comparison

Planning

- Waterfall: heavy upfront, fixed scope
- Agile: continuous, adaptive

Delivery cadence

- Waterfall: single major release
- Agile: small releases or increments each Sprint

Change management

- Waterfall: formal change control, higher cost to change
- Agile: change expected; backlog reordered between Sprints

Risk

- Waterfall: issues surface late
- Agile: issues surface early via increments

Governance

- Waterfall: stage gates and sign‑offs
- Agile: inspect‑and‑adapt events, Definition of Done

---

## Choosing an Approach

Prefer Waterfall when

- Requirements are stable and regulated documentation is paramount
- Integration or verification is extremely costly to repeat

Prefer Agile when

- Requirements evolve, or early usability feedback is critical
- You can deliver value in slices and iterate with stakeholders

Hybrid option

- Use Waterfall‑like gates for compliance artifacts
- Deliver product increments iteratively (Agile) between gates

---

## Practical Starter Checklists

Waterfall stage gate checklist (sample)

- [ ]  BRD and SRS approved
- [ ]  Architecture and data model signed off
- [ ]  Test plan, traceability matrix ready
- [ ]  Deployment plan and rollback defined

Agile Definition of Done (sample)

- [ ]  Code reviewed
- [ ]  Tests added and passing in CI
- [ ]  Security lint and dependency scan pass
- [ ]  Docs updated
- [ ]  Deployed to staging or behind a feature flag

---

## Next Actions

- Decide whether this project leans Waterfall, Agile, or Hybrid based on constraints
- If Agile: confirm Sprint length, roles, and DoD, then start Sprint 1
- If Waterfall: finalize requirements and design artifacts, then plan implementation and test phases