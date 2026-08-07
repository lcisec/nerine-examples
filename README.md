# nerine-examples

Example scripts for [Nerine](https://lcisec.dev/products/nerine). Each one is a plain text
file: a request, then the checks the response has to pass.

## The scripts

| Script | What it demonstrates | What it asserts |
|---|---|---|
| [`tests/up.txt`](tests/up.txt) | The smallest useful script — one request, no explicit checks | The target is reachable and returns `200` |
| [`tests/nav.txt`](tests/nav.txt) | Checking rendered content across several pages in one script | Five pages on lcisec.com each return `200` and carry all six navigation links |
| [`tests/security-headers.txt`](tests/security-headers.txt) | Header assertions, including negative ones and a regex | CSP falls back to `'self'` and allows no wildcard source, no `unsafe-inline`, no `unsafe-eval`; HSTS sets a max age with subdomain coverage and preloading |
| [`tests/security-yoursite.txt`](tests/security-yoursite.txt) | A template to point at your own site | The `.well-known` files a site is expected to publish are present and served |

## Running them

Every test case carries an implicit `compare status == 200`, so a request with no `compare`
lines is already an assertion that the request succeeded.

Scripts that use `${TARGET_SERVER}` are run against a target you supply — `-s` for a single
server, `-S` for a file of servers. Both flags are Nerine Professional and Enterprise:

```
nerine-pro -S targets.txt tests/up.txt tests/security-headers.txt
```

`nav.txt` names its URLs directly and runs under any version:

```
nerine tests/nav.txt
```

`targets.txt` points at lcisec.dev, and `up.txt`, `nav.txt` and `security-headers.txt` all
pass against it on a fresh clone. **`security-yoursite.txt` is the exception and is meant to
fail** until you point it somewhere that publishes those files — treat it as a starting
point, and delete whichever cases do not apply to you.
