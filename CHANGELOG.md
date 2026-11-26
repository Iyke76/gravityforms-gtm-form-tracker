# Changelog

## 3.5.0 (2026-11-26)

Privacy and data minimisation (the v3.4.0 changes) plus form abandonment, in one release. Every change is a config option: `payload: 'full'`, `failedPayload: 'full'`, `piiMode: 'plain'` and `trackAbandon: false` reproduce the v3.3.0 payload apart from the fixes marked *.

### Privacy and payload

- `event` is the first key in every push.
- `piiMode`: `'plain'` (default, as before), `'hashed'` (email and phone only as SHA-256) or `'none'` (no contact data). `'hashed'` waits up to `hashWaitMs` (300 ms) for the hash so an instant submit still gets it.
- `rawFields` accepts an array as an allowlist. In hashed or none mode without one, `ec_raw_fields` keeps only select, radio and checkbox values.
- `formNameField`: a Hidden field (CSS class `form_name`, or its input name) pushed as `form_name`. Skipped by the contact-field classifier.
- `failedPayload: 'minimal'` (default): `gf_form_failed` carries no contact data, hashes or raw fields. Field names and the count stay.
- `payload: 'lean'` (default): no `ec_attribution` object, first-touch detail, `ec_time_to_submit_ms` or `form_url`, debug-only keys only with `debug`, blanks removed.
- Stale keys from an earlier push are cleared before the next push, so one form's data never leaks into another event.
- In hashed and none mode the copy kept for thank-you pages has no plain contact data and lives in sessionStorage only.
- Heartbeat adds `ec_pii_mode` and `ec_abandon`; `status()` lists the new options and per-form abandon state.
- \* API key, secret, access/auth token, CSRF and nonce fields are always stripped. v3.3.0 let an `api_key` field into `ec_raw_fields` and joined it onto `ec_name` (or used it as `ec_name` on a form with no name field).
- \* With PII consent denied, all contact keys are stripped. v3.3.0 still sent `ec_postal_code` and the other address keys.

### Form abandonment

- New event `gf_form_abandon`: a started form left without submitting, on `pagehide`, the tab going hidden or an SPA route change. One per form per page load.
- Payload: form metadata, `ec_abandon_stage` (`before_submit` or `after_error`), `ec_abandon_field`, `ec_fields_completed`, `ec_fields_total`, `ec_form_step` (multi-page forms), `ec_time_on_form_s`. Never field values, contact keys, hashes or raw fields, in any mode.
- Follows the Gravity Forms form number across AJAX re-renders (validation errors, page changes). Next/Previous clicks and submits that are still in flight never count as abandons.
- A success after the visitor came back to a hidden tab carries `ec_resumed: true`.
- `trackAbandon` (default `true`) and `abandonOnHidden` (default `true`).

### Fixes

- \* K1: a success never borrows another form's stored data. In v3.3.0 four paths could: the `gform_confirmation_loaded` handler (a confirmation for one form pushed a success for every form on the page, using the last stored submission), the confirmation render observer (read the last stored submission first, and any form's confirmation retired every other form, so a second form's real success was lost), the postback confirmation page (credited the last stored form whatever form confirmed) and the thank-you page (recalled a form that was only typed into, never submitted).
- \* K2: `form_url` no longer reaches the dataLayer in lean mode. It stays in `payload: 'full'`.

## 3.3.0 (2026-11-06)

Rebuilt from the universal tracker v3.3.0. Existing event names and keys are unchanged, so current triggers and variables keep working.

- `ec_first_name`, `ec_last_name`, `ec_company`, `ec_street`, `ec_city`, `ec_region`, `ec_postal_code` and `ec_country`
- Phone validation to E.164, with `defaultCountryCode` and `countryCodeField`
- Consent queue: events raised while analytics consent is denied replay when it is granted
- First and last touch attribution with click IDs
- Failure diagnostics: `ec_submit_attempt`, `ec_time_to_submit_s`, `ec_fast_submit`, `ec_failed_fields`
- `strictSuccess` to drop inferred successes
- `rawFields` and `redactFields`. The GTM file now ships `rawFields: false`, so `ec_raw_fields` is off unless you turn it on
- Password, card, SSN and IBAN style field names are always stripped

## 3.0.1

Initial standalone Gravity Forms release prepared for public GitHub publishing.

### Included

- Gravity Forms-only confirmation-render-based tracking.
- `gf_form_success` and `gf_form_failed` dataLayer events.
- First-interaction event: `ec_form_interaction`.
- Load diagnostic event: `ec_tracker_initialized`.
- Consent detection for Google Consent Mode v2, OneTrust, and Cookiebot.
- `window.EC_FORM_CONSENT_CHECK` override hook.
- PII redaction when PII consent is denied.
- Optional SHA-256 hashing for normalized email and phone values.
- Duplicate success suppression using a 5-minute session dedupe ledger.
- SPA route-change rescanning.
- GTM Custom HTML paste-ready file and raw JavaScript file.

### Privacy hardening

- Debug logging defaults to disabled.
- Dedupe runs after PII redaction.
- Analytics-denied consent prevents event firing and storage writes.
- PII-denied consent redacts both stored and in-memory recall data.

### Fixed before public release

- Corrected heartbeat tracker version reference for the Gravity Forms standalone.
- Removed `<script>` wrapper from the raw `.js` file.
- Added jQuery listener support for `gform_confirmation_loaded`.
- Scoped confirmation-wrapper detection by form ID where available to reduce multi-form false positives.
