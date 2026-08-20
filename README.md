# nerine-examples

Example scripts for [Nerine](https://lcisec.dev/products/nerine). A script is a plain text
file. It holds a request and the checks the response has to pass.

The scripts in `tests/` are single files, each one demonstrating a part of the language.
[`spec-driven/`](spec-driven) is a longer worked example that uses Nerine to hold an LLM
agent to a specification while it builds a web application. [`writing-tests/`](writing-tests)
holds a reference for handing an LLM a specification and getting Nerine scripts back.

## The scripts

| Script | What it demonstrates | What it asserts |
|---|---|---|
| [`tests/up.txt`](tests/up.txt) | The smallest script that does anything, which is one request with no checks written out | The target answers and returns `200` |
| [`tests/nav.txt`](tests/nav.txt) | Checking page content across several pages in one script | Five pages on lcisec.com return `200` and carry all six navigation links |
| [`tests/security-headers.txt`](tests/security-headers.txt) | Header checks, including negative checks and a regex | CSP falls back to `'self'` and allows no wildcard source, no `unsafe-inline`, and no `unsafe-eval`. HSTS sets a max age, covers subdomains, and preloads |
| [`tests/security-yoursite.txt`](tests/security-yoursite.txt) | A template to point at your own site | The `.well-known` files a site is expected to publish are served |

## Running them

Every test case carries an implicit `compare status == 200`. A request with no `compare`
lines still asserts that the request succeeded. That is the whole of `up.txt`.

A script that uses `${TARGET_SERVER}` needs a target. Use the `-s` flag for a single server
or the `-S` flag for a file of servers. Both flags are Nerine Professional and Enterprise.

```
nerine-pro -S targets.txt tests/security-headers.txt
```

This command checks the CSP and HSTS headers on every server in `targets.txt`.

`nav.txt` names its URLs directly, so it takes no target and runs under any version.

```
nerine tests/nav.txt
```

This command reads five pages on lcisec.com and fails if a navigation link is missing.

Nerine takes one file pattern, not a list of files. Quote the pattern so the shell hands it
to Nerine instead of expanding it first.

```
nerine-pro -S targets.txt "tests/*.txt"
```

This command runs all four scripts against every server in `targets.txt`.

`targets.txt` points at lcisec.dev. Run `up.txt`, `nav.txt`, or `security-headers.txt`
against it on a fresh clone and they pass.

Keep in mind, `security-yoursite.txt` is meant to fail. It asks for `.well-known` files that
most sites do not serve, and lcisec.dev serves one of the three. Point it at your own site
and delete the cases that do not apply to you.

## Building an application with an agent

[`spec-driven/`](spec-driven) holds a specification for a small task list API, six Nerine
scripts that cover it, and the instructions to hand an LLM agent. The agent writes the
application. The scripts decide whether it is finished, and the agent is told it may not
touch them.

Unlike the scripts above, this example ships no application to run against. The
specification and the scripts are the input, and the agent produces the rest. Like
`nav.txt`, the scripts name their URLs directly, so they take no target and run under any
edition. See [`spec-driven/README.md`](spec-driven/README.md).

## Writing tests with an LLM

[`writing-tests/REFERENCE.md`](writing-tests/REFERENCE.md) is a different kind of
reference: not an example to run, but a complete, self-contained description of the
Nerine scripting language, meant to be handed to an LLM in the same prompt as a
specification so the model can write the Nerine scripts that check it directly, with
nothing else. Where `spec-driven/` hands an agent a specification and scripts that
already exist and has it build the application those scripts hold it to, here there is
no application and no scripts yet — the model's job is to produce the scripts
themselves.
