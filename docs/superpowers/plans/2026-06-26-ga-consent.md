# Google Analytics + GDPR Consent Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add GA4 event tracking with a GDPR-compliant consent-first banner and a privacy policy page to GrowFocus.app.

**Architecture:** A plain `<script>` block after the `</x-dc>` element handles all consent logic and GA loading — no build tools, no framework, no new dependencies. Event hooks call a global `window.trackEvent(name)` guarded by existence check, so they silently no-op when GA has not been loaded (user declined or first visit). The consent banner is a plain DOM element rendered outside the reactive component system.

**Tech Stack:** Vanilla HTML/JS, `localStorage` for consent persistence, GA4 (`gtag.js`) loaded dynamically on consent.

## Global Constraints

- No build tooling — changes go directly in `index.html` and a new `privacy.html`.
- No npm packages or external consent libraries.
- Measurement ID: `G-Z2PHHVFF39`.
- `localStorage` key for consent: `ga_consent` — values `'granted'` or `'denied'`.
- Global event helper: `window.trackEvent(name)` — always guard call sites with `window.trackEvent &&`.
- All UI must match the app's aesthetic: `rgba(20,26,16,.6)` glass background, `blur(24px)`, border `rgba(176,198,140,.2)`, font `'Nunito Sans'`, accent green `#8fae5e`.
- Testing is manual in a browser — open `index.html` directly or via a local server.

---

### Task 1: Consent Banner + GA Loading Infrastructure

**Files:**
- Modify: `index.html:641` (just before `</body>`)

**Interfaces:**
- Produces: `window.trackEvent(name: string): void` — fires a GA event if consent is granted and GA is loaded; silently no-ops otherwise.
- Produces: `localStorage.getItem('ga_consent')` → `'granted' | 'denied' | null`

- [ ] **Step 1: Add the consent banner HTML and infrastructure script**

  Insert the following block immediately before the closing `</body>` tag at `index.html:641` (between `</x-dc>` and `</body>`):

  ```html
  <!-- ===== GDPR consent banner ===== -->
  <div id="consent-banner" style="display:none; position:fixed; bottom:28px; left:50%; transform:translateX(-50%); z-index:9999; align-items:center; gap:16px; padding:13px 18px; border-radius:999px; background:rgba(20,26,16,.75); backdrop-filter:blur(24px) saturate(1.3); -webkit-backdrop-filter:blur(24px) saturate(1.3); border:1px solid rgba(176,198,140,.2); box-shadow:0 12px 30px -12px rgba(0,0,0,.6); font-family:'Nunito Sans',sans-serif; font-size:13px; color:rgba(238,232,216,.82); white-space:nowrap; transition:opacity .35s ease;">
    <span>We use analytics to understand how GrowFocus is used. No ads, no selling data. <a href="privacy.html" style="color:#a9c97f; text-decoration:none;">Privacy policy</a></span>
    <button id="consent-decline" style="cursor:pointer; padding:7px 14px; border-radius:999px; border:1px solid rgba(176,198,140,.25); background:transparent; font-family:'Nunito Sans',sans-serif; font-size:13px; font-weight:600; color:rgba(238,232,216,.65);">Decline</button>
    <button id="consent-accept" style="cursor:pointer; padding:7px 16px; border-radius:999px; border:none; background:#8fae5e; font-family:'Nunito Sans',sans-serif; font-size:13px; font-weight:700; color:#16200c;">Accept</button>
  </div>

  <script>
    (function () {
      var GA_ID = 'G-Z2PHHVFF39';

      function loadGA() {
        var s = document.createElement('script');
        s.async = true;
        s.src = 'https://www.googletagmanager.com/gtag/js?id=' + GA_ID;
        document.head.appendChild(s);
        window.dataLayer = window.dataLayer || [];
        window.gtag = function () { dataLayer.push(arguments); };
        gtag('js', new Date());
        gtag('config', GA_ID);
      }

      window.trackEvent = function (name) {
        if (window.gtag) gtag('event', name);
      };

      var banner = document.getElementById('consent-banner');
      var consent = null;
      try { consent = localStorage.getItem('ga_consent'); } catch (e) {}

      if (consent === 'granted') {
        loadGA();
      } else if (consent === null) {
        banner.style.display = 'flex';
      }

      document.getElementById('consent-accept').addEventListener('click', function () {
        try { localStorage.setItem('ga_consent', 'granted'); } catch (e) {}
        banner.style.opacity = '0';
        setTimeout(function () { banner.style.display = 'none'; }, 350);
        loadGA();
      });

      document.getElementById('consent-decline').addEventListener('click', function () {
        try { localStorage.setItem('ga_consent', 'denied'); } catch (e) {}
        banner.style.opacity = '0';
        setTimeout(function () { banner.style.display = 'none'; }, 350);
      });
    })();
  </script>
  ```

