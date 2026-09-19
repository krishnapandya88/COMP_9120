
# Blood Bank Supply Network

COMP9120 group database project.

## 1. Project Overview

We are building a database-backed blood bank supply system that tracks
donations, blood components, hospital requests, reservations, and shipments.

A donation can produce multiple component units, such as red cells, plasma,
and platelets. Each component has its own expiry and compatibility rules.

The system should allocate compatible, unexpired inventory to hospitals,
prioritising units that expire soonest.

### Core transaction

Fulfil an urgent hospital request by:

1. Finding eligible component units.
2. Checking that sufficient inventory is available.
3. Reserving the selected units.
4. Creating a shipment and its shipment items.
5. Updating the request status.
6. Committing all changes together.

If there are insufficient eligible units or another step fails, the
transaction must roll back without leaving partial changes.

### Concurrency scenario

Two hospitals request the last available O-negative red-cell units at the
same time.

The system must prevent the same unit from being allocated to both
hospitals.

## 2. Scope and Assumptions

This is an educational prototype, not a clinical decision-support system.

Blood compatibility and donation eligibility must use documented,
project-approved rules. We will not assume that ABO/Rh compatibility alone
is sufficient for real clinical use.

### In scope

- Donor registration and donation history.
- Donation eligibility based on a documented minimum interval.
- Component production and expiry tracking.
- Component-specific compatibility rules.
- Hospital requests.
- Inventory allocation and reservation.
- Shipment creation.
- Atomic request fulfilment.
- Concurrent allocation testing.
- ER modelling, relational mapping, and normalisation analysis.

### Out of scope unless required by the assessment

- Actual clinical crossmatching and patient treatment decisions.
- Integration with hospital information systems.
- Transport routing and temperature-monitoring hardware.
- Production authentication and deployment.

### Decisions to confirm before implementation

- Minimum donation interval and whether it varies by donation type.
- Supported component types and expiry rules.
- Compatibility rules for each supported component.
- Whether requests represent patients or hospital inventory needs.
- Whether request lines use whole component-unit counts.
- Whether requests permit partial fulfilment.
- Reservation cancellation and release behaviour.
- Required assessment deliverables and SQL/database version.

Initial transaction assumption: a fulfilment attempt is all-or-nothing.
Changes to this assumption must be documented.

## 3. Team and Ownership

Each deliverable has an owner and a reviewer. Ownership means driving the
work to completion, not working without discussion.

| Member | Role | Main ownership | Reviewer |
| --- | --- | --- | --- |
| Member A| data modeller | Requirements, ERD, integration, final review | Member B |
| Member B — replace name | Database engineer | Schema, constraints, allocation transaction, concurrency | Member C |
| Member C — replace name | Application and QA engineer | Prototype interface, seed data, tests, demo evidence | 1 |

All members must understand the full model and contribute to documentation
and review.

### 1 — Lead and Data Modeller

Responsibilities:

- Confirm the assessment scope and submission requirements.
- Maintain the requirements and business-rule checklist.
- Design the conceptual ERD.
- Coordinate modelling decisions with Member B.
- Lead the normalisation analysis.
- Review whether the implementation matches the requirements.
- Maintain the task board and coordinate integration.
- Assemble the final submission after team approval.

Primary outputs:

- `docs/requirements.md`
- `docs/business-rules.md`
- `docs/erd/`
- `docs/normalisation.md`
- `docs/decisions.md`

Definition of done:

- Every requirement has a corresponding model element or implementation.
- Cardinalities, participation, and identifiers are reviewed.
- Assumptions are explicit.
- The final ERD agrees with the approved relational mapping.

### Member B — Database Engineer

Responsibilities:

- Map the ERD into a relational schema.
- Implement PostgreSQL tables and appropriate data types.
- Define primary keys, foreign keys, nullability, and other constraints.
- Document deletion behaviour for each relationship.
- Implement atomic request fulfilment.
- Implement locking or another justified concurrency strategy.
- Generate and maintain the relational-model diagram.
- Support Member C with database integration and test setup.

Primary outputs:

- `sql/01_schema.sql`
- `sql/02_functions.sql`
- `sql/04_queries.sql`
- `docs/rm/`
- `docs/transactions.md`

Definition of done:

- Schema creation succeeds on a fresh database.
- Constraints enforce the documented rules where applicable.
- Fulfilment either completes entirely or rolls back.
- Concurrent requests cannot allocate the same unit.
- The RM diagram matches the implemented schema.

### Member C — Application and QA Engineer

Responsibilities:

- Build a minimal interface for the agreed use cases.
- Prepare realistic synthetic seed data.
- Implement functional, integrity, rollback, and concurrency tests.
- Integrate the interface with Member B's database operations.
- Record reproducible test instructions and results.
- Prepare the demo flow and supporting evidence.
- Maintain GenAI records if required by the assessment.

Primary outputs:

- `app/`
- `sql/03_seed.sql`
- `tests/`
- `docs/testing.md`
- `docs/demo.md`
- `docs/genai/`, if required

Definition of done:

