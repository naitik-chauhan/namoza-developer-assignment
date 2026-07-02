# Task 03 — Integration Design: OrthoNow Landing Page → HubSpot + WhatsApp + Google Ads

---

## Architecture Diagram

```
                          BROWSER (Client-Side)
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│   Form Submit                                                │
│     │                                                        │
│     ├──① window.dataLayer.push('consultation_form_submitted')│
│     │        │                                               │
│     │        ▼                                               │
│     │   ┌──────────┐    ┌────────────────┐                   │
│     │   │   GTM    │───▶│  GA4 Event Tag │                   │
│     │   │Container │    └────────────────┘                   │
│     │   │          │    ┌────────────────┐                   │
│     │   │          │───▶│  Google Ads    │                   │
│     │   │          │    │  Conversion Tag│                   │
│     │   └──────────┘    └────────────────┘                   │
│     │                                                        │
│     └──② fetch('POST /api/lead', {name, phone, clinic})     │
│              │                                               │
└──────────────┼───────────────────────────────────────────────┘
               │
               ▼  HTTPS
┌──────────────────────────────────────────────────────────────┐
│              BACKEND API (Cloud Function / Lambda)            │
│                                                              │
│   ┌─────────────────────────────────────────────────┐        │
│   │  POST /api/lead                                 │        │
│   │                                                 │        │
│   │  1. Validate + normalize phone → +91XXXXXXXXXX  │        │
│   │  2. Check idempotency key (prevent duplicates)  │        │
│   │  3. Search HubSpot by phone ──────────────┐     │        │
│   │     ├─ Found → PATCH (update contact)     │     │        │
│   │     └─ Not found → POST (create contact)  │     │        │
│   │  4. Send WhatsApp via Karix ──────────────┤     │        │
│   │     ├─ Success → log delivery             │     │        │
│   │     └─ Failure → retry (3x) → SMS fallback│     │        │
│   │  5. Return {success: true} to browser     │     │        │
│   └───────────────────────────────────────────┘     │        │
│                                                      │        │
│   ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │        │
│   │  Retry   │  │  Dead    │  │  Structured Logs │  │        │
│   │  Queue   │  │  Letter  │  │  (CloudWatch /   │  │        │
│   │  (SQS)   │  │  Queue   │  │   Datadog)       │  │        │
│   └──────────┘  └──────────┘  └──────────────────┘  │        │
└──────────────────────────────────────────────────────────────┘
               │                    │
               ▼                    ▼
    ┌──────────────────┐  ┌──────────────────┐
    │   HubSpot CRM    │  │   Karix          │
    │   Contacts API   │  │   WhatsApp       │
    │   (v3 REST)      │  │   Business API   │
    └──────────────────┘  └──────────────────┘
```

---

## Core Architecture (Written Answer — 380 words)

When a patient submits the consultation form, two independent paths fire simultaneously
from the browser — a **client-side path** for analytics and a **server-side path** for
CRM and messaging. This separation ensures that Google Ads conversion tracking
(which must fire from the browser for proper click attribution) is decoupled from
backend processing, so neither blocks the other.

**Client-side (immediate):** The `dataLayer.push()` already implemented in Task 2 triggers
GTM, which fires two tags — the GA4 event tag (`consultation_form_submitted`) and the
Google Ads Conversion Tracking tag. Both execute within the browser — Google Ads requires
the GCLID cookie from the user's click session, which is only accessible client-side.
No server involvement needed.

**Server-side (async):** The form simultaneously sends a `POST` request to a lightweight
backend API (Node.js Cloud Function on GCP, co-located with Indian traffic for low
latency). This endpoint orchestrates two sequential operations:

**Step 1 — HubSpot contact upsert via Contacts API (v3).** I use the Contacts API — not
the Forms API or a Zapier integration — for one critical reason: **phone-based
deduplication**. HubSpot's default dedup key is `email`, and our form doesn't collect
email (standard for Indian healthcare lead gen — phone is the primary identifier).
The Forms API has no mechanism to deduplicate on phone. Instead, my backend first
calls `POST /crm/v3/objects/contacts/search` with a filter on the `phone` property.
If a match exists, I `PATCH` the existing contact. If not, I `POST` a new one.
Contact properties set: `firstname`, `phone`, `clinic_preference` (custom property),
`hs_lead_status = "New Enquiry"`, and a custom `lead_source = "Google Ads - Consultation
Landing Page"`.

