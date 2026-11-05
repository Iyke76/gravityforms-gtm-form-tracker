# Gravity Forms GTM Form Tracker

Confirmation-render-based form tracking for Gravity Forms in Google Tag Manager.

This standalone tracker fires dataLayer events when Gravity Forms produces an observable confirmation, validation failure, or postback confirmation page. It is built for GTM, GA4, server-side GTM, Meta CAPI, and Google Enhanced Conversions workflows where button-click tracking is too noisy.

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

### Diagnostic events

`ec_form_interaction` fires once per form after the first real user input. It contains metadata only, not PII.

`ec_tracker_initialized` fires once per page load so you can confirm the script loaded even before a test submission.

## Payload example

```js
{
  event: "gf_form_success",
  ec_event_id: "uuid-v4",
  ec_detection_method: "confirmation_event",
  ec_tracker_version: "3.3.0",
  ec_name: "Jane Doe",
  ec_email: "jane@example.com",
  ec_phone: "+15551234567",
  form_url: "https://example.com/contact",
  form_cta: "Submit",
  form_platform: "gravityforms",
  form_position: "top",
  form_index: 1,
  form_id: "gform_3",
  ec_raw_fields: {
    "input_1": {
      value: "jane@example.com",
      type: "email",
      _bucket: "email"
    }
  }
}
```

When PII consent is denied, PII fields are removed:

```js
{
  event: "gf_form_success",
  ec_event_id: "uuid-v4",
  ec_detection_method: "confirmation_event",
  ec_tracker_version: "3.3.0",
  form_url: "https://example.com/contact",
  form_platform: "gravityforms",
  form_position: "top",
  form_index: 1,
  form_id: "gform_3",
  ec_pii_redacted: true
}
```

## Quick start in GTM

1. Open Google Tag Manager.
2. Create a new **Custom HTML** tag.
3. Paste the contents of `gravityforms-form-tracker.gtm.html`.
4. Trigger it on **All Pages - Page View** or **Consent Initialization** if your consent setup needs the tracker available early.
5. Preview the container.
6. Submit a real Gravity Forms test form.
7. Confirm `gf_form_success` or `gf_form_failed` appears in the dataLayer.
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
| `hashPii` | `false` | Adds `ec_email_sha256` and `ec_phone_sha256` when possible. |
| `dedupe` | `true` | Suppresses duplicate success events for the same form/email/phone within 5 minutes. |
| `allowRecoverySuccess` | `true` | Kept for config consistency across EC standalone trackers. Gravity Forms success can fire after a prior failure when a confirmation appears. |
| `submitWindowMs` | `15000` | Kept for config consistency across EC standalone trackers. Gravity Forms success/failure is confirmation/validation-render based. |
| `ttlMs` | `600000` | Freshness window for stored form data used by same-domain thank-you recall. |
| `defaultCountryCode` | `"1"` | Country code used for best-effort phone normalization. Set `""` to disable. |
| `extraThankYouPatterns` | `[]` | Additional URL substrings that identify thank-you pages. |

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
| `pii: false` | Events can fire, but raw PII and hashes are removed. Storage and recall data are redacted. |

If no CMP or override is detected, the tracker defaults to full consent. You are responsible for matching this behavior to your site and legal requirements.

## PII warning

The success and failure payloads can include:

- `ec_email`
- `ec_phone`
- `ec_name`
- `ec_raw_fields`

Do not send raw PII to GA4. GA4 terms prohibit PII in event parameters.

Recommended approaches:

- Use server-side GTM and hash values before forwarding to ad platforms.
- Enable `hashPii: true` and send only hashed fields where appropriate.
- Explicitly exclude raw PII fields from GA4 Event tags.
- Use raw fields only for first-party CRM workflows where you are allowed to process the data.

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

## Suggested GTM triggers

Success trigger:

```text
Event name equals gf_form_success
```

Failure trigger:

```text
Event name equals gf_form_failed
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

Current version: `3.3.0`

## License

MIT. See `LICENSE`.

## Author

Iyke Abel  
LinkedIn: https://www.linkedin.com/in/iykeabel/  
Email: abeliyke05@gmail.com
