# Task List API

A small JSON API for a single user's task list. This document is the
specification. The Nerine scripts in `tests/` encode the parts of it a running
server can be checked against.

## 1. The server

1.1. The server listens on `127.0.0.1:8080`. The scripts in `tests/` name that
address directly, so a server listening anywhere else fails every test case for
want of a connection.

1.2. The implementation language, framework, and storage are unconstrained.
Tasks may be held in memory and lost when the process exits.

1.3. The server starts with one command. Document that command in a `README.md`
next to the source.

## 2. Data

2.1. A task has three fields.

| Field | Type | Notes |
|---|---|---|
| `id` | string | Server assigned, a lowercase UUID, immutable |
| `title` | string | 1 to 200 characters after trimming surrounding whitespace |
| `done` | boolean | `false` on creation |

2.2. There is one account. The username is `demo` and the password is
`s3cr3t-demo-pw`. The server runs on the loopback interface and holds no real
data, so the credentials live in the specification rather than in a store.

2.3. Tasks are one shared collection. Every session sees every task.

## 3. Sessions

3.1. `POST /api/session` authenticates. The request body is
`application/x-www-form-urlencoded` and carries `username` and `password`.

3.2. On success the response is `201` with the body `{"username":"demo"}` and
two cookies.

| Cookie | Value | Attributes |
|---|---|---|
| `session` | 32 lowercase hexadecimal characters | `Path=/; HttpOnly; SameSite=Strict` |
| `csrf_token` | 32 lowercase hexadecimal characters | `Path=/; SameSite=Strict` |

The `session` cookie is `HttpOnly` because no client script needs to read it.
The `csrf_token` cookie is not, because the client has to read it to echo it
back per 4.2.

3.3. On failure the response is `401` with the body
`{"error":"invalid credentials"}` and no cookies.

3.4. `DELETE /api/session` ends the session and responds `204`. The response
expires the `session` cookie. Each login creates an independent session, and
ending one session leaves every other session working.

## 4. Access control

4.1. Every `/api/tasks` request requires a valid `session` cookie. A request
without one is `401` with the body `{"error":"authentication required"}`.

4.2. Every request that changes state, which is every `POST`, `PATCH`, and
`DELETE`, requires an `X-CSRF-Token` header holding the value of the
`csrf_token` cookie for that session. A request with a missing or wrong token
is `403` with the body `{"error":"invalid csrf token"}`. `POST /api/session` is
the exception, because it is the request that issues the token.

4.3. Reads, which are `GET` requests, need no `X-CSRF-Token` header.

## 5. Tasks

Every response in this section is JSON. A task serializes as
`{"id":"...","title":"...","done":false}`.

| Request | Success | Notes |
|---|---|---|
| `GET /api/tasks` | `200` | A JSON array, `[]` when there are no tasks |
| `POST /api/tasks` | `201` | Body `{"title":"..."}`. Sets `Location` to `/api/tasks/{id}` |
| `GET /api/tasks/{id}` | `200` | The task |
| `PATCH /api/tasks/{id}` | `200` | Body carries `title`, `done`, or both. Returns the updated task |
| `DELETE /api/tasks/{id}` | `204` | No body |

5.1. `POST` and `PATCH` bodies are `application/json`. A request with any other
content type is `415` with the body
`{"error":"unsupported media type"}`.

5.2. A body that is not valid JSON is `400` with the body
`{"error":"malformed json"}`.

5.3. A `POST` with no `title`, or a `title` that is empty or only whitespace, is
`400` with the body `{"error":"title is required"}`. A `title` longer than 200
characters is `400` with the body `{"error":"title is too long"}`.

5.4. A request for an `{id}` that does not exist is `404` with the body
`{"error":"task not found"}`.

5.5. Any other method is `405` with the body `{"error":"method not allowed"}`.
The `Allow` header lists `GET, POST` on `/api/tasks` and `GET, PATCH, DELETE`
on `/api/tasks/{id}`.

## 6. Every response

6.1. Responses carry these headers, whatever the status code.

| Header | Value |
|---|---|
| `Content-Security-Policy` | `default-src 'none'; frame-ancestors 'none'` |
| `X-Content-Type-Options` | `nosniff` |
| `Cache-Control` | `no-store` |

The API returns no HTML and loads nothing, so its policy denies everything
rather than falling back to `'self'`. There is no `Strict-Transport-Security`
header, because the server speaks plain HTTP on the loopback interface and has
no business claiming otherwise.

6.2. A response with a body sets `Content-Type` to exactly `application/json`,
with no charset parameter. A `204` response has no body and sets no
`Content-Type`.

## 7. Order of checks

A request that is wrong in more than one way gets one status code, so the
checks run in a fixed order and the first failure wins.

1. Path and method, giving `404` or `405` per 5.5
2. Authentication, giving `401` per 4.1
3. CSRF token, giving `403` per 4.2
4. Content type, giving `415` per 5.1
5. Body parsing and validation, giving `400` per 5.2 and 5.3
6. Task lookup, giving `404` per 5.4

An unauthenticated `PUT /api/tasks` is therefore `405` and not `401`, and an
unauthenticated `POST /api/tasks` with no CSRF token is `401` and not `403`.
