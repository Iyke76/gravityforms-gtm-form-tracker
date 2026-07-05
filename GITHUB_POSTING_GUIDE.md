# GitHub Posting Guide

Use this guide when creating the public GitHub repository.

## Suggested repository name

```text
gravityforms-gtm-form-tracker
```

Alternative names:

```text
gravity-forms-form-tracker
gravityforms-form-tracker
gravityforms-dataLayer-form-tracker
gravityforms-confirmation-form-tracker
```

## Repository description

```text
Confirmation-render-based Gravity Forms tracking for Google Tag Manager. Fires success/failure events to dataLayer with consent gating, PII redaction, dedupe, optional hashing, and SPA support.
```

## Suggested topics

```text
google-tag-manager
gtm
dataLayer
ga4
gravity-forms
wordpress
form-tracking
conversion-tracking
server-side-gtm
meta-capi
enhanced-conversions
analytics
javascript
```

## Files to publish

```text
README.md
LICENSE
CHANGELOG.md
.gitignore
gravityforms-form-tracker.js
gravityforms-form-tracker.gtm.html
GITHUB_POSTING_GUIDE.md
```

## First commit message

```text
Initial Gravity Forms GTM form tracker release
```

## Release title

```text
Gravity Forms GTM Form Tracker v3.0.1
```

## Release notes

```text
Initial standalone Gravity Forms release.

Includes:
- Confirmation-render-based success tracking
- Gravity Forms validation failure tracking
- dataLayer events for success, failure, interaction, and initialization
- Consent detection for Google Consent Mode v2, OneTrust, and Cookiebot
- PII redaction when consent is denied
- Optional SHA-256 hashing for email and phone
- Duplicate success suppression
- SPA route-change support
- Raw JS and GTM Custom HTML versions
```

## Publish from local terminal

If the GitHub repo already exists and is empty:

```bash
cd "/Users/iyke-abel/Documents/Form Trackers/gravityforms-gtm-form-tracker"
git init
git add README.md LICENSE CHANGELOG.md .gitignore gravityforms-form-tracker.js gravityforms-form-tracker.gtm.html GITHUB_POSTING_GUIDE.md
git commit -m "Initial Gravity Forms GTM form tracker release"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/gravityforms-gtm-form-tracker.git
git push -u origin main
```

If this folder is already a Git repo, skip `git init`.

## GitHub UI checklist

1. Create a new public repository.
2. Use the suggested description above.
3. Do not initialize with README, license, or gitignore if you are pushing these files from local.
4. Push the local files.
5. Add the suggested topics in the repo sidebar.
6. Confirm the README renders correctly.
7. Open `gravityforms-form-tracker.js` and confirm it does not include `<script>` tags.
8. Open `gravityforms-form-tracker.gtm.html` and confirm it does include `<script>` tags.
9. Create release `v3.0.1`.
10. Paste the release notes above.

## Pre-publish QA

Before announcing it publicly:

1. Install `gravityforms-form-tracker.gtm.html` in a GTM Preview container.
2. Submit a real Gravity Forms form with valid data.
3. Confirm `ec_tracker_initialized` fires once.
4. Confirm `ec_form_interaction` fires once after input.
5. Confirm `gf_form_success` fires after success.
6. Submit invalid data and confirm `gf_form_failed` fires.
7. Test AJAX confirmation and non-AJAX/postback confirmation if both are used.
8. Test a page with multiple Gravity Forms if relevant.
9. Test `debug:true` only in preview.
10. Test `pii:false` through `EC_FORM_CONSENT_CHECK` and confirm raw PII is removed.
11. Confirm raw PII is not mapped into GA4 event parameters.
12. Check `window._ecFormTrackers.gf.status()` in the browser console.

## Important README promise

Keep the README privacy wording aligned with the code. If you change dedupe, storage, hashing, or consent behavior later, update the README in the same commit.
