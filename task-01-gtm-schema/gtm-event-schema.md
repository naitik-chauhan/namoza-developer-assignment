# Task 01 — GTM Event Schema for OrthoNow

## Context

OrthoNow operates 9 orthopaedic clinics across Bengaluru, Hyderabad, and Chennai. The existing
WordPress site has zero event tracking — GA4 is configured with pageviews only, no GTM container,
and no CRM integration. Before any paid campaigns go live, we need full event instrumentation so the
performance marketing team can measure what matters.

This document defines the complete GTM event schema, the booking form funnel tracking implementation,
and the primary Google Ads conversion action.

---

## 1. Complete GTM Event Schema

All event names follow GA4's `snake_case` convention. Parameters are designed to support both
GA4 reporting dimensions and Google Ads audience building.

| # | Event Name | Trigger Type | Key Parameters | GA4 Report / Audience |
|---|-----------|-------------|----------------|----------------------|
| 1 | `booking_step_complete` | Custom Event (dataLayer) | `step_number` (int: 1–3), `step_name` (string), `clinic_location` (string), `specialty` (string), `form_id` (string) | **Funnel Exploration**: 3-step booking funnel with step-level drop-off. **Audience**: "Started Booking But Did Not Complete" (step 1 or 2 without step 3) |
| 2 | `booking_abandoned` | Custom Event (dataLayer) | `last_step_completed` (int: 0–2), `clinic_location` (string), `specialty` (string), `time_on_form_sec` (int), `form_id` (string) | **Exploration report**: Abandonment analysis by step × clinic. **Audience**: "Booking Abandoners" for remarketing |
| 3 | `consultation_booked` | Custom Event (dataLayer) | `clinic_location` (string), `specialty` (string), `booking_date` (string, YYYY-MM-DD), `form_id` (string), `page_path` (string) | **Conversions report**: Primary conversion. **Audience**: "Completed Booking" for exclusion lists and lookalikes |
| 4 | `click_call_now` | GTM Click Trigger (CSS selector or `tel:` link click) | `link_url` (string, the `tel:` href), `page_path` (string), `clinic_location` (string), `button_position` (string: header/hero/footer) | **Events report**: Call click volume by page and clinic. **Audience**: "High Intent — Called" |
| 5 | `click_whatsapp_chat` | GTM Click Trigger (CSS selector on WhatsApp widget / `wa.me` link) | `link_url` (string), `page_path` (string), `clinic_location` (string), `device_category` (string: mobile/desktop) | **Events report**: WhatsApp engagement by page. **Audience**: "WhatsApp Engaged" for remarketing |
| 6 | `patient_guide_form_submit` | Custom Event (dataLayer) | `guide_name` (string), `page_path` (string), `form_id` (string) | **Events report**: Guide download conversion rate. **Audience**: "Downloaded Guide — Nurture" |
| 7 | `patient_guide_download` | GTM Click Trigger (PDF link click, filtered by file extension `.pdf`) | `file_name` (string), `guide_name` (string), `page_path` (string) | **Events report**: Actual download completions vs. form fills |
| 8 | `clinic_page_view` | GTM Page View Trigger (fires on `/clinic/*` URL pattern) | `page_path` (string), `clinic_location` (string, extracted via GTM Lookup Table variable from URL), `city` (string) | **Pages and screens report**: Clinic page popularity. **Audience**: "Viewed [City] Clinics" for geo-targeted campaigns |
| 9 | `blog_scroll_depth` | GTM Scroll Depth Trigger (thresholds: 25%, 50%, 75%, 90%) | `scroll_threshold` (int: 25/50/75/90), `page_path` (string), `article_title` (string, via DOM scraping variable), `content_category` (string) | **Exploration report**: Content engagement quality. **Audience**: "Deep Readers (75%+)" for content remarketing |
| 10 | `blog_article_view` | GTM Page View Trigger (fires on `/blog/*` URL pattern) | `page_path` (string), `article_title` (string), `content_category` (string), `author` (string) | **Pages and screens report**: Article traffic. Combine with scroll depth for engagement scoring |
| 11 | `form_field_interaction` | Custom Event (dataLayer) | `form_id` (string), `field_name` (string), `page_path` (string) | **Exploration report**: Form field-level engagement. Identifies UX friction points |

