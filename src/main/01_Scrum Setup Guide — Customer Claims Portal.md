# Scrum Setup Guide — Customer Claims Portal

## Purpose

Practical, copy‑ready Scrum setup for a small team building an insurance Customer Claims Portal. Includes roles, events, artifacts, and short‑term standards you can adopt immediately.

---

## Core Setup

- Sprint length: 2 weeks
- Team: 6 Developers, 1 Product Owner, 1 Scrum Master
- Daily Scrum: 11:00–11:15
- Working agreement highlights: low WIP, pair on risky changes, fast feedback

## Roles

- Product Owner: orders Product Backlog for maximum value, clarifies acceptance criteria
- Scrum Master: coaches on Scrum, removes impediments, facilitates events
- Developers: design, build, test, document as one unit; collectively own quality

## Events (Ceremonies)

- Sprint Planning: define 1 Sprint Goal and select items for the Sprint Backlog
- Daily Scrum: 15 min for Developers to re‑plan toward the Sprint Goal
- Sprint Review: demo the Increment, capture feedback, update Product Backlog
- Sprint Retrospective: inspect how we worked, choose 1–2 improvements

## Artifacts

- Product Backlog: ordered list of outcomes and stories
- Sprint Backlog: selected items + plan to deliver the Sprint Goal
- Increment: working, potentially shippable software at sprint end
- Definition of Done (DoD): shared quality checklist below

---

## Starter Product Backlog (example)

EPIC: Submit and track insurance claims online

- Story 1: As a policyholder, I can submit a basic claim with policy number, claim type, and description.
    - Acceptance: required fields validated; claim saved; confirmation shown
- Story 2: As a policyholder, I can upload supporting documents up to 20 MB per file.
    - Acceptance: virus scan; allowed file types only; clear error messages
- Story 3: As a policyholder, I can see my claim status (Submitted, In Review, Approved, Rejected)
- Story 4: As an auditor, I can view an immutable audit log of claim actions
- Story 5: As an admin, I can configure claim types and reason codes

Non‑functional backlog items

- P95 latency under 300 ms for submit and status endpoints
- Security headers, CSRF protection, and server‑side input validation on all forms
- Accessibility WCAG 2.1 AA for forms and errors

---

## Sprint 0 (lightweight) Setup Tasks

- Repos, CI pipeline, staging environment, feature flags
- Definition of Ready (DoR): clear acceptance, testable, dependencies identified, designs attached
- Working agreements: PRs same day; pair on risky changes; max WIP 2 per developer; impediments channel

---

## Sprint 1 Plan (example)

Sprint Goal: Enable policyholders to submit a basic claim end‑to‑end

- Backend: POST /claims, validations, persistence
- Frontend: form fields, errors, success page
- Tests: unit, API happy path and edge cases
- Pipeline: add security scan job

Definition of Done (initial)

- [ ]  Code reviewed
- [ ]  Unit and API tests added; critical paths covered
- [ ]  Security lint and dependency scan pass
- [ ]  Feature flag added if needed
- [ ]  Docs and API contracts updated
- [ ]  Deployed to staging; meets acceptance criteria

---

## Templates

User Story

- As a [role], I want [capability], so that [value].
- Acceptance criteria:
    - Given [context], when [action], then [result]
    - Negative and edge cases listed
- Notes: links to designs, APIs, data model

Daily Scrum prompt

- Progress toward Sprint Goal since yesterday
- Plan for today
- Impediments

Sprint Review agenda

- Restate Sprint Goal
- Demo Increment
- Collect feedback and changes
- PO updates backlog order

Retrospective prompts

- What helped or hindered delivery?
- Which change will have the biggest impact next sprint?
- One experiment to try

---

## Short‑Term Standards (next 4–6 weeks)

Process

- Sprint length fixed at 2 weeks; scope changes require re‑negotiation of the Sprint Goal
- WIP limits: max 2 in‑progress items per developer; swarm to finish before starting new
- Story size: target 1–3 days each; split if larger
- Daily Scrum at 11:00; timebox to 15 minutes; decisions recorded in task comments

Quality

- DoD applies to every story; no partial “done”
- Code review SLA: request before 4 pm, review same business day
- Tests: at least one automated test per acceptance criterion; API tests for critical flows
- Security: server‑side validation, CSRF protection, and security headers required for any new endpoint or form

Delivery

- Every sprint produces a working Increment in staging or behind a feature flag
- Demo from staging with realistic data during Sprint Review
- Performance check: submit endpoint p95 < 300 ms in CI or flagged

Backlog & Readiness

- Definition of Ready enforced for items pulled into a sprint
- Acceptance criteria must be testable and include failure cases
- Designs or mocks attached when UI is involved

---

## Common Pitfalls to Avoid

- Treating Daily Scrum as status to managers instead of re‑planning
- Starting too many items at once; finish vertically sliced work
- Vague acceptance criteria; make them testable and observable
- Changing scope mid‑sprint without adjusting the Sprint Goal

---

## Next Actions

- Confirm Sprint length and ceremony times with the team
- Import the Starter Product Backlog into your tool
- Adopt the Short‑Term Standards immediately for the next two sprints
- In the next Retrospective, adjust DoD and standards based on evidence