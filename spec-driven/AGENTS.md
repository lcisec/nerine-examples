# Instructions for the agent

Copy this file into the project you are building, as `AGENTS.md` or as
`CLAUDE.md`, next to `SPEC.md` and `tests/`.

## What you are building

Build the server described in `SPEC.md`. Pick the language, the framework, and
the storage yourself. The specification constrains what goes over the wire and
nothing else.

## The contract

`SPEC.md` says what to build. The Nerine scripts in `tests/` say how the result
is checked. The scripts are the acceptance criteria, and they are not yours to
change.

1. Do not edit, delete, rename, or add to anything in `tests/`. If you believe a
   script is wrong, stop and say so instead of changing it.
2. Do not write code whose purpose is to satisfy a test rather than the
   specification. Matching on a request path to return a canned response, or
   special casing the exact titles the scripts use, is failing the task with a
   passing score.
3. Where `SPEC.md` and a script disagree, stop and ask. One of the two is wrong,
   and guessing buries the question.
4. `SPEC.md` covers more than the scripts check. Passing every script is
   necessary, not sufficient. Build the whole specification.

## The loop

Work in this order and do not skip ahead.

1. Read `SPEC.md` all the way through, including section 7 on the order of
   checks. Read every script in `tests/`.
2. Implement. Start with sessions in section 3, because everything else is
   behind them.
3. Start the server.
4. Run the scripts. Read every error.
5. Fix the cause of the first error, not the symptom. Repeat from step 3.
6. Stop when the scripts run clean and you have implemented the rest of the
   specification.

## Running the scripts

Start the server on `127.0.0.1:8080`, which is the address the scripts name, then,
from the project root:

```
nerine "tests/*.txt"
```

Nerine takes one file pattern rather than a list of files, so quote the pattern
and let Nerine expand it. Add `-v` for verbose output while you are debugging.

Nerine prints an error for every failed comparison and prints nothing for a
comparison that passes. Silence is a pass. An error names the comparison, the
script, the line, and the test case:

```
ERR comparison `compare status == 201` failed line=14 script=02-tasks.txt test="POST http://127.0.0.1:8080/api/tasks"
```

## What the scripts do not check

Build these from the specification. Nothing will tell you that you got them
wrong.

- The `HttpOnly`, `Path`, and `SameSite` attributes on both cookies, per 3.2.
- The 200 character limit on a title, per 5.3.
- That ending one session leaves other sessions working, per 3.4.
- That `GET /api/tasks` returns `[]` rather than `null` when there are no
  tasks, per 5.

## Things that will cost you a run

- A test case with no `compare status` line still asserts `200`. Every other
  status code has to be stated.
- Nerine sends `application/x-www-form-urlencoded` unless a test case says
  otherwise, which is why the JSON cases carry `modify type application/json`.
- The scripts match the body with regular expressions that allow whitespace
  after a colon, so serialize JSON however your language does it by default,
  per 6.3. The field names and the values are what the scripts read.
- `Content-Type` is exactly `application/json`. A charset parameter fails the
  comparison.
- Scripts run in parallel against one server, and they share the task
  collection. Do not make a request handler depend on the collection being
  empty, on a task being the only one, or on any particular ordering.
