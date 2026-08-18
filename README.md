# Anonmyz

**A local secret-masking DLP proxy for AI tools.** Anonmyz detects supported secrets in a prompt, replaces them with request-scoped placeholders before the request leaves your machine, sends the sanitized request to the configured AI provider, and restores known placeholders in the response locally.

There is no Anonmyz cloud service, account, database, or telemetry.

```text
AI client -> Anonmyz on localhost -> configured AI provider
                 |                         |
                 | mask before upload      | placeholders only
                 |                         |
                 +---- restore locally <---+
```

Anonmyz reduces accidental disclosure in proxied request bodies. It is not a semantic classifier, a sandbox, or a guarantee that every secret will be detected.

## A concrete example

Given this request body:

```json
{"prompt":"Deploy with password=correct-horse-battery-staple"}
```

Anonmyz keeps the original value in a request-scoped, in-memory vault and sends a body shaped like this upstream:

```json
{"prompt":"Deploy with password=[[SECRET_7B31A2C4]]"}
```

If the provider returns `[[SECRET_7B31A2C4]]`, Anonmyz replaces that exact placeholder with `correct-horse-battery-staple` before the local client receives the response. The placeholder suffix above is illustrative.

## The three questions to ask before trusting it

### How reliable is restoration?

Restoration is deterministic for an exact, known placeholder within the same request/response exchange:

- Each request gets an isolated in-memory vault. A placeholder from another request cannot restore a value from this request.
- Repeated occurrences of the same secret in one request reuse the same placeholder, and every exact occurrence is restored.
- Random placeholder collisions are rejected instead of overwriting an existing mapping; Anonmyz generates another placeholder. If it cannot reserve storage safely, the request is blocked rather than forwarded partially masked.
- Unknown placeholders remain unchanged and never resolve to an unrelated value.
- Streaming responses use a bounded rolling buffer so placeholders split at any byte boundary can be reassembled before restoration.

There is an important limit: the model must return the placeholder exactly. If it edits, translates, truncates, or omits the placeholder, Anonmyz cannot infer the intended original and does not guess.

### What happens to an unrecognized secret?

It passes through unchanged. Detection is pattern-based and best-effort; there is no honest way to claim otherwise.

Anonmyz can miss proprietary or newly introduced token formats, arbitrary passwords without recognizable assignment context, secrets expressed as natural language, and values hidden inside encoded, encrypted, compressed, image, or other non-matching content. It scans request bodies that actually traverse the proxy. Required authentication headers pass through, and traffic that bypasses the configured proxy is outside its scope.

Treat Anonmyz as a guardrail against common accidental disclosure, not as proof that a prompt is safe to publish.

### What formats are in scope?

The current registry contains 28 pattern detectors:

- API keys and tokens: Anthropic, OpenAI, Google, GitHub PAT/OAuth/Actions tokens, GitLab PAT, AWS access-key IDs and secret-access keys, Stripe, Slack, JWTs, HTTP Bearer tokens, and PEM private-key blocks.
- Contextual credentials: common inline assignments such as `password=...` or `api_key: ...`, plus shell `export`/`set` assignments.
- Personal and financial data: email addresses, credit-card numbers, Turkish and international IBANs, and Turkish phone numbers.
- Checksum- or structure-validated national IDs: Turkish TC Kimlik, Brazilian CPF, Spanish DNI, Indian Aadhaar, and Italian Codice Fiscale.
- Optional local context: Unix and Windows absolute filesystem paths.

This list is the scope, not a claim of universal secret detection.

## Security boundaries

- Only request bodies routed through the local proxy are inspected.
- Detection can produce false negatives and false positives.
- Required provider authentication headers are not masked.
- Traffic that bypasses the configured proxy is not protected.
- Restoration only applies to exact placeholders known to the current request.
- The request-scoped local vault is wiped when the request/response exchange completes.
- An inability to mask safely must block the request rather than forward a partially sanitized body.

## Run the local proof

Prerequisite: Go 1.22 or later. The demo uses loopback networking only and does not require an API key.

```bash
go build -trimpath -o anonmyz .
./anonmyz demo --non-interactive
```

Windows PowerShell:

```powershell
go build -trimpath -o anonmyz.exe .
.\anonmyz.exe demo --non-interactive
```

The demo starts the production proxy and a local mock model, proves the original fake values are absent upstream, verifies that placeholders are present, splits a placeholder across SSE writes, and confirms local restoration. It exits non-zero when an invariant fails.

## Source map

- [`patterns/`](patterns/) — supported pattern registry and validators
- [`masker/`](masker/) — masking and exact placeholder restoration
- [`vault/`](vault/) — request-scoped in-memory mappings
- [`proxy/`](proxy/) — explicit HTTP proxy and streaming response path
- [`mitm/`](mitm/) — optional transparent proxy mode
- [`THREAT_MODEL.md`](THREAT_MODEL.md) — security boundaries and non-goals

## Verify

```bash
go test ./...
go test -race ./...
go vet ./...
```

## License

Apache License 2.0.