- The interface demonstrates the agreed end-to-end workflow.
- Every implemented table has suitable example data.
- Success, failure, and concurrency cases are tested.
- Tests are reproducible by another teammate.
- Demo results match actual database behaviour.

### Shared Responsibilities

Everyone must:

- Review at least one other member's work.
- Attend agreed check-ins.
- Record contributions and decisions.
- Keep meeting records on Canvas if required.
- Acknowledge GenAI assistance according to unit policy.
- Understand and be able to explain the final submission.

## 4. Proposed Data Model

The following entities are a starting point, not a final approved schema.

| Entity | Purpose |
| --- | --- |
| Donor | Donor identity and relevant eligibility information |
| Donation | A collection event linked to a donor |
| ComponentType | Supported components and associated rule references |
| ComponentUnit | Individually tracked inventory with expiry and state |
| BloodType | Supported blood-group classifications |
| CompatibilityRule | Permitted source/recipient combinations by component |
| Hospital | Requesting organisation |
| HospitalRequest | Request header, priority, and lifecycle status |
| RequestLine | Component, recipient requirements, and requested quantity |
| Reservation | Allocation of a component unit to a request line |
| Shipment | Dispatch record associated with fulfilment |
| ShipmentItem | Units included in a shipment |

### Modelling questions

- Where should the blood-group result be stored: donor, donation, or unit?
- Is the stored result a current donor property or a historical test result?
- How is a unit prevented from having multiple active allocations?
- Can a request generate multiple shipments?
- How are cancelled or expired reservations represented?
- Which values should be derived rather than duplicated?

### Normalisation discussion

Copying blood type onto every unit may introduce redundancy, but the
correct dependency depends on what the attribute represents.

We must identify actual functional dependencies before labelling a design
as a normalisation violation. Historical test results and deliberate
snapshots must be distinguished from duplicated current donor data.

## 5. Business Rules

Rules must be finalised in `docs/business-rules.md`.

| ID | Rule | Planned enforcement |
| --- | --- | --- |
| BR01 | A donor must satisfy the documented donation interval | Transactional eligibility check |
| BR02 | Each component unit belongs to a donation | Foreign key |
| BR03 | A unit has a valid component type and expiry | Keys and validation |
| BR04 | Expired units cannot be newly allocated | Allocation-time check |
| BR05 | Allocation must satisfy component-specific compatibility | Compatibility lookup |
| BR06 | Eligible units are selected by earliest expiry first | Ordered allocation |
| BR07 | A unit cannot have multiple active allocations | Constraint and concurrency control |
| BR08 | Requested quantities must be positive | Check constraint |
| BR09 | Fulfilment must be atomic | Database transaction |
| BR10 | Insufficient stock must not leave partial fulfilment | Rollback |
| BR11 | Shipment items must correspond to the allocated units | Keys and transaction validation |

Important notes:

- Red-cell compatibility must not be reused unchanged for plasma.
- Platelet rules require their own documented scope and assumptions.
- Expiry eligibility must be checked at allocation time; a unit can expire
  without any row being updated.
- Donation-gap checks must also consider simultaneous donation attempts
  for the same donor.
- FEFO means first-expiring, first-out among eligible units.
- Use a deterministic tie-breaker, such as unit ID, for equal expiries.

## 6. Transaction and Concurrency Design

Member B owns the implementation. Member C owns independent testing.
1 reviews consistency with the business rules.

### Expected fulfilment workflow

1. Begin a transaction.
2. Lock and validate the hospital request.
3. Prevent an already fulfilled request from being fulfilled again.
4. Identify compatible, available, unexpired units.
5. Lock candidate inventory using the agreed concurrency strategy.
6. Select units in expiry order with a deterministic tie-breaker.
7. Confirm sufficient inventory for every required request line.
8. Create reservations and shipment records.
9. Update inventory and request state consistently.
10. Commit.

On failure, roll back all changes.

### Design decisions to document

- Transaction isolation level.
- Lock acquisition order.
- Behaviour when another transaction holds eligible inventory.
- Whether callers wait, retry, or receive a stock-conflict result.
- How deadlocks and retryable failures are handled.
- Whether FEFO is strict or relaxed under contention.

Do not use `SKIP LOCKED` without explaining its consequences: it can skip
earlier-expiring locked inventory or temporarily report insufficient
available stock.

### Required concurrency demonstration

- Session A requests the last eligible units.
- Session B requests the same inventory before Session A finishes.
- Only one request can receive each unit.
- The other request waits, retries, or fails according to the documented
  policy.
- No duplicate active reservations or partial shipments remain.

## 7. Repository Structure

Planned structure; files will be added as implementation progresses.

```text
.
├── README.md
├── .gitignore
├── .env.example
├── app/
├── sql/
│   ├── 01_schema.sql
│   ├── 02_functions.sql
│   ├── 03_seed.sql
│   └── 04_queries.sql
├── tests/
│   ├── integrity/
│   ├── fulfilment/
│   └── concurrency/
└── docs/
    ├── requirements.md
    ├── business-rules.md
    ├── decisions.md
    ├── normalisation.md
    ├── transactions.md
    ├── testing.md
    ├── demo.md
    ├── erd/
    ├── rm/
    ├── meetings/
    └── genai/
```

