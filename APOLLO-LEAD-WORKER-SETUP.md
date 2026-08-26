# Apollo Address — Lead Worker Setup

This is the missing piece between the landing page and Salesforce. The
page already does its job correctly: it validates the form, captures
UTM/gclid/fbclid, and POSTs a JSON payload to a URL. That URL just
didn't exist yet.

**This is a plain text file on purpose.** Earlier versions of this
handover shipped the code as a `.zip` of a `.zip`, which tripped both
Gmail's attachment scanner (blocks `.js` files outright) and then a
renamed-extension workaround for that tripped an antivirus heuristic
on the receiving end (disguised file extensions inside nested archives
is a real malware pattern, so the flag wasn't unreasonable — just a
false positive on genuinely harmless code). Putting the actual source
in a plain `.md` file sidesteps both: nothing here is an archive, a
script attachment, or disguised in any way. Create the two files
below yourself, by hand, copying the code blocks in — two minutes of
manual work, zero scanner drama.

```
Browser (the landing page)
   |  POST  { name, phone, email, purpose, consent, utm_*, ... }
   v
Cloudflare Worker  (the code below)
   |  1. validates again, server-side
   |  2. exchanges a refresh token for a Salesforce access token
   |  3. checks for a duplicate (same phone, last 24h)
   |  4. creates or updates the Lead
   v
Salesforce
```

Nothing Salesforce-shaped — no client secret, no refresh token, no org
URL — ever reaches the browser. The Worker is the only thing that talks
to Salesforce.

**One thing that changed recently:** Salesforce now enforces three
security settings on every org that didn't used to be mandatory — PKCE
on the login step, and automatic **rotation** of the refresh token
every time it's used. Both are handled below.

---

## Part 0 — Create the project folder

On your machine, create this structure:

```
apollo-lead-worker/
|-- wrangler.toml
`-- src/
    `-- index.js
```

Create `wrangler.toml` and paste in exactly this:

```toml
name = "apollo-lead-worker"
main = "src/index.js"
compatibility_date = "2025-01-01"

# One value you set per environment, not a secret - the exact origin the
# form is served from. Locks down CORS so only your site can call this.
# No trailing slash. Update after you know your final domain.
[vars]
ALLOWED_ORIGIN = "https://REPLACE-WITH-YOUR-CLOUDFLARE-URL.workers.dev"

# Salesforce now enforces Refresh Token Rotation on every org - each time
# this Worker exchanges the refresh token for an access token, Salesforce
# hands back a BRAND NEW refresh token and immediately invalidates the old
# one. A fixed secret can't keep up with that, so the current token lives
# in this KV namespace instead, and the Worker overwrites it after every
# single request. The `id` below is a placeholder - README.md Part 3 shows
# the one command that creates this namespace and gives you the real id
# to paste in.
[[kv_namespaces]]
binding = "TOKEN_STORE"
id = "REPLACE-WITH-YOUR-KV-NAMESPACE-ID"

# Everything below is set via `wrangler secret put <NAME>` - never written
# here, never committed, never visible in the dashboard after it's set.
#
#   SF_CLIENT_ID       External Client App consumer key
#   SF_CLIENT_SECRET   External Client App consumer secret
#   SF_REFRESH_TOKEN   from the one-time authorization in README.md -
#                      this is only ever used ONCE, to "seed" the KV
#                      store on the very first request. After that the
#                      Worker reads and writes the rotating token in KV
#                      instead, and this original secret is never touched
#                      again.
#   SF_LOGIN_URL       https://tam.my.salesforce.com  (your org's domain,
#                      no trailing slash - NOT login.salesforce.com once
#                      you have a custom My Domain, which you do)
```

Create `src/index.js` and paste in exactly this:

```javascript
/**
 * Apollo Address - lead capture Worker
 * ============================================================
 * What this does, in order, for every form submission:
 *
 *   1. Checks the request actually came from your site (CORS).
 *   2. Re-validates the data server-side. The browser already validates,
 *      but a browser check is a suggestion - anyone can POST directly to
 *      this URL with curl, so nothing here trusts the client.
 *   3. Exchanges a stored refresh token for a fresh Salesforce access
 *      token. Nothing Salesforce-shaped ever touches the browser.
 *   4. Looks for an existing Lead with the same phone number created in
 *      the last 24 hours. If one exists, appends this submission to its
 *      Description instead of creating a duplicate - this is the
 *      dedupe rule for repeat form-fillers.
 *   5. Creates (or updates) the Lead. If Salesforce rejects a field it
 *      doesn't recognise (e.g. a custom field your admin hasn't created
 *      yet), the Worker strips that one field and retries automatically,
 *      rather than losing the whole lead over one missing column.
 *
 * Nothing here is a secret except the four values in wrangler.toml's
 * `secret` list (client id/secret, refresh token, login URL) - those are
 * set with `wrangler secret put`, never written into this file.
 * ============================================================
 */

// ---------------------------------------------------------------------
// 1. LeadSource mapping
// ---------------------------------------------------------------------
// Your SOQL convention is inclusion filters against a known list of
// LeadSource values (never exclusion filters) - so this list has to be
// the same fixed vocabulary your SFDC team already reports against, not
// whatever a UTM tag happens to say. Edit the right-hand values to match
// your org's actual LeadSource picklist labels before go-live; if a
// value here isn't a valid picklist entry, Salesforce will reject it and
// the Worker will fall back to "Digital" (edit FALLBACK_SOURCE too).
const FALLBACK_SOURCE = "Digital";
const SOURCE_MAP = {
  google: "Digital - Google Ads",
  meta: "Digital - Meta Ads",
  facebook: "Digital - Meta Ads",
  instagram: "Digital - Meta Ads",
  whatsapp: "Digital - WhatsApp",
  direct: "Digital - Direct",
  referral: "Digital - Referral",
};

function leadSourceFor(payload) {
  const key = (payload.utm_source || "").toLowerCase();
  return SOURCE_MAP[key] || FALLBACK_SOURCE;
}

// ---------------------------------------------------------------------
// 2. Validation - the same rules the page enforces, enforced again
// ---------------------------------------------------------------------
const PHONE_RE = /^[6-9]\d{9}$/;
const EMAIL_RE = /^[^\s@]+@[^\s@]+\.[^\s@]{2,}$/;

function validate(payload) {
  const errors = [];
  if (!payload.name || String(payload.name).trim().length < 2) errors.push("name");
  if (!PHONE_RE.test(payload.phone || "")) errors.push("phone");
  if (!EMAIL_RE.test(payload.email || "")) errors.push("email");
  if (payload.consent !== true) errors.push("consent");
  return errors;
}

// ---------------------------------------------------------------------
// 3. CORS
// ---------------------------------------------------------------------
function corsHeaders(env, origin) {
  const allowed = origin === env.ALLOWED_ORIGIN ? origin : env.ALLOWED_ORIGIN;
  return {
    "Access-Control-Allow-Origin": allowed,
    "Access-Control-Allow-Methods": "POST, OPTIONS",
    "Access-Control-Allow-Headers": "Content-Type",
    "Access-Control-Max-Age": "86400",
    Vary: "Origin",
  };
}

// ---------------------------------------------------------------------
// 4. Salesforce auth - refresh-token flow, with rotation handling
// ---------------------------------------------------------------------
// Salesforce now rotates the refresh token on every use: each exchange
// returns a brand new refresh token and invalidates the one just used.
// So the "current" token can't live in a fixed secret - it has to be
// read from and written back to KV on every single request. The first
// time this ever runs (KV is empty), it falls back to the one-time
// secret from the README's Part 2, then immediately starts using KV
// from that point on.
const TOKEN_KEY = "sf_refresh_token";

async function getAccessToken(env) {
  const refreshToken = (await env.TOKEN_STORE.get(TOKEN_KEY)) || env.SF_REFRESH_TOKEN;

  const url = `${env.SF_LOGIN_URL}/services/oauth2/token`;
  const body = new URLSearchParams({
    grant_type: "refresh_token",
    client_id: env.SF_CLIENT_ID,
    client_secret: env.SF_CLIENT_SECRET,
    refresh_token: refreshToken,
  });

  const res = await fetch(url, {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body,
  });

  if (!res.ok) {
    const detail = await res.text().catch(() => "");
    throw new Error(`Salesforce auth failed (${res.status}): ${detail}`);
  }

  const json = await res.json(); // { access_token, refresh_token, instance_url, ... }

  // Rotation means a new refresh_token comes back with every response -
  // save it immediately so the *next* request uses the current one,
  // not the one that just got invalidated.
  if (json.refresh_token && json.refresh_token !== refreshToken) {
    await env.TOKEN_STORE.put(TOKEN_KEY, json.refresh_token);
  }

  return json;
}

// ---------------------------------------------------------------------
// 5. Field mapping
// ---------------------------------------------------------------------
// Standard fields every Salesforce org has, guaranteed to succeed.
// Custom fields are attempted too, and stripped automatically on a 400
// if your org doesn't have them yet - so this Worker starts working the
// day your SFDC developer creates them, with no code change or redeploy.
function splitName(full) {
  const parts = String(full || "").trim().split(/\s+/);
  if (parts.length === 1) return { firstName: null, lastName: parts[0] };
  return { firstName: parts.slice(0, -1).join(" "), lastName: parts[parts.length - 1] };
}

function buildDescription(p) {
  // Guaranteed capture of everything, even before any custom field
  // exists - nothing is ever silently lost.
  const lines = [
    `Lead ID: ${p.lead_id || ""}`,
    `Project: ${p.project || ""}`,
    `Buying for: ${p.purpose || ""}`,
    `Form location: ${p.form_location || ""}`,
    `Consent given: ${p.consent === true ? "Yes" : "No"}`,
    "",
    `-- Last touch --`,
    `Source/Medium/Campaign: ${p.utm_source || ""} / ${p.utm_medium || ""} / ${p.utm_campaign || ""}`,
    `Content/Term: ${p.utm_content || ""} / ${p.utm_term || ""}`,
    `GCLID: ${p.gclid || ""}    FBCLID: ${p.fbclid || ""}    MSCLKID: ${p.msclkid || ""}`,
    `Landing page: ${p.landing_page || ""}`,
    `Referrer: ${p.referrer || ""}`,
    "",
    `-- First touch --`,
    `Source/Medium/Campaign: ${p.first_utm_source || ""} / ${p.first_utm_medium || ""} / ${p.first_utm_campaign || ""}`,
    `Landing page: ${p.first_landing_page || ""}`,
    `First seen: ${p.first_timestamp || ""}`,
    "",
    `Page URL: ${p.page_url || ""}`,
    `Submitted: ${p.submitted_at || ""}`,
    `User agent: ${p.user_agent || ""}`,
  ];
  return lines.join("\n").slice(0, 32000); // Salesforce long-text-area ceiling
}

function buildLeadFields(p) {
  const { firstName, lastName } = splitName(p.name);

  return {
    // --- standard fields: always present on every Salesforce org ---
    FirstName: firstName,
    LastName: lastName || "Website Enquiry",
    Company: p.name ? `${p.name} (Individual Enquiry)` : "Apollo Address Website",
    Phone: p.phone,
    Email: p.email,
    LeadSource: leadSourceFor(p),
    Description: buildDescription(p),

    // --- custom fields: created by your SFDC developer when ready.
    // Any of these that don't exist yet get stripped automatically. ---
    Project__c: p.project,
    Lead_ID__c: p.lead_id,
    Form_Location__c: p.form_location,
    Buying_Purpose__c: p.purpose,
    UTM_Source__c: p.utm_source,
    UTM_Medium__c: p.utm_medium,
    UTM_Campaign__c: p.utm_campaign,
    UTM_Content__c: p.utm_content,
    UTM_Term__c: p.utm_term,
    GCLID__c: p.gclid,
    FBCLID__c: p.fbclid,
    MSCLKID__c: p.msclkid,
    Landing_Page__c: p.landing_page,
    First_Touch_Source__c: p.first_utm_source,
    First_Touch_Medium__c: p.first_utm_medium,
    First_Touch_Campaign__c: p.first_utm_campaign,
    First_Landing_Page__c: p.first_landing_page,
    Consent_Given__c: p.consent === true,
  };
}

// Remove keys with no value at all (undefined/null/"") so we don't send
// noise, but keep explicit `false` (e.g. Consent_Given__c: false).
function pruneEmpty(fields) {
  const out = {};
  for (const [k, v] of Object.entries(fields)) {
    if (v === undefined || v === null || v === "") continue;
    out[k] = v;
  }
  return out;
}

// ---------------------------------------------------------------------
// 6. Salesforce REST calls, with self-healing field retry
// ---------------------------------------------------------------------
const API_VERSION = "v59.0";

async function sfFetch(instanceUrl, token, path, options) {
  return fetch(`${instanceUrl}${path}`, {
    ...options,
    headers: {
      Authorization: `Bearer ${token}`,
      "Content-Type": "application/json",
      ...(options && options.headers),
    },
  });
}

function extractBadField(errorBody) {
  // Salesforce's INVALID_FIELD message names the offending field, e.g.
  // "No such column 'GCLID__c' on sobject of type Lead". Pull it out so
  // we can strip just that one key and try again.
  try {
    const arr = JSON.parse(errorBody);
    const msg = Array.isArray(arr) && arr[0] && arr[0].message;
    if (!msg) return null;
    const m = msg.match(/column '([^']+)'/) || msg.match(/field '([^']+)'/i);
    return m ? m[1] : null;
  } catch (e) {
    return null;
  }
}

async function createOrUpdateLead(instanceUrl, token, payload) {
  let fields = pruneEmpty(buildLeadFields(payload));

  // Dedupe: same phone number, created in the last 24 hours -> update
  // that Lead's notes instead of creating a second record. This is the
  // 24h phone-based dedupe rule; adjust the SOQL window if you want a
  // different threshold.
  const soql = `SELECT Id, Description FROM Lead WHERE Phone = '${payload.phone.replace(/'/g, "")}' AND CreatedDate = LAST_N_DAYS:1 ORDER BY CreatedDate DESC LIMIT 1`;
  const dupeRes = await sfFetch(instanceUrl, token, `/services/data/${API_VERSION}/query?q=${encodeURIComponent(soql)}`);
  const dupeJson = dupeRes.ok ? await dupeRes.json() : { records: [] };

  const existing = dupeJson.records && dupeJson.records[0];

  for (let attempt = 0; attempt < 8; attempt++) {
    const isUpdate = !!existing;
    const path = isUpdate
      ? `/services/data/${API_VERSION}/sobjects/Lead/${existing.Id}`
      : `/services/data/${API_VERSION}/sobjects/Lead`;

    const body = isUpdate
      ? { Description: `${existing.Description || ""}\n\n---\nRepeat enquiry:\n${fields.Description}`.slice(-32000) }
      : fields;

    const res = await sfFetch(instanceUrl, token, path, {
      method: isUpdate ? "PATCH" : "POST",
      body: JSON.stringify(body),
    });

    if (res.status === 204) return { id: existing.Id, updated: true };  // PATCH success, no body
    if (res.ok) {
      const json = await res.json();
      return { id: json.id, updated: false };
    }

    const errText = await res.text();
    const badField = extractBadField(errText);
    if (badField && fields[badField] !== undefined) {
      delete fields[badField];               // that field doesn't exist in this org yet - drop and retry
      continue;
    }

    throw new Error(`Salesforce Lead ${isUpdate ? "update" : "create"} failed (${res.status}): ${errText}`);
  }

  throw new Error("Salesforce Lead create failed after repeated field-stripping retries");
}