### Naming Conventions & Standards

- All custom events use `snake_case` — aligned with GA4's recommended format.
- `page_path` is included on every event as a default parameter for cross-page analysis.
- `clinic_location` uses consistent values matching the 9 clinic names (e.g., `"Koramangala"`, `"Jubilee Hills"`) — this must be standardized across the WordPress site's data attributes and GTM variables.
- `form_id` enables disambiguation when multiple forms exist on the same page.

---

## 2. Booking Form Funnel — Step-Level Drop-Off Tracking

### How This Works (Critical Implementation Detail)

OrthoNow's booking form is a **3-step multi-step form** — meaning all 3 steps render on a single page
URL. The user progresses through steps via JavaScript that shows/hides form sections (or swaps
DOM content). **The URL does not change between steps.**

This is critical because:

> **GTM cannot natively detect multi-step form transitions.** GTM's built-in triggers (Page View,
> Click, Form Submission) have no awareness of JavaScript-driven step changes inside a single-page
> form. There is no "form step" trigger in GTM.

**Therefore, the front-end developer must write `window.dataLayer.push()` calls** that fire at each
step transition. The GTM specialist (me) defines the schema and the trigger configuration. The
front-end developer implements the actual push calls in the form's JavaScript logic.

### Implementation Workflow

1. **I (GTM/analytics developer)** define the exact dataLayer schema below and create a
   developer briefing document.
2. **The front-end developer** adds `window.dataLayer.push(...)` calls at each step transition
   in the form's JavaScript.
3. **I configure GTM** to listen for the `booking_step_complete` Custom Event and create
   corresponding GA4 event tags with parameter mappings.
4. **QA**: I verify in GTM Preview Mode + GA4 DebugView that events fire correctly at each step.

---

### dataLayer Push — Step 1: Clinic Location & Specialty Selected

This fires when the user selects a clinic and specialty, then clicks "Next" to proceed to Step 2.

```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "Koramangala",
  "specialty": "Knee Replacement",
  "form_id": "booking_main",
  "page_path": "/book-consultation"
}
```

**Front-end implementation note**: This push should fire inside the click handler of the "Next" button
for Step 1, **after** form validation passes (i.e., both `clinic_location` and `specialty` dropdowns
have valid selections). If validation fails, the push must NOT fire — otherwise we inflate funnel
entry numbers.

---

### dataLayer Push — Step 2: Patient Details Entered

This fires when the user fills in name, phone, and preferred date, then clicks "Next" to proceed
to the confirmation screen.

```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "patient_details_entered",
  "clinic_location": "Koramangala",
  "specialty": "Knee Replacement",
  "patient_name_provided": true,
  "phone_provided": true,
  "preferred_date": "2026-07-15",
  "form_id": "booking_main",
  "page_path": "/book-consultation"
}
```

