---
name: Shieldlabs
description: Use when building visitor identification, fraud detection, and abuse prevention systems. Reach for this skill when implementing risk scoring, anonymity detection (VPN/proxy/Tor), account security (login, signup, payment), or traffic quality analysis. Apply when you need to detect multi-accounting, account takeover, credential stuffing, payment fraud, promo abuse, or bot traffic.
metadata:
    mintlify-proj: shieldlabs
    version: "1.0"
---

# ShieldLabs Skill

## Product summary

ShieldLabs is a visitor identification and risk-scoring platform that detects anonymity signals (VPN, proxy, Tor, anti-detect browsers) and assigns a 0–100 Risk Score to every visit. It works via a browser snippet that collects signals, a server-side scoring engine, and webhooks or API reads to deliver results. Agents use ShieldLabs to prevent fraud, detect abuse, and make access-control decisions (allow, challenge, review, block) based on visitor risk.

**Key files and endpoints:**
- Snippet: `https://cdn.shieldlabs.ai/snippet.js?publicKey=YOUR_PUBLIC_KEY` (browser-side)
- Webhook delivery: POST to your registered endpoint with signed `X-Shield-Signature` header
- History API: `https://account.shieldlabs.ai/api/v1/history/{search_type}/{value}` (Private API Key)
- Management API: `https://api.shieldlabs.ai/v1/profile` (Secret Key)
- Dashboard: `https://app.shieldlabs.ai/`

**Primary docs:** https://docs.shieldlabs.ai

## When to use

Reach for ShieldLabs when:
- **Building login/signup flows** — detect account takeover, credential stuffing, multi-accounting, or bot signups
- **Processing payments** — flag high-risk checkouts, prevent payment fraud, detect card testing
- **Protecting sensitive actions** — gate password resets, withdrawals, or high-value transfers behind step-up authentication
- **Analyzing traffic quality** — rank traffic sources by risk, measure real vs. bot visitors, detect promo abuse
- **Implementing account security** — recognize returning users on known devices, detect account sharing, prevent ban evasion
- **Investigating abuse** — correlate devices, IPs, and accounts to find patterns (multi-accounting, affiliate fraud, sybil attacks)

Do not use ShieldLabs for: authentication (it identifies visitors, not users), authorization (it scores risk, not permissions), or as a standalone blocker (it surfaces evidence; your code decides).

## Quick reference

### Credentials and keys

| Key | Where it goes | What it does | Exposure |
|---|---|---|---|
| **Public Key** | Snippet URL `?publicKey=...` | Identifies domain in browser | Safe to expose |
| **Private API Key** | Backend only (`sec_…`) | Authenticates History API reads | Backend-only secret |
| **Secret Key** | Backend only (hex) | Authenticates Management API | Backend-only secret |
| **Webhook signing secret** | Backend only (`whsec_…` per endpoint) | Verifies `X-Shield-Signature` header | Backend-only secret |

### Risk Score bands

| Band | Range | Meaning | Typical action |
|---|---|---|---|
| **Clean** | 0–9 | No signals | Allow, no friction |
| **Low** | 10–29 | One minor signal | Allow, log it |
| **Medium** | 30–59 | Several signals or one moderate signal | Challenge, 2FA, or review |
| **High** | 60–100 | Strong anonymity signals | Block, verify, or manual review |

### Common signal weights

| Signal | Weight | Meaning |
|---|---|---|
| Tor | 99 | Tor exit node |
| JavaScript Disabled | 90 | Headless or automation |
| OS Mismatch | 60 | OS inconsistency |
| Anti-detect Browser | 60 | Fingerprint spoofing |
| Browser Automation | 60 | Selenium, Puppeteer, etc. |
| VPN | 15 | VPN connection |
| Privacy Relay | 15 | iCloud Private Relay |
| Proxy | 10 | Proxy IP |
| Datacenter IP | 10 | Hosting/datacenter range |
| Timezone Mismatch | 10 | TZ ≠ IP geolocation |

### Snippet calls