Never commit passwords, connection secrets, or real donor/patient data.

## 8. Execution Plan

### Milestone 1 — Scope and rules

- [ ] All: read the actual assessment brief.
- [ ] 1: draft requirements and business rules.
- [ ] B: identify database and transaction risks.
- [ ] C: identify demo cases and test data needs.
- [ ] All: agree compatibility, expiry, and donation-gap assumptions.

Exit condition: the team agrees what the system must enforce.

### Milestone 2 — Design

- [ ] 1: draft ERD.
- [ ] B: review ERD and draft relational mapping.
- [ ] C: check whether proposed data supports the demo and tests.
- [ ] All: review identifiers, cardinalities, and lifecycle states.

Exit condition: an approved baseline model exists.

### Milestone 3 — Implementation

- [ ] B: implement schema and constraints.
- [ ] C: prepare seed data and interface skeleton.
- [ ] 1: document normalisation and review rule coverage.
- [ ] B and C: agree the fulfilment operation's inputs and outputs.

Exit condition: a fresh database can be created and populated.

### Milestone 4 — Transactions and integration

- [ ] B: implement allocation and shipment transaction.
- [ ] C: connect the interface and implement tests.
- [ ] 1: review end-to-end behaviour.
- [ ] All: demonstrate successful fulfilment and insufficient-stock rollback.

Exit condition: the main workflow works end to end.

### Milestone 5 — Concurrency and submission

- [ ] B and C: execute simultaneous allocation tests.
- [ ] 1: coordinate final requirements audit.
- [ ] B: synchronise RM diagram with final SQL.
- [ ] C: document setup, test results, and demo steps.
- [ ] All: review submission files and contribution records.

Exit condition: another teammate can reproduce setup and demonstrations.

## 9. Git Workflow

- Keep `main` runnable.
- Use feature branches.
- Open a pull request for each meaningful change.
- Get at least one teammate's review before merging.
- Update relevant documentation in the same pull request.
- Discuss schema changes before modifying shared tables.

Example branch names:

- `docs/requirements-erd`
- `feat/schema-constraints`
- `feat/fulfilment-transaction`
- `feat/request-interface`
- `test/concurrent-allocation`

### Pull request checklist

- [ ] The change addresses an agreed task.
- [ ] Relevant tests pass.
- [ ] No credentials or private data are committed.
- [ ] Schema changes include updated documentation and diagrams.
- [ ] A teammate has reviewed the change.

## 10. Testing Checklist

### Integrity

- [ ] Invalid foreign keys are rejected.
- [ ] Non-positive request quantities are rejected.
- [ ] Mandatory values cannot be null.
- [ ] Duplicate active unit allocations are prevented.
- [ ] Deletion behaviour matches the documented rules.

### Business behaviour

- [ ] Donation eligibility respects the agreed interval.
- [ ] Expired units are excluded.
- [ ] Incompatible units are excluded.
- [ ] Compatibility depends on component type.
- [ ] Earliest-expiring eligible units are allocated first.
- [ ] Equal-expiry selection is deterministic.

### Atomicity and concurrency

- [ ] Sufficient stock produces complete fulfilment.
- [ ] Insufficient stock leaves no partial changes.
- [ ] A simulated intermediate failure rolls back all writes.
- [ ] Repeating fulfilment cannot allocate additional units accidentally.
- [ ] Concurrent requests cannot receive the same unit.
- [ ] Locking and retry behaviour match the documented policy.

## 11. Local Setup

Target database: PostgreSQL.

The exact version, application stack, dependency commands, database setup
commands, and test commands will be added once agreed.

Before the implementation milestone is complete, this section must let a
teammate reproduce the project from a fresh clone.

## 12. Meetings and Contribution Records

Hold two short check-ins per week, plus milestone reviews as needed.

Each member reports:

1. Work completed.
2. Work planned next.
3. Blockers or decisions needed.

Rotate the meeting recorder.

Record:

- Date and attendees.
- Decisions and reasons.
- Tasks, owners, and deadlines.
- Links to issues and pull requests.

Repository records supplement, but do not replace, any required Canvas
diary.

## 13. Assessment Alignment

The Assignment 1 rubric previously discussed and the real-world project
brief may have different deliverables.

Before submission, map the applicable brief to an explicit checklist.
Do not assume that a prototype, GenAI analysis, video, or poster is
required unless the relevant assessment asks for it.

If GenAI evidence is required, preserve exact prompts, original outputs,
tool details, and the team's independent analysis.

## 14. Definition of Project Completion

The project is complete when:

- The requirements, ERD, RM diagram, and SQL are consistent.
- Business rules are implemented or their limitations are documented.
- Every table has appropriate synthetic example data.
- Fulfilment is atomic and concurrency-safe.
- Setup and tests are reproducible.
- Required assessment artefacts are present.
- Contribution records are up to date.
- Every member can explain the design and transaction behaviour.
````
