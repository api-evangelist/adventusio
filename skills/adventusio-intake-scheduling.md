---
name: adventusio-intake-scheduling
description: Read intake availability and create or cancel course intake schedules in Adventus.io — the one write surface with a reversal.
generated: '2026-09-09'
method: generated
source: graphql/adventusio.graphql
api: adventusio-graphql
endpoint: https://api.adventus.io/graphql
operations:
  - intake_control_hello
  - intake_control_listSchedule
  - intake_control_getStatus
  - intake_control_getIntakeForCourse
  - intake_control_createSchedule
  - intake_control_cancelSchedule
---

# Intake control: schedules and availability

Requires a bearer token. This namespace is a separately built subsystem exposed
through the same gateway — it defines its **own** pagination, sort, country and
error types rather than reusing the shared ones, so do not carry conventions
over from the rest of the schema.

## Its own types

| Shared schema | intake_control equivalent |
|---|---|
| `PaginationInput { offset, limit }` | `intake_control_PaginationInput` |
| `SortOrder { ASC, DESC }` | `intake_control_SortOrder` |
| `Country` | `intake_control_Country` |
| GraphQL `errors[]` | also `intake_control_errorObject` in payloads |

## Read

- `intake_control_hello` — a liveness field returning `String`. Useful as a cheap
  reachability check for this subsystem specifically.
- `intake_control_listSchedule(courseId: Int, institutionId: Int, paginate: intake_control_PaginationInput, sort: intake_control_SortOrder)`
  → `intake_control_CourseScheduleList`.
- `intake_control_getStatus(courseId: [Int]!, metadata: intake_control_MetadataInput)`
  → `[intake_control_ControlStatus]`. Note `courseId` is a **list** and is
  required — batch your lookups here rather than looping.
- `intake_control_getIntakeForCourse(intakeRequest: [intake_control_IntakeResponseInput])`
  → `[intake_control_IntakeResponse]`.

`intake_control_Schedule` carries `id`, `userId`, `institutionId`, `courseId`,
`intakeId`, `tenantId` and a `countries` collection, plus
`intake_control_CurrentSchedule`, `intake_control_CourseLevel` and
`intake_control_IntakeLevel` detail.

## Write — and the only reversal in the whole API

```graphql
mutation {
  intake_control_createSchedule(input: { … intake_control_CreateScheduleInput … }) {
    id
  }
}
```

```graphql
mutation {
  intake_control_cancelSchedule(input: { … intake_control_CancelScheduleInput … }) {
    id
  }
}
```

`intake_control_cancelSchedule` is the **only** reversal operation Adventus.io
publishes. Every other write in the schema — student creation, notes, messages,
document upload, Connect acceptance — has no undo.

**No cancellation window is stated.** Adventus.io publishes no documentation for
this subsystem at all, so you do not know how long after creation a cancel is
still accepted, or whether an already-started intake can be cancelled. Do not
assume one. If a cancel must happen by a deadline, verify with
`intake_control_listSchedule` that the state actually changed rather than
trusting the mutation's return.

## Retry safety

`intake_control_createSchedule` has no idempotency key. Before retrying a
timed-out create, call `intake_control_listSchedule(courseId:, institutionId:)`
and check whether the schedule already exists — you can cancel a duplicate, but
only if you notice it.
