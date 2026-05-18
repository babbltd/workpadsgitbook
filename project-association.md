# §PA — Project Association

**Status:** v0.1 (2026-05-18)  
**SUI:** SUI-013  
**Kaios source:** `draft_specs/PROJECT-ASSOCIATION-DESIGN.md` design 2026-05-17

---

## 1. Core Rule

**Financial records: maximum 1 project association.**  
A financial record (invoice, payment, expense, COGS) may be associated with at most one project. Multiple associations would create an accounting split problem without an explicit allocation, guaranteeing double-counting or omission.

**Service/job records: 1 or more project associations.**  
A service record (job note, site visit, activity log) describes work. Work can legitimately span multiple projects — a site visit covering two concurrent contracts, a meeting about three ongoing engagements. No accounting implications; the record is descriptive.

This constraint is **enforced at the app layer**, not the wire format. The wire format carries whatever is in the tag field. The app refuses to encode more than one `proj:` prefix on a financial record.

---

## 2. Wire Mechanism

Project associations are encoded in the `tag` field (FLAGS3 bit 1), using a `proj:` prefix:

```
Financial record (max 1 project):
  tag = "proj:uid_alpha,urgent"           ✓ valid
  tag = "proj:uid_alpha,proj:uid_beta"    ✗ app rejects (financial + 2 projects)

Service record (multiple OK):
  tag = "proj:uid_alpha,proj:uid_beta,site-visit"   ✓ valid
```

**Parsing rule:** prefix `proj:` identifies a project association. App extracts all `proj:` prefixed segments; remaining segments are free-form tags.

**Project UID in tag:** project UIDs are referenced by their stable identifier. The app resolves UIDs to project names via a local project directory cached from project records. On first encounter with an unknown UID, the app displays the truncated UID until the project record is fetched.

---

## 3. Project Record Type

A project is a workpads record. Specifically: `BASE_TEMPLATE=000` (Service) with a dedicated template variant that marks it as a project container.

| Field | Flag | Content |
|-------|------|---------|
| job | bit 0 | Project name — "Westfield Mall Fit-Out" |
| date | bit 2 | Project start date |
| date_end | FLAGS3 bit 3 | Project end date (expected completion) |
| uid | FLAGS3 bit 5 | Project UID — what other records reference via `proj:uid` |
| customer | bit 1 | Client name |
| ref_number | bit 13 | Project code / contract number |
| story | bit 11 | Project description / scope |
| tag | FLAGS3 bit 1 | Project category tags |

The dedicated template variant marking `project_container` is registered in the template registry. Decoders recognising this template treat the record as a project index entry rather than a single-event record.

---

## 4. Project Aggregation

A project accumulates records over time. App-side aggregation:

```
Fetch all records where tag CONTAINS "proj:<uid>"
Filter financial_block=1 → cost records
Sum customer_amount where EXPENSE_CAT=00 → billed items (charged to customer)
Sum worker_amount   where EXPENSE_CAT=01 → absorbed COGS (internal cost)
Sum both            where EXPENSE_CAT=10 → running costs allocated to project
```

The single-project-per-financial-record constraint makes this aggregation unambiguous.

### Project Summary via State Commit

A project phase close uses `BASE_TEMPLATE=101` (State Commit), `COMMIT_TYPE=00`, with compound block lines:

```
Line 1: total revenue (all I>I financial records in chain)
Line 2: total expenses (all O<O financial records in chain)
Line 3: net margin
Summary: project status
```

Chain link `&c=<project_uid>` ties the State Commit to the project record. Chain depth signals which phase close this is (final = project close with `CHAIN_COMPLETE=1`).

---

## 5. Multi-Project Scenarios

**Worker with two concurrent contracts:**

```
Record A: "Electrical inspection, Westfield" (service note, no financial block)
  tag = "proj:uid_westfield,inspection"
  ✓ Multiple projects allowed for service records

Record B: "Cable purchase, £45" (expense, O<O)
  tag = "proj:uid_westfield"
  ✓ Single project — cost absorbed by Westfield

Record C: "Cable purchase, £30" (expense, O<O)
  tag = "proj:uid_eastgate"
  ✓ Separate record for different project
```

Worker does NOT encode a £75 expense with two project tags. They create two records: £45 to Westfield, £30 to Eastgate. Each is a clean, auditable entry with unambiguous project ownership.

---

## 6. Post-MVP: Cost Allocation

For costs that genuinely span two projects and must be explicitly allocated:

**Split record pattern:** create two expense records with explicit amounts — one per project. Clean, auditable, no wire format change.

**Allocation field (future, FLAGS4/5):** a future `allocation_pct` field (uint8, value = percentage 1–100) could signal partial cost allocation. A financial record with `allocation_pct=60` means "60% of this cost belongs to the associated project; 40% to the overhead pool." Wire encoding deferred to FLAGS5 (post-v1.0).

---

## 7. Open Items

- **OQ-PA3** — Project UID stability: if a project record is amended, does `uid` in FLAGS3 bit 5 refer to the original record UID or a stable project identifier? Recommendation: stable project identifier stored in a dedicated `project_uid` field (FLAGS4 slot, post-MVP) separate from record UID, so amendments don't change the project's canonical identifier.
- **OQ-PA5** — Tag field size with long project UIDs: a UUID v4 is 36 chars; `proj:` prefix adds 5 chars = 41 chars per project. Tag field max is 60B. One project + one category tag = ~52 chars (fits). Two projects on a service record = ~87 chars (exceeds 60B). Resolution: either increase tag max to 120B, or truncate project UIDs to 12 significant chars (low collision risk at typical project counts of <1000 per user).
