# Building an application with a constrained agent

An LLM agent writing an application will tell you it is finished even if it
isn't. The problem is that the agent is both the builder and the judge, and it
grades its own work against its own memory of what you asked for.

This example takes the judging away from it. A specification says what to build,
a set of Nerine scripts says what the finished thing has to do, and the agent is
told it may not touch the scripts. The agent still writes all the code. It just
no longer decides whether the code is right.

## What is here

| File | Role |
|---|---|
| [`SPEC.md`](SPEC.md) | The specification for a small task list API. Written for a person to read and an agent to implement |
| [`tests/`](tests) | Six Nerine scripts covering the acceptance criteria for the specification |
| [`AGENTS.md`](AGENTS.md) | The instructions handed to the agent. Copy it into the project as `AGENTS.md` or `CLAUDE.md` |

There is no application here yet, so the scripts have nothing to run against
until your agent builds one.

| Script | What it validates |
|---|---|
| [`01-session.txt`](tests/01-session.txt) | Logging in, the two cookies it sets, and logging back out |
| [`02-tasks.txt`](tests/02-tasks.txt) | One task created, read, listed, updated, deleted, and confirmed gone |
| [`03-access-control.txt`](tests/03-access-control.txt) | Anonymous requests, missing and wrong CSRF tokens, and which failure wins |
| [`04-validation.txt`](tests/04-validation.txt) | Bad bodies, bad content types, unknown ids, and unsupported methods |
| [`05-response-headers.txt`](tests/05-response-headers.txt) | The security headers, checked on six different status codes |
| [`06-session-isolation.txt`](tests/06-session-isolation.txt) | Two independent logins, proving that ending one leaves the other working |

## Using it

1. Create an empty project and copy `SPEC.md`, `AGENTS.md`, and `tests/` into
   it.
2. Point your agent at the project and tell it to read `AGENTS.md`.
3. Let it build. It starts the server and runs the scripts itself.
4. Read the diff when it says it is done.

The scripts run against the server the agent starts:

```
nerine "tests/*.txt"
```

They name `http://127.0.0.1:8080` directly rather than taking a target from the
`-s` flag, so they run under any edition of Nerine, Personal included. Point the
server somewhere else and the address in the scripts has to change with it.

## What to expect

Every script fails on the first run, because there is no server listening yet.
That is the starting point rather than a problem.

Nerine logs a line for every failed comparison and says nothing about the ones
that pass, so a clean run prints nothing:

```
ERR comparison `compare status == 201` failed line=14 script=02-tasks.txt test="POST http://127.0.0.1:8080/api/tasks"
```

A silent run is not the same as a finished application. The scripts cover most
of the specification but not all of it -- SPEC.md still has to be read in full,
not just satisfied one script at a time.