**Why `patient_name_provided: true` instead of the actual name?** PII (Personally Identifiable
Information) should never be sent to GA4 — this violates Google's Terms of Service and DPDP Act
(India's data protection law). We track **whether** the field was filled, not **what** was entered.
The actual PII flows to the CRM, not the analytics layer.

**Front-end implementation note**: This push fires in the Step 2 "Next" button handler, after
phone number format validation (Indian 10-digit mobile). Carry forward `clinic_location` and
`specialty` from Step 1 so the funnel report can segment by these dimensions without requiring
the analyst to stitch steps manually.

---

### dataLayer Push — Step 3: Booking Confirmed

This fires when the user reviews their details on the confirmation screen and clicks "Confirm Booking."

```json
{
  "event": "booking_step_complete",
  "step_number": 3,
  "step_name": "booking_confirmed",
  "clinic_location": "Koramangala",
  "specialty": "Knee Replacement",
  "preferred_date": "2026-07-15",
  "form_id": "booking_main",
  "page_path": "/book-consultation",
  "booking_method": "website"
}
```

**Additionally**, Step 3 also fires the conversion event for Google Ads:

```json
{
  "event": "consultation_booked",
  "clinic_location": "Koramangala",
  "specialty": "Knee Replacement",
  "preferred_date": "2026-07-15",
  "form_id": "booking_main",
  "page_path": "/book-consultation",
  "booking_method": "website"
}
```

**Front-end implementation note**: Two pushes fire in sequence at Step 3 — `booking_step_complete`
(for funnel completeness) and `consultation_booked` (for the conversion tag). Alternatively,
`consultation_booked` can be configured as a GTM trigger that fires based on
`booking_step_complete` where `step_number == 3`, reducing the dev's implementation to a single
push. I prefer the explicit two-push approach for clarity and separation of concerns.

---

### dataLayer Push — Abandonment Tracking

This fires when the user navigates away from the form (via `beforeunload` event) or remains
idle for 60+ seconds after interacting with a step.

```json
{
  "event": "booking_abandoned",
  "last_step_completed": 1,
  "clinic_location": "Koramangala",
  "specialty": "Knee Replacement",
  "time_on_form_sec": 34,
  "form_id": "booking_main",
  "page_path": "/book-consultation",
  "abandonment_trigger": "page_exit"
}
```

**Front-end implementation note**: Use `window.addEventListener('beforeunload', ...)` to detect
exit. Track `time_on_form_sec` using a `Date.now()` timestamp captured when the form first
renders. The `abandonment_trigger` can be `"page_exit"`, `"idle_timeout"`, or `"back_button"`.
Note: `beforeunload` has limited reliability on mobile browsers — this is a best-effort signal,
not a guaranteed capture.

---

### GTM Configuration for Booking Form Events

| GTM Element | Type | Configuration |
|------------|------|---------------|
| **Trigger**: `CE - booking_step_complete` | Custom Event | Event name = `booking_step_complete` |
| **Trigger**: `CE - booking_abandoned` | Custom Event | Event name = `booking_abandoned` |
| **Trigger**: `CE - consultation_booked` | Custom Event | Event name = `consultation_booked` |
| **Variable**: `DLV - step_number` | Data Layer Variable | Variable name = `step_number` |
| **Variable**: `DLV - step_name` | Data Layer Variable | Variable name = `step_name` |
| **Variable**: `DLV - clinic_location` | Data Layer Variable | Variable name = `clinic_location` |
| **Variable**: `DLV - specialty` | Data Layer Variable | Variable name = `specialty` |
| **Variable**: `DLV - form_id` | Data Layer Variable | Variable name = `form_id` |
| **Variable**: `DLV - preferred_date` | Data Layer Variable | Variable name = `preferred_date` |
| **Variable**: `DLV - last_step_completed` | Data Layer Variable | Variable name = `last_step_completed` |
| **Variable**: `DLV - time_on_form_sec` | Data Layer Variable | Variable name = `time_on_form_sec` |
| **Tag**: `GA4 - booking_step_complete` | GA4 Event | Event name: `booking_step_complete`. Parameters: all DLVs above. Fires on: `CE - booking_step_complete` |
| **Tag**: `GA4 - booking_abandoned` | GA4 Event | Event name: `booking_abandoned`. Parameters: `last_step_completed`, `clinic_location`, `specialty`, `time_on_form_sec`. Fires on: `CE - booking_abandoned` |
| **Tag**: `GA4 - consultation_booked` | GA4 Event | Event name: `consultation_booked`. Parameters: `clinic_location`, `specialty`, `preferred_date`, `booking_method`. Fires on: `CE - consultation_booked` |
| **Tag**: `Google Ads Conversion - consultation_booked` | Google Ads Conversion Tracking | Conversion ID + Label from Google Ads. Fires on: `CE - consultation_booked` |

### GA4 Funnel Exploration Setup

To surface step-level drop-off in GA4:

1. Go to **GA4 → Explore → Funnel Exploration**
2. Configure a **Closed Funnel** (users must enter at Step 1):
   - **Step 1**: Event = `booking_step_complete`, condition: `step_number == 1`
   - **Step 2**: Event = `booking_step_complete`, condition: `step_number == 2`
   - **Step 3**: Event = `booking_step_complete`, condition: `step_number == 3`
3. Add **Breakdown dimension**: `clinic_location` — this reveals if specific clinics have
   higher drop-off (could indicate UX issues with certain clinic's available specialties).
4. Add **Breakdown dimension**: `specialty` — reveals if certain specialties have higher
   abandonment (could indicate pricing concerns or availability issues).
5. Set the **funnel completion window** to 30 minutes — booking should be a single-session
   activity.

**Expected output**: A visual funnel showing completion rate and absolute drop-off between each
step. For example: 1,000 → 620 → 410 gives Step 1→2 drop-off of 38% and Step 2→3 drop-off
of 34%.

---

## 3. Google Ads Conversion Action

### Selected Conversion: `consultation_booked` (Step 3 — Booking Confirmed)

**Why this event over the others:**

| Alternative Considered | Why Not |
|----------------------|---------|
| `booking_step_complete` (Step 1) | Too early in funnel. High volume, low intent signal. Smart Bidding would optimise for users who start forms, not for users who book. Leads to wasted ad spend on tire-kickers. |
| `booking_step_complete` (Step 2) | Better than Step 1, but still not a completed action. The patient hasn't confirmed — they might abandon at the review screen. |
| `click_call_now` | Valuable signal, but a click ≠ a connected call. Without call tracking (e.g., CallRail), we can't verify the call happened. Importing this inflates conversion counts. |
| `click_whatsapp_chat` | Same issue — a click opens `wa.me` but doesn't guarantee a conversation started. Low-quality signal for bidding. |
| `patient_guide_form_submit` | Top-of-funnel action. These users are researching, not booking. Wrong optimization target for a "Book a Consultation" campaign. |

**Why `consultation_booked` is correct:**

1. **Highest-quality intent signal available without backend verification.** The user has selected
   a clinic, entered their details, and confirmed. This is the strongest client-side signal of
   booking intent.
2. **Appropriate conversion volume.** For Smart Bidding (tCPA/tROAS) to work effectively, Google
   recommends 30–50 conversions per campaign per month. A Step 3 conversion will have lower
   volume than Step 1, but across 9 clinics with paid traffic, this should clear the threshold.
3. **Clean signal for campaign optimization.** By importing only completed bookings, we give
   Google Ads a clear signal: "Find me more users like the ones who actually complete a booking."
   This produces better lookalike modeling and bid optimization.
4. **Avoids double-counting.** One conversion action = one import. If we imported multiple steps,
   a single user journey would register as 3 conversions, inflating ROAS calculations.

### Google Ads Import Setup

1. In GA4: Mark `consultation_booked` as a **Key Event** (formerly "conversion").
2. In Google Ads: Go to **Tools → Conversions → New Conversion Action → Import → Google Analytics 4**.
3. Select `consultation_booked` from the list.
4. Set **counting**: "One" (one conversion per user session — prevents duplicate bookings from
   inflating numbers).
5. Set **conversion window**: 30 days (standard for healthcare consideration cycles).
6. Set **attribution model**: Data-driven (Google's default, uses ML to distribute credit across
   touchpoints).

---

## 4. Developer Briefing Note — How I Would Brief the Dev Team

When the interviewer asks: *"Who writes the dataLayer push — you or the front-end dev?"*

**Answer**: I define the schema. The front-end developer writes the code.

Here's how I would brief them:

> **To: Front-End Dev Team**
> **Re: dataLayer implementation for booking form event tracking**
>
> We need `window.dataLayer.push()` calls added to the multi-step booking form at three points.
> I've defined the exact JSON payloads above. Here's what you need to know:
>
> 1. **Ensure `window.dataLayer = window.dataLayer || [];` exists** before GTM container loads.
>    This should already be in the GTM snippet, but verify it's present before any push calls.
>
> 2. **Push timing matters.** Each push must fire AFTER form validation passes for that step,
>    inside the success callback of the "Next" / "Confirm" button handler. Never push before
>    validation — we don't want to count failed attempts as completions.
>
> 3. **Carry forward parameters.** Steps 2 and 3 must include `clinic_location` and `specialty`
>    from Step 1. Store these in a JavaScript variable or form state object when Step 1 completes.
>
> 4. **No PII in pushes.** Do not include patient name, phone number, or email in any dataLayer
>    push. Use boolean flags (`patient_name_provided: true`) instead.
>
> 5. **Test with GTM Preview Mode.** After implementation, I'll share a GTM Preview link. Open it,
>    walk through the form, and verify that each push appears in the Preview console at the correct
>    step.
>
> **Estimated dev effort**: 2–3 hours for a developer familiar with the form codebase.

---

*Document prepared by Naitik Kumar for Namoza Developer Assignment — Task 01*
