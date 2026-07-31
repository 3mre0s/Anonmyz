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
- Placeholder collisions must be rejected rather than allowed to overwrite an existing mapping.
- Unknown placeholders remain unchanged and never resolve to an unrelated value.
- Streaming implementations must buffer incomplete placeholder prefixes so network chunk boundaries do not break restoration.

There is an important limit: the model must return the placeholder exactly. If it edits, translates, truncates, or omits the placeholder, Anonmyz cannot infer the intended original and does not guess.

### What happens to an unrecognized secret?

It passes through unchanged. Detection is pattern-based and best-effort; there is no honest way to claim otherwise.

Anonmyz can miss proprietary or newly introduced token formats, arbitrary passwords without recognizable assignment context, secrets expressed as natural language, and values hidden inside encoded, encrypted, compressed, image, or other non-matching content. It scans request bodies that actually traverse the proxy. Required authentication headers pass through, and traffic that bypasses the configured proxy is outside its scope.

Treat Anonmyz as a guardrail against common accidental disclosure, not as proof that a prompt is safe to publish.

### What formats are in scope?

The initial detector scope is:

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
- The local vault must be wiped when the request/response exchange completes.
- An inability to mask safely must block the request rather than forward a partially sanitized body.

## Status

This repository is the new home for Anonmyz. Source, tests, runnable examples, release artifacts, and the complete threat model will be published here separately from the original hackathon repository.

## License

Apache License 2.0.
