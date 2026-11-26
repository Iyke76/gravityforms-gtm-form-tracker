# Gravity Forms GTM Form Tracker

Confirmation-render-based form tracking for Gravity Forms in Google Tag Manager.

This standalone tracker fires dataLayer events when Gravity Forms produces an observable confirmation, validation failure, or postback confirmation page, and when someone starts a form and leaves without submitting it. It is built for GTM, GA4, server-side GTM, Meta CAPI, and Google Enhanced Conversions workflows where button-click tracking is too noisy.

## Why this exists

Most form tracking is wired to a submit button click or a thank-you page view.

Click tracking counts validation failures, repeated clicks, and rage clicks. Thank-you-page tracking can count refreshes, direct visits, and bookmarked URLs. This script watches Gravity Forms' own confirmation and validation signals so the dataLayer event is closer to a real form outcome.

## Files

| File | Use case |
| --- | --- |
| `gravityforms-form-tracker.js` | Raw JavaScript for GitHub, CDN, or custom hosting. Do not wrap this file in another `<script>` tag when loading by URL. |
| `gravityforms-form-tracker.gtm.html` | GTM Custom HTML paste-ready version. Includes the `<script>` wrapper. |

## Events

The tracker pushes these events to `window.dataLayer`:

```text
gf_form_success
gf_form_failed
gf_form_abandon
ec_form_interaction
ec_tracker_initialized
```

### Success event

`gf_form_success` fires when the script detects one of these success paths:

- `gform_confirmation_loaded` event
- `.gform_confirmation_wrapper` render
- Gravity Forms postback confirmation page with wrapper present and no form
- Same-domain thank-you URL recall when recent stored form data exists

### Failure event

`gf_form_failed` fires when Gravity Forms renders an observable validation failure after the user has interacted with the form.

By default (`failedPayload: 'minimal'`) a failed event carries no contact data, hashes or raw field values. It keeps the form metadata, `ec_detection_method`, `ec_failed_fields` (field names only, never values) and `ec_failed_field_count`. Set `failedPayload: 'full'` for the v3.3.0 behaviour.

### Abandon event

