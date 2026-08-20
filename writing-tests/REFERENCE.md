# Nerine Syntax Reference

A complete reference to the Nerine scripting language: every command, every
option each command takes, and the exact rules the parser and the comparison
engine follow. Written to be handed to an LLM alongside a specification, so
the model can write correct Nerine scripts without access to anything else.

## The model

A script is a plain text file holding one or more test cases. A test case is
one HTTP request plus the commands that change it before it is sent, extract
data from the response, and check the response against what is expected.

```
GET https://example.com/
compare status == 200
```

A test case begins with a request line and ends at the first blank line or
the end of the file. Comments start with `#` at the beginning of a line and
can appear anywhere, including between test cases.

A script runs its test cases in order, top to bottom. Multiple scripts run in
parallel and independently of each other.

Every script has its own cookie jar. Every response's `Set-Cookie` headers
are stored in it, and it is attached automatically to every later request in
that script — nothing has to read a cookie back and resend it by hand.
Cookie jars are not shared between scripts.

## The request line

```
METHOD URL
```

`METHOD` is one of `GET`, `HEAD`, `POST`, `PUT`, `PATCH`, `DELETE`. `URL` is
the full request URL, including the scheme.

Every test case starts with two implicit commands, present unless
overridden:

```
modify type application/x-www-form-urlencoded
compare status == 200
```

## Line syntax

Every command line except a `modify body` continuation is split on
whitespace and rejoined with single spaces before it is parsed. Consecutive
spaces inside a value collapse to one. A `modify body` value continued on
tab-indented lines (below) is not affected by this and keeps its whitespace
exactly as written.

## `modify`

Changes the request before it is sent.

```
modify cookie name value
modify header name value
modify body value
modify type content-type
```

- `cookie` and `header` add a cookie or set a header on the request. `name`
  is required for both. Setting a `header` whose name is `Content-Type`
  (case-insensitive) is equivalent to `modify type` and takes effect at the
  same point in the request regardless of where the line appears in the test
  case.
- `body` sets the request body. A value split across multiple lines
  continues on lines that begin with a tab character; the tab is stripped
  and the rest of the line is kept as written, unaffected by whitespace
  collapsing.

  ```
  modify body {
  	"key": "value"
  	}
  ```

- `type` sets the request's `Content-Type` header. It takes exactly one
  word — a content type with parameters needs quoting handled by the
  caller, since Nerine does not do any itself.

A test case can carry any number of `modify` lines. `header` and `type`
lines apply in the order written and each replaces whatever the header held
before (a later line for the same header wins). `cookie` lines add a cookie
rather than replacing one — if the jar already holds a cookie under that
name, the request carries both, and the jar's is always the one a server
sees first when it reads a cookie by name. `modify cookie` therefore cannot
override a cookie the jar already set. To do that, replace the whole
`Cookie` header instead:

```
modify header Cookie session=newvalue; other=value
```

`modify header` replaces a header outright, where `modify cookie` only adds
to it, and `Cookie` is a header like any other.

Every `modify` line's value has `${var}` references resolved against the
script's variables (see Variables) before it is applied.

## `extract`

Pulls data out of the response into a variable for later test cases to use.

```
extract header var name regex
extract cookie var name regex
extract body var regex
```

- `var` is the name to store the match under — the token that comes right
  after `header`, `cookie`, or `body`, not a literal word.
- `header` and `cookie` read from a header or cookie on the response;
  `name` is required for both. `extract cookie` reads the whole
  `Set-Cookie` line for that cookie — the value and every attribute it
  carries (`Path`, `HttpOnly`, `SameSite`, and so on) — not just the value.
  `extract header` reads the header's value; if the response repeats that
  header, only the first occurrence is read.
- `body` reads the response body.

`regex` is a regular expression. It is compiled in multiline mode, so `^`
and `$` match at the start and end of each line, not only the start and end
of the whole value. It cannot contain a capture group — a pattern with one
is rejected when the script loads. It cannot contain a `${var}` reference;
`extract` has no variables of its own to resolve one against.

`extract.FindString` returns the first match, or an empty string if there is
none, and that (possibly empty) result is what gets stored.

A test case can carry any number of `extract` lines, and they run in the
order written, against the same response, before any `compare` line in that
test case runs.