- [ ] **Step 2: Manual verification — first visit (consent null)**

  1. Open `index.html` in a browser (or `http://localhost:<port>`).
  2. Open DevTools → Application → Local Storage → clear `ga_consent` key if present.
  3. Hard-refresh the page.
  4. **Expected:** Consent banner appears at bottom-center, styled as a dark pill with Accept (green) and Decline buttons.
  5. Check DevTools → Network — confirm no `googletagmanager.com` request has fired.

- [ ] **Step 3: Manual verification — Accept flow**

  1. Click **Accept**.
  2. **Expected:** Banner fades out and disappears.
  3. DevTools → Network → confirm `gtag/js?id=G-Z2PHHVFF39` request fires.
  4. DevTools → Application → Local Storage → confirm `ga_consent = 'granted'`.
  5. Hard-refresh page → banner must NOT reappear, GA must load automatically (check Network tab).

- [ ] **Step 4: Manual verification — Decline flow**

  1. Clear `ga_consent` from Local Storage, hard-refresh.
  2. Click **Decline**.
  3. **Expected:** Banner fades out. No `googletagmanager.com` request. `ga_consent = 'denied'` in storage.
  4. Hard-refresh → banner must NOT reappear, no GA load.

- [ ] **Step 5: Commit**

  ```bash
  git add index.html
  git commit -m "feat: add GDPR consent banner and GA4 loading infrastructure"
  ```

---

### Task 2: GA Event Hooks in Component

**Files:**
- Modify: `index.html:279-287` (`addTask` method)
- Modify: `index.html:376-410` (`tick` method — phase-complete branch)
- Modify: `index.html:587` (`toggleTimer` handler in `renderVals`)
- Modify: `index.html:623` (`toggleMusicPanel` handler in `renderVals`)

**Interfaces:**
- Consumes: `window.trackEvent(name)` from Task 1

- [ ] **Step 6: Hook `timer_start` and `timer_pause` into `toggleTimer`**

  Find the `toggleTimer` line in `renderVals()` (around line 587). Replace:

  ```js
  toggleTimer: (e) => { e && e.stopPropagation(); if (!this.state.running) this.chime && (this._ac && this._ac.state === 'suspended' && this._ac.resume()); this.setState((p) => ({ running: !p.running }), () => this.updateTitle()); },
  ```

  With:

  ```js
  toggleTimer: (e) => {
    e && e.stopPropagation();
    if (!this.state.running) {
      if (this._ac && this._ac.state === 'suspended') this._ac.resume();
      window.trackEvent && window.trackEvent('timer_start');
    } else {
      window.trackEvent && window.trackEvent('timer_pause');
    }
    this.setState((p) => ({ running: !p.running }), () => this.updateTitle());
  },
  ```

- [ ] **Step 7: Hook `focus_session_complete` and `break_complete` into `tick`**

  Find the phase-complete `else` branch inside `tick()` (around line 393). Replace:

  ```js
                  const set = s.settings, cycle = set.cycle;
                  if (set.sound) setTimeout(() => this.chime(), 0);
                  next.running = !!set.autoStart;
  ```

  With:

  ```js
                  const set = s.settings, cycle = set.cycle;
                  if (set.sound) setTimeout(() => this.chime(), 0);
                  if (s.mode === 'focus') {
                    setTimeout(() => { window.trackEvent && window.trackEvent('focus_session_complete'); }, 0);
                  } else {
                    setTimeout(() => { window.trackEvent && window.trackEvent('break_complete'); }, 0);
                  }
                  next.running = !!set.autoStart;
  ```

