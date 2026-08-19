---
name: Shieldlabs
description: Use when implementing visitor identification, detecting anonymity (VPNs, proxies, Tor, anti-detect browsers), scoring fraud risk, preventing account abuse, or analyzing traffic quality. Agents should reach for this skill when building fraud prevention, account security, or traffic quality features that require persistent device identification and risk scoring.
metadata:
    mintlify-proj: shieldlabs
    version: "1.0"
---

# ShieldLabs Skill

## Product summary

ShieldLabs is a visitor identification and risk-scoring platform that detects anonymity signals (VPNs, proxies, Tor, anti-detect browsers) and assigns a 0–100 Risk Score to every visit. It works by loading a browser snippet that collects 100+ signals, sending them to ShieldLabs servers for scoring, and delivering results via webhooks or API. The platform generates persistent DeviceID and VisitorID identifiers that survive cleared cookies and IP changes, enabling you to recognize returning users and detect multi-accounting, account takeover, and abuse patterns.

**Key files and endpoints:**
- **Snippet:** ES module from `cdn.shieldlabs.ai/snippet.js?publicKey=YOUR_PUBLIC_KEY`
- **Webhook delivery:** POST to your registered endpoints with `X-Shield-Signature` header
- **History API:** `account.shieldlabs.ai/api/v1/history/request_id/{requestID}` (read-only, no billing)
- **Management API:** `api.shieldlabs.ai/v1/profile` (profile, balance, per-row-billed history)
- **Dashboard:** `app.shieldlabs.ai` (register domains, manage keys, view analytics)

**Primary docs:** https://docs.shieldlabs.ai

## When to use

Reach for ShieldLabs when you need to:

- **Detect anonymous traffic:** Identify visitors behind VPNs, proxies, Tor, or anti-detect browsers
- **Prevent account abuse:** Detect multi-accounting, account takeover, credential stuffing, and account sharing by comparing DeviceID and VisitorID across accounts
- **Reduce login friction:** Allow returning users on known devices to sign in without extra verification
- **Gate sensitive actions:** Require step-up authentication (2FA, email verification, phone confirmation) for high-risk logins, payments, or withdrawals
- **Analyze traffic quality:** Rank traffic sources by risk and anonymity share to measure real vs. bot/abusive traffic
- **Implement fraud prevention:** Score signups, payments, and withdrawals; block or challenge high-risk visits

Do not use ShieldLabs for: authentication (it identifies devices, not people), IP geolocation alone (it detects masking, not location), or real-time blocking (it scores asynchronously with ~1s latency).

## Quick reference

### Snippet exports

| Export | Use case |
|---|---|
| `checkAnonymous(userHID?, callback?)` | Untagged visitor (logged-out traffic) |
| `checkAuthenticatedUser(userHID, callback?)` | Logged-in user (pass hashed id only) |
| `forceCheckAnonymous(callback?)` | Fresh check, reset session first |
| `forceCheckAuthenticatedUser(userHID, callback?)` | Right after login or before sensitive action |

### Risk Score bands

| Band | Range | Meaning | Default action |
|---|---|---|---|
| Clean | 0–9 | No signals | Allow, no friction |
| Low | 10–29 | One minor signal | Allow, log it |
| Medium | 30–59 | Multiple signals or one moderate | Challenge, 2FA, review |
| High | 60–100 | Strong anonymity signals | Block, verify, manual review |

### API credentials (per domain)

| Credential | Where it goes | Purpose |
|---|---|---|
| Public Key | Snippet URL (`?publicKey=...`) | Safe to expose; identifies domain |
| Private API Key (`sec_…`) | Backend only | Authenticates History API reads |
| Secret Key (hex) | Backend only | Authenticates Management API |
| Webhook secret (`whsec_…`) | Backend only, per endpoint | Verifies `X-Shield-Signature` header |

### Webhook payload structure

```json
{
  "event_type": "identification.scored",
  "data": {
    "request_id": "UUID",
    "device_id": "UUID",
    "visitor_id": "UUID",
    "user_hid": "your_hashed_id_or_null",
    "risk_score": 0-100,
    "signals": [
      { "name": "Proxy", "weight": 10 },
      { "name": "Timezone Mismatch", "weight": 10 }
    ],
    "detection_flags": {
      "tor": false,
      "vpn": false,
      "proxy": true,
      "anti_detect_browser": false,
      "abuser": false,
      "os_mismatch": false
    }
  }
}
```

