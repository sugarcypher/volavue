# Volavue — Launch Runbook

> **Status as of 2026-05-26: LAUNCHED.** `https://volavue.fyi` is live with TLS,
> security headers, enforced CSP, Cal.com booking, and atomic Cal-Stripe payment
> at $349 USD per session. The end-to-end flow was verified with a real $1 test
> charge (then refunded; the test booking was cancelled). The remainder of this
> document is now reference material — useful if the site goes down (rollback
> via Cloudflare Pages), if anything moves (CDN, registrar), or if the legal
> pages need updating with counsel.
>
> Outstanding (non-blocking) items:
> - **Auto-deploy from GitHub.** Currently deploys are pushed manually via
>   `wrangler pages deploy assets --project-name volavue`. One dashboard click
>   (Pages → volavue → Settings → Connect to Git) wires up auto-deploy.
> - **API key rotation.** Both the Cal.com API key and the Stripe restricted
>   key used during launch should be rotated before this `.env.local` file is
>   shared or before machines change hands.

The original launch checklist below is kept verbatim for reference — it is the
order in which the integrations were originally brought up. Step-by-step
useful if you ever need to migrate to a new Cloudflare account, switch
registrars, or rebuild from scratch.

---

## 0. Pre-flight

- [ ] You can log in to:
  - GitHub (account: `sugarcypher`)
  - Cloudflare
  - Porkbun (the registrar where `volavue.fyi` lives)
  - Cal.com (account `next-level-etpkqv`)
  - Stripe
  - The inbox for `volavue@volavue.fyi`
- [ ] You have a password manager and hardware key / authenticator app ready —
  every account below should end this session with 2FA on.

---

## 1. Push to GitHub

The local repo already has the production code committed on `main`. Add the
remote and push:

```sh
cd ~/VolaVue-web
git remote add origin https://github.com/sugarcypher/volavue.git
git push -u origin main
```

If the push prompts for credentials, use a GitHub Personal Access Token (not
your password). Settings → Developer settings → Personal access tokens →
Fine-grained tokens → "Only select repositories" → `volavue` → permission
`Contents: Read and write`.

Then in the GitHub repo settings:

- [ ] **Branch protection** on `main`: require PR review, prohibit force-push.
- [ ] **2FA** on the account (hardware key preferred). Settings → Password
  and authentication.
- [ ] **Vigilant Mode** on. Settings → SSH and GPG keys → "Vigilant mode".
- [ ] **Secret scanning** on (free for public repos).

---

## 2. Cloudflare Pages — connect the repo

1. Cloudflare dashboard → **Workers & Pages** → **Create application** →
   **Pages** → **Connect to Git**.
2. Authorise Cloudflare to read the `sugarcypher/volavue` repo.
3. Project setup:
   - **Project name:** `volavue` (this becomes `volavue.pages.dev`)
   - **Production branch:** `main`
   - **Framework preset:** *None*
   - **Build command:** *(leave empty)*
   - **Build output directory:** `assets`
   - **Root directory:** *(leave empty)*
   - **Environment variables:** none required
4. Click **Save and Deploy**. First build should complete in <1 minute.
5. Open the `*.pages.dev` URL it produced. Click through each page.
   - Confirm fonts load (Fraunces serif headings, Hanken Grotesk body).
   - Open DevTools → Console. If you see CSP violations from
     `Content-Security-Policy-Report-Only`, list them — they need to be added
     to `_headers` before the enforcing flip in §6.

---

## 3. Custom domain — `volavue.fyi`

### In Cloudflare Pages
1. Project → **Custom domains** → **Set up a custom domain**.
2. Enter `volavue.fyi`. Add.
3. Cloudflare will display **either** an A-record target **or** a CNAME target
   (it varies by zone). **Copy the exact value(s) it shows.** Leave this tab
   open.
4. Repeat for `www.volavue.fyi` — Cloudflare expects you to use the apex as the
   canonical and either redirect or alias `www` to it.

### In Porkbun (DNS panel)
1. Porkbun → **DNS** → `volavue.fyi`.
2. Delete any existing A / AAAA / CNAME records for the apex and `www` that
   are not from Cloudflare (parking, default Porkbun records, etc.).
3. Add the records Cloudflare gave you. Typical shape:
   - **Type:** `CNAME` · **Host:** *(empty for apex / `@`)* · **Answer:**
     `volavue.pages.dev` · **TTL:** `600`
     (Porkbun supports CNAME flattening at the apex.)
   - **Type:** `CNAME` · **Host:** `www` · **Answer:** `volavue.pages.dev` ·
     **TTL:** `600`
4. Save. Propagation usually takes 1–10 minutes on Porkbun.

### Back in Cloudflare Pages
1. The custom-domain row goes from "Pending" → "Active" once DNS resolves
   and the cert is issued. Wait for **Active**.
2. Open `https://volavue.fyi/` in a fresh **incognito** window. Confirm:
   - HTTPS padlock present, no certificate warning.
   - All pages reachable (Home, Booking, Intake, Privacy, Terms, Cookies,
     Disclaimer, Accessibility, Contact).
   - `https://volavue.fyi/booking` (no `.html`) redirects to
     `https://volavue.fyi/booking.html` (301). Same for `/intake`,
     `/privacy`, etc. — the `_redirects` file handles these.