- [ ] **Step 8: Hook `music_opened` into `toggleMusicPanel`**

  Find `toggleMusicPanel` in `renderVals()` (around line 623). Replace:

  ```js
  toggleMusicPanel: (e) => { e && e.stopPropagation(); this.setState((p) => ({ musicOpen: !p.musicOpen }), () => { if (this.state.musicOpen) this.initYt(); }); },
  ```

  With:

  ```js
  toggleMusicPanel: (e) => {
    e && e.stopPropagation();
    this.setState((p) => {
      if (!p.musicOpen) window.trackEvent && window.trackEvent('music_opened');
      return { musicOpen: !p.musicOpen };
    }, () => { if (this.state.musicOpen) this.initYt(); });
  },
  ```

- [ ] **Step 9: Hook `task_added` into `addTask`**

  Find the `addTask()` method (around line 279). Replace:

  ```js
  addTask() {
    const text = (this.state.newTask || '').trim();
    if (!text) return;
    this.setState((p) => {
  ```

  With:

  ```js
  addTask() {
    const text = (this.state.newTask || '').trim();
    if (!text) return;
    window.trackEvent && window.trackEvent('task_added');
    this.setState((p) => {
  ```

- [ ] **Step 10: Manual verification — event tracking**

  1. Accept consent (or set `localStorage.setItem('ga_consent','granted')` then refresh).
  2. Open DevTools → Console. Run: `window.gtag` — must not be `undefined`.
  3. Add a spy: in Console run `const orig = window.gtag; window.gtag = function(...a){ console.log('gtag', ...a); orig(...a); }`.
  4. Click **Start session** → Console must log `gtag event timer_start`.
  5. Click **Pause** → Console must log `gtag event timer_pause`.
  6. Open the Music player → Console must log `gtag event music_opened`.
  7. Add a task → Console must log `gtag event task_added`.
  8. (Optional) Speed-test phase completion: in Console run `this.state` is not accessible directly — instead use the app normally through a cycle; or verify in GA4 DebugView (enable by adding `&debug_mode=1` to the URL if needed via gtag config).

- [ ] **Step 11: Commit**

  ```bash
  git add index.html
  git commit -m "feat: add GA4 event hooks for timer, music, and task interactions"
  ```

---

### Task 3: Privacy Policy Page

**Files:**
- Create: `privacy.html`

**Interfaces:**
- Linked from: consent banner `<a href="privacy.html">` (Task 1)

