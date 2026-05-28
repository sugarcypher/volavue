# VolaVue — Security & Privacy Hardening

This document captures the pre-launch security posture for the VolaVue site
(`volavue.fyi`), operated by **ThinkWell Labs, LLC** (Wyoming). It explains
*what the architecture protects against by design*, the headers and settings
to apply at the edge (Cloudflare), and the manual checks you should walk
through before going live.

It is **not** a substitute for legal advice. The legal pages are published as-is;
review with counsel is recommended but not a launch blocker.

---

## 1. Threat model — what we're protecting against

The site is a static brochure + scheduling/payment funnel. Realistic threats:

1. **Data extraction.** Someone pulling intake answers, booking info, or
   payment data from our systems.
2. **Account compromise.** An attacker taking over the GitHub, Cloudflare,
   Cal.com, Stripe or registrar account.
3. **Site tampering.** An attacker modifying what visitors see (defacement,
   phishing, malicious script injection).
4. **DDoS / scraping / form abuse.** Noise that disrupts the site or wastes
   resources.
5. **Privacy non-compliance.** GDPR / CCPA / CPRA exposure from how data is
   handled.

## 2. Why the architecture already minimises most of this

The biggest single security win is structural: **we hold almost no data
ourselves.** That removes most of the "data extraction" surface before any
control is even applied.

- **No own database.** Static HTML/CSS/JS only. Nothing to breach on our
  side.
- **No own backend.** Static hosting on GitHub Pages — no server-side
  code we maintain.
- **No own analytics setting cookies.** If we want stats later, use
  Cloudflare Web Analytics (cookieless).
- **No third-party dependencies in the repo.** No `package.json`, no npm
  supply chain to compromise.
- **Cal.com** holds the intake + booking. Their security programme covers
  it. We delete bookings after a short retention window.
- **Stripe** holds payment data. We never see card numbers.
- **Self-hosted fonts.** No request to Google Fonts on page load.
- **Intake answers travel via `sessionStorage`**, not the URL — keeping PII
  out of browser history, server logs, and referrer headers.
- **Strictly-necessary cookies only.** Cal.com is the only third-party embed
  on the booking page. Its cookies are required for the scheduler to work
  and qualify as strictly-necessary under GDPR — no consent banner is
  shown. We set no analytics, advertising or tracking cookies of our own.
- **Cloudflare in front.** DDoS, bot, and bad-actor noise is absorbed at
  the edge before reaching the origin.

The remaining substantive risk is **account-level** (someone taking over
GitHub / Cloudflare / Cal / Stripe / the registrar / the email). The
checklist below addresses that.

---

## 3. Account hardening — do this before launch

For each of these accounts, in order of how damaging a takeover would be:

1. **Email — `volavue@volavue.fyi`** (the master recovery for everything)
   - Long unique password stored in a password manager.
   - **Hardware-key or app-based 2FA** (not SMS).
   - No SMS recovery option. Disable it if present.
   - Configure recovery codes; store offline.
2. **Domain registrar (where `volavue.fyi` is registered)**
   - Strong unique password + 2FA.
   - **Registrar lock / transfer lock** enabled.
   - 2-step domain-transfer confirmation enabled if available.
3. **Cloudflare**
   - Strong unique password + 2FA (hardware key or TOTP).
   - "Require 2FA for all members" if you ever add team members.
   - Configure account-takeover-alert email to a different inbox.
4. **GitHub (`sugarcypher`)**
   - 2FA enabled (hardware key strongly preferred).
   - Vigilant Mode on — flags unverified commits.
   - Branch protection on `main`: require PR review, require status checks,
     prohibit force-push.
   - Secret scanning enabled (free for public repos).
   - Dependabot alerts on (no-op for us today — no dependencies — but
     future-safe).
5. **Cal.com**
   - 2FA on.
   - Set a short booking-retention policy. Delete bookings after the
     session output is delivered + N days (decide N with counsel — common
     defaults: 30 or 90 days).
6. **Stripe**
   - 2FA on (hardware key preferred).
   - Restricted-access API keys only if you ever script anything — never
     embed live keys in the site repo.
   - Enable Stripe's email alerts for high-value events.

Store recovery codes for every account in your password manager AND
offline (e.g. printed and locked in a drawer).

---

## 4. Cloudflare configuration

Once `volavue.fyi` is on Cloudflare:

### DNS & TLS
- DNS for `volavue.fyi` set to **proxied** (orange cloud) so traffic goes
  through Cloudflare. CNAME for `flawlessfini.sh` set up similarly,
  redirecting via a Cloudflare Page Rule to `volavue.fyi`.
- **SSL/TLS mode: Full (Strict)** — requires GitHub Pages' certificate to
  match.
