# Contributing

Contributions are welcome — especially new backend adapters and additional
retrieval-scope test cases.

## Running the checks

No dependencies to install. Python 3.10+ only.

```bash
python -m unittest discover -s tests -v

# end-to-end: must exit 0
python retrieval_scope_tester.py --plan examples/test_plan.json --fixture examples/fixture_secure.json

# end-to-end: must exit 2 (four violations detected)
python retrieval_scope_tester.py --plan examples/test_plan.json --fixture examples/fixture_vulnerable.json
```

CI runs exactly these on 3.10 and 3.13.

## What belongs here

In scope: retrieval authorisation, tenant isolation, chunk-level ACLs, and
propagation of security-state changes (role change, ACL edit, revocation) to
the retriever or index.

Out of scope: prompt injection, generation behaviour, model safety, embedding
inversion, corpus poisoning, parser vulnerabilities. Those are separate RAG
security tests — see the OWASP AI Exchange section on
[RAG system security testing](https://owaspai.org/go/ragtesting).

If a change adds a check, it also adds the fixture that proves the check
fires. A test that cannot fail is not a test.

## Adding a backend

Don't add vendor SDK dependencies. The tester stays stdlib-only, and backends
are reached through one of the existing adapters:

- `--fixture` — a JSON file, for offline demonstration and CI.
- `--http-config` — any JSON retrieval endpoint, with identity templated into
  request headers.
- `--command` — an arbitrary executable that reads a request on stdin and
  prints `{"chunks": [...]}`. Use this for anything needing an SDK, custom
  auth, or provider-specific query filters.

If your backend genuinely cannot be reached through any of the three, open an
issue describing it before writing an adapter.

## Test cases

New canonical cases follow the existing `RAG-AUTH-00N` numbering in
`examples/test_plan.json`, and should state the security property being
asserted, not just the mechanics.

## Reporting problems

Open an [issue](https://github.com/vishnu-77/retrieval-scope-tester/issues).
For a vulnerability in this tool itself, see [SECURITY.md](SECURITY.md).