- [ ] **Step 12: Create `privacy.html`**

  Create the file `privacy.html` in the repo root with the following content:

  ```html
  <!DOCTYPE html>
  <html lang="en">
  <head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Privacy Policy — GrowFocus</title>
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  <link href="https://fonts.googleapis.com/css2?family=Newsreader:ital,opsz,wght@0,6..72,400;0,6..72,500;1,6..72,400&family=Nunito+Sans:opsz,wght@6..12,400;6..12,500;6..12,600&display=swap" rel="stylesheet">
  <style>
    *, *::before, *::after { box-sizing: border-box; }
    html, body { margin: 0; padding: 0; background: #0a0c07; color: #ece7d6; font-family: 'Nunito Sans', sans-serif; font-size: 16px; line-height: 1.7; }
    body { padding: 60px 24px 80px; }
    .wrap { max-width: 680px; margin: 0 auto; }
    a { color: #a9c97f; text-decoration: none; }
    a:hover { text-decoration: underline; }
    h1 { font-family: 'Newsreader', serif; font-weight: 500; font-size: 42px; line-height: 1.15; color: #f3eedf; margin: 0 0 8px; }
    .meta { font-size: 13px; color: rgba(238,232,216,.45); margin-bottom: 48px; }
    h2 { font-family: 'Newsreader', serif; font-weight: 500; font-size: 22px; color: #f3eedf; margin: 40px 0 10px; }
    p { margin: 0 0 16px; color: rgba(238,232,216,.82); }
    ul { margin: 0 0 16px; padding-left: 20px; color: rgba(238,232,216,.82); }
    li { margin-bottom: 6px; }
    .back { display: inline-block; margin-bottom: 40px; font-size: 13px; font-weight: 600; color: #a9c97f; }
    .reset-box { margin-top: 12px; padding: 14px 18px; border-radius: 14px; background: rgba(20,26,16,.6); border: 1px solid rgba(176,198,140,.2); font-size: 14px; }
    code { font-size: 13px; background: rgba(255,255,255,.08); padding: 2px 6px; border-radius: 5px; font-family: monospace; }
  </style>
  </head>
  <body>
  <div class="wrap">
    <a href="index.html" class="back">← Back to GrowFocus</a>
    <h1>Privacy Policy</h1>
    <p class="meta">Last updated: 26 June 2026</p>

    <h2>What we collect</h2>
    <p>GrowFocus uses Google Analytics 4 to collect anonymised usage data — but <strong>only if you consent</strong>. No data is collected before you click Accept on the consent banner, and none is collected if you click Decline.</p>
    <p>When you consent, Google Analytics may collect:</p>
    <ul>
      <li>Pages visited and time spent</li>
      <li>Interactions (timer start/pause, focus sessions completed, music opened, tasks added)</li>
      <li>Approximate location (country and city — derived from IP address, not stored precisely)</li>
      <li>Device type and browser</li>
    </ul>
    <p>We do <strong>not</strong> collect your name, email address, or any account information. GrowFocus has no user accounts.</p>

    <h2>Why we collect it</h2>
    <p>Analytics data helps us understand how the app is used so we can improve it. We do not use this data for advertising, and we do not sell it to third parties.</p>

    <h2>Legal basis</h2>
    <p>We process analytics data on the basis of your <strong>consent</strong> (Article 6(1)(a) of the GDPR). You may withdraw consent at any time — see below.</p>

    <h2>Who processes the data</h2>
    <p>Analytics data is processed by <strong>Google LLC</strong> via Google Analytics 4. Google acts as a data processor under a Data Processing Agreement. You can read Google's privacy policy at <a href="https://policies.google.com/privacy" target="_blank" rel="noopener">policies.google.com/privacy</a>.</p>
    <p>Data may be transferred to servers in the United States. Google is certified under the EU–US Data Privacy Framework.</p>

    <h2>Data retention</h2>
    <p>Google Analytics retains event data for 14 months by default. After that period it is automatically deleted.</p>

    <h2>Your rights</h2>
    <p>Under the GDPR you have the right to access, rectify, or erase your personal data, and to withdraw consent at any time.</p>
    <p><strong>To withdraw consent and stop analytics collection:</strong></p>
    <div class="reset-box">
      Open your browser's developer tools (F12), go to <strong>Application → Local Storage</strong>, and delete the key <code>ga_consent</code>. Then reload the page — the consent banner will reappear and you can choose Decline.
    </div>
    <p style="margin-top:16px;"><strong>To request deletion of your analytics data:</strong> because Google Analytics does not link data to personally identifiable information in GrowFocus, we cannot identify your specific records. You can opt out of Google Analytics globally by installing the <a href="https://tools.google.com/dlpage/gaoptout" target="_blank" rel="noopener">Google Analytics Opt-out Browser Add-on</a>.</p>

    <h2>Cookies</h2>
    <p>Google Analytics sets cookies (<code>_ga</code>, <code>_ga_*</code>) to distinguish users and sessions. These cookies are only set after you consent. They expire after 2 years (<code>_ga</code>) and 24 hours (<code>_ga_*</code>).</p>

    <h2>Contact</h2>
    <p>For any questions about this privacy policy or to exercise your data rights, contact: <a href="mailto:giffcik.pl@gmail.com">giffcik.pl@gmail.com</a></p>
  </div>
  </body>
  </html>
  ```

- [ ] **Step 13: Manual verification — privacy policy**

  1. Open `privacy.html` in the browser.
  2. Verify it renders correctly: dark background, correct fonts, green accent links, all sections visible.
  3. Click "← Back to GrowFocus" — must navigate to `index.html`.
  4. From `index.html`, click "Privacy policy" in the consent banner — must open `privacy.html`.

- [ ] **Step 14: Commit**

  ```bash
  git add privacy.html
  git commit -m "feat: add GDPR privacy policy page"
  ```