---

## 4. Cal.com — finalise the booking event

The site embeds Cal.com event **`next-level-etpkqv/60min`**. Verify the event
is configured to match the site copy:

1. Cal.com → **Event Types** → open the **60 min** event under
   `next-level-etpkqv`.
2. **Title:** `Structured Decision Review`.
3. **Duration:** 60 minutes (or 90, if you prefer — `assets/booking.html`
   advertises "60–90 minutes").
4. **Booking Questions** → add these five questions (exact wording matches
   the intake page so prefill works) and make all of them **required**:
   1. `Briefly describe the decision you are facing.` (long text)
   2. `Who else is directly involved?` (long text)
   3. `What feels most uncertain or unclear right now?` (long text)
   4. `Is there a timeline or deadline influencing this decision?` (text)
   5. `What outcome are you most concerned about?` (long text)
5. **Availability:** set the windows when you can take a session.
   Timezone defaults to the calendar account; verify it matches your real
   timezone.
6. **Calendar connections:** connect a Google / iCal calendar so Cal.com
   blocks already-booked time.
7. **Notifications:** enable booking-confirmed, rescheduled, and cancelled
   emails. Verify the **from**-address is the one you control.
8. **Buffer time & notice:** set a minimum notice (e.g. 24 hours) and a
   buffer between sessions (e.g. 15 min) to prevent back-to-back collisions.
9. Account-level: **2FA on**, retention policy set (delete bookings after
   N days post-session — common default 90).

---

## 5. Stripe — wire up via the Cal.com app

You picked Cal.com's native Stripe integration, so payment lives inside
Cal.com. **Do not paste Stripe keys into this repo** — the site is fully
client-side and there's no place secret keys should live.

1. Stripe Dashboard → confirm the account is **out of test mode**
   (top-right toggle on **Live**). Live mode requires identity verification;
   if you are still in test mode, complete it now.
2. Stripe → 2FA on (hardware key preferred), email alerts on for
   high-value events.
3. Cal.com → **Apps** → **Stripe** → **Install / Connect**.
   Authorise. You're brought back to Cal.com with the Stripe account linked.
4. Cal.com → Event Types → the **60 min** event → **Payments** →
   **Enable Stripe** → set the **session price** (USD or AUD per your
   business preference; you live in AU, so AUD likely).
5. Confirm: in the event settings, Cal.com should now say something like
   "Booking requires payment of $X" and the booker won't be confirmed until
   payment succeeds. **This is what removes the orphan-payment risk.**
6. (Optional, recommended) Stripe → Settings → Email receipts → enable
   "Successful payments". The booker gets a Stripe-branded receipt in
   addition to Cal.com's confirmation.

---

## 6. CSP — flip from Report-Only to enforcing

The site ships `Content-Security-Policy-Report-Only` so Cal.com / Stripe
loading can be observed in a real deploy without blocking anything. After §2
and §3 succeed:

1. Open `https://volavue.fyi/booking` in a real browser.
2. DevTools → **Console**. Accept the cookie banner; Cal.com loads.
3. Walk through to "Pick a time" and trigger the Stripe payment step inside
   the Cal embed (you don't need to complete a charge — just reach the
   payment screen).
4. If you see any messages like *"[Report Only] Refused to load …"*, copy
   the violated directive + origin. Common additions if Cal renews CDNs:
   `cdn.cal.com`, `eu.app.cal.com`, `m.stripe.network`. Add to the
   corresponding directive in `assets/_headers`.
5. When the page loads end-to-end with **zero** Report-Only violations,
   open `assets/_headers` and rename the header:
   `Content-Security-Policy-Report-Only:` → `Content-Security-Policy:`
   Commit, push. Cloudflare Pages will redeploy in ~30 seconds. Reload and
   confirm no console errors.

---

## 7. End-to-end smoke test on production

Do this in **incognito**, on a real device, with a real email you control
that is not the operator email.

- [ ] **Home loads.** Hero, framework diagram, testimonials, footer.
- [ ] **All footer links resolve** (Privacy, Terms, Cookies, Disclaimer,
  Accessibility, Contact).
- [ ] **Intake form:** fill all 5 questions + name + email → Submit.
  - URL on the next page is **clean** — no `?q1=…`. The values went into
    `sessionStorage`, not the URL.
  - Booking page shows your answers in the "Your intake summary" panel.
- [ ] **Cookie banner:** appears on first visit. **Decline** disables the
  calendar and shows the mailto fallback. Reload, **Accept** loads the
  Cal.com embed.
- [ ] **Booking happy path:**
  1. Pick a time.
  2. Cal.com asks for the 5 booking questions — they should arrive
     **prefilled** from your intake.
  3. Stripe payment step appears.
  4. Complete a real charge with a real card (your own). Use the smallest
     test price you're comfortable with for the live test — you can refund
     after.
  5. Confirmation page loads. Confirmation email arrives at your inbox.
  6. Stripe Dashboard → Payments → confirms the charge.
  7. Cal.com Dashboard → Bookings → shows the booking with the 5 answers.
  8. Your connected Google/iCal calendar holds the event in the right
     timezone.
