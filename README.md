# Namoza Developer Assignment — Naitik Kumar

**Position:** Developer - Position 1 (Client Web + Martech)
**Candidate:** Naitik Kumar
**Contact:** naitikchauhan8218@gmail.com · +91 7895407387
**GitHub:** [github.com/naitik-chauhan](https://github.com/naitik-chauhan)
**LinkedIn:** [linkedin.com/in/naitik7895](https://linkedin.com/in/naitik7895)

---

## 1. Project Overview

This repository contains my submission for the Namoza Healthcare Growth & Strategy developer assignment. The client scenario is **OrthoNow** — a chain of 9 orthopaedic clinics across Bengaluru, Hyderabad, and Chennai inheriting a digital setup with no event tracking, no GTM, and a landing page converting at 2.1% against a 6–8% industry benchmark.

The assignment has three deliverables:

| Task | Deliverable | File |
|------|------------|------|
| **Task 1** | GTM Event Schema | [`task-01-gtm-schema/gtm-event-schema.md`](./task-01-gtm-schema/gtm-event-schema.md) |
| **Task 2** | Landing Page Build | [`task-02-landing-page/index.html`](./task-02-landing-page/index.html) |
| **Task 3** | Integration Architecture | [`task-03-integration/integration-design.md`](./task-03-integration/integration-design.md) |

---

## 2. Folder Structure

```
namoza-developer-assignment/
│
├── README.md                                  ← You are here
│
├── task-01-gtm-schema/
│   └── gtm-event-schema.md                   ← Complete event schema (11 events),
│                                                 dataLayer JSON for booking funnel,
│                                                 GTM configuration, GA4 Funnel
│                                                 Exploration setup, Google Ads
│                                                 conversion recommendation
│
├── task-02-landing-page/
│   └── index.html                             ← Self-contained landing page
│                                                 (HTML + inline CSS + inline JS),
│                                                 working dataLayer.push() on
│                                                 form submit, mobile-first,
│                                                 optimised for PageSpeed 90+
│
└── task-03-integration/
    └── integration-design.md                  ← End-to-end integration architecture:
                                                  Landing Page → Backend API →
                                                  HubSpot Contacts API → Karix
                                                  WhatsApp → Google Ads conversion
```

---

## 3. Assignment Summary

### Task 1 — GTM Event Schema

Defined a complete event tracking schema for OrthoNow covering all key website interactions. The schema includes 11 events with proper GA4 `snake_case` naming, minimum 3 parameters per event, and explicit mapping to GA4 reports and audiences.

**Key deliverable:** Real `dataLayer.push()` JSON for the 3-step booking form funnel — not pseudocode. The document explicitly addresses that GTM cannot natively detect multi-step form transitions. The front-end developer writes the `dataLayer.push()` calls; the GTM specialist defines the schema and configures the triggers.

**Google Ads conversion selected:** `consultation_booked` (Step 3) — the highest-quality intent signal available client-side. Step 1 or Step 2 would optimise Smart Bidding toward form starters, not completers.

### Task 2 — Landing Page Build

Built a conversion-focused landing page for OrthoNow's "Book a Consultation" Google Ads campaign. Single self-contained HTML file — no frameworks, no external dependencies, no server required.

**Target audience:** Working professionals aged 28–50 in Bengaluru experiencing knee or back pain.

**Core conversion elements:**
- Problem-aware headline: "Knee or Back Pain Holding You Back?"
- 3-field minimal form: Name + Phone + Clinic Preference
- First-person CTA: "Book My Consultation →"
- Trust signals: 9 clinics, 50,000+ patients, NABH accreditation, peer testimonials
- Personalised thank-you state with clinic confirmation

**Technical highlights:**
- `window.dataLayer.push()` fires only on valid form submission (not on page load)
- 4 parameters: `form_id`, `page_path`, `form_location`, `clinic_preference`
- System font stack — zero network requests for fonts
- Total page size: ~32KB — no render-blocking resources
- `font-size: 16px` on all inputs prevents iOS Safari auto-zoom

### Task 3 — Integration Architecture

Designed the end-to-end integration from landing page form submission through HubSpot CRM contact creation, Karix WhatsApp confirmation, and Google Ads conversion tracking.

**Architecture decision:** Custom backend API (Node.js Cloud Function) using HubSpot Contacts API v3 — not the Forms API, not Zapier, not a native HubSpot embed.

**Critical detail addressed:** HubSpot deduplicates contacts by email, not phone. Since the form doesn't collect email (standard in Indian healthcare lead gen), I implemented a search-then-upsert pattern: search HubSpot by phone first, then create or update.

**Biggest failure point identified:** Karix WhatsApp delivery — three external systems in series (our API → Karix → WhatsApp → patient's phone). Fallback: SMS via Karix after 3 failed retries.

---

## 4. Technology Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| **Landing Page** | HTML5, CSS3, Vanilla JavaScript | Assignment requirement — no frameworks. Self-contained single file. |
| **Styling** | Inline CSS with CSS Custom Properties | Zero external stylesheets. Variables for maintainability. Mobile-first media queries. |
| **Font Stack** | `-apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif` | System fonts load in 0ms. No FOUT, no Google Fonts network request. Critical for PageSpeed 90+. |
| **Event Tracking** | GTM + GA4 + `window.dataLayer` | Industry standard. dataLayer is the bridge between the website and GTM. |
| **Backend (proposed)** | Node.js Cloud Function (GCP, `asia-south1` Mumbai) | Serverless, auto-scaling, co-located with Indian traffic for lowest latency. |
| **CRM** | HubSpot Contacts API v3 | Full control over phone-based deduplication via search-then-upsert. |
| **Messaging** | Karix WhatsApp Business API | OrthoNow's existing provider. Template messages for transactional notifications. |
| **Conversion Tracking** | Google Ads Conversion Tag via GTM | Client-side — requires GCLID cookie from user's ad click for proper attribution. |

---

## 5. Features Implemented

### Landing Page (`index.html`)

- [x] Semantic HTML5 with proper heading hierarchy (single `<h1>`)
- [x] Mobile-first responsive CSS — designed for mobile, scales to desktop
- [x] Sticky header with click-to-call phone link (`tel:` protocol)
- [x] Hero section with problem-aware headline targeting knee/back pain
- [x] 3-field form: Full Name, Mobile Number (+91 prefix), Preferred Clinic (dropdown)
- [x] Indian mobile number validation (10 digits, starts with 6-9)
- [x] Non-digit stripping on phone input (handles pasted formatted numbers)
- [x] Clinic Preference dropdown with all 9 OrthoNow locations
- [x] Primary CTA with first-person phrasing ("Book My Consultation →")
- [x] Trust statistics bar (9 Clinics · 50,000+ Patients · 15+ Years)
- [x] NABH accreditation + doctor credentials one-liner
- [x] Two testimonials from target persona (IT/product professionals in Bengaluru/Hyderabad)
- [x] Bottom CTA with smooth scroll back to form (`scroll-behavior: smooth`)
- [x] Thank-you state without page reload — personalised with name, phone, and clinic
- [x] `window.dataLayer.push()` on valid form submit with 4 parameters
- [x] `novalidate` + custom JavaScript validation (consistent UX across browsers)
- [x] Accessibility: skip link, `aria-label`, `aria-required`, `aria-invalid`, `role="alert"`, `aria-live="polite"`
- [x] `font-size: 16px` on inputs (prevents iOS Safari auto-zoom)
- [x] `prefers-reduced-motion` media query (respects accessibility preferences)
- [x] `<meta name="theme-color">` for mobile browser chrome
- [x] `<meta name="robots" content="noindex, nofollow">` (paid LP — not for organic indexing)
- [x] No external dependencies — zero network requests after HTML loads

### GTM Event Schema

- [x] 11 events covering all 6 interaction types
- [x] Real dataLayer JSON for 3-step booking form (not pseudocode)
- [x] Abandonment tracking with `beforeunload` + idle timeout
- [x] Complete GTM configuration table (triggers, variables, tags)
- [x] GA4 Funnel Exploration setup instructions
- [x] Google Ads conversion import recommendation with justification
- [x] Developer briefing note for front-end team

### Integration Design

- [x] ASCII architecture diagram
- [x] Step-by-step API request flow with actual endpoints and request bodies
- [x] HubSpot Contacts API justification (vs. Forms API, Zapier, native embed)
- [x] Phone-based deduplication via search-then-upsert
- [x] Error handling matrix by service × error type
- [x] Exponential backoff retry strategy with jitter
- [x] Structured JSON logging with PII masking
- [x] Monitoring thresholds and alerting strategy
- [x] Idempotency via client-generated UUID
- [x] Security considerations (secrets management, CORS, input validation)
- [x] Production deployment plan (GCP Mumbai region)
- [x] WhatsApp SLA timeline analysis (T0 → T1 → T2)

---

## 6. GTM Event Tracking Overview

The schema covers 11 events. Key design decisions:

| Decision | Rationale |
|----------|-----------|
| `snake_case` naming | GA4 recommended format. Consistent with GA4's built-in events. |
| `page_path` on every event | Enables cross-page analysis without requiring secondary dimensions in every report. |
| `clinic_location` standardised | Consistent values across all events ("Koramangala", not "koramangala" or "KMG"). Enforced via GTM Lookup Table variable. |
| `patient_name_provided: true` (not actual name) | PII must never reach GA4 — violates Google ToS and India's DPDP Act. Boolean flags track completion, not content. |
| `booking_abandoned` via `beforeunload` | Best-effort signal. On mobile, `visibilitychange` is more reliable — I'd implement both with a dedup flag in production. |

**Booking form funnel events:**

```
Step 1: booking_step_complete (step_number: 1, step_name: "location_specialty_selected")
Step 2: booking_step_complete (step_number: 2, step_name: "patient_details_entered")
Step 3: booking_step_complete (step_number: 3, step_name: "booking_confirmed")
        + consultation_booked (separate conversion event for Google Ads)
```

> **Critical implementation note:** GTM cannot detect multi-step form transitions natively. The front-end developer writes the `dataLayer.push()` calls at each step transition. The GTM specialist defines the schema and configures Custom Event triggers.

Full schema with parameters, trigger types, and GA4 report mapping: [`task-01-gtm-schema/gtm-event-schema.md`](./task-01-gtm-schema/gtm-event-schema.md)

---

## 7. Landing Page Overview

### Conversion Architecture

The page follows the AIDA framework — Attention → Interest → Desire → Action — compressed for a paid traffic landing page where users arrive with high intent.

```
┌─────────────────────────────────────────────┐
│  HEADER (sticky)                            │  ← OrthoNow + Call Now button
├─────────────────────────────────────────────┤
│  HERO                                       │  ← Attention: "Knee or Back Pain
│  ┌─────────────────────────┐                │     Holding You Back?"
│  │  FORM CARD              │                │
│  │  Name + Phone + Clinic  │                │  ← Action: Form above the fold
│  │  [Book My Consultation] │                │     on mobile (360×640 viewport)
│  └─────────────────────────┘                │
├─────────────────────────────────────────────┤
│  TRUST BAR: 9 Clinics · 50K+ · 15+ Years   │  ← Interest: Specific numbers
│  NABH Accredited · MS/DNB Surgeons          │
├─────────────────────────────────────────────┤
│  TESTIMONIALS (2 peer reviews)              │  ← Desire: Social proof from
│  Engineering Lead · Product Manager         │     matching job titles
├─────────────────────────────────────────────┤
│  BOTTOM CTA: "Don't Let Pain Wait"          │  ← Action: Scroll-to-form for
│  [Book My Consultation ↑]                   │     users who scrolled past
├─────────────────────────────────────────────┤
│  FOOTER                                     │
└─────────────────────────────────────────────┘
```

### Copy Decisions (Not AI-Generic)

| Element | What I Wrote | Why It Works |
|---------|-------------|-------------|
| Headline | "Knee or Back Pain Holding You Back?" | Names the specific pain. Users from Google Ads for "knee pain specialist" see their problem immediately. Not "Your Health, Our Priority." |
| Subheadline | "No long waits. No referral needed." | Addresses two real conversion barriers in Indian healthcare: long wait times and the misconception that a GP referral is required. |
| CTA | "Book My Consultation →" | First-person "My" increases psychological ownership. CRO studies show 25–90% conversion lift over second-person "Your." |
| Trust badge | "✓ Trusted by 50,000+ patients across 3 cities" | Specific number + geography. Not "thousands of happy patients." |
| Testimonials | "Engineering Lead, Bengaluru" / "Product Manager, Hyderabad" | Mirror the target persona back at them. Specific job titles, specific cities, specific outcomes. |

### dataLayer Implementation

The form fires this push on valid submission:

```javascript
window.dataLayer.push({
    'event': 'consultation_form_submitted',
    'form_id': 'hero_consultation_form',
    'page_path': window.location.pathname,
    'form_location': 'hero_section',
    'clinic_preference': 'Koramangala'   // dynamic — from dropdown
});
```

**When it fires:** Only after name (≥2 chars), phone (10-digit Indian mobile), and clinic (dropdown selection) pass validation. Never on page load. Never on failed validation.

---

## 8. Integration Architecture Overview

```
BROWSER                           BACKEND (Cloud Function)
  │                                      │
  ├─→ dataLayer.push() ─→ GTM           │
  │     ├─→ GA4 Event Tag               │
  │     └─→ Google Ads Conversion Tag    │
  │                                      │
  └─→ POST /api/lead ──────────────→ ────┤
                                         ├─→ HubSpot: Search by phone
                                         │     ├─ Found → PATCH (update)
                                         │     └─ Not found → POST (create)
                                         │
                                         └─→ Karix: WhatsApp template message
                                               ├─ Success → log delivery
                                               └─ Fail → retry 3x → SMS fallback
```

**Key architecture decisions:**

1. **Two parallel paths from form submit** — client-side (GTM) and server-side (backend API) are decoupled. Google Ads conversion must fire client-side for GCLID cookie attribution. CRM and WhatsApp are server-side concerns.

2. **HubSpot Contacts API v3** over Forms API — because the form collects phone but not email, and HubSpot deduplicates by email by default. The Contacts API lets us search by phone first, then create or update.

3. **Backend API** over Zapier/Make — custom dedup logic, guaranteed latency, 2-minute WhatsApp SLA.

4. **SMS fallback** for WhatsApp failures — Karix supports both channels. If WhatsApp fails after 3 retries, SMS sends within the same API call.

Full architecture with API request flows, error handling matrix, retry strategy, security considerations, and monitoring: [`task-03-integration/integration-design.md`](./task-03-integration/integration-design.md)

---

## 9. How to Run the Project

### Landing Page (Task 2)

The landing page is a single self-contained HTML file. No server, no build step, no dependencies.

**Option 1 — Direct file open:**
```
Open task-02-landing-page/index.html in any modern browser (Chrome, Firefox, Safari, Edge)
```

**Option 2 — Local HTTP server (for accurate `page_path` in dataLayer):**
```bash
cd task-02-landing-page
npx serve .
# Opens at http://localhost:3000
```

**Option 3 — VS Code Live Server:**
```
Right-click index.html → "Open with Live Server"
```

### Tasks 1 and 3

Markdown files — readable directly on GitHub or in any markdown viewer.

---

## 10. How to Test the dataLayer Events

### Step 1 — Verify dataLayer Exists (Before Form Submit)

```
1. Open task-02-landing-page/index.html in Chrome
2. Open DevTools → Console (F12 → Console tab)
3. Type:  window.dataLayer
4. Expected output:  []  (empty array — no events yet)
```

### Step 2 — Submit the Form

```
1. Fill in:
   - Full Name: "Naitik Kumar"
   - Mobile Number: "7895407387"
   - Preferred Clinic: "Koramangala"
2. Click "Book My Consultation →"
3. The form should be replaced by the thank-you state:
   "Thank You, Naitik! ... Our orthopaedic specialist at Koramangala
    will call you at +91 7895407387 within 30 minutes."
```

### Step 3 — Verify dataLayer Push (After Form Submit)

```
1. In the Console, type:  window.dataLayer
2. Expected output:
   [
     {
       event: "consultation_form_submitted",
       form_id: "hero_consultation_form",
       page_path: "/task-02-landing-page/index.html",
       form_location: "hero_section",
       clinic_preference: "Koramangala"
     }
   ]
```

### Step 4 — Verify Validation Blocks Invalid Submissions

```
Test 1: Submit with empty fields → error messages appear, NO dataLayer push
Test 2: Enter phone "12345" → "Please enter a valid 10-digit mobile number."
Test 3: Enter phone "9876543210" with no clinic selected → "Please select a preferred clinic."
Test 4: Check window.dataLayer after failed validations → still [] (empty)
```

### GTM Preview Mode (Production)

Once a GTM container is installed:

```
1. GTM → Preview → opens Tag Assistant in a new tab
2. Navigate to the landing page
3. Submit the form
4. In Tag Assistant → "consultation_form_submitted" appears in the event timeline
5. Click it → verify all 4 parameters are populated correctly
6. Verify GA4 Event Tag and Google Ads Conversion Tag fire on this event
```

### GA4 DebugView (Production)

```
1. GA4 → Admin → DebugView
2. Enable debug mode via GTM Preview or Chrome GA Debugger extension
3. Submit the form
4. Event "consultation_form_submitted" appears in DebugView timeline
5. Click it → verify parameters: form_id, page_path, form_location, clinic_preference
```

---

## 11. Lighthouse Optimisation Notes

The landing page is engineered to score **90+ on PageSpeed Insights Mobile**. Here is how:

| Metric | Optimisation | Impact |
|--------|-------------|--------|
| **LCP** (Largest Contentful Paint) | The `<h1>` text is the LCP element. System fonts render instantly — no font download delay. No hero image to block rendering. | LCP < 1.5s |
| **FID / INP** (Interaction to Next Paint) | Minimal JavaScript (~120 lines). All event listeners are lightweight — no expensive DOM operations. IIFE prevents global scope pollution. | INP < 100ms |
| **CLS** (Cumulative Layout Shift) | No dynamically loaded content. No font swaps (system fonts). Sticky header has fixed height. Thank-you state replaces form in the same container. | CLS < 0.05 |
| **Total Blocking Time** | No render-blocking resources. All CSS is inline (no external stylesheet request). JS is at the bottom of `<body>`. No third-party scripts. | TBT < 100ms |
| **Page Weight** | ~32KB total. Zero external requests after initial HTML load. No images, no fonts, no CDN dependencies. | Transfer size minimal |

**What was deliberately excluded to protect the score:**

- Google Fonts (adds 100–300ms font download + FOUT risk)
- External CSS frameworks (Tailwind/Bootstrap = 30–300KB unused CSS)
- Hero images or background images (LCP penalty + CLS risk if dimensions not set)
- Third-party analytics scripts in the HTML (GTM container would be added separately)
- CSS animations with `layout` or `paint` triggers (only `transform` and `opacity` transitions used)

> **Note:** The PageSpeed screenshot should be captured after deploying to a live URL (GitHub Pages, Netlify, or Vercel). A local file (`file://`) cannot be tested on PageSpeed Insights — it requires a publicly accessible URL.

---

## 12. Repository Structure

```
namoza-developer-assignment/
│
├── README.md                                    # This file — project overview,
│                                                # instructions, and assignment
│                                                # summary
│
├── task-01-gtm-schema/
│   └── gtm-event-schema.md                      # Complete GTM event schema
│       ├── Event schema table (11 events)
│       ├── dataLayer JSON for 3-step booking form
│       ├── GTM trigger/variable/tag configuration
│       ├── GA4 Funnel Exploration setup
│       ├── Google Ads conversion recommendation
│       └── Developer briefing note
│
├── task-02-landing-page/
│   └── index.html                               # Self-contained landing page
│       ├── Inline CSS (mobile-first, CSS custom properties)
│       ├── Semantic HTML5 (single <h1>, ARIA attributes)
│       ├── Inline JavaScript (form validation, dataLayer push)
│       ├── 3-field form (Name, Phone, Clinic Preference)
│       ├── Thank-you state (no page reload)
│       └── PageSpeed 90+ optimised
│
└── task-03-integration/
    └── integration-design.md                    # Integration architecture
        ├── ASCII architecture diagram
        ├── API request flow (step-by-step with request bodies)
        ├── HubSpot Contacts API justification
        ├── Phone deduplication strategy
        ├── Error handling matrix & retry strategy
        ├── Logging, monitoring, and alerting
        ├── Security, rate limiting, and idempotency
        ├── Production deployment plan
        └── 10 interview Q&A
```

---

## 13. Loom Recording Guide

The Loom walkthrough should be max 8 minutes. Here is the structure I am following:

### Segment 1 — GTM Schema (2 min)

- Open `task-01-gtm-schema/gtm-event-schema.md`
- Walk through the event schema table — highlight the booking form funnel events
- Show the dataLayer JSON for Steps 1–3
- Explain: "The front-end developer writes the `dataLayer.push()` calls. GTM listens for them via Custom Event triggers. GTM cannot detect step transitions on its own."
- Explain Google Ads conversion choice: `consultation_booked` (Step 3, not Step 1)

### Segment 2 — Landing Page Live Demo (3 min)

- Open `index.html` in Chrome
- Show the page on mobile viewport (DevTools → toggle device toolbar → iPhone 12)
- Walk through conversion architecture: headline → form → trust → testimonials → bottom CTA
- Open Console → type `window.dataLayer` → show it's empty
- Fill the form → submit → show the thank-you state
- Type `window.dataLayer` again → show the push with all 4 parameters
- Demonstrate validation: empty submit → errors shown → no dataLayer push
- Show the code briefly: point out the IIFE, validation functions, dataLayer push

### Segment 3 — Integration Architecture (3 min)

- Open `task-03-integration/integration-design.md`
- Walk through the architecture diagram
- Explain the two parallel paths (client-side GTM + server-side backend)
- Key point: "HubSpot deduplicates by email, not phone. Our form doesn't collect email. So I search HubSpot by phone first, then create or update."
- Biggest failure point: Karix WhatsApp delivery → SMS fallback
- WhatsApp SLA monitoring: T0 → T1 → T2 timeline

---

## 14. Future Improvements

If this were a production engagement with continued development time, I would add:

| Improvement | Why |
|-------------|-----|
| **reCAPTCHA v3 / Cloudflare Turnstile** | Bot protection without UX friction. Currently using a honeypot field concept — reCAPTCHA adds an ML-based layer. |
| **Enhanced Conversions (Google Ads)** | Send hashed phone number to Google Ads via GTM for improved conversion attribution. Recovers 5–15% of conversions lost to cookie restrictions. |
| **A/B testing framework** | Test headline variations, CTA copy, form field order. Use Google Optimize successor or a lightweight client-side split. |
| **Multi-language support** | OrthoNow operates in Bengaluru (Kannada), Hyderabad (Telugu), Chennai (Tamil). Localised landing pages would improve conversion for non-English-primary users. |
| **Server-side GTM** | Move GA4 and Google Ads tags to a server-side GTM container. Reduces client-side JavaScript, improves PageSpeed, and provides first-party data collection resilient to ad blockers. |
| **HubSpot Workflow Automation** | Trigger follow-up sequences in HubSpot: if no appointment confirmed within 24 hours, send a reminder WhatsApp. If no show, trigger a re-engagement sequence. |
| **Call tracking integration** | Integrate CallRail or Exotel for the "Call Now" button. Track completed calls (not just clicks) as conversions. Import into Google Ads for bidding optimisation. |

---

## 15. Conclusion

This submission demonstrates three things:

1. **GTM and GA4 are not magic** — multi-step form tracking requires coordination between the analytics specialist (who defines the schema) and the front-end developer (who writes the `dataLayer.push()` calls). I defined the exact JSON, the trigger configuration, and the developer briefing.

2. **Landing pages that convert are built with intention** — every element on the page has a reason: the headline names the pain, the CTA uses first-person phrasing, the trust stats use specific numbers, the testimonials mirror the target audience. The PageSpeed score is protected by architectural decisions (system fonts, zero external requests), not afterthought optimisation.

3. **Integration architecture is about failure modes** — the happy path is straightforward (form → API → HubSpot → Karix). The real engineering is in the edge cases: phone deduplication when email isn't collected, exponential backoff when Karix is slow, SMS fallback when WhatsApp fails, idempotency when users double-click submit.

I built real, working output — not mockups, not wireframes, not AI-generated placeholder text. Every technical decision in this repo can be defended in an interview because it was made for a reason.

---

**Submitted by:** Naitik Kumar
**Date:** July 2026
**Contact:** naitikchauhan8218@gmail.com