```javascript
// Anonymous visitor
mod.checkAnonymous();

// Logged-in user (pass hashed/pseudonymous ID, never raw email)
mod.checkAuthenticatedUser('hashed_user_id');

// With callback to capture requestID
mod.checkAnonymous({
  onInitialized: (result) => {
    if (result.status === 'initialized') {
      const requestID = result.requestID;
      // Forward to backend to correlate with webhook
    }
  }
});
```

### Webhook verification (Node.js)

```javascript
import crypto from 'crypto';

const SECRET = process.env.SHIELDLABS_WEBHOOK_SECRET; // whsec_…
const received = req.get('X-Shield-Signature') ?? '';
const expected = 'sha256=' + 
  crypto.createHmac('sha256', SECRET).update(req.rawBody).digest('hex');

const a = Buffer.from(expected);
const b = Buffer.from(received);
if (a.length !== b.length || !crypto.timingSafeEqual(a, b)) {
  return res.sendStatus(401);
}
res.sendStatus(200);
handleScore(req.body.data); // idempotent on request_id
```

### History API read

```bash
curl "https://account.shieldlabs.ai/api/v1/history/request_id/{requestID}?limit=1" \
  -H "Authorization: Bearer sec_your_private_api_key"
```

## Decision guidance

### When to use webhook vs. History API

| Scenario | Use | Why |
|---|---|---|
| Real-time decision (login, checkout) | Webhook | ~1 second latency, push delivery |
| Fallback for missed webhook | History API | Guaranteed read, no retries on webhook |
| Batch investigation (fraud review) | History API | Poll by device_id, user_hid, or IP |
| High-stakes action (withdrawal) | Both | Webhook + History fallback for certainty |

### When to use which action threshold

| Action | Clean | Low | Medium | High |
|---|---|---|---|---|
| **Signup** | Allow | Allow | Email verify / CAPTCHA | Reject / review |
| **Login** | Allow | Allow | Require 2FA | Block / recovery |
| **Checkout** | Allow | Allow | 3-D Secure / review | Block / review |
| **Withdrawal** | Allow | Verify | Verify | Manual hold |
| **Content/post** | Publish | Publish | Rate-limit / review | Hold / verify account |

### When to override score with detection_flags

Branch on `detection_flags` (stable booleans) for hard rules:

```javascript
if (detection_flags.tor || detection_flags.anti_detect_browser || detection_flags.abuser) {
  return 'verify'; // Always step up on strongest tells
}
return defaultAction(risk_score); // Otherwise use band ladder
```

## Workflow

### 1. Set up the domain and keys

1. Sign up at https://app.shieldlabs.ai/ (free tier: 5,000 identifications)
2. Create a domain (e.g., `myshop.com`)
3. Copy the **Public Key** (safe for browser), **Private API Key** (History API), and **Secret Key** (Management API)
4. Store Private and Secret keys as backend environment variables only

### 2. Install the snippet

1. Load the ES module on pages where you identify visitors:
   ```html
   <script type="module">
     const mod = await import('https://cdn.shieldlabs.ai/snippet.js?publicKey=YOUR_PUBLIC_KEY');
     mod.checkAnonymous(); // or checkAuthenticatedUser('hashed_id')
   </script>
   ```
2. Capture the `requestID` from the `onInitialized` callback and forward to your backend
3. Verify the snippet loads (check browser Network tab for `cdn.shieldlabs.ai/snippet.js` with 200 status)
4. Verify CSP allows `script-src https://cdn.shieldlabs.ai` and `connect-src https://rest.shieldlabs.ai`

### 3. Register and verify a webhook endpoint

1. Open the dashboard, go to your domain's **Webhooks** tab
2. Click **Add endpoint**, enter your HTTPS URL (e.g., `https://myshop.com/webhooks/shieldlabs`)
3. Copy the `whsec_…` signing secret and store it as a backend environment variable
4. Click **Verify** to send a test ping; status should flip to **Active**
5. Implement the webhook handler: verify `X-Shield-Signature`, respond `200` fast, then process the score

### 4. Implement decision logic

