# Apollo Address — Handover to Development Team

## What this is

A landing page for Apollo Address (Address Maker), built for lead
generation: hero video, project details, amenities, gallery, and a
lead-capture form/popup wired for Salesforce + GTM/UTM tracking.

Two separate deliverables are included:

1. **`apollo-site.zip`** — the landing page itself. Static HTML/CSS/JS,
   no build step, no framework. Deploy it anywhere that serves static
   files (Netlify, Cloudflare Pages/Workers, S3+CloudFront, your own
   Nginx box — whatever you already use for the live microsites).
2. **`APOLLO-LEAD-WORKER-SETUP.md`** — a single plain-text document
   containing both the full setup instructions and the complete
   Cloudflare Worker source code (Salesforce integration), inline. It's
   a `.md` file rather than a zip of code on purpose — an earlier
   version shipped the code as a zip, which tripped Gmail's attachment
   scanner, and a workaround for that then tripped an antivirus
   heuristic on the receiving end. Plain text sidesteps both entirely.
   You'll create two small files by hand (instructions are the first
   thing in that document), then follow the setup from there.

---

## Status: what's done vs. what's blocked

**Done and working:**
- The site is complete and tested (desktop + mobile), including the
  popup form, sticky mobile CTA bar, hero video with autoplay
  fallbacks, and full UTM/gclid/fbclid capture on both first-touch and
  last-touch.
- GTM container wiring and a defined set of `dataLayer` events (see
  `index.html`, search for `dataLayer.push` — every event is commented).
- The Salesforce side: an **External Client App** ("Apollo Address
  Website") already exists in the org, with OAuth enabled, correct
  scopes, and "All users can self-authorize" set. Consumer Key and
  Consumer Secret have been generated and are with the marketing team
  (not included in this handover for security — request them
  separately, see "What you need" below).

**Blocked, unresolved:**
- The one-time OAuth login (needed to get a refresh token so the
  Worker can authenticate to Salesforce) is failing with:
  ```
  error=redirect_uri_mismatch&error_description=redirect_uri must match configuration
  ```
  This happened consistently even after confirming the Callback URL in
  Salesforce matches exactly (`https://login.salesforce.com/services/oauth2/callback`)
  and waiting the required propagation time after edits. It was not
  resolved — likely a subtle encoding issue in the authorize URL, a
  stale cache, or a mismatch between the Consumer Key used and the app
  it's checked against. This needs someone with real browser dev tools
  (Network tab) to inspect the actual outgoing request, which is hard
  to do over screenshots. Should be quick to diagnose properly.

---

## What you need from the marketing team / Salesforce admin

- The **Consumer Key** and **Consumer Secret** for the "Apollo Address
  Website" External Client App (Setup → App Manager → find it → Settings
  tab → OAuth Settings → Consumer Key and Secret).
- Confirmation of which Salesforce user should be the integration
  user (the README below asks for a dedicated non-personal login).
- The org's exact My Domain URL (used `tam.my.salesforce.com` in
  testing — confirm this is correct).

---

## Where to pick up

Everything past this point is documented in detail inside
`APOLLO-LEAD-WORKER-SETUP.md`. It covers, in
order:

- **Part 1** — the Salesforce App setup (already done, see above)
- **Part 2** — the one-time OAuth login to get a refresh token (**this
  is the blocked step** — the README includes the exact PKCE
  `code_verifier`/`code_challenge` pair already generated, so you
  shouldn't need to regenerate it, just get past the redirect_uri
  error)
- **Part 3** — deploying the Worker via Wrangler, including a Cloudflare
  KV namespace (Salesforce now force-rotates refresh tokens on every
  use — the Worker stores the current one in KV rather than a static
  secret, since a fixed secret would break after the first lead)
- **Part 4** — pointing the site's `ENDPOINT` constant at the deployed
  Worker URL (currently a placeholder in `index.html`:
  `https://REPLACE-WITH-YOUR-WORKER.workers.dev`)
- **Part 5** — a test `curl` command to confirm the whole path works
  end-to-end before trusting the live form

## One important note on Salesforce's current API version

Salesforce disabled creating new **Connected Apps** earlier this year
(Spring '26) in favor of **External Client Apps**, and now force-enables
PKCE and refresh token rotation on new apps in most orgs. If you've done
Salesforce OAuth integrations before but not recently, these are the
parts that'll look different from what you're used to — the README
handles all three, but it's worth knowing going in.

## Also worth knowing

- **Click-to-call is intent, not outcome.** The Call button fires an
  analytics event on tap for ad attribution — it cannot detect whether
  a call connected or how long it lasted. That requires a telephony/
  call-tracking number in front of it, which is a separate integration
  not built here.
- **Before going live**, `index.html` still has a few placeholder
  values to replace: the canonical/OG URLs (`REPLACE-WITH-YOUR-...`),
  and the GTM container ID (`GTM-XXXXXXX`, appears twice — once in the
  head script, once in the noscript fallback).
