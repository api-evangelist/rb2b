---
generated: '2026-08-12'
method: generated
name: Resolve an anonymous visitor to a named person
description: Walk an anonymous website visitor's IP address through the RB2B identity graph to a LinkedIn profile, a business profile, and contact details — spending the fewest credits for the answer you actually need.
api: mcp/rb2b-mcp.yml
surface: https://api.rb2b.com/api/v1
operations: [ip_to_company, ip_to_hem, hem_to_best_linkedin, hem_to_business_profile, linkedin_to_best_personal_email, linkedin_to_mobile_phone, check_credits]
source: >-
  RB2B publishes no OpenAPI, so every tool name, HTTP path, request body key and
  credit cost below is taken from the provider's own published MCP server,
  @rb2b/rb2b-apis-mcp 1.1.7 — tool definitions in mcp/rb2b-mcp-tools.json
  (extracted verbatim from dist/server.js) and the path bindings in
  mcp/rb2b-tool-crosswalk.yml (read from dist/api.js). Nothing here is invented.
---

# Resolve an anonymous visitor to a named person

RB2B's Identity API is a graph, not a CRUD API. Each call is one edge: you hand it a weak identifier and it hands back a stronger one. This skill walks the common path — IP address in, person out — and tells you where to stop.

## Auth

- Static account key in the `Api-Key` request header. Get one at <https://ui.api.rb2b.com>; the API Partner Program account is separate from the consumer account at app.rb2b.com and the keys are not interchangeable.
- Also send `Content-Type: application/json`, `Accept: application/json` and a `User-Agent`.
- See `authentication/rb2b-authentication.yml`.

## Before you start — this is metered, and there is no idempotency key

- Every successful call spends 1–4 credits. Check the balance first: `check_credits` (`GET /credits`, free).
- **There is no `Idempotency-Key` and no replay window.** A retry after a timeout is a second billable call. Cache every result you get and never retry blind. See `conventions/rb2b-conventions.yml`.
- Rate limit: 50 requests/second per endpoint. Exceeding it returns `429` with no `Retry-After`.

## Steps

1. **Decide whether you need a person at all.** If company-level attribution is enough, call `ip_to_company` (`POST /ip_to_company`, body `{"ip_address": "..."}`, 1 credit) and stop. Company-level resolution works globally; person-level resolution is US-only, so outside the US this is the only step that will return anything.
2. **IP → hashed email.** `ip_to_hem` (`POST /ip_to_hem`, body `{"ip_address": "..."}`, 1 credit) returns a **ranked list** of MD5 and SHA256 hashes with confidence scores. Take the highest-confidence MD5 — do not fan out across the whole list, that is one billable call per hash downstream.
3. **Hashed email → LinkedIn.** `hem_to_best_linkedin` (`POST /hem_to_best_linkedin`, body `{"md5": "..."}`, 1 credit) returns the single highest-confidence profile URL. Use `hem_to_linkedin_slug` (`POST /hem_to_linkedin` — note the path is shorter than the tool name) if you want the raw vanity slug instead.
4. **Enrich, once you have a person.** From the LinkedIn slug:
   - `linkedin_to_business_profile` (`POST /linkedin_to_business_profile`, 4 credits) — title, company, industry.
   - `linkedin_to_best_personal_email` (`POST /linkedin_to_best_personal_email`, 1 credit) — one high-confidence personal email. `linkedin_to_personal_email` returns all candidates for the same 1 credit; prefer `_best_` unless you are deduping against a list you already hold.
   - `linkedin_to_mobile_phone` (`POST /linkedin_to_mobile_phone`, 3 credits) — the most expensive per-answer call on the surface. Gate it behind a real qualification decision.
5. **If you already hold an email address, skip steps 2–3 entirely.** The `email_*` tools hit the same `hem_*` endpoints with an `email` key instead of `md5`: `email_to_best_linkedin`, `email_to_business_profile` (2 credits), `email_to_maid`.

## Cheapest path to each answer

| You want | Call | Credits |
|---|---|---|
| Which company is visiting | `ip_to_company` | 1 |
| Which person is visiting | `ip_to_hem` → `hem_to_best_linkedin` | 2 |
| That person's employer + title | + `hem_to_business_profile` (2) instead of the LinkedIn route (4) | 4 total |
| That person's personal email | + `linkedin_to_best_personal_email` | 3 total |
| That person's mobile | + `linkedin_to_mobile_phone` | 5–6 total |
| Ad-targetable device ID | `ip_to_maid` | 1 |

## Errors

- `401 {"error":"missing_api_key"}` — no `Api-Key` header. `401 {"error":"invalid_api_key_format"}` — malformed key.
- `404` means **no identity match**, not a bad route. Do not retry the same input; it will miss again and the retry may be billed.
- `429` — you crossed 50 req/s on that endpoint. Back off; no `Retry-After` is sent.
- Every response carries an undocumented `x-request-id`. Capture it — it is the only handle support can use.
- See `errors/rb2b-error-codes.yml`.

## Handling what comes back

This API returns person-level PII — names, personal emails, mobile numbers — resolved from an anonymous page view. Treat every response as regulated personal data: log the identifiers, not the payloads; honour the opt-out at <https://app.retention.com/optout/>; and confirm your own basis for processing before you enrich. RB2B publishes SOC 2 Type 2, CCPA and GDPR posture (`conformance/rb2b-conformance.yml`), which covers RB2B's obligations, not yours.

## Via MCP

The same 19 operations are available as MCP tools without writing HTTP: `npx -y @rb2b/rb2b-apis-mcp init`, then `claude mcp add rb2b -- npx -y @rb2b/rb2b-apis-mcp`. The server checks your balance before each paid call and appends `[NOTICE]`/`[WARNING]`/`[CRITICAL]` credit warnings at 500/50/10 remaining. See `mcp/rb2b-mcp.yml`.