**Step 2 — WhatsApp confirmation via Karix.** After the HubSpot upsert succeeds, the
backend calls Karix's WhatsApp Business API to send a pre-approved template message:
*"Hi {name}, thank you for booking at OrthoNow {clinic}. Our specialist will call you
shortly."* This is a template message (not a session message), so it can be sent
without the patient initiating a conversation first.

**The single biggest failure point is the Karix WhatsApp delivery.** It depends on three
external systems in sequence: our API → Karix → WhatsApp servers → patient's device.
If Karix is down or WhatsApp delivery is delayed, the 2-minute SLA breaks. My
fallback: after 3 retries with exponential backoff (1s, 3s, 9s), the backend falls
back to sending an SMS via Karix's SMS API (same provider, different channel).
Failed messages are pushed to a dead-letter queue, and an ops alert fires in Slack.

To **monitor the SLA**, I log the timestamp at form receipt and the Karix delivery
callback timestamp. A CloudWatch alarm triggers if P95 end-to-end latency exceeds
90 seconds — giving a 30-second buffer before the 2-minute SLA breaches.

---

## API Request Flow (Step-by-Step)

### Step 0 — Client-Side (Browser)

```
Form Submit
  │
  ├──→ window.dataLayer.push({event: 'consultation_form_submitted', ...})
  │       └──→ GTM fires: GA4 Event Tag + Google Ads Conversion Tag
  │            (uses GCLID cookie for click attribution — must be client-side)
  │
  └──→ fetch('https://api.orthonow.in/api/lead', {
          method: 'POST',
          headers: {'Content-Type': 'application/json'},
          body: JSON.stringify({
            name: 'Naitik Kumar',
            phone: '7895407387',
            clinic: 'Koramangala',
            idempotency_key: 'uuid-v4-generated-on-client'
          })
        })
```

### Step 1 — Backend Receives Request

```
POST /api/lead
  │
  ├── Validate inputs (name ≥ 2 chars, phone matches /^[6-9]\d{9}$/, clinic in whitelist)
  ├── Normalize phone → "+917895407387" (E.164 format)
  ├── Check idempotency_key against Redis/DynamoDB cache
  │     ├── Key exists → return cached response (prevents duplicate processing)
  │     └── Key new → proceed, store key with 24h TTL
  │
  └── Continue to Step 2
```

### Step 2 — HubSpot Contact Upsert

```
SEARCH existing contact by phone:
  POST https://api.hubapi.com/crm/v3/objects/contacts/search
  Authorization: Bearer {HUBSPOT_ACCESS_TOKEN}
  Body: {
    "filterGroups": [{
      "filters": [{
        "propertyName": "phone",
        "operator": "EQ",
        "value": "+917895407387"
      }]
    }],
    "properties": ["firstname", "phone", "hs_lead_status"]
  }

  IF results.total > 0 → UPDATE (existing contact found):
    PATCH https://api.hubapi.com/crm/v3/objects/contacts/{contactId}
    Body: {
      "properties": {
        "firstname": "Naitik",
        "phone": "+917895407387",
        "clinic_preference": "Koramangala",
        "hs_lead_status": "New Enquiry",
        "lead_source": "Google Ads - Consultation Landing Page"
      }
    }

  IF results.total == 0 → CREATE (new contact):
    POST https://api.hubapi.com/crm/v3/objects/contacts
    Body: {
      "properties": {
        "firstname": "Naitik",
        "lastname": "Kumar",
        "phone": "+917895407387",
        "clinic_preference": "Koramangala",
        "hs_lead_status": "New Enquiry",
        "lead_source": "Google Ads - Consultation Landing Page"
      }
    }
```

### Step 3 — WhatsApp via Karix

```
POST https://api.karix.io/message/
Authorization: Bearer {KARIX_API_KEY}
Content-Type: application/json
Body: {
  "channel": "whatsapp",
  "source": "+91ORTHONOW_BUSINESS_NUMBER",
  "destination": ["+917895407387"],
  "content": {
    "type": "template",
    "template": {
      "name": "consultation_booking_confirmation",
      "language": {"code": "en"},
      "body_params": ["Naitik", "Koramangala"]
    }
  },
  "callback_url": "https://api.orthonow.in/webhooks/karix"
}

RESPONSE: 202 Accepted → message queued for delivery
  └── Karix sends delivery status to callback_url
      {"status": "delivered", "timestamp": "2026-07-01T10:02:15Z"}
```