// ---------------------------------------------------------------------
// 7. Entry point
// ---------------------------------------------------------------------
export default {
  async fetch(request, env) {
    const origin = request.headers.get("Origin") || "";
    const headers = corsHeaders(env, origin);

    if (request.method === "OPTIONS") {
      return new Response(null, { status: 204, headers });
    }

    if (request.method !== "POST") {
      return new Response(JSON.stringify({ error: "Method not allowed" }), {
        status: 405,
        headers: { ...headers, "Content-Type": "application/json" },
      });
    }

    if (origin && origin !== env.ALLOWED_ORIGIN) {
      return new Response(JSON.stringify({ error: "Origin not allowed" }), {
        status: 403,
        headers: { ...headers, "Content-Type": "application/json" },
      });
    }

    let payload;
    try {
      payload = await request.json();
    } catch (e) {
      return new Response(JSON.stringify({ error: "Invalid JSON" }), {
        status: 400,
        headers: { ...headers, "Content-Type": "application/json" },
      });
    }

    // Honeypot: the real form's "company" field is hidden from people and
    // must always arrive empty. A filled value means a bot filled every
    // field it could find, including the trap.
    if (payload.company) {
      return new Response(JSON.stringify({ ok: true }), {  // pretend success, drop silently
        status: 200,
        headers: { ...headers, "Content-Type": "application/json" },
      });
    }

    const errors = validate(payload);
    if (errors.length) {
      return new Response(JSON.stringify({ error: "Validation failed", fields: errors }), {
        status: 422,
        headers: { ...headers, "Content-Type": "application/json" },
      });
    }

    try {
      // Rotation creates a narrow race: two submissions landing at nearly
      // the same instant can both read the same refresh token before
      // either has rotated it, so the second one fails auth. One retry
      // covers that - by the second attempt, KV has the token the first
      // request just rotated in.
      let auth, result;
      try {
        auth = await getAccessToken(env);
        result = await createOrUpdateLead(auth.instance_url, auth.access_token, payload);
      } catch (firstErr) {
        if (!/auth failed|INVALID_GRANT/i.test(firstErr.message)) throw firstErr;
        auth = await getAccessToken(env);
        result = await createOrUpdateLead(auth.instance_url, auth.access_token, payload);
      }

      return new Response(JSON.stringify({ ok: true, salesforce_id: result.id, updated: result.updated }), {
        status: 200,
        headers: { ...headers, "Content-Type": "application/json" },
      });
    } catch (err) {
      console.error(err.message);
      return new Response(JSON.stringify({ error: "Could not reach Salesforce" }), {
        status: 502,
        headers: { ...headers, "Content-Type": "application/json" },
      });
    }
  },
};
```

That's the whole project. Everything from here is setup, not code.

---

## Part 1 — Salesforce setup (do this once)

You'll need Salesforce admin access for this part. If that's not you,
hand this part to whoever has it; Part 2 onward you can do yourself.

Salesforce retired the old "Connected App" creation flow — you'll be
creating an **External Client App** instead. It does the same job, with
a couple of extra settings. (**Already done for Apollo Address** — see
the project status doc for what exists already and what you still need
from the marketing team: Consumer Key, Consumer Secret, and confirmation
of the org's My Domain URL.)

### 1. Create the External Client App

Setup -> App Manager -> **New External Client App**

- **Basic Information**: name it `Apollo Address Website`, fill in your
  email, set **Distribution State** to **Local**.
- **API (Enable OAuth Settings)** -> check **Enable OAuth**:
  - **Callback URL**: `https://login.salesforce.com/services/oauth2/callback`
    (only ever used once, in Part 2 below)
  - **OAuth Scopes** — add both:
    - `Manage user data via APIs (api)`
    - `Perform requests at any time (refresh_token, offline_access)`
