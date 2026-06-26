# Google Analytics + GDPR Consent Design

**Date:** 2026-06-26  
**Status:** Approved

## Overview

Add Google Analytics 4 (GA4) event tracking to GrowFocus.app with a GDPR-compliant consent-first flow. GA never loads until the user explicitly accepts. A custom consent banner and a privacy policy page are included.

---

## 1. Consent Flow & GA Loading

A consent check runs on every page load before any GA code executes:

1. Read `localStorage.getItem('ga_consent')`.
2. If `'granted'` → dynamically inject the GA4 `<script>` tag and initialize `gtag`.
3. If `'denied'` → do nothing. GA never loads this session or any future session.
4. If `null` (first visit) → show the consent banner.

**Banner actions:**
- **Accept** → save `'granted'` to `localStorage`, dynamically load GA, dismiss banner.
- **Decline** → save `'denied'` to `localStorage`, dismiss banner. GA never loads.

GA is loaded via a dynamically created `<script>` tag — never a static tag in the HTML — so it is technically impossible for GA to execute before consent is granted.

The user can reset their choice by clearing `localStorage` (documented in the privacy policy).

---

## 2. Consent Banner UI

A bottom-center pill bar with the app's existing glassmorphism design language:

- **Background:** `rgba(20,26,16,.6)` with `backdrop-filter: blur(24px) saturate(1.3)`
- **Border:** `1px solid rgba(176,198,140,.2)`, `border-radius: 999px`
- **Font:** `Nunito Sans`, same sizes as existing pills
- **Copy:** _"We use analytics to understand how GrowFocus is used. No ads, no selling data."_
- **Privacy policy link** → opens `privacy.html`
- **Decline button** → ghost style (transparent background, muted text)
- **Accept button** → green fill (`#8fae5e`, matching the Start timer button)

The banner sits above all UI (`z-index: 9999`) and fades out smoothly on either choice. It never reappears once a choice is saved.

---

## 3. GA4 Setup

A GA4 property must be created at [analytics.google.com](https://analytics.google.com):

1. Create a new property → Web stream → enter the site URL.
2. Measurement ID: **`G-Z2PHHVFF39`** (already created).

The GA snippet is the standard `gtag.js` async load, injected only after consent:

```js
const GA_ID = 'G-Z2PHHVFF39';

function loadGA() {
  const s = document.createElement('script');
  s.async = true;
  s.src = `https://www.googletagmanager.com/gtag/js?id=${GA_ID}`;
  document.head.appendChild(s);
  window.dataLayer = window.dataLayer || [];
  function gtag(){ dataLayer.push(arguments); }
  window.gtag = gtag;
  gtag('js', new Date());
  gtag('config', GA_ID);
}
```

---

## 4. Events to Track

All events are fired via `gtag('event', name)`. No PII or user IDs are sent.

| Event name | Trigger |
|---|---|
| `page_view` | Automatic on GA load |
| `timer_start` | User clicks Start on the Pomodoro |
| `timer_pause` | User clicks Pause mid-session |
| `focus_session_complete` | A full focus interval completes (not skipped) |
| `break_complete` | A short or long break completes |
| `music_opened` | User opens the music player panel |
| `task_added` | User adds a task to the session list |

Events hook into existing `onClick` handlers in the reactive component system. Each call is guarded by a check that `window.gtag` exists (i.e., consent was granted and GA loaded).

---

## 5. Privacy Policy Page (`privacy.html`)

A standalone `privacy.html` in the repo root, styled to match the app (same fonts, dark background, green accent).

**Sections:**
- What data is collected (anonymized usage events, approx. location, device/browser — no names or emails)
- Why it is collected (product improvement)
- Legal basis: consent, Article 6(1)(a) GDPR
- Data processor: Google Analytics (link to Google's privacy policy)
- Data retention: per GA4 defaults (14 months event data)
- User rights: withdraw consent (clear `localStorage`), request data deletion (Google's deletion tools)
- Contact for data requests: giffcik.pl@gmail.com
- Last updated: 2026-06-26

---

## 6. Files Changed

| File | Change |
|---|---|
| `index.html` | Add consent banner HTML + consent/GA JS logic + event calls in handlers |
| `privacy.html` | New file — privacy policy page |

No build tooling changes. No new dependencies.

---

## Out of Scope

- Server-side analytics
- Cookie preference center with granular toggles
- Consent logging/audit trail (not required for single-operator apps at this scale)
- Automated GDPR data export/deletion workflows
