---
generated: '2026-08-12'
method: generated
name: Authorize an OEM domain and receive visitor webhooks
description: Register a customer domain through the RB2B OEM API, deploy the OEM tracking script, and stand up a webhook receiver that will not get disconnected on the first hiccup.
api: asyncapi/rb2b-webhooks.yml
surface: https://app.rb2b.com/api/v1
operations: [add_domain, domains, delete_domain, credit_usage]
source: >-
  Grounded in RB2B's own OEM API & Webhook Guide
  (https://support.rb2b.com/en/articles/12880800-using-the-rb2b-oem-program-api-webhook-guide),
  the webhook response requirements
  (https://support.rb2b.com/en/articles/14300384-responding-to-webhook-requests)
  and the webhook FAQs
  (https://support.rb2b.com/en/articles/9806566-webhook-faqs). Endpoint
  behaviour confirmed by live probe on 2026-08-12 (401
  {"error":"missing_api_key"}). RB2B publishes no OpenAPI for this surface.
---

# Authorize an OEM domain and receive visitor webhooks

The RB2B OEM program embeds RB2B identification inside your own platform. There are exactly four API endpoints and one webhook, and the ordering matters: **a domain that has not been added via the API is ignored even when the script is installed.**

## Auth

- `Api-Key` request header, plus `Content-Type: application/json`, `Accept: application/json`, and a `User-Agent` (RB2B documents the User-Agent as required).
- The OEM key is on the **API Key** card at <https://app.rb2b.com/oem_dashboard>. It is a different key from an API Partner Program key.
- There is no self-service rotation — contact RB2B support to regenerate, then update every dependent system. See `authentication/rb2b-authentication.yml`.

## Steps

1. **Authorize the domain first.** `POST https://app.rb2b.com/api/v1/add_domain` with `{"domain": "sub.domain.com"}`. Use the full host including subdomain. Success returns `{"domain": "sub.domain.com", "added": true}`. The API is the only supported way to do this — there is no dashboard equivalent.
2. **Confirm the authorised set.** `GET https://app.rb2b.com/api/v1/domains` returns the account's active domains. Call this before deleting anything; `delete_domain` on an unknown host returns `{"error": "domain does not exist for account"}`.
3. **Install the OEM script.** Take the bespoke snippet from the **Script Setup** card at <https://app.rb2b.com/oem_dashboard> and place it in the `<head>` of each page you want tracked. The OEM script **cannot run alongside the standard RB2B script** — remove the standard one first. To attribute page views back to your own tenant, set the `customer_id` value on the script; RB2B echoes it onto the webhook payload.
4. **Configure the webhook.** Add your HTTPS receiver URL under **Webhook Configuration** on the OEM dashboard. RB2B will then push **every page view** on your authorised domains, in real time — not one event per identified visitor. Size the receiver for page-view volume, not lead volume.
5. **Retire domains when tenants churn.** `POST /delete_domain` with `{"domain": "..."}` returns `{"domain_deleted": true}`. Collection stops; a still-installed script on an unauthorised domain simply produces nothing.
6. **Watch spend.** `GET https://app.rb2b.com/api/v1/credit_usage` returns `{"credits_used", "last_billing_date", "next_billing_date"}`. This is the OEM counter and is separate from `GET /credits` on the Identity API.

## Building the receiver — the part that breaks

RB2B's delivery contract is strict and unforgiving, and there are no retries:

- **Return `200` to every request, immediately.** Any other status — 4xx, 5xx — or no response within **15 seconds** is treated as your endpoint being offline and **disconnects the integration**. It then has to be re-enabled by hand.
- **Acknowledge first, process later.** Write the payload to a queue and return 200 in the same breath. Never do enrichment, CRM writes, or anything else inline.
- The response body is ignored — empty or a trivial JSON ack is fine.
- **Accept nulls.** Only `LinkedIn URL`, `City`, `State`, `Zipcode`, `Seen At` and `Captured URL` are guaranteed non-null. Schema mismatches ("Expected String but got Null") are RB2B's own documented top cause of auto-disconnects.
- **Remap the field names.** The payload uses Title Case keys with spaces — `"LinkedIn URL"`, `"First Name"`, `"Business Email"`, `"Estimate Revenue"`. The shape is fixed and not customisable; do the mapping on your side.
- `Employee Count` arrives as either an integer or a string. Parse defensively.

## Security you have to add yourself

RB2B sends **unsigned** webhooks and will not attach custom headers — its own guidance is to put any authentication in the URL as query parameters. So:

- Use a long, high-entropy path or query secret, and treat the URL itself as a credential.
- Terminate on HTTPS only and validate the secret before parsing.
- You cannot verify origin cryptographically. If provenance matters, correlate against `customer_id` and your own authorised-domain list, and rate-limit the endpoint.
- See `asyncapi/rb2b-webhooks.yml` and `conformance/rb2b-conformance.yml`.

## Errors

- `401 {"error":"missing_api_key"}` — no `Api-Key` header; `401 {"error":"invalid_api_key_format"}` — malformed key.
- `{"error": "domain does not exist for account"}` on `delete_domain` — list `/domains` first.
- No rate limit is published for this surface, and no `RateLimit-*` headers are returned. See `errors/rb2b-error-codes.yml` and `rate-limits/rb2b-rate-limits.yml`.