### Signal weights (highest to lowest)

| Signal | Weight | Meaning |
|---|---|---|
| Tor | 99 | Tor exit node |
| JavaScript Disabled | 90 | Headless/automation client |
| OS Mismatch | 60 | Browser OS ≠ network OS |
| Anti-detect Browser | 60 | Fingerprint spoofing detected |
| Browser Automation | 60 | `navigator.webdriver` flag set |
| Stun not checked | 30 | Network check incomplete |
| OS not Detected | 30 | OS unknown |
| Browser VPN/Proxy | 30 | In-browser VPN extension |
| VPN | 15 | VPN corroborated across signals |
| Privacy Relay | 15 | iCloud Private Relay |
| Proxy | 10 | IP flagged as proxy |
| Datacenter IP | 10 | Hosting/datacenter IP |
| Abuser Flag | 10 | IP on abuse reputation list |
| Timezone Mismatch | 10 | Device TZ ≠ IP geolocation |

## Decision guidance

### When to use webhook vs. History API

| Scenario | Use | Why |
|---|---|---|
| Real-time decisions (login, checkout) | Webhook | ~1s latency, push delivery |
| Guaranteed reads (withdrawal, KYC) | History API fallback | At-most-once webhook delivery; History is reliable |
| Batch analysis, analytics | History API | Query by device_id, visitor_id, user_hid, ip |
| Dashboard-only integration | Neither | Results visible in app.shieldlabs.ai |

### When to challenge vs. block by action

| Action | Clean (0–9) | Low (10–29) | Medium (30–59) | High (60–100) |
|---|---|---|---|---|
| Signup | Allow | Allow | Email verify or CAPTCHA | Reject or review |
| Login | Allow | Allow | Require 2FA | Block, require recovery |
| Checkout | Allow | Allow, log | Step-up (3-D Secure) | Block or review |
| Withdrawal | Allow | Verify | Verify | Manual review hold |
| Content/posting | Publish | Publish | Rate-limit or queue | Hold for moderation |

### When to act on detection_flags vs. risk_score

| Scenario | Use |
|---|---|
| Hard rules (always block Tor) | `detection_flags.tor`, `detection_flags.anti_detect_browser` |
| Contextual decisions (login vs. payment) | `risk_score` band + action context |
| Logging and transparency | Both: log signals, decide on band |

## Workflow

### 1. Plan your implementation

- Decide where to identify: every page, only at login, only at checkout, or all three
- Decide what to do with the score: allow, challenge (2FA), review, or block
- Plan webhook endpoint(s) for real-time scoring; use History API as fallback for critical actions
- Check CSP: if your site sends a strict Content Security Policy, allowlist `cdn.shieldlabs.ai`, `rest.shieldlabs.ai`, `webrtc.shieldlabs.ai`, and your webhook endpoint hosts

### 2. Register domain and get keys

- Sign up at `app.shieldlabs.ai` (free tier: 5,000 identifications)
- Create a domain (e.g., `myshop.com`)
- Copy Public Key (for snippet), Private API Key (for History API), Secret Key (for Management API)
- Store Private API Key and Secret Key in backend environment variables only; never expose in browser or logs

### 3. Install the snippet

- Load the ES module in your page(s) with `import()` (not npm, not classic script)
- Call `checkAnonymous()` for logged-out visitors or `checkAuthenticatedUser(hashedUserId)` for logged-in users
- Pass the `requestID` from the callback to your backend so you can match the webhook or History API result
- Test: load a page with the snippet, check the dashboard within a few seconds to see the visit and its Risk Score

### 4. Register and verify webhook endpoint

- Go to dashboard → your domain → **Webhooks** tab
- Click **Add endpoint**, enter your backend URL (HTTPS only), save
- Copy the `whsec_…` signing secret to a backend environment variable
- Click **Verify** to test; when it returns 2xx, the endpoint is **Active**
- Implement signature verification: HMAC-SHA256 of raw request body with `whsec_…` secret, compare to `X-Shield-Signature` header using constant-time comparison

### 5. Handle the webhook and decide