`gf_form_abandon` fires when someone started a form and left without submitting it. See [Form abandonment](#form-abandonment).

### Diagnostic events

`ec_form_interaction` fires once per form after the first real user input. It contains metadata only, not PII.

`ec_tracker_initialized` fires once per page load so you can confirm the script loaded even before a test submission.

## Payload example

Default (lean) success payload, `event` first:

```js
{
  event: "gf_form_success",
  ec_name: "Jane Doe",
  ec_email: "jane@example.com",
  ec_phone: "+15551234567",
  form_name: "quote_request",
  form_cta: "Submit",
  form_platform: "gravityforms",
  form_position: "top",
  form_index: 1,
  form_id: "gform_3",
  ec_event_id: "uuid-v4",
  ec_detection_method: "confirmation_class",
  ec_submit_attempt: 1,
  ec_time_to_submit_s: 42,
  ec_fast_submit: false,
  ec_last_source: "google",
  ec_click_id: "Cj0KCQ...",
  ec_click_id_type: "gclid"
}
```

Lean rules (`payload: 'lean'`, the default):

- `event` is always the first key.
- No `ec_attribution` object, `ec_first_landing`, `ec_first_referrer`, `ec_first_ts` or `ec_time_to_submit_ms`. The compact attribution keys stay.
- No `form_url`. GA4 already records the page as `page_location`.
- `ec_tracker_version` and `ec_dedupe_basis` only with `debug: true`.
- Blank, null and undefined keys are removed.
- `payload: 'full'` restores the v3.3.0 key set, including `form_url`.

Other payload rules:

- `rawFields` adds `ec_raw_fields`. It also accepts an array as an allowlist, for example `rawFields: ['input_5']`.
- `hashPii: true` adds `ec_email_sha256` and `ec_phone_sha256`.
- `form_name` comes from a Hidden field (see [form_name](#form_name)).
- Before each push, keys from an earlier push that the new push does not set are cleared, so one form's raw fields or failed fields never leak into the next event.

When PII consent is denied, contact keys, hashes and raw fields are removed:

```js
{
  event: "gf_form_success",
  form_cta: "Submit",
  form_platform: "gravityforms",
  form_position: "top",
  form_index: 1,
  form_id: "gform_3",
  ec_event_id: "uuid-v4",
  ec_detection_method: "confirmation_class",
  ec_pii_redacted: true
}
```

## Privacy modes

`piiMode` decides what contact data leaves the browser.

| `piiMode` | Payload | Browser storage (thank-you recall) |
| --- | --- | --- |
| `'plain'` (default) | Normalized `ec_email`, `ec_phone`, name and address keys | Full copy in localStorage and sessionStorage, as in v3.3.0 |
| `'hashed'` | No plain contact keys. Email and phone only as `ec_email_sha256` / `ec_phone_sha256` | Minimal copy in sessionStorage only (form name, allowlisted and choice fields, hashes). localStorage copy removed |
| `'none'` | No contact keys and no hashes | Minimal copy in sessionStorage only. localStorage copy removed |

- `'hashed'` turns `hashPii` on and waits up to `hashWaitMs` (300 ms) for the hash, so an instant submit still gets it.
- In `'hashed'` and `'none'` mode without an allowlist, `ec_raw_fields` keeps only select, radio and checkbox values.
- Failed events carry no contact data in any mode unless `failedPayload: 'full'`.

Recommended config when GA4 is the only consumer:

```js
window.EC_FORM_CONFIG = {
  piiMode: 'none',
  rawFields: false,
  payload: 'lean',
  failedPayload: 'minimal'
};
```

Need the v3.3.0 payload exactly? Use `payload: 'full'`, `failedPayload: 'full'`, `piiMode: 'plain'` and `trackAbandon: false`. Only the safety fixes marked * in the changelog stay.

### form_name

A stable label per form purpose keeps triggers working when a form is rebuilt with a new ID.

1. Add a **Hidden** field to the form and set its Default Value to your label, for example `quote_request`.
2. Either give the field the Custom CSS Class `form_name`, or set `formNameField` to its input name (for example `formNameField: 'input_5'`).

The value is pushed as `form_name` on success, failed and abandon events. It is never treated as a name or other contact field.

## Form abandonment

`gf_form_abandon` tells you who started a form and left without submitting, which field they stopped on, how far they got and how long they spent. It never sends what was typed.

- **Started**: the first input or change that leaves a field with a value (browser autofill counts). Focus alone does not count. Hidden, honeypot, redacted and `form_name` fields are ignored. A page that loads with Gravity Forms validation errors (postback forms) or on page 2+ of a multi-page form counts as started, since the visitor already submitted something.
- **Exit signals**: `pagehide`, the tab going hidden (`abandonOnHidden`, catches mobile app closes), and an SPA route change to another path. The first signal wins, at most one abandon per form per page load.
- **Never fires** when the form succeeded, a submit or Next/Previous click is still pending (page navigation or redirect in progress), analytics consent is denied, or `trackAbandon` is false.
- **Stages**: `before_submit` (never submitted) and `after_error` (submitted, Gravity Forms showed validation errors, no success).
- If the visitor switches tabs (abandon with `visibility_hidden`), comes back and submits, the success carries `ec_resumed: true`.

```js
{
  event: "gf_form_abandon",
  form_id: "gform_3",
  form_name: "quote_request",
  form_platform: "gravityforms",
  form_position: "top",
  form_index: 1,
  ec_detection_method: "pagehide",   // pagehide | visibility_hidden | spa_route
  ec_abandon_stage: "after_error",   // before_submit | after_error
  ec_abandon_field: "input_4",       // last field focused, same key as ec_raw_fields
  ec_fields_completed: 3,
  ec_fields_total: 6,
  ec_form_step: 2,                   // multi-page forms only
  ec_time_on_form_s: 47,
  ec_event_id: "uuid-v4"
}
```

Gravity Forms notes:

- AJAX forms replace the whole form after a validation error or a page change. Abandonment follows the Gravity Forms form number, so progress and the error stage carry across the re-render.
- Multi-page forms report `ec_form_step` from Gravity Forms' own page number.
- On postback (non-AJAX) multi-page forms each page is a new page load, so `ec_time_on_form_s` counts from the start of the current page.
- Turn it off with `trackAbandon: false`.

## Quick start in GTM

1. Open Google Tag Manager.
2. Create a new **Custom HTML** tag.
3. Paste the contents of `gravityforms-form-tracker.gtm.html`.
4. Trigger it on **All Pages - Page View** or **Consent Initialization** if your consent setup needs the tracker available early.
5. Preview the container.
6. Submit a real Gravity Forms test form.
7. Confirm `gf_form_success` or `gf_form_failed` appears in the dataLayer. Type into a field and close the tab to see `gf_form_abandon`.
8. Create GA4, server-side GTM, Meta CAPI, or Enhanced Conversions tags that listen for the events.

## Loading as raw JavaScript

If you host `gravityforms-form-tracker.js` yourself, load it like this:

```html
<script src="https://your-domain.example/gravityforms-form-tracker.js"></script>
```

Do not use `gravityforms-form-tracker.gtm.html` as a hosted JavaScript file. It is only for GTM Custom HTML paste-in use.

## Configuration

Define `window.EC_FORM_CONFIG` before loading the tracker:

```html
<script>
window.EC_FORM_CONFIG = {
  debug: false,
  hashPii: false,
  piiMode: "plain",
  payload: "lean",
  failedPayload: "minimal",
  formNameField: "form_name",
  trackAbandon: true,
  abandonOnHidden: true,
  dedupe: true,
  allowRecoverySuccess: true,
  submitWindowMs: 15000,
  ttlMs: 600000,
  defaultCountryCode: "1",
  extraThankYouPatterns: []
};
</script>
```

| Option | Default | What it does |
| --- | --- | --- |
| `debug` | `false` | Enables `[EC]` console logs. Use in GTM Preview/testing only. |
| `hashPii` | `false` | Adds `ec_email_sha256` and `ec_phone_sha256` when possible. Forced on by `piiMode: 'hashed'`. |
| `piiMode` | `'plain'` | `'plain'`, `'hashed'` (email and phone only as SHA-256) or `'none'` (no contact data). See [Privacy modes](#privacy-modes). |
| `hashWaitMs` | `300` | In hashed mode, how long to wait for the SHA-256 before pushing. |
| `rawFields` | `false` | `true` adds `ec_raw_fields`. An array is an allowlist, e.g. `['input_5']`. |
| `redactFields` | `[]` | Extra field names always stripped. |
| `payload` | `'lean'` | `'lean'` sends only keys with a use; `'full'` restores the v3.3.0 key set. |
| `failedPayload` | `'minimal'` | `'minimal'` keeps contact data off failed events; `'full'` is the v3.3.0 behaviour. |
| `formNameField` | `'form_name'` | CSS class or input name of the Hidden field pushed as `form_name`. |
| `trackAbandon` | `true` | Push `gf_form_abandon` when a started form is left unsubmitted. `false` registers no listeners. |
| `abandonOnHidden` | `true` | Also treat the tab going hidden as an exit. `false` uses only `pagehide` and route changes. |
| `strictSuccess` | `false` | `true` drops inferred successes. |
| `countryCodeField` | `''` | Exact field name holding an ISO2 or dial code for phone normalization. |
| `dedupe` | `true` | Suppresses duplicate success events for the same form/email/phone within 5 minutes. |
| `allowRecoverySuccess` | `true` | Kept for config consistency across EC standalone trackers. Gravity Forms success can fire after a prior failure when a confirmation appears. |
| `submitWindowMs` | `15000` | Kept for config consistency across EC standalone trackers. Gravity Forms success/failure is confirmation/validation-render based. |
| `ttlMs` | `600000` | Freshness window for stored form data used by same-domain thank-you recall. |
| `defaultCountryCode` | `"1"` | Country code used for best-effort phone normalization. Set `""` to disable. |
| `extraThankYouPatterns` | `[]` | Additional URL substrings that identify thank-you pages. |

Always stripped, whatever the config: password, card, SSN and IBAN style fields, plus API key, secret, access/auth token, CSRF and nonce fields.

## Consent

The tracker checks consent before firing events or writing storage.

Auto-detected consent systems:

- Google Consent Mode v2
- OneTrust
- Cookiebot

You can override consent with `window.EC_FORM_CONSENT_CHECK` before the tracker loads:

```html
<script>
window.EC_FORM_CONSENT_CHECK = function () {
  return {
    analytics: true,
    pii: false
  };
};
</script>
```

Consent behavior:

| Consent state | Behavior |
| --- | --- |
| `analytics: false` | No events fire and no storage is written. |
| `pii: false` | Events can fire, but contact keys, hashes and raw fields are removed. Storage and recall data are redacted. |

If no CMP or override is detected, the tracker defaults to full consent. You are responsible for matching this behavior to your site and legal requirements.

## PII warning

The success and failure payloads can include:

- `ec_email`
- `ec_phone`
- `ec_name`
- `ec_raw_fields`

`form_name` is a label you choose, so keep personal data out of it.

Do not send raw PII to GA4. GA4 terms prohibit PII in event parameters.

Recommended approaches:

- Use server-side GTM and hash values before forwarding to ad platforms.
- Use `piiMode: 'hashed'` (only hashed email and phone reach the browser's dataLayer) or `piiMode: 'none'` when no tag needs contact data.
- Explicitly exclude raw PII fields from GA4 Event tags.
- Use raw fields only for first-party CRM workflows where you are allowed to process the data.
- Never send `gf_form_abandon` to ad platforms as a conversion, and never attach contact data to it.

## GA4 reporting

Failed events:

- `failure_reason` = `{{Dlv - ec_detection_method}}`, `failed_fields` = a Custom JavaScript variable that joins `ec_failed_fields` with commas, `failed_field_count` = `{{Dlv - ec_failed_field_count}}`.
- Register `failure_reason` and `failed_fields` as event-scoped **custom dimensions**, and `failed_field_count` as an event-scoped **custom metric** (unit Standard).

Abandonment: a GA4 event tag named `form_abandon` on a Custom Event trigger matching `_form_abandon$` (regex), with:

- `form_id`, `form_name`;
- `abandon_stage` = `{{Dlv - ec_abandon_stage}}`, `abandon_field` = `{{Dlv - ec_abandon_field}}`, `abandon_trigger` = `{{Dlv - ec_detection_method}}`, `form_step` = `{{Dlv - ec_form_step}}` (multi-page forms);
- `fields_completed`, `fields_total`, `time_on_form` (from `ec_time_on_form_s`).

Register `abandon_stage`, `abandon_field`, `abandon_trigger` (and `form_step`) as event-scoped custom dimensions, and `fields_completed`, `fields_total`, `time_on_form` as event-scoped custom metrics (unit Standard; `time_on_form` can use Seconds). Then build a funnel exploration from `ec_form_interaction` (form start) to `form_abandon` / `form_submit`, broken down by `abandon_field` and `abandon_stage`.

Don't send `form_url` (GA4 has `page_location`), and drop `form_platform` on single-platform sites.

## Detection methods

Every success/failure event includes `ec_detection_method`.

Common success methods:

```text
confirmation_event
confirmation_class
stored_recall
```

Common failure methods:

```text
validation_message
```

Abandon methods:

```text
pagehide
visibility_hidden
spa_route
```

Gravity Forms is a strong candidate for confirmation-render tracking because it exposes native confirmation events and predictable confirmation wrappers.

## Debugging

Console logs are disabled by default for privacy. To enable logs in GTM Preview:

```html
<script>
window.EC_FORM_CONFIG = {
  debug: true
};
</script>
```

You can also inspect tracker status in the browser console:

```js
window._ecFormTrackers.gf.status()
```

The status output includes:

- Tracker version
- Consent state
- Forms attached
- Last event
- Secure-context status
- Runtime config

## Browser support

The script uses standard browser APIs including:

- `MutationObserver`
- `localStorage`
- `sessionStorage`
- `history.pushState`
- `jQuery` event hooks when jQuery is present
- `crypto.randomUUID` with fallback
- `crypto.subtle` for optional hashing

Optional SHA-256 hashing requires a secure context, usually HTTPS.

## Known limitations

- Gravity Forms and WordPress plugins can customize markup. Always test on a real page before publishing GTM changes.
- Cross-domain redirects cannot use same-domain stored recall after navigation.
- Bot or spam submissions that pass Gravity Forms validation can still count as success.
- Phone normalization is best-effort. Raw values remain in `ec_raw_fields` when PII consent allows them.
- Multi-form pages are scoped by Gravity Forms form ID where available, with a one-form generic wrapper fallback.
- When an AJAX form re-renders with server-side validation errors, no `gf_form_failed` is pushed (unchanged from v3.3.0). Abandonment still records the `after_error` stage.

## Suggested GTM triggers

Success trigger:

```text
Event name equals gf_form_success
```

Failure trigger:

```text
Event name equals gf_form_failed
```

Abandon trigger (its own GA4 `form_abandon` tag):

```text
Event name matches RegEx _form_abandon$
```

Interaction trigger:

```text
Event name equals ec_form_interaction
```

Diagnostic trigger:

```text
Event name equals ec_tracker_initialized
```

## Version

Current version: `3.5.0`

## License

MIT. See `LICENSE`.

## Author

Iyke Abel  
LinkedIn: https://www.linkedin.com/in/iykeabel/  
Email: abeliyke05@gmail.com
