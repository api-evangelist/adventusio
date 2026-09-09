---
name: adventusio-manage-students
description: Create, find, update and enrich student records in Adventus.io, with the safety rules the API does not give you.
generated: '2026-09-09'
method: generated
source: graphql/adventusio.graphql
api: adventusio-graphql
endpoint: https://api.adventus.io/graphql
operations:
  - students
  - student
  - addStudent
  - updateStudent
  - updateStudentAcademicAchievement
  - addStudentNote
  - studentNotes
  - studentNoteTypes
  - studentActivities
  - studentLeadSource
  - gradingSystems
  - studyLevels
---

# Manage student records in Adventus.io

Requires a recruiter/agent bearer token — see `adventusio-authenticate`.

## Find students

`students` is page-numbered, **not** offset/limit:

```graphql
query {
  students(page: 1, keyword: "surname", status: "Active") {
    # StudentList wrapper — read its fields from the SDL
  }
}
```

`status` is an untyped `String` with no published vocabulary. The value `Active`
is the one the API itself uses as its default (visible in the upstream URL the
error envelope echoes). Do not guess others; confirm against real data.

Fetch one with `student(id: Int)`.

## Create a student

```graphql
mutation {
  addStudent(student: { … StudentInput … }) { id }
}
```

**Before you call this, know two things the API will not tell you:**

1. **There is no idempotency key.** If `addStudent` times out you cannot tell
   whether it landed. Search with `students(page: 1, keyword: …)` before
   retrying, every time.
2. **There is no delete and no undo.** Nothing in the schema removes a student
   record. A duplicate you create is permanent from this API's point of view.

Read the exact required fields off `StudentInput` in
`graphql/adventusio.graphql` — do not assume them.

## Update a student

`updateStudent(id: Int!, student: StudentInput)` overwrites. There is no
revision history, no restore, and no way to read a prior value. **Read the
record with `student(id:)` and keep the response before you write**, or the
overwritten value is gone.

## Academic achievement

`updateStudentAcademicAchievement(studentId: Int!, academicAchievement: AcademicAchievementInput)`.

`AcademicAchievementInput` requires `studyLevel: String!`,
`gradingSystemCountryId: Int!`, `gradingSystemId: Int!` and
`gradingSystemScore: JSON!`. Resolve the first three from the anonymous
reference queries first:

- `studyLevels` returns the 11 valid levels (High School, Language Pathway,
  Undergraduate - Foundation/Certificate/Diploma/Associate Degree/Bachelor,
  Postgraduate - Certificate/Diploma, Masters, Doctorate / PHD).
- `gradingSystems(countryId: Int)` returns the systems for a country.
- `countries` returns country ids.

`gradingSystemScore` is a free-form `JSON` scalar; its shape is defined by the
grading system you selected, so read `GradingSystem` / `GradingSystemTable` /
`GradingSystemScore` from the SDL rather than inventing a payload.

## Notes and activity

- `studentNoteTypes` — the valid note types. Call it first.
- `addStudentNote(studentId: Int!, note: StudentNoteInput)` — **irreversible**.
  There is no edit and no delete for a note.
- `studentNotes(studentId: Int!)` and `studentActivities(studentId: Int!)` are
  unpaginated — they return a bare array with no total.

## Student's own view

A student-authenticated caller uses the `my*` mirror of the same surface:
`myStudentProfile`, `updateMyStudentProfile`, `myDocuments` /
`myDocumentsV2`, `updateMyAcademicAchievement`, `myAgent`, `myOrderSummary`.