- Verify `X-Shield-Signature` before trusting the payload
- Respond `200` immediately (webhook timeout is ~1s)
- Upsert on `request_id` to handle redelivery idempotently
- Read `data.risk_score` and `data.signals`; branch on `data.detection_flags` for hard rules
- Map score band + action context to allow / challenge / review / block
- For critical actions (withdrawal, KYC), also read the History API as a fallback

### 6. Verify and tune

- Ship with logging only: record score and signals for every check
- Watch your analytics dashboard for traffic distribution
- Correlate Risk Score bands with your conversion, chargeback, and abuse data
- Gradually tighten thresholds, starting with highest-stakes actions
- Test with real CSP and ad blockers before relying on scores

## Common gotchas

- **Calling the snippet on every page load:** Each call bills 1 request. Move `checkAnonymous()` to the specific touchpoint where you act on the result (login, checkout, signup). Unconditional calls on every navigation waste budget.
- **Passing raw user ids to `checkAuthenticatedUser`:** Always hash the user id (SHA256 or similar). Never pass email, phone, or raw numeric id. ShieldLabs stores this as `user_hid` for correlation; it must not be reversible.
- **Trusting the score alone:** A legitimate user on a corporate VPN or privacy browser can score 60+. Always decide on score + signals + action context. A 30 at signup is different from a 30 at a $5,000 withdrawal.
- **Relying on webhook delivery:** Webhooks are at-most-once with no retries and a ~1s timeout. For guaranteed reads (withdrawal, payout, KYC), poll the History API by `request_id` as a fallback.
- **Re-serializing JSON before signature verification:** HMAC the raw request body bytes exactly as received. Parsing and re-serializing changes the bytes and breaks the signature.
- **Storing Private API Key or Secret Key in the browser:** These are backend-only credentials. If exposed, rotate them immediately from the dashboard. The Public Key is the only safe-to-expose credential.
- **Ignoring CSP:** If your site sends a strict CSP, the snippet is silently blocked. Allowlist `cdn.shieldlabs.ai` (script-src) and `rest.shieldlabs.ai`, `webrtc.shieldlabs.ai` (connect-src), plus your webhook endpoint.
- **Branching on signal label strings:** Signal names like `"Proxy"` are display labels and can change. Branch on stable `detection_flags` booleans and the Risk Score band instead.
- **Not handling the 999 rate-limit marker:** A score of 999 is a rate-limit ban (HTTP 429), not a 0–100 Risk Score. Guard `score > 100` before reading the band.
- **Double-applying effects:** If you read both a webhook and the History API for the same `request_id`, upsert idempotently so the same check does not trigger two business effects (e.g., two blocks, two emails).

## Verification checklist

Before shipping:

- [ ] Snippet loads without errors; check browser console and Network tab
- [ ] Callback fires and returns `requestID`; log it to confirm
- [ ] Webhook endpoint receives signed POST within ~1s of snippet call
- [ ] Signature verification passes; constant-time comparison used
- [ ] Risk Score appears in dashboard within a few seconds
- [ ] Webhook handler is idempotent on `request_id` (upsert, not insert)
- [ ] Decision logic branches on `risk_score` band and `detection_flags`, not signal label strings
- [ ] CSP allows `cdn.shieldlabs.ai`, `rest.shieldlabs.ai`, `webrtc.shieldlabs.ai`, webhook endpoint
- [ ] Private API Key and Secret Key are in backend environment variables only
- [ ] Webhook secret (`whsec_…`) is in backend environment variables only
- [ ] Logging captures score, signals, and decision for audit trail
- [ ] Thresholds tuned gradually; no hard blocks on low-stakes actions
- [ ] History API fallback implemented for critical actions (withdrawal, KYC)
- [ ] Tested with real CSP and ad blocker enabled

## Resources

**Comprehensive page-by-page navigation:** https://docs.shieldlabs.ai/llms.txt

**Critical pages:**
- [Quickstart](https://docs.shieldlabs.ai/quickstart) — 5-minute end-to-end walkthrough
- [Acting on the Risk Score](https://docs.shieldlabs.ai/guides/acting-on-risk-score) — decision logic, per-scenario thresholds, complete handler examples
- [Webhooks setup](https://docs.shieldlabs.ai/setup/webhooks) — registration, signature verification, delivery guarantees, Node/Go/Python handlers

---

> For additional documentation and navigation, see: https://docs.shieldlabs.ai/llms.txt