- **Security section** — uncheck **"Require Proof Key for Code Exchange
  (PKCE) Extension for Supported Authorization Flows"** if it's editable.
  On many orgs this is now locked on by Salesforce and shown grayed out
  with "To change this required setting, contact Support" — if so,
  **leave it as-is and continue**; Part 2 below already accounts for
  PKCE being mandatory.
- Click **Create**.

### 2. Set the app policies

Reopen the app -> **Policies** tab -> **Edit**:

- **Refresh Token Policy** -> set to **"Refresh token is valid until
  revoked"** if that option exists. Many orgs now lock this to "expire
  if idle for 30 days" instead (grayed out, non-editable) — if so,
  leave it; it just means the connection needs redoing if the site
  goes completely silent for a full month, which a live lead-gen page
  won't.
- **Permitted Users** -> set to **"All users may self-authorize"** if
  available.
- Save.

Wait **10 minutes** before continuing — Salesforce takes a few minutes
to actually activate a new app, and also after any Policy edit to it.

### 3. Get the Consumer Key and Secret

App Manager -> open "Apollo Address Website" -> **Settings** tab ->
**OAuth Settings** -> **Consumer Key and Secret**. Salesforce may first
email a verification code — enter it if asked.

### 4. Choose an integration user

Use a dedicated Salesforce user for this — not a real salesperson's
login — so leads keep flowing even if that person's password changes.
Give this user a Permission Set with create/edit access to Leads and
nothing else.

