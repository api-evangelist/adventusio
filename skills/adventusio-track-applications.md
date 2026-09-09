---
name: adventusio-track-applications
description: Read student applications (orders), their milestones and course statistics in Adventus.io.
generated: '2026-09-09'
method: generated
source: graphql/adventusio.graphql
api: adventusio-graphql
endpoint: https://api.adventus.io/graphql
operations:
  - orders
  - orderMilestones
  - studentOrderSummary
  - myOrderSummary
  - instituteStats
  - courseStats
  - institution
  - threadMessageList
  - sendThreadMessage
  - markThreadReadStatus
---

# Track applications and their progress

Requires a bearer token. This surface is **read-only** except for messaging —
nothing here creates, withdraws or cancels an application.

## List applications

```graphql
query {
  orders(page: 1, keyword: "…", status: "…", style: "…", sort: "…", filter: "…") {
    # [StudentsOrders]
  }
}
```

`status`, `style`, `sort` and `filter` are all untyped `String` arguments with
no published vocabulary. Send only values you have observed in real responses.
`orders` returns a bare list, so there is no total count — page until short.

`StudentsOrders` carries `studentId` and a `studentOrders: [Application]`
collection. `Application` carries an `orderId: String` and `studentId: Int`.

## Progress

- `orderMilestones(orderId: String)` → `OrderMilestones`, built from `Milestone`,
  `MilestoneDetails` and `CurrentMilestone`.
- `studentOrderSummary(studentId: Int!)` → `OrderSummaryList` for one student.
- `myOrderSummary` → the caller's own summary.

`OrderSummary` links out to `Milestone`, `Country` (via both `countryDetail` and
`country_id`), an `applicationId: String`, a `courseId: Int`, and a
`messages: ThreadMessageList`.

**There is no Course type in this schema.** `courseId` is referenced from
`OrderSummary`, `CourseStats` and `intake_control_Schedule`, but no query
returns a course object. Course detail lives in the web application's search
surface, not this API. Do not try to resolve a `courseId` to a course here.

## Statistics

- `instituteStats(instituteId: Int!)` → `[InstituteStats]`, unpaginated.
- `courseStats(instituteId: Int!, search: String, paginate: PaginationInput, sort: SortInput)`
  → `CourseStatsList`. Note this one uses **`paginate`** with
  `PaginationInput { offset: Int!, limit: Int! }` — a different convention from
  `orders`, which uses `page: Int!`. Getting this wrong returns a
  `GRAPHQL_VALIDATION_FAILED` error at HTTP 400.

## Messaging on an application

- `threadMessageList(type: String!, id: Int!, offset: Int, limit: Int)` — a
  **third** pagination convention: loose `offset`/`limit` arguments, not an
  input object.
- `sendThreadMessage(threadType: String!, threadId: Int!, message: String)` —
  **irreversible**. There is no edit, retract or delete for a sent message, and
  no idempotency key, so a retry after a timeout sends it twice. Read the thread
  back before resending.
- `markThreadReadStatus(threadType: String!, threadId: Int!, read: Boolean)` is
  the one genuinely safe write here — it is a toggle and can be set back.