### Step 4 — Return Response to Browser

```
HTTP 200 OK
{
  "success": true,
  "message": "Consultation request received",
  "hubspot_contact_id": "12345678",
  "whatsapp_status": "queued"
}
```

---

## Technical Deep Dive

### Why HubSpot Contacts API — Not Forms API, Not Zapier

| Approach | Why Rejected |
|----------|-------------|
| **HubSpot Forms API** | Creates contacts asynchronously — no immediate response confirming creation. No phone-based deduplication. Cannot search-before-create. No control over upsert logic. |
| **HubSpot Native Form Embed** | Requires HubSpot JavaScript on the landing page (adds ~200KB). Creates dependency on HubSpot CDN. No dedup control. Slower page load, hurts PageSpeed score. |
| **Zapier / Make** | Each step adds 1–5 seconds of latency (webhook receive → process → API call). Free tiers have execution limits (100 tasks/month on Zapier). No custom phone dedup logic. Cannot guarantee 2-minute WhatsApp SLA under load. Adds a third-party dependency for a critical path. |
| **Direct API from Browser** | Exposes HubSpot API key in client-side JavaScript. CORS issues. No input sanitization on the server. Anyone could inspect the key and abuse the API. |

**HubSpot Contacts API (v3) via backend** gives us:
- Full control over phone-based search-then-upsert logic
- Synchronous response confirming contact creation
- Server-side API key storage (never exposed to browser)
- Custom error handling, retry, and logging

---

### Phone Number Deduplication — The Critical Detail

> **The trap**: HubSpot's default deduplication is on **email**, not phone. Since our form
> does not collect email (standard in Indian healthcare lead gen where phone is the primary
> communication channel), creating contacts directly would produce duplicates every time a
> returning patient submits the form.

**Solution — Search-then-upsert by phone:**

```
1. Normalize phone to E.164 format: "7895407387" → "+917895407387"
   (Strip spaces, hyphens, leading zeros, ensure +91 prefix)

2. Search HubSpot: POST /crm/v3/objects/contacts/search
   Filter: phone = "+917895407387"

3. Decision:
   ├── Contact exists → PATCH (update name, clinic_preference, lead_status)
   └── No match → POST (create new contact)
```

**Edge case — same phone, different names:**

If Naitik Kumar submits on Monday and "Naitik C" submits on Wednesday with the same
phone number, the search finds the existing contact and **updates** it. The latest name
overwrites the previous one. This is intentional — in healthcare, the same person may
enter their name differently across visits. The phone number is the source of truth.

If the business requires preserving name history, we log the previous value before
overwriting:

```json
{
  "level": "info",
  "action": "hubspot_contact_updated",
  "contact_id": "12345678",
  "phone": "+917895XXXXX",
  "previous_name": "Naitik Kumar",
  "new_name": "Naitik C",
  "reason": "phone_dedup_match"
}
```

**Why not set phone as a unique property in HubSpot?**

HubSpot allows marking custom properties as "unique" (available on Professional+ plans),
which would enable the `POST /crm/v3/objects/contacts` endpoint to deduplicate on phone
natively. However, this has two risks:
1. Phone format inconsistency (with/without +91) causes false negatives
2. It's a HubSpot account-level setting — other teams using the same HubSpot instance
   may break if they don't normalize phone numbers the same way

The search-then-upsert pattern gives us **explicit control** without depending on
HubSpot's property configuration.

---

### Error Handling & Retry Strategy