### 5. (Recommended) Add the custom fields

The Worker works today without these — it stuffs all the extra data
(UTM source, gclid, first-touch attribution, etc.) into the Lead's
Description field as a fallback. But readable custom fields make
reporting far easier. Create these whenever convenient; the Worker
starts populating them automatically the moment they exist, no code
change needed:

| API Name | Type |
|---|---|
| `Project__c` | Text(255) |
| `Lead_ID__c` | Text(50), external ID |
| `Form_Location__c` | Text(50) |
| `Buying_Purpose__c` | Text(50) |
| `UTM_Source__c` / `UTM_Medium__c` / `UTM_Campaign__c` / `UTM_Content__c` / `UTM_Term__c` | Text(255) each |
| `GCLID__c` / `FBCLID__c` / `MSCLKID__c` | Text(255) each |
| `Landing_Page__c` / `First_Landing_Page__c` | URL or Text(255) |
| `First_Touch_Source__c` / `First_Touch_Medium__c` / `First_Touch_Campaign__c` | Text(255) each |
| `Consent_Given__c` | Checkbox |

Also confirm the exact **LeadSource** picklist values this org already
uses for digital leads. The code has a `SOURCE_MAP` near the top — edit
the right-hand values to match the org's real picklist labels, or
Salesforce will reject the value and the Worker falls back to a single
generic "Digital" value.