1. Parse the webhook `data.risk_score` and `data.signals`
2. Map the score to a band (Clean 0–9, Low 10–29, Medium 30–59, High 60–100)
3. Apply the per-action threshold from the decision table above
4. Cross-reference your own data (known-bad devices, account history)
5. Make the decision (allow, challenge, review, block) in your application code
6. Log the score, signals, and decision for audit and tuning

### 5. Verify and tune

1. Load a page with the snippet; confirm the visit appears in the dashboard within seconds
2. Trigger a webhook to your endpoint; verify the signature and payload
3. Watch the analytics dashboard to see how real traffic distributes across bands
4. Gradually tighten thresholds, starting with the highest-stakes actions
5. Use the History API to investigate patterns (multi-accounting, account takeover, etc.)

## Common gotchas

- **Reusing keys across domains:** Each domain gets its own key set. Do not use one domain's keys on another site.
- **Exposing Secret Key or Private API Key:** These are backend-only secrets like database passwords. Never put them in the browser, snippet, client logs, or public repositories. Rotate immediately if leaked.
- **Forgetting to verify `X-Shield-Signature`:** Always constant-time compare the HMAC-SHA256 of the raw request body. Re-serializing JSON changes the bytes and breaks the signature.
- **Hashing the user ID:** Pass a hashed or pseudonymous ID to `checkAuthenticatedUser()`, never a raw email or user ID. ShieldLabs does not store it; it is for your own correlation.
- **Blocking on score alone:** A legitimate user can score high (corporate proxy, VPN, privacy browser). Always decide on score + signals + action context, never the number alone.
- **Relying on webhook alone:** Webhooks are at-most-once with no retries. For guaranteed reads, poll the History API by `request_id`.
- **Ignoring signal names:** Branch on stable `detection_flags` booleans or signal `name` slugs, not on display labels from the dashboard (which can change).
- **Forgetting to make handlers idempotent:** Upsert on `request_id` so a redelivered webhook does not double-apply effects (e.g., double-charging, double-blocking).
- **Confusing score fields:** Webhook uses `risk_score`, History API uses `score`, Management API uses `Score`. Normalize when a decision path can be fed by either.
- **Treating 999 as a score:** The value 999 is a rate-limit ban marker, not a 0–100 score. Guard `score > 100` before reading the band.

## Verification checklist

Before submitting work:

- [ ] Snippet loads on the page (check Network tab for `cdn.shieldlabs.ai/snippet.js` with 200)
- [ ] `checkAnonymous()` or `checkAuthenticatedUser()` is called after the import resolves
- [ ] `requestID` is captured and forwarded to the backend
- [ ] Webhook endpoint is HTTPS with a valid certificate
- [ ] Webhook handler verifies `X-Shield-Signature` using constant-time comparison
- [ ] Webhook handler responds `200` within ~1 second
- [ ] Webhook handler is idempotent on `request_id` (upsert, not insert)
- [ ] Decision logic branches on `risk_score` band and `signals` (not score alone)
- [ ] Decision logic uses `detection_flags` for hard rules (Tor, anti-detect, abuser)
- [ ] CSP allows `script-src https://cdn.shieldlabs.ai` and `connect-src https://rest.shieldlabs.ai`
- [ ] Private API Key and Secret Key are backend-only environment variables
- [ ] Public Key is in the snippet URL (safe to expose)
- [ ] Webhook signing secret is backend-only environment variable
- [ ] History API fallback is implemented for high-stakes actions
- [ ] Thresholds are tuned gradually, starting in log-only mode
- [ ] Signals are logged for audit and later review

## Resources

- **Full documentation index:** https://docs.shieldlabs.ai/llms.txt (comprehensive page-by-page navigation for agents)
- **API Overview:** https://docs.shieldlabs.ai/api/overview (authentication, hosts, three surfaces)
- **Acting on the Risk Score:** https://docs.shieldlabs.ai/guides/acting-on-risk-score (decision playbook with per-scenario thresholds)
- **Webhook Setup:** https://docs.shieldlabs.ai/setup/webhooks (registration, verification, delivery guarantees)

---

> For additional documentation and navigation, see: https://docs.shieldlabs.ai/llms.txt