# Retrieval Scope Tester

[![tests](https://github.com/vishnu-77/retrieval-scope-tester/actions/workflows/test.yml/badge.svg)](https://github.com/vishnu-77/retrieval-scope-tester/actions/workflows/test.yml)
[![python: 3.10+](https://img.shields.io/badge/python-3.10%2B-blue)](https://www.python.org/downloads/)
[![license: MIT](https://img.shields.io/badge/license-MIT-green)](LICENSE)

A provider-agnostic reference implementation for testing **retrieval-scope enforcement** in RAG systems before any LLM or generation layer is involved.

It addresses the gap described in OWASP AI Exchange issue #211: a RAG application can appear safe at the prompt/model layer while its retriever or vector index still returns chunks that the current identity should not be able to access.

## Relationship to the OWASP AI Exchange

This is an **external** reference implementation of the retrieval-scope enforcement approach documented in the OWASP AI Exchange under [RAG system security testing](https://owaspai.org/go/ragtesting). It is not an OWASP project and is not maintained by the AI Exchange, which deliberately stays at the level of threats, controls and methodology; executable tooling lives in its own repository.

Discussed in [OWASP/www-project-ai-security-and-privacy-guide#213](https://github.com/OWASP/www-project-ai-security-and-privacy-guide/pull/213), addressing issue [#211](https://github.com/OWASP/www-project-ai-security-and-privacy-guide/issues/211).

## Security property

For every retrieval result:

```text
authorised(identity, document_id, chunk_id, tenant) == true
```

The tester evaluates the retrieval result itself. A downstream model refusing to quote or use an unauthorised chunk does not make the retrieval safe because the data has already crossed the retrieval trust boundary.

## Baseline checks

The bundled plan covers:

- `RAG-AUTH-001` Cross-tenant retrieval isolation.
- `RAG-AUTH-002` Chunk-level ACL enforcement, including different chunks from the same document.
- `RAG-AUTH-003` Role downgrade / least-privilege propagation.
- `RAG-AUTH-004` Permission revocation / stale-index access.

The lifecycle cases run multiple query steps so access can be compared before and after a role or ACL change.

## Requirements

Python 3.10+. No third-party Python packages are required.

## Offline demonstration

Secure fixture:

```bash
python retrieval_scope_tester.py \
  --plan examples/test_plan.json \
  --fixture examples/fixture_secure.json
```

Expected exit code: `0`.

Deliberately vulnerable fixture:

```bash
python retrieval_scope_tester.py \
  --plan examples/test_plan.json \
  --fixture examples/fixture_vulnerable.json
```

Expected exit code: `2`.

The vulnerable fixture demonstrates four failures: cross-tenant leakage, an unauthorised chunk from an otherwise allowed document, stale finance access after role downgrade, and stale access after permission revocation.

## Direct HTTP retrieval adapter

For a retriever or vector-search service that exposes a JSON endpoint, use the built-in HTTP adapter:

```bash
python retrieval_scope_tester.py \
  --plan examples/test_plan.json \
  --http-config examples/http_config.example.json
```

Example configuration:

```json
{
  "url": "http://localhost:8080/retrieve",
  "chunks_field": "chunks",
  "headers": {
    "X-User-ID": "{subject}",
    "X-Tenant-ID": "{tenant}",
    "X-Roles": "{roles}"
  }
}
```

The placeholders `{subject}`, `{tenant}` and `{roles}` are populated from each test identity.

The endpoint receives a JSON POST containing the case, step, query and identity, and must return a chunk list such as:

```json
{
  "chunks": [
    {
      "document_id": "a-finance-1",
      "chunk_id": "revenue-q4",
      "tenant": "tenant-a"
    }
  ]
}
```

This adapter is intentionally generic. It does not require OpenAI, an OpenAI SDK, an LLM provider, or any particular vector database.

## Command adapter

For systems that need custom authentication, SDKs, query filters or provider-specific calls, use `--command`:

```bash
python retrieval_scope_tester.py \
  --plan my_test_plan.json \
  --command "python my_retriever_adapter.py"
```

The request JSON is written to the adapter on stdin. The adapter prints a JSON object containing `chunks`.

This keeps the evaluator independent of Pinecone, Weaviate, Qdrant, Elasticsearch, OpenSearch, pgvector, Chroma, or any other retrieval backend.

## Chunk-level scope

Prefer `allowed_resources` and `denied_resources` for new tests.

A resource can be written as:

```json
"allowed_resources": [
  "a-handbook#public-benefits",
  {
    "document_id": "a-public-1",
    "chunk_id": "overview"
  }
]
```

A document-only reference such as `a-public-1` authorises any returned chunk from that document. A `document_id#chunk_id` reference authorises only that exact chunk.

The older `allowed_document_ids` and `denied_document_ids` fields remain supported for compatibility.

## Lifecycle testing

A case can contain multiple steps. This allows the same test to prove that a security state change is enforced by the retrieval layer instead of merely checking an already-changed static identity.

For offline fixtures, the before/after states are modelled directly in the fixture.

For a live target, provide `--mutator-command` when a step contains a `mutation`. The mutator receives JSON describing the case, step, requested mutation and identity. It can update a source ACL, IAM binding, application policy or test fixture. After the hook succeeds, the tester queries the retriever again and evaluates the post-change result.

Example mutation:

```json
{
  "type": "revoke",
  "subject": "alice",
  "document_id": "a-finance-revoked"
}
```

## Result reasons

Each violating resource can report one or more reasons:

- `outside_authorised_scope` — the chunk is not covered by the step's allowed resources.
- `explicitly_denied` — the chunk matches a denied resource.
- `cross_tenant` — the chunk carries a tenant other than the expected one.
- `missing_tenant` — a tenant was expected but the chunk carried none.

A returned chunk may carry multiple reasons at the same time.

`missing_tenant` fails closed on purpose. A retriever that simply omits the
tenant field cannot demonstrate isolation, so it is treated as a violation
rather than a pass. Single-tenant systems opt out of tenant checking entirely
by leaving `expected_tenant` (and the identity's `tenant`) null, in which case
neither tenant reason is ever raised.

## Tests

Run the regression suite:

```bash
python -m unittest discover \
  -s tests \
  -v
```

The tests cover exact chunk matching, same-document chunk isolation, cross-tenant detection, explicit deny handling, backward-compatible document allow-lists, multi-step lifecycle parsing, fixture step selection and HTTP header templating.

## Exit codes

| Code | Meaning |
| --- | --- |
| `0` | All retrieval-scope cases passed |
| `1` | Configuration, adapter, HTTP or mutation execution error |
| `2` | One or more retrieval-scope violations were detected |

This makes the tester suitable for CI and scheduled security regression checks.

## Contributing

Issues and pull requests are welcome — particularly new backend adapters and
additional canonical test cases.

- [Open an issue](https://github.com/vishnu-77/retrieval-scope-tester/issues)
- [Contribution guide](CONTRIBUTING.md) — how to run the checks, what is in and out of scope, how to add a backend
- [Security policy](SECURITY.md) — how to report a false negative or an injection issue privately
- [Discussion on the OWASP AI Exchange PR](https://github.com/OWASP/www-project-ai-security-and-privacy-guide/pull/213)

The tester stays Python 3.10+ stdlib only. Backends are reached through the
fixture, HTTP, or command adapters rather than by adding vendor SDK
dependencies.

## Scope

This reference implementation tests retrieval authorisation, tenant isolation and security-state propagation. It does not test prompt injection, generation behaviour, model safety, embedding inversion, corpus poisoning or parser vulnerabilities. Those require separate RAG security tests.
