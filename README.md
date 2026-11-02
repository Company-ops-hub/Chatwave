# Chatwave
WhatsApp broadcast and automation dashboard built on **Google Apps Script + Google Sheets** (backend) and a **single-file dashboard** (`index.html`) hosted on GitHub Pages. All data stays in the client's own Google Workspace. Nothing is stored on any third-party server.

```
Browser (GitHub Pages: index.html)  --POST text/plain-->  Apps Script web app (Code.gs)  -->  Google Sheet
                                                              ^        |
                       Meta WhatsApp Cloud API webhook -------+        +--> Meta Graph API (v21.0)
```

## Contents

1. [What you need](#1-what-you-need)
2. [Deploy the backend (Apps Script)](#2-deploy-the-backend-apps-script)
3. [Deploy the dashboard (GitHub Pages)](#3-deploy-the-dashboard-github-pages)
4. [Connect Meta (WhatsApp Cloud API)](#4-connect-meta-whatsapp-cloud-api)
5. [Test checklist](#5-test-checklist)
6. [Sample contacts CSV](#6-sample-contacts-csv)
7. [Automation API (ingest_event)](#7-automation-api-ingest_event)
8. [Security notes (read this)](#8-security-notes-read-this)
9. [Quotas and how the design stays inside them](#9-quotas-and-how-the-design-stays-inside-them)
10. [Data retention and GDPR](#10-data-retention-and-gdpr)
11. [Assumptions and what to verify with real Meta credentials](#11-assumptions-and-what-to-verify-with-real-meta-credentials)
12. [Troubleshooting](#12-troubleshooting)

---

## 1. What you need

- A Google account (ideally the client's Google Workspace account, since the data lives there).
- A GitHub account.
- For live sending (not needed for the demo): a Meta Business account, a WhatsApp Business Account (WABA), a dedicated phone number, at least one **approved** message template, and a permanent access token (System User token).

## 2. Deploy the backend (Apps Script)

1. Create a new Google Sheet. Name it e.g. `Broadcast data`.
2. Open **Extensions > Apps Script**. Delete the default code and paste the full contents of `Code.gs`. Save.
3. **Set the time zone:** Project Settings (gear icon) > tick *Show "appsscript.json" manifest file* > open `appsscript.json` and set `"timeZone": "Europe/Dublin"`. Save.
4. **Create the sheets:** select `setup` in the function dropdown and click **Run**. Accept the permission prompts (it needs Sheets, external requests, and triggers). This creates the sheets `Contacts`, `Campaigns`, `Messages`, `Replies`, `Suppression`, `AuditLog`, and generates the webhook verify token and the automation API key.
5. **Set the admin password:**
   - In `runSetPassword()`, replace `CHANGE-ME-NOW` with a password of at least 12 characters.
   - Run `runSetPassword`.
   - **Immediately change the text back to `CHANGE-ME-NOW`** so the password is not left in the code.
6. **Install the triggers:** run `setupTriggers`. This installs `processQueue` (every minute) and `purgeOldData` (daily, around 03:00 Dublin time). It is safe to run again.
7. **Deploy as a web app:** *Deploy > New deployment > type: Web app*.
   - Execute as: **Me**
   - Who has access: **Anyone**
   - Click **Deploy** and copy the **Web app URL** (ends in `/exec`).
8. Run `showWebhookInfo` and open *Execution log*. It prints the webhook URL, the verify token and the automation API key. Keep these private.
9. Run `runSelfTest`. Every line should read `PASS` (DEMO mode shows a `WARN`, which is expected for now).

> **Updating the code later:** after editing `Code.gs`, use *Deploy > Manage deployments > pencil icon > Version: New version > Deploy*. This keeps the same URL. Creating a *new* deployment gives a new URL.

> **Workspace note:** if the Workspace admin blocks "Anyone" access for web apps, the dashboard and the Meta webhook cannot reach the script. Ask the admin to allow it, or use a personal Google account.

## 3. Deploy the dashboard (GitHub Pages)

1. Create a **public** repository named `pingdesk` (free GitHub Pages needs a public repo). Do not put the client's name in it.
2. Upload `index.html` and `README.md` to the repository root. The file must be called `index.html`.
3. *Settings > Pages > Build and deployment > Source: Deploy from a branch > Branch: `main` / `(root)` > Save*.
4. After a minute the site is live at `https://<username>.github.io/pingdesk/`.
5. Open the site. On the login screen paste the **Web app URL** from step 2.7, then enter the admin password. The URL is saved in that browser only.

**Do not** paste the web app URL or any secret into `index.html` in a public repository. The login screen field exists for this reason (`API_URL` in the file stays empty).

**Branding:** edit the `BRAND` object at the top of `index.html` (brand name, logo URL, primary colour, support email). **Features:** set `FEATURES.automations`, `inbox` or `integrations` to `true` to switch them on. No layout changes are needed.

## 4. Connect Meta (WhatsApp Cloud API)

Do this after the demo has been approved. The system starts in **DEMO mode**.

1. In the dashboard open **Settings** and enter:
   - Access token (permanent System User token)
   - Phone Number ID
   - WABA ID
   - Verify token (use the one printed by `showWebhookInfo`, or choose your own and enter it in both places)
2. Click **Save changes**, then **Test connection**. You should see the display number and quality rating.
3. In the Meta developer dashboard: *WhatsApp > Configuration > Webhook*:
   - Callback URL: the Web app URL (`/exec`)
   - Verify token: the same token as above
   - Click **Verify and save**, then subscribe to the **messages** field.
4. Turn **Demo mode** off in Settings. You will be asked to confirm, because demo contacts, demo messages and demo campaigns are deleted when going live.
5. Click **Refresh from Meta** on the Templates page. Only `APPROVED` templates can be sent.
6. Use **Send test message** (for example the built-in `hello_world` template) to your own number first.

## 5. Test checklist

### A. Backend and access
- [ ] `runSelfTest` shows no `FAIL` lines
- [ ] Login with a wrong password 5 times shows the 15-minute lockout message
- [ ] Login with the right password works after the lock expires (or after the lock is cleared)
- [ ] Change password works and other sessions are signed out
- [ ] Opening the web app URL in a browser returns the health JSON

### B. Demo mode (no Meta needed)
- [ ] Settings shows Demo mode on and the DEMO badge is visible in the top bar
- [ ] Import the [sample CSV](#6-sample-contacts-csv): the report shows invalid, duplicate and valid rows as described
- [ ] Contacts table search, filters and pagination work
- [ ] New campaign wizard: audience preview, template preview, schedule, review with cost and the consent checkbox, confirm dialog
- [ ] Campaign moves Sending > Completed and the progress bar fills
- [ ] Dashboard KPIs and the 7-day chart update
- [ ] A few simulated replies appear, including one `STOP` tagged OPT-OUT
- [ ] The STOP number appears in Opt-outs and its queued messages are skipped
- [ ] Pause, Resume, Cancel and Duplicate work from the Campaigns table
- [ ] Removing an opt-out requires typing `DELETE`
- [ ] Dark mode and the mobile layout look right
- [ ] Activity log shows logins, imports, campaign actions and settings changes

### C. Live mode (with real Meta credentials)
- [ ] **Test connection** succeeds
- [ ] Webhook verification succeeds in Meta
- [ ] Templates list matches WhatsApp Manager and approval statuses are right
- [ ] **Send test message** reaches your own phone
- [ ] Tiny campaign (2 to 3 of your own numbers): messages move Accepted > Sent > Delivered > Read
- [ ] Reply from your phone appears in Replies
- [ ] Reply `STOP` from your phone: number lands in Opt-outs, further sends are skipped
- [ ] A bad number or a template error shows the Meta error code and message in the row
- [ ] Campaign larger than the daily limit continues automatically on the next day
- [ ] Day-after check: purge ran (Activity log has a `retention.purge` entry once the retention period has passed)

## 6. Sample contacts CSV

Save as `sample-contacts.csv`. The `DEMO sample` consent source keeps these rows out of live sending, and they are removed when you switch Demo mode off. The last four rows are deliberately bad, to test the validation report.

```csv
phone,first_name,last_name,email,custom1,tags,opted_in,consent_source,consent_date
0835550101,Aoife,Murphy,aoife@example.com,Deep clean,"pilot,dublin",YES,DEMO sample,2026-10-01
+353835550102,Ciaran,Kelly,ciaran@example.com,Carpet clean,pilot,YES,DEMO sample,2026-10-02
353835550103,Niamh,Walsh,niamh@example.com,Deep clean,"pilot,vip",YES,DEMO sample,2026-10-03
00353835550104,Sean,Byrne,sean@example.com,Window clean,cork,YES,DEMO sample,03/10/2026
083 555 0105,Orla,Doyle,orla@example.com,Deep clean,pilot,YES,DEMO sample,2026-10-05
0835550106,Declan,Ryan,declan@example.com,Carpet clean,pilot,NO,DEMO sample,2026-10-06
12345,Bad,Number,bad@example.com,,pilot,YES,DEMO sample,2026-10-07
0835550107,No,Consent,noconsent@example.com,,pilot,YES,,2026-10-08
0835550108,Future,Date,future@example.com,,pilot,YES,DEMO sample,2099-01-01
0835550101,Aoife,Duplicate,dup@example.com,,pilot,YES,DEMO sample,2026-10-01
```

Required columns: `phone`, `consent_source`, `consent_date` (`YYYY-MM-DD` or `DD/MM/YYYY`). In the import wizard you can also supply one consent source or date for all rows instead of a column. Irish local numbers starting with `0` are converted to `+353`. Tags are comma separated. `opted_in` defaults to `YES` when the column is empty or missing.

Expected result: 6 new contacts (one with `opted_in = NO`, so it will not receive campaigns), 3 invalid rows, 1 duplicate.

## 7. Automation API (ingest_event)

Phase 2 is built in the backend already. External systems (booking system, HubSpot workflow, Zapier or Make) call `ingest_event` with the **API key** (not a session token). The dashboard keeps Automations disabled until you set `FEATURES.automations = true`.

The key is shown masked in Settings. Run `showWebhookInfo` for the full value, or click **Generate new key** (the new key is shown once).

```bash
curl -L "https://script.google.com/macros/s/XXXX/exec" \
  -H "Content-Type: text/plain" \
  -d '{
    "action": "ingest_event",
    "api_key": "wak_...",
    "payload": {
      "phone": "0871234567",
      "template_name": "appointment_reminder",
      "language": "en",
      "variables": ["10:00", "Paul"],
      "send_at": "2026-11-10T09:00:00Z",
      "idempotency_key": "booking-1234-reminder"
    }
  }'
```

Do not add `-X POST` together with `-L`: Apps Script answers with a redirect and curl must switch to GET for the second hop.

- **Batch:** send `"payload": {"events": [ {...}, {...} ]}` (up to 200 events per call).
- **Idempotency:** the same `idempotency_key` is never queued twice (`status: "duplicate"`).
- **Day-before reminders** are just events with a future `send_at`. The queue sends them when due.
- **Consent rules:** the number must already be an opted-in contact. If it is not in Contacts, the event must include `consent_source` and a valid `consent_date`, which creates the contact. Suppressed numbers are always rejected.
- Optional fields: `header_image_url` (https), `button_var`, `url_index`, `first_name`, `last_name`, `email`.
- Events use the same pipeline as campaigns: daily limit, duplicate protection, retry rules, status tracking.

Response: `{ "ok": true, "data": { "queued": 1, "results": [ { "idempotency_key": "...", "status": "queued", "send_at": "..." } ] } }`.

## 8. Security notes (read this)

- **Webhook signature cannot be verified.** Apps Script web apps cannot read request headers, so Meta's `X-Hub-Signature-256` is not checked. Mitigation: status updates only change rows whose WhatsApp message ID is already known, unknown payloads are ignored, the `phone_number_id` in the payload must match the configured one, and replies are de-duplicated by message ID. If the client needs signature verification, put a small relay (for example a Cloudflare Worker) in front that verifies the signature and forwards to the web app.
- **Secrets** (access token, IDs, verify token, API key, password hash) live only in Apps Script **Script Properties**. The API returns them masked. Nothing secret is in `index.html` or the repository.
- **Session tokens** are random, stored hashed in CacheService, and expire after 6 hours. Changing the password signs out every other session.
- **Login lockout:** 5 failed attempts lock the login for 15 minutes.
- The web app runs as the account owner and is reachable by anyone with the URL. All dashboard actions except `login` and `ingest_event` require a valid session; `ingest_event` requires the API key.
- Contact data is escaped before display (XSS safe) and CSV exports guard against spreadsheet formula injection.
- Rotate the Meta token and API key if the Apps Script project is ever shared with someone who should not have them.

## 9. Quotas and how the design stays inside them

| Apps Script limit | How the design copes |
|---|---|
| 6 minutes per execution | `processQueue` stops after about 4.5 minutes; the next minute trigger continues |
| URL fetch calls per day (about 20,000 consumer, 100,000 Workspace) | One message = one call. A 5,000-message campaign is far below this |
| Trigger runtime per day (about 90 min consumer, 6 h Workspace) | When the queue is empty an idle flag makes the minute run almost free |
| CacheService 100 KB per value, 6 h TTL | Large values (template list) are chunked; sessions use the 6 h maximum |
| Script Properties 9 KB per value | Only small values are stored |
| Spreadsheet 10 million cells | The retention purge keeps `Messages` small |
| Concurrent writes | A script lock wraps every sheet write (30 s wait) |

Sheets are read in one batch and written in one batch (no per-cell loops). Sends run in parallel chunks of 20 with `UrlFetchApp.fetchAll`. Tables are paginated in both the API and the UI, so 5,000 contacts stay smooth.

## 10. Data retention and GDPR

- A daily trigger deletes rows containing phone numbers (`Contacts`, `Messages`, `Replies`) once **retention days** (default 15) have passed after the campaign completed.
- Kept: `Campaigns` (counts only, no phone numbers), `AuditLog` (phone-like numbers are masked), and `Suppression` (permanent: number, date, source). The purge never reads or writes `Suppression`.
- Every contact needs `consent_source` and `consent_date`. Contacts can be exported (CSV) and any number can be erased with **Erase number** (removes it from Contacts, Messages and Replies; the opt-out entry stays, so the person is never contacted again).
- The client remains the data controller. Make sure a privacy notice covers WhatsApp messaging and that a data processing agreement with Meta and Google is in place as required.

## 11. Assumptions and what to verify with real Meta credentials

**Assumptions**
- Demo mode is on after setup; switching it off removes demo data.
- Per-message rates (marketing 0.05, utility 0.01, authentication 0.01 EUR) are editable estimates, not Meta's official prices. Check the current Irish/EU rate card.
- The daily limit (default 250 unique recipients per rolling 24 hours) mirrors a new number's starting tier. Raise it in Settings only after Meta raises the number's messaging limit.
- Templates are created and approved in WhatsApp Manager. The dashboard only reads them.
- Template language codes in campaigns must match the approved template exactly (for example `en` vs `en_US`).

**Verify with real credentials**
1. Webhook verification and delivery: Apps Script answers `POST` requests with a redirect after running the code. Confirm that Meta shows successful deliveries and that statuses and replies arrive. Retries are safe because updates are idempotent.
2. Error code handling: the transient and permanent error code lists in `Code.gs` (`TRANSIENT_CODES`, `AUTH_CODES`) are based on Meta's documentation. Check them against the errors you actually see.
3. Image header templates: Meta must be able to fetch the image URL (public https, correct file type and size).
4. URL button templates with a dynamic suffix: confirm the button variable produces the right link.
5. Quick-reply templates: payload mapping is supported by the backend but not editable in the UI yet.
6. Quality rating and messaging tier on the Dashboard appear only with valid credentials.
7. The 24-hour customer service window and template category rules still apply: marketing content must use a marketing template.
8. Time zone: the script time zone must be `Europe/Dublin` for the daily purge and date logic.

## 12. Troubleshooting

| Problem | Fix |
|---|---|
| Login says it cannot reach the server | Check the web app URL, and that access is set to **Anyone** and the latest version is deployed |
| "Admin password is not set yet" | Run `runSetPassword` (step 2.5) |
| Dashboard warns that triggers are not installed | Run `setupTriggers` |
| Campaign stays in Sending with nothing going out | Check the daily limit warning, the Meta credentials, and `runSelfTest` output |
| Messages in **Needs review** | A send was interrupted. Check WhatsApp Manager before resending; these are never resent automatically |
| Webhook verification fails in Meta | Verify token mismatch, or the latest code is not deployed as a new version |
| Template not found when creating a campaign | Click **Refresh from Meta** and check the language code |
| "The system is busy" | Another write held the lock; try again in a few seconds |