```
┌─────────────────────────────────────────────────────────┐
│                   ERROR HANDLING MATRIX                  │
├──────────────┬──────────────┬──────────────┬────────────┤
│   Service    │   Error      │   Action     │  Fallback  │
├──────────────┼──────────────┼──────────────┼────────────┤
│ HubSpot API  │ 429 (rate    │ Retry after  │ Queue in   │
│              │  limit)      │ Retry-After  │ DLQ, alert │
│              │              │ header       │ ops        │
├──────────────┼──────────────┼──────────────┼────────────┤
│ HubSpot API  │ 500/503      │ Retry 3x     │ Queue lead │
│              │ (server err) │ exp backoff  │ for manual │
│              │              │ (1s,3s,9s)   │ entry      │
├──────────────┼──────────────┼──────────────┼────────────┤
│ HubSpot API  │ 401 (auth)   │ No retry     │ Alert ops  │
│              │              │ (won't fix   │ immediately│
│              │              │  itself)     │            │
├──────────────┼──────────────┼──────────────┼────────────┤
│ Karix WA     │ 5xx / timeout│ Retry 3x     │ Fallback   │
│              │              │ exp backoff  │ to SMS     │
├──────────────┼──────────────┼──────────────┼────────────┤
│ Karix WA     │ Template     │ No retry     │ Alert ops, │
│              │ rejected     │ (config err) │ send SMS   │
├──────────────┼──────────────┼──────────────┼────────────┤
│ Karix WA     │ No delivery  │ Monitor via  │ SMS after  │
│              │ callback in  │ webhook,     │ 90s, alert │
│              │ 90 seconds   │ timeout check│ dashboard  │
├──────────────┼──────────────┼──────────────┼────────────┤
│ Our API      │ Validation   │ Return 400   │ Show error │
│              │ failure      │ to browser   │ in form UI │
└──────────────┴──────────────┴──────────────┴────────────┘
```

**Retry strategy: Exponential backoff with jitter**

```
Attempt 1: wait 1s + random(0-500ms)
Attempt 2: wait 3s + random(0-500ms)
Attempt 3: wait 9s + random(0-500ms)
Attempt 4: give up → dead-letter queue + alert
```

The jitter prevents thundering herd problems if multiple requests fail simultaneously
(e.g., during a HubSpot outage when queued retries all fire at the same second).

**Critical design rule: HubSpot failure must NOT block WhatsApp delivery.**

The backend processes HubSpot and Karix as independent operations. If HubSpot is down,
the lead is queued for later processing, but the WhatsApp confirmation still sends. The
patient's experience is not degraded by a CRM outage.

---

### Logging

All API interactions produce structured JSON logs (not plain text) for machine parsing:

```json
{
  "timestamp": "2026-07-01T10:01:45.123Z",
  "level": "info",
  "service": "lead-api",
  "action": "hubspot_contact_created",
  "idempotency_key": "a1b2c3d4-e5f6-7890",
  "hubspot_contact_id": "12345678",
  "clinic": "Koramangala",
  "phone_masked": "+91789540XXXX",
  "latency_ms": 342,
  "http_status": 201
}
```

**PII handling**: Phone numbers are masked in logs (`+91789540XXXX`). Full phone exists
only in HubSpot (the CRM, which is the system of record for PII). Log system (CloudWatch /
Datadog) never stores identifiable patient data — this is important for India's DPDP Act
compliance.

---

### Monitoring & Alerting

| Metric | Threshold | Alert Channel |
|--------|-----------|---------------|
| WhatsApp delivery latency (P95) | > 90 seconds | Slack #ops-critical |
| HubSpot API error rate | > 5% in 5 minutes | Slack + PagerDuty |
| Karix API error rate | > 5% in 5 minutes | Slack + PagerDuty |
| Dead-letter queue depth | > 0 items | Slack #ops-alerts |
| API endpoint latency (P99) | > 5 seconds | CloudWatch alarm |
| Daily lead volume vs. ad spend | < expected ratio | Daily Slack digest |

**WhatsApp SLA monitoring (2-minute requirement):**

```
Timeline tracked:
  T0 = form_received_at (our API receives POST)
  T1 = whatsapp_queued_at (Karix returns 202)
  T2 = whatsapp_delivered_at (Karix webhook callback)

  SLA metric = T2 - T0

  If T2 - T0 > 120 seconds → SLA breach logged
  If T1 not received within 10 seconds → Karix is down, trigger SMS fallback
  If T2 not received within 90 seconds → delivery stalled, trigger SMS + alert
```

---

### Security Considerations