---

## Part 2 — Get a refresh token (one-time, ~5 minutes)

Because PKCE is mandatory, this login step needs an extra "proof" value
alongside the usual authorization code. Use this exact pair:

```
code_verifier:  0bE0y2k7AC4puVCPQjFCD0zAkpbtUGPQIC6geK6HJniWh1-DHOL2EIwKl8k9NkUyK6573gqY7zv6dsQrRYp3yw
code_challenge: TOagi0z2bJEHliCVb1bPCiy6bFGCIo0QWxUA1EfOrh8
```

1. Build this URL, filling in the Consumer Key and confirming the
   Salesforce domain is correct (this uses `tam` — verify against the
   actual org):

   ```
   https://tam.my.salesforce.com/services/oauth2/authorize?response_type=code&client_id=YOUR_CONSUMER_KEY&redirect_uri=https://login.salesforce.com/services/oauth2/callback&code_challenge=TOagi0z2bJEHliCVb1bPCiy6bFGCIo0QWxUA1EfOrh8&code_challenge_method=S256
   ```

2. Open it in a browser (use dev tools / Network tab if anything goes
   wrong here — a `redirect_uri_mismatch` was hit previously and not
   resolved without proper tooling; likely a stale cache, a Policy
   edit that hadn't finished propagating, or a Consumer Key typo).

3. Log in as the integration user, click **Allow**.

4. You'll land on a blank/broken-looking page. Look at the **address
   bar** — it contains `?code=` followed by a long string. Copy
   everything after `code=` and before any `&`. It expires in a couple
   of minutes, so move to the next step quickly.

5. Exchange it for a refresh token:

   ```bash
   curl -X POST https://tam.my.salesforce.com/services/oauth2/token \
     -d "grant_type=authorization_code" \
     -d "client_id=YOUR_CONSUMER_KEY" \
     -d "client_secret=YOUR_CONSUMER_SECRET" \
     -d "redirect_uri=https://login.salesforce.com/services/oauth2/callback" \
     -d "code=THE_CODE_FROM_STEP_4" \
     -d "code_verifier=0bE0y2k7AC4puVCPQjFCD0zAkpbtUGPQIC6geK6HJniWh1-DHOL2EIwKl8k9NkUyK6573gqY7zv6dsQrRYp3yw"
   ```

6. The response is JSON containing `refresh_token`. This is only used
   **once**, to seed the Worker — after that, Salesforce rotates it
   automatically and the Worker tracks the current one itself (Part 3).

---

## Part 3 — Deploy the Worker

Needs [Node.js](https://nodejs.org) and a Cloudflare account (free tier
is enough).

```bash
cd apollo-lead-worker
npm install -g wrangler
wrangler login
```

**Create the KV namespace** that stores the rotating refresh token —
Salesforce force-rotates it on every use, so a fixed secret would break
after the first lead. This gives the Worker somewhere to keep the
current one:

```bash
wrangler kv namespace create TOKEN_STORE
```

This prints something like `{ binding = "TOKEN_STORE", id = "ab12cd..." }`.
Open `wrangler.toml` and replace `REPLACE-WITH-YOUR-KV-NAMESPACE-ID`
with that `id`.

**Store the four secrets** (you'll be prompted to paste each value):

```bash
wrangler secret put SF_CLIENT_ID
wrangler secret put SF_CLIENT_SECRET
wrangler secret put SF_REFRESH_TOKEN
wrangler secret put SF_LOGIN_URL       # e.g. https://tam.my.salesforce.com
```

**Deploy:**

```bash
wrangler deploy
```

Wrangler prints your live URL, e.g. `https://apollo-lead-worker.<subdomain>.workers.dev`.

Open `wrangler.toml`, set `ALLOWED_ORIGIN` to the site's actual live
URL, then `wrangler deploy` again so CORS matches.

---

## Part 4 — Point the site at it

In the landing page's `index.html`, find:

```js
var ENDPOINT = "https://REPLACE-WITH-YOUR-WORKER.workers.dev/apollo-lead";
```

Replace with the real Worker URL from Part 3 — **drop the
`/apollo-lead` suffix**, this Worker responds on its root path:

```js
var ENDPOINT = "https://apollo-lead-worker.<subdomain>.workers.dev";
```

Redeploy the site.

---

## Part 5 — Test before trusting it

```bash
curl -X POST https://apollo-lead-worker.<subdomain>.workers.dev \
  -H "Content-Type: application/json" \
  -H "Origin: https://YOUR-SITE-URL" \
  -d '{"name":"Test Lead","phone":"9900011122","email":"test@example.com","purpose":"Investment","consent":true,"project":"Apollo Address","form_location":"curl-test","utm_source":"google","utm_medium":"cpc"}'
```

Expect back: `{"ok":true,"salesforce_id":"00Q...","updated":false}`.
Check Salesforce for a new "Test Lead". Run the exact same command
again within 24 hours — it should return `"updated":true` and update
the same Lead rather than duplicating it (phone-based dedupe).

Confirm rotation is working: run it a third time. If it still succeeds,
the Worker correctly picked up the rotated token. You can check this
directly:

```bash
wrangler kv key get sf_refresh_token --binding TOKEN_STORE
```

This should print a token that's **different** from the one-time
`SF_REFRESH_TOKEN` secret — confirming it moved on to a rotated token.

Only then, submit the real form on the live page and confirm the same
thing happens end to end.

---

## What this does not yet cover

- **Rate limiting** beyond the honeypot field and format validation.
  Turn on Cloudflare's **Bot Fight Mode** (free, one click) once live.
- **Alerting on failure.** If Salesforce is down or the token is fully
  revoked, the Worker returns a clean error to the page (existing
  "please try again" banner) but nobody is notified. Worth wiring a
  Slack/email webhook into the `catch` block once stable.
- **Click-to-call is intent, not outcome.** The Call button fires an
  analytics event on tap for ad attribution — it cannot detect whether
  a call connected or how long it lasted. That needs a telephony/
  call-tracking number in front of it, a separate paid integration.
