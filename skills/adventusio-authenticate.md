---
name: adventusio-authenticate
description: Obtain and use a bearer token for the Adventus.io GraphQL API, and know which fields need one.
generated: '2026-09-09'
method: generated
source: graphql/adventusio.graphql + live probes of https://api.adventus.io/graphql
api: adventusio-graphql
endpoint: https://api.adventus.io/graphql
operations:
  - userLogin
  - studentLogin
  - userLogout
  - studentLogout
  - userForgotPassword
  - userResetPassword
---

# Authenticate against the Adventus.io GraphQL API

Adventus.io publishes no authentication guide. Everything here was read from the
live schema and confirmed against the running endpoint.

## Endpoint

`POST https://api.adventus.io/graphql` with `Content-Type: application/json`.
`GET` is not supported — it returns HTTP 400 `GET query missing.`

## 1. Decide whether you need a token at all

These fields answer with **no credential**:

- `countries(hasCallingCode: Boolean)`
- `languages`
- `studyLevels`
- `gradingSystems(countryId: Int)` / `gradingSystem(gradingId: Int!)`
- `__schema` (introspection is open)

Everything else — `students`, `student`, `orders`, `institution`, `myAgent`,
`studentDocuments`, `courseStats`, `connectInvites`, all `intake_control_*`
fields, and every mutation other than the login/forgot-password pair — is gated.

## 2. Log in

There are two distinct audiences and two distinct mutations.

For a recruiter, agent or institution staff account:

```graphql
mutation {
  userLogin(email: "you@example.com", password: "…") {
    # select the fields UserAuthPayload actually declares —
    # read them from graphql/adventusio.graphql before sending
  }
}
```

For a student account, use `studentLogin(email:, password:)`, which returns
`StudentAuthPayload`.

Credentials are issued with an Adventus partner account. There is no self-serve
API key and no signup flow that produces one — see
`https://adventus.io/recruiter-signup/`.

## 3. Send the token

Put the token in the standard header:

```
Authorization: Bearer <token>
```

No token lifetime, refresh mechanism or expiry is documented anywhere. Treat
expiry as unknown: handle `UNAUTHENTICATED` at any time and re-login rather than
assuming a session length.

## 4. Recognise a failure

The API is GraphQL, so an auth failure arrives as **HTTP 200** with:

```json
{"errors":[{"message":"401: Unauthorized","path":["students"],
  "extensions":{"code":"UNAUTHENTICATED"}}],"data":{"students":null}}
```

Read `errors[].extensions.code`, not the HTTP status. See
`errors/adventusio-error-codes.yml`.

## 5. Adventus Connect uses a different credential

`pendingConnectCount(accessToken: String!)` and
`connectInvites(accessToken: String!, …)` take the credential as a **GraphQL
argument**, not a header. That token is delivered out of band (the
`ConnectSource` enum is `EMAIL` or `WEB`). Because it travels in the query body
it can be captured by query logging and by the API's own error payloads — do not
reuse the session bearer token there and do not log the query.

## 6. Log out

`userLogout` / `studentLogout` revoke the session server-side. Call one when you
are done; there is no other revocation surface.