| Concern | Mitigation |
|---------|-----------|
| API key exposure | All keys (HubSpot, Karix) stored in GCP Secret Manager / AWS Secrets Manager. Never in code, environment variables in CI/CD, or client-side JS. |
| Endpoint abuse | Rate limiting: 10 requests/minute/IP via API gateway (Cloudflare / AWS API Gateway). Honeypot hidden field in form to catch bots. |
| Input injection | Server-side validation: name (string, 2–100 chars, strip HTML), phone (exactly 10 digits, 6-9 start), clinic (enum whitelist — reject anything not in the 9 valid values). |
| CORS | Allow-Origin restricted to `orthonow.in` domain only. No wildcard. |
| Data in transit | HTTPS/TLS 1.3 on all endpoints — API, HubSpot, Karix. No HTTP fallback. |
| PII compliance (DPDP Act) | Phone numbers masked in logs. Full PII only in HubSpot (system of record). Data retention policy: logs expire after 90 days. |

---

### Rate Limiting

| Service | Limit | Our Expected Volume | Risk Level |
|---------|-------|-------------------|------------|
| HubSpot API (Standard) | 100 requests / 10 sec | ~2 requests per lead (search + create/update) | **Low** — OrthoNow at peak gets ~50 leads/hour = 100 API calls/hour. Nowhere near 100/10s. |
| Karix WhatsApp API | Varies by plan (typically 30–60 msg/sec) | 1 message per lead | **Low** |
| Our API endpoint | 10 requests/min/IP (configured) | 1 request per form submit | **Low** — rate limit exists for abuse prevention, not capacity. |

**If HubSpot returns 429 (rate limited):** Read the `Retry-After` header, wait that
duration, retry once. If still 429, queue the lead for processing in 60 seconds.

---

### Idempotency

**Problem:** User double-clicks "Submit". Browser retries on network timeout. Result:
duplicate contacts, duplicate WhatsApp messages.

**Solution:** Client generates a UUID v4 `idempotency_key` before the fetch call:

```javascript
// On the frontend (before fetch):
var idempotencyKey = crypto.randomUUID();  // e.g., "a1b2c3d4-e5f6-7890-..."
```

Backend flow:
```
1. Receive POST /api/lead with idempotency_key
2. Check Redis: EXISTS idempotency:{key}
   ├── Key exists → return cached response (200 OK, same body as original)
   └── Key not found → process request, SET idempotency:{key} with 24h TTL
3. Store response in Redis alongside the key for cache-hit returns
```

This guarantees that no matter how many times the same submission is sent, the lead
is created once, the WhatsApp is sent once, and the response is identical.

---

### Production Deployment Considerations

| Aspect | Decision | Rationale |
|--------|----------|-----------|
| **Compute** | GCP Cloud Function (Mumbai region, `asia-south1`) | Co-located with Indian users for lowest latency. Serverless = auto-scaling, no server maintenance. Cold start ~200ms (Node.js). |
| **Secrets** | GCP Secret Manager | API keys for HubSpot and Karix. Rotated quarterly. Accessed via IAM role, not env vars. |
| **Retry Queue** | Cloud Tasks (GCP) or SQS (AWS) | Persistent queue for failed HubSpot/Karix calls. Survives function restarts. |
| **Dead-Letter Queue** | Separate queue with alerting | Messages that fail all retries land here. Ops team manually processes within 1 business day. |
| **Idempotency Store** | Redis (Memorystore) or Firestore | Low-latency key lookups. 24h TTL auto-expiry. |
| **CI/CD** | GitHub Actions → Cloud Functions deploy | Automated deployment on merge to `main`. Staging environment for pre-prod testing. |
| **Environments** | Staging + Production | Staging uses HubSpot sandbox account + Karix test credentials. No real patients contacted in staging. |
| **Health Check** | Synthetic monitor (every 5 min) | Sends a test request to `/api/health`. Alerts if endpoint is unreachable. |

---

## Biggest Failure Point — Detailed Analysis

**The Karix WhatsApp delivery is the single biggest failure point.**

Why:

1. **Three external dependencies in series**: Our API → Karix servers → WhatsApp
   servers → patient's phone. Any one failing breaks the chain.
2. **WhatsApp template approval**: If the message template is modified and re-submitted,
   WhatsApp's review process takes 24–48 hours. During review, messages fail silently.
3. **Karix uptime is outside our control**: Even with an SLA from Karix, their outages
   directly impact our 2-minute commitment to OrthoNow.