- **Always Use HTTPS** = on.
- **Automatic HTTPS Rewrites** = on.
- **Minimum TLS version**: 1.2 (or 1.3).
- **HSTS**: enable in Cloudflare's HSTS settings with the values shown
  below.

### Security
- **Bot Fight Mode** = on (free).
- **Browser Integrity Check** = on.
- **Security Level**: Medium or High.
- **Email obfuscation** = on (Cloudflare rewrites mailto links).
- **WAF managed rules** = the free tier is sufficient for this site.

### Performance / privacy
- **Cloudflare Web Analytics**: enable (cookieless).
- **Turnstile**: not required today (no public form submits to a server).
  Worth adding if you ever add a contact form that POSTs.

### Security headers (Cloudflare Transform Rules → Modify Response Header)

Add one Transform Rule that applies to "all incoming requests" on
`volavue.fyi`, setting these response headers:

| Header | Value |
| --- | --- |
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains; preload` |
| `X-Content-Type-Options` | `nosniff` |
| `Referrer-Policy` | `strict-origin-when-cross-origin` |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=(), interest-cohort=(), payment=(self "https://*.stripe.com")` |
| `Cross-Origin-Opener-Policy` | `same-origin` |
| `X-Frame-Options` | `DENY` |
| `Content-Security-Policy` | *(see below)* |

**CSP value** (single line; the inline-heavy build needs `'unsafe-inline'`
for styles and scripts — Cloudflare's edge sets the header and you can
tighten later if/when we refactor inline styles out):

```
default-src 'self'; script-src 'self' 'unsafe-inline' https://app.cal.eu https://app.cal.com https://*.stripe.com; style-src 'self' 'unsafe-inline'; img-src 'self' data: https://app.cal.eu https://app.cal.com https://*.stripe.com; font-src 'self'; connect-src 'self' https://app.cal.eu https://app.cal.com https://*.stripe.com; frame-src https://app.cal.eu https://app.cal.com https://*.stripe.com; form-action 'self'; base-uri 'self'; frame-ancestors 'none'; upgrade-insecure-requests
```

The CSP is currently **enforced** (not Report-Only). Verified against a
real-money smoke test of the Cal+Stripe booking flow on 2026-05-26. If
new third-party origins are added later (e.g. analytics), test with
`Content-Security-Policy-Report-Only` first and only flip to enforced
after a real-browser walk of the affected page shows zero violations.

---

## 5. Repo / GitHub hygiene

- **No secrets in the repo**, ever. The site is fully client-side; if you
  later add a Cloudflare Worker or any backend, secrets go in the host's
  secret store, not the repo.
- `.gitignore` excludes `.DS_Store` and the cloud-sync conflict files.
- **Branch protection on `main`**: require PR review and prohibit
  force-push.
- **Signed commits** (optional, nice to have): turn on Vigilant Mode and
  sign commits with a GPG/SSH key.

---

## 6. Pre-launch verification checklist

1. Visit `https://volavue.fyi/`. Confirm the page serves over HTTPS with
   no mixed-content warnings.
2. Run the site through:
   - **Mozilla Observatory** (<https://observatory.mozilla.org/>) — target
     grade A or higher.
   - **securityheaders.com** — target A or A+.
   - **Cloudflare's SSL/TLS test** — confirm Full (Strict).
3. Open the booking page. Confirm:
   - The cookie banner appears on first visit.
   - "Decline" leaves the calendar disabled with the email fallback.
   - "Accept" loads the Cal.com calendar.
   - Refreshing remembers your choice.
4. Submit the intake form. Confirm the URL on the booking page is *clean*
   (no `?q1=...` query string) — the data is in `sessionStorage` only.
5. View page source on each main page. Confirm:
   - `<link rel="canonical">` is set.
   - No reference to `fonts.googleapis.com` remains.
   - `volavue@volavue.fyi` is the only contact email.
6. Confirm `/robots.txt`, `/sitemap.xml`, `/llms.txt`, and
   `/.well-known/security.txt` all return 200.

---

## 7. After launch — keep it boring

- Quarterly: re-run Observatory / securityheaders and confirm grades.
- Quarterly: rotate the `Expires` date in `security.txt`.
- Annually: review and refresh the legal pages.
- Monitor: a single inbox alert from Cloudflare ("attack mode triggered",
  unusual traffic) is normal noise. A login alert from one of the
  account-level services is *not* — investigate immediately.

If something does go wrong, the response order is:
1. Lock the account at risk (revoke sessions, rotate password).
2. Audit recent activity (Cloudflare events, GitHub audit log, Cal/Stripe
   activity).
3. Disclose to users only if their data was actually affected, in line
   with the Privacy Policy.

---

*Maintained by ThinkWell Labs, LLC. Last updated 2026-05-24.*