- [ ] **Refund the test charge** in Stripe. Confirm Cal.com sees the
  cancellation (it sometimes requires explicit cancellation in Cal as well —
  do both, decide which is the canonical workflow for your future refunds).
- [ ] **Failure paths:**
  - Card declined → Cal.com does not confirm the booking. Confirm no time
    is held.
  - Attempt to book the **same time you just held** in a second tab → Cal
    rejects it as taken.
  - Reschedule a booking via Cal's email link → calendar updates.
- [ ] **Mobile:** open on phone in incognito. Header, side menu, cookie
  banner, calendar, Stripe step all work.
- [ ] **DevTools console:** no errors on any page.
- [ ] **Network:** no requests to `fonts.googleapis.com` (fonts are
  self-hosted) and no requests to any analytics endpoint we don't own.

---

## 8. External audits

Run once `https://volavue.fyi/` is live and §6 is done:

- [ ] **<https://securityheaders.com>** — target **A** or **A+**. If lower,
  the missing header will be obvious; add it to `assets/_headers`.
- [ ] **<https://observatory.mozilla.org>** — target **A** or higher.
- [ ] **<https://www.ssllabs.com/ssltest/>** — target **A** at minimum.
- [ ] Lighthouse (Chrome DevTools) on Home + Booking — Performance,
  Accessibility, Best Practices, SEO. Target 90+ on each.

---

## 9. Cloudflare account-level hardening

Inside the Cloudflare dashboard, on the `volavue.fyi` zone (which Pages
creates / manages):

- **SSL/TLS** mode: **Full (Strict)**.
- **Always Use HTTPS:** on.
- **Automatic HTTPS Rewrites:** on.
- **Minimum TLS Version:** 1.2 (1.3 if all your visitors support it).
- **HSTS** in Cloudflare's HSTS settings — *only after* the site has been
  live and stable for a few days. Once you set HSTS at the edge with
  `preload`, mistakes become expensive.
- **Bot Fight Mode:** on (free).
- **Browser Integrity Check:** on.
- **Cloudflare Web Analytics:** on (cookieless).

The `_headers` file in this repo already sets `Strict-Transport-Security` at
the Pages layer — that's the floor. Cloudflare's HSTS toggle is the belt and
braces.

---

## 10. Failure-mode dry runs

Before announcing the site:

- **What if Cal.com goes down?** Booking page should still load; cookie
  banner won't matter because there's no calendar to gate. The mailto in
  the footer + the email in the placeholder is the fallback.
- **What if Stripe declines a card?** Cal.com refuses the booking; no
  calendar block, no confirmation email. Re-test once.
- **What if a visitor declines cookies?** They see a clear message and
  the `mailto:volavue@volavue.fyi` — that's the documented fallback.
- **What if you need to take the site down quickly?** Cloudflare Pages →
  Deployments → roll back to a prior deployment (one click). Or pause the
  custom domain → site returns Cloudflare's 522 / your error page until
  you re-add it.

---

## 11. Things this repo does **not** do (and why that's fine)

These items appear on a "production checklist" template but **do not apply
to this site**:

- **Rate limiting / WAF rules on form submits.** No form on this site posts
  to our origin. Cal.com handles its own scheduling abuse defences;
  Stripe handles payment abuse defences. Cloudflare's free WAF + Bot
  Fight Mode is sufficient for the static surface.
- **CSRF tokens.** No state-changing endpoints on `volavue.fyi`.
- **Webhook signature verification.** No webhooks land on our origin. The
  Stripe ↔ Cal.com webhook is verified inside Cal.com, not here.
- **Server-side input validation.** No server; the only inputs touched by
  our code are intake answers rendered with `textContent` (XSS-safe) and
  passed to Cal as runtime config (not interpolated into HTML or script
  source).
- **Secret rotation.** No secrets in the repo. (Verify with
  `grep -rE 'sk_(live|test)|pk_live' assets/` — should return nothing.)
- **Database backups / DR plan.** No database.
- **Container scanning, SBOM, dependency audit.** No package.json, no
  dependencies, no supply chain.

If any of these change later (you add a Cloudflare Worker, a Pages Function,
an analytics endpoint, a contact form that posts somewhere), they become
applicable. Until then, they are N/A.

---

## 12. Recovery contacts

If something goes wrong out-of-hours:

- **Site down:** Cloudflare status page (<https://www.cloudflarestatus.com>)
  → if green, roll back the Pages deployment.
- **Booking down:** Cal.com status (<https://status.cal.com>).
- **Payments down:** Stripe status (<https://status.stripe.com>).
- **DNS:** Porkbun status (<https://status.porkbun.com>).

---

*Maintained by ThinkWell Labs, LLC. Update this document whenever the
deploy topology changes — it is the single source of truth for getting the
site back up after an outage.*