## `compare`

Checks the response against what is expected.

```
compare header name operator value
compare cookie name operator value
compare cookie-raw name operator value
compare body operator value
compare redirect operator value
compare status operator value
```

- `header` compares a header's value. If the response repeats that header,
  only the first occurrence is compared.
- `cookie` compares a cookie's value only. `cookie-raw` compares the whole
  `Set-Cookie` line for that cookie instead — the value and every attribute,
  the same text `extract cookie` reads. Both find the cookie by name, so
  either reaches a cookie regardless of how many `Set-Cookie` headers the
  response carries or what order they arrive in.
- `body` compares the response body.
- `redirect` compares the response's `Location` header. Its value is a
  single word — a redirect target containing a space cannot be compared
  this way.
- `status` compares the numeric status code as a string. Its value is a
  single word, the same as `redirect`. A test case can carry only one
  `status` comparison; a second line overwrites the first rather than
  adding a second check.

`name` is required for `header`, `cookie`, and `cookie-raw`, and is not
present at all for the other three.

`operator` is one of:

| Operator | Meaning |
|---|---|
| `==` | equal |
| `!=` | not equal |
| `contains` | value is a substring of the response data |
| `!contains` | value is not a substring of the response data |
| `~` | response data matches value as a regular expression |
| `!~` | response data does not match value as a regular expression |

For `~` and `!~`, `value` is a regular expression, compiled in multiline
mode like `extract`'s. Unlike `extract`, a capture group in the pattern is
allowed — Nerine never reads submatch data, so a group behaves as a plain,
non-capturing group. `${var}` is allowed inside the pattern: it is resolved
against the script's variables, and the resolved text is matched literally,
so a variable holding a regex metacharacter (`.`, `+`, `(`, and so on) does
not change what the pattern matches — only the variable's exact value does.

Every operator's `value` has `${var}` resolved before the comparison runs,
the same as `modify`.

A test case can carry any number of `compare` lines, all checked against the
same response. Every failure is reported; the first one does not stop the
rest from running.

## Variables

`${name}` in a `modify` or `compare` value, or in a request URL, is replaced
with the value of the variable `name` before that line takes effect. A
reference to a variable that does not exist is left as literal text.

Variables are written only by `extract`, in the same script, in an earlier
test case (or an earlier line in the same test case). They are read only by
`modify` and `compare`. `extract` cannot read a variable in its own pattern.

A variable's scope is the script that extracted it. It cannot be read by
another script running alongside it.

## Running Nerine

```
nerine [options] file_pattern
```

`file_pattern` is a single shell-style glob, quoted so the shell passes it
through rather than expanding it — `nerine "tests/*.txt"`, not
`nerine tests/*.txt`.

Nerine ships as three editions with the same script language: `nerine`
(personal), `nerine-pro`, and `nerine-ent`. Pro and Enterprise add the `-s`
flag (a single target server) and the `-S` flag (a file of target servers),
and make `${TARGET_SERVER}` available in scripts as the server currently
under test. Enterprise additionally loads any environment variable prefixed
`NERINE_` into scripts with the prefix stripped, so `NERINE_HEADING` becomes
`${HEADING}`.

A script that names its URLs directly, rather than through
`${TARGET_SERVER}`, runs unchanged under any edition.

## Output

A comparison that passes is silent. One that fails logs one line naming the
script, the line number, the test case, the comparison as written, what the
response actually held, and what was expected:

```
ERR comparison `compare status == 201` failed line=14 script=tests/session.txt test="POST https://example.com/api/session"
```

A clean run of a script prints nothing.

## Worked example

Given this fragment of a specification:

> `GET /health` returns `200` with the body `{"status":"ok"}` and a
> `Content-Type` of `application/json`.
>
> `GET /health/deep` requires a `session` cookie. Without one it returns
> `401` with the body `{"error":"unauthorized"}`.

The Nerine script that checks it:

```
GET https://example.com/health
compare header content-type == application/json
compare body contains "status":"ok"

GET https://example.com/health/deep
compare status == 401
compare body contains "error":"unauthorized"
```

Each sentence of the specification becomes one or more `compare` lines on
the request it describes. The `200` case needs no `compare status` line of
its own, since `200` is the implicit default; `401` is stated because it
overrides that default.
