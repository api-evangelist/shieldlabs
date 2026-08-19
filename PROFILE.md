# ShieldLabs

Anonymous visitor identification and fraud-prevention platform. A browser ES-module snippet collects 100+ device and network signals and returns six persistent identifiers plus an explainable 0-100 Risk Score; the customer's own code makes the allow/challenge/block decision. Delivery is a signed, at-most-once webhook, backed by two server-side REST surfaces described by a public OpenAPI 3.1 specification.

- **Provider:** https://shieldlabs.ai
- **Documentation:** https://docs.shieldlabs.ai/
- **Base URL:** https://api.shieldlabs.ai (Management API) · https://account.shieldlabs.ai/api (History API)
- **OpenAPI:** https://github.com/ShieldLabs-ai/shieldlabs-openapi · https://docs.shieldlabs.ai/references/openapi.yaml
- **Tags:** fraud-detection, abuse-prevention, visitor-identification, device-fingerprinting, bot-detection, vpn-proxy-detection, risk-scoring, identity, security
- **Public API:** yes
- **Agent-native (MCP / llms.txt / agent card / skills):** yes

## APIs

- **ShieldLabs Server API** — Three operations plus one OpenAPI 3.1 webhook, across two hosts implemented by two different internal services. History API on `account.shieldlabs.ai/api` (Private API Key `sec_…`, snake_case, `{data,total}` envelope, free — does not consume request balance). Management API on `api.shieldlabs.ai` (Secret Key + `X-Shield-Domain`, PascalCase arrays, bills one request per returned row, minimum one). (`https://api.shieldlabs.ai`)

## What the enrichment pass found

The original submission profile recorded "no discoverable machine-readable contract (OpenAPI/AsyncAPI/GraphQL/Postman)" and treated the SDK claims as unverified. Contract discovery overturned the first finding and confirmed a sharper version of the second.

**Found, and now harvested:**

- A real **OpenAPI 3.1.0** specification, published twice by the provider — at `https://docs.shieldlabs.ai/references/openapi.yaml` (linked from the docs `llms.txt`, under an `## OpenAPI Specs` heading) and in the company's own MIT-licensed repo `github.com/ShieldLabs-ai/shieldlabs-openapi`, which describes itself as the source of truth. The two copies differ by one field: the repo copy carries a 19th detection flag, `stun_request_seen`, that the docs copy omits.
- A **JSON Schema 2020-12** document for the `identification.scored` webhook, derived by the provider 1:1 from the live service code, plus a sample payload and a signature-verification guide.
- A live, **unauthenticated MCP server** at `https://docs.shieldlabs.ai/mcp`, advertised at `/.well-known/mcp.json`. `tools/list` answers anonymously with three tools. It is a *documentation* MCP server — none of its tools reach the Server API.
- An **A2A agent card** at `https://docs.shieldlabs.ai/.well-known/agent-card.json` (grades `flavored` against A2A 1.0.0 — it uses `supportedInterfaces` rather than `additionalInterfaces`, and `protocolVersion` is the pre-1.0 `0.3`).
- A **provider-authored Agent Skill** at `/.well-known/agent-skills/shieldlabs/skill.md`, referenced from the agent card. Saved verbatim; not generated on the provider's behalf.
- A first-party **GitHub organization**, `github.com/ShieldLabs-ai`, with eight client libraries, an examples repo and the spec repo.

**Confirmed absent:**

- **No published SDK.** All eight libraries exist as source and none are on a registry — npm 404s for `@shieldlabs/js`, `/react`, `/vue`, `/next`, `/node`; PyPI 404s for `shieldlabs`; Packagist 404s for `shieldlabs/shieldlabs`; the Go module proxy returns no versions because no repo carries a tag. Four of the manifests still read `0.0.0`. The only installable client artifact is the **unpinned** CDN snippet.
- **No status page.** `status.shieldlabs.ai` does not resolve; `shieldlabs.statuspage.io` is an unclaimed subdomain serving Atlassian's own marketing page.
- **No security.txt** on any of five probed hosts, though a genuine responsible-disclosure statement with a safe-harbour sentence is published in the docs.
- **No AsyncAPI**, and no compliance program — the privacy page states outright that it "makes no regulatory compliance claim". No SOC 2, ISO 27001 or trust center.
- **No rate-limit response headers.** The limits are documented precisely (20 req/min per IP, then a one-hour IP ban; 512 concurrent connections; 512 KB bodies) but nothing is signalled at runtime, and a banned request is written to history with a score of `999` — outside the documented 0-100 range — where it can reach a webhook handler as a maximum-risk visitor.

_Profiled by API Evangelist from the provider's own public surface._
