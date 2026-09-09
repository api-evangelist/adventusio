---
name: adventusio-handle-documents
description: Upload, list and delete student documents in Adventus.io, including the V1/V2 field split.
generated: '2026-09-09'
method: generated
source: graphql/adventusio.graphql
api: adventusio-graphql
endpoint: https://api.adventus.io/graphql
operations:
  - studentDocumentTypes
  - studentDocuments
  - studentDocumentsV2
  - myDocumentTypes
  - myDocuments
  - myDocumentsV2
  - uploadStudentDocument
  - uploadMyDocument
  - deleteStudentDocument
---

# Handle student documents

Requires a bearer token.

## Choose the right field — there are two generations

| V1 | V2 | Difference |
|---|---|---|
| `studentDocuments(studentId: Int!)` | `studentDocumentsV2(studentId: Int!, page: Int!, keyword: String)` | V1 returns a bare `[StudentDocument]`; V2 returns a paginated `StudentDocumentList` and adds keyword search |
| `myDocuments` | `myDocumentsV2(page: Int!, keyword: String)` | same change on the caller's own documents |

**Neither V1 field is marked `@deprecated`** — the schema uses the
`@deprecated` directive nowhere at all — so the contract gives you no signal
that V1 is superseded. Prefer V2: it is the one that paginates, and an
unpaginated V1 call on a large record set has no way to page.

## Discover the valid classification first

`studentDocumentTypes(studentId: Int!)` returns `[StudentDocumentSection]`,
which contains `StudentDocumentCategory` and `StudentDocumentType`. The upload
mutation demands `category: String!` and `documentTypeId: Int!`, and those
values come from here. Call it before every upload; the sections are
student-specific.

`myDocumentTypes` is the student-authenticated equivalent.

## Upload

```graphql
mutation UploadStudentDocument($document: Upload!) {
  uploadStudentDocument(
    studentId: 123
    fileName: "transcript.pdf"
    description: "Final transcript"
    section: "…"
    category: "…"          # from studentDocumentTypes
    documentTypeId: 7      # from studentDocumentTypes
    data: { }              # JSON scalar, optional
    document: $document
  ) { id }
}
```

`Upload` is the GraphQL multipart request specification scalar — send a
`multipart/form-data` request with `operations`, `map` and the file part, not a
base64 string in JSON.

`uploadMyDocument` is the same call without `studentId`, for a
student-authenticated caller.

## Delete

`deleteStudentDocument(studentId: Int!, documentId: Int!)` →
`DeleteStudentDocumentResponse`.

**This is terminal.** There is no restore, no trash and no undelete operation
in the schema, and no retention window is published. Confirm the
`documentId` against `studentDocumentsV2` immediately before deleting, and
never delete as part of an unattended retry path.

## Retry safety

No upload or delete supports an idempotency key. A timed-out upload may have
succeeded — list the student's documents and match on `fileName` before
uploading again, or you will create a duplicate that only a delete can remove.
