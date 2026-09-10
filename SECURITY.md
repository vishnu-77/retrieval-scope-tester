# Security

This is a testing tool, not a runtime component — it is not intended to sit in
a production request path.

Two things are still worth reporting privately rather than in a public issue:

- A way to make the tester report **PASS** on a retrieval result that actually
  violates the configured scope (a false negative is the failure mode that
  matters here — it hides a real authorisation bug).
- Command, header, or path injection through a test plan, fixture, or adapter
  configuration.

Report these through
[GitHub private vulnerability reporting](https://github.com/vishnu-77/retrieval-scope-tester/security/advisories/new).

Anything else — crashes, parsing errors, adapter bugs — can go straight to a
public [issue](https://github.com/vishnu-77/retrieval-scope-tester/issues).

## Note on running it

Test plans, fixtures, and adapter configs are executable input: `--command`
and `--mutator-command` run programs, and `--http-config` sends identity
headers to whatever URL it names. Only run plans and configs you trust, and
point the tester at systems you are authorised to test.