4. **Mobile network variability**: Even if Karix delivers to WhatsApp servers instantly,
   the patient's phone may be offline, on poor network, or have WhatsApp notifications
   disabled.

**Fallback architecture:**

```
WhatsApp attempt (Karix)
  │
  ├── Success (delivery callback within 90s) → done
  │
  └── Failure (timeout or error after 3 retries)
        │
        ├── Fallback to SMS (Karix SMS API, same provider)
        │     └── SMS is more reliable — works on feature phones,
        │         doesn't require internet, simpler delivery path
        │
        ├── Log to dead-letter queue for manual follow-up
        │
        └── Alert #ops-critical in Slack with patient details
              (masked phone, clinic, timestamp)
```

---

## Interview Preparation: 10 Difficult Questions & Answers

### Q1: "If two patients submit with the same phone number but different names, what happens in your setup?"

**Answer:** My backend searches HubSpot by phone before creating a contact. If the phone
already exists, it updates the existing contact — the latest name overwrites the
previous one. This is by design: in Indian healthcare, the same patient may enter
"Naitik Kumar" on one visit and "Naitik C" on another. The phone number is the
canonical identifier, not the name.

I log the previous name before overwriting so the ops team has an audit trail. If the
business needs stricter dedup (e.g., distinguishing family members sharing a phone),
we'd need to collect a second identifier — likely DOB or email — which is a product
decision, not a technical one.

---

### Q2: "Why not use Zapier or Make? It would be faster to set up."

**Answer:** Three reasons:

1. **Latency**: Each Zapier step adds 1–5 seconds. The form submit → webhook → HubSpot
   → Karix chain through Zapier would take 10–20 seconds on average. With a custom
   backend, the full chain completes in under 2 seconds.

2. **No custom dedup logic**: Zapier's HubSpot integration creates contacts using the
   Forms API or basic create — it doesn't support the search-then-upsert pattern I
   need for phone-based deduplication.

3. **SLA risk**: Zapier's webhook processing has no guaranteed latency. During high-load
   periods, tasks queue. I can't guarantee the 2-minute WhatsApp SLA through a
   third-party automation tool.

Zapier is excellent for prototyping and low-volume flows. For a production system
with SLA requirements and custom dedup logic, a lightweight backend is more reliable.

---

### Q3: "Why HubSpot Contacts API and not the Forms API?"

**Answer:** The Forms API creates contacts asynchronously — there's no immediate
response confirming whether the contact was created or deduplicated. More critically,
the Forms API uses HubSpot's native dedup which keys on email. Since our form doesn't
collect email, every submission would create a new contact, producing duplicates for
returning patients.

The Contacts API (v3) gives me a synchronous response, and the Search endpoint lets me
implement phone-based dedup explicitly — search first, then create or update.

---

### Q4: "What happens if HubSpot is down when someone submits the form?"

**Answer:** The backend treats HubSpot and Karix as independent operations. If HubSpot
returns a 5xx or times out after 3 retries, the lead payload is written to a persistent
retry queue (Cloud Tasks / SQS) with a 60-second delay. The WhatsApp message still
sends — the patient's experience is not degraded by a CRM outage.

The queued lead will retry automatically. If it fails again after 3 queue attempts, it
moves to a dead-letter queue and triggers a Slack alert. The ops team manually enters
the lead into HubSpot — this has happened zero times in systems I've built with this
pattern, but the fallback exists.

The frontend always receives a success response (200 OK) because the lead has been
accepted by our API — the HubSpot write is an internal concern, not the patient's.

---

### Q5: "How do you handle phone number format variations?"

**Answer:** All phone numbers are normalised to E.164 format (`+917895407387`) at the
API boundary — the first thing the backend does. The normalisation strips spaces,
hyphens, parentheses, leading zeros, and ensures the +91 prefix.

This normalised format is used for both HubSpot search and HubSpot create/update. If
the phone is stored as "+917895407387" in HubSpot, and a new submission comes in as
"07895407387" or "+91-78954-07387", the normalisation ensures the search matches
correctly.

Without normalisation, the same patient could exist three times in HubSpot with
different phone formats — which defeats the entire dedup strategy.

---

### Q6: "The WhatsApp message must fire within 2 minutes. What could break this?"

**Answer:** Five things:

1. **Karix API outage** — their servers are unreachable. Mitigation: 3 retries with
   exponential backoff + SMS fallback.
2. **WhatsApp template suspended** — Meta/WhatsApp flags or pauses the template for
   policy review. Mitigation: monitor template status via Karix dashboard, maintain
   a backup template.
3. **Cold start latency** — if using serverless, the first invocation after idle may take
   500ms–2s. Mitigation: provisioned concurrency or a keep-alive ping every 5 minutes.
4. **HubSpot latency cascading** — if HubSpot is slow (but not down), the total chain
   takes longer. Mitigation: fire WhatsApp in parallel with HubSpot, not sequentially.
5. **Patient's phone offline** — WhatsApp can't deliver. This is outside our control and
   technically the message is "sent" (Karix accepted it), but "delivered" only happens
   when the patient's phone connects. Our SLA should be measured against Karix
   acceptance (sent), not end-device delivery.

---

### Q7: "How would you monitor this in production?"

**Answer:** Three layers:

1. **Per-request tracing**: Every API call (HubSpot, Karix) is logged with latency, HTTP
   status, and a correlation ID that ties the entire lead lifecycle together. Structured
   JSON logs, shipped to CloudWatch or Datadog.

2. **Dashboard metrics**: P95/P99 latency for each API call, success/failure rates,
   dead-letter queue depth, daily lead volume. I'd use Grafana or CloudWatch dashboards.

3. **Alerting**: CloudWatch alarm if WhatsApp P95 delivery time > 90 seconds (30-second
   buffer before 2-minute SLA). PagerDuty escalation if error rate > 5% for any service.
   Daily digest in Slack with lead counts vs. ad spend for the marketing team.

---

### Q8: "What if someone spams the form with 10,000 submissions?"

**Answer:** Four layers of protection:

1. **Client-side**: Disable submit button after first click. Generate idempotency key
   per page load (same key = same request even if button clicked 10 times).

2. **API Gateway**: Rate limit at 10 requests/minute/IP via Cloudflare or AWS API
   Gateway. Returns 429 after threshold.

3. **Honeypot field**: Hidden form field (CSS `display:none`). Bots fill it, humans
   don't. Backend rejects submissions where the honeypot is filled.

4. **Idempotency**: Even if 10,000 requests come through, each unique phone number
   produces only one HubSpot contact (search-before-create) and one WhatsApp message
   (idempotency key prevents re-processing).

If spam is sustained, we add reCAPTCHA v3 (invisible, no UX impact) or Cloudflare
Turnstile as a step-up.

---

### Q9: "If you had to add email collection later, how would your architecture change?"

**Answer:** Minimal change:

1. **Form**: Add an email field.
2. **Backend**: Include `email` in the HubSpot create/update payload.
3. **Dedup**: Switch from phone-search to email-based (HubSpot's native dedup handles
   this automatically). Keep phone-search as a secondary check for contacts that were
   created before email was collected.
4. **Google Ads**: Enable Enhanced Conversions — send hashed email and phone to Google
   Ads via the Measurement Protocol or GTM's Enhanced Conversions tag. This improves
   conversion attribution by 5–15% (Google's published benchmarks).

The architecture doesn't change structurally — it's the same API, same flow, with one
more field.

---

### Q10: "Walk me through how you'd test this end-to-end before going live."

**Answer:** Three stages:

1. **Staging environment**: Backend deploys to a staging Cloud Function. Uses HubSpot's
   developer sandbox (free with any HubSpot account) and Karix's test/sandbox
   credentials. Submit test leads with known phone numbers. Verify:
   - Contact appears in HubSpot sandbox with correct properties
   - WhatsApp template renders correctly (Karix test mode shows template preview)
   - Duplicate phone submission updates (not duplicates) the contact
   - Idempotency key prevents re-processing on retry

2. **Integration test script**: Automated script that sends 5 test submissions with
   edge cases: duplicate phones, special characters in names, all 9 clinics. Asserts
   HubSpot contact count and properties via API.

3. **Production smoke test**: After deployment, submit one real lead with a team
   member's phone number. Verify: HubSpot contact created, WhatsApp received, Google
   Ads conversion visible in Google Ads console (takes 1–3 hours to appear). Then
   delete the test contact.

---

*Document prepared by Naitik Kumar for Namoza Developer Assignment — Task 03*
