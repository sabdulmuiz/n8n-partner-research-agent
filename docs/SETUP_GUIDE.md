# Setup Guide

Step-by-step setup for the Partner Research & Outreach Agent. Written for macOS and Chrome; on Windows, use PowerShell instead of Terminal and everything else is the same.

**Time needed:** about 1 hour the first time.

| Part | What you do |
| --- | --- |
| 1 | Install Node.js and start n8n |
| 2 | Build the Google Sheet |
| 3 | Get free API keys (Gemini, Serper, Hunter.io) |
| 4 | Set up Google Cloud so n8n can use your sheet |
| 5 | Import the workflow and edit Config |
| 6 | Connect every node |
| 7 | Test on one company, then run the list |
| 8 | Review the results |
| 9 | Troubleshooting |

---

## Part 1: Install Node.js and start n8n

1. Go to **nodejs.org**, download the **LTS** version and install it with the defaults.
2. Open **Terminal** and check:
   ```
   node -v
   npm -v
   ```
3. Start n8n:
   ```
   npx n8n
   ```
   Type `y` if asked to proceed. Yellow `npm warn` lines and deprecation notices are harmless. Wait for **"Editor is now accessible via: http://localhost:5678"**.
4. Open **http://localhost:5678** in **Chrome**. (Safari shows a "secure cookie" error; to use Safari, restart with `N8N_SECURE_COOKIE=false npx n8n`.)
5. Create your n8n owner account. It's stored on your machine only.

Keep Terminal open while n8n is running. Workflows and credentials are saved between sessions.

**Keeping your Mac awake during long runs:** in a second Terminal window, run `caffeinate -dimsu` and keep the lid open. Press Ctrl+C to stop it afterwards.

---

## Part 2: Build the Google Sheet

1. **sheets.google.com** → **Blank spreadsheet** → name it, e.g. **Partner Research Agent**.

**Targets** (your input)

2. Rename **Sheet1** to `Targets`.
3. **File → Import → Upload** → `sample-data/targets.csv` → **Replace current sheet** → **Import data**. (Or type the header row and your own targets.)

| Column | Required | Meaning |
| --- | --- | --- |
| `name` | yes | Organisation name |
| `website` | yes | Homepage URL |
| `domain` | recommended | Email domain, e.g. `google.com`. Worked out from the website if blank. |
| `linkedin_company` | recommended | Name as shown on LinkedIn, e.g. `Amazon Web Services` |
| `type` | optional | Short description, e.g. `Startup accelerator` |
| `contact_location` | recommended | Where the contact must be based, e.g. `United Arab Emirates`, `United Kingdom` |
| `contact_titles` | optional | Role keywords separated by `;`, e.g. `Partnerships; Startups` |

**Briefs** (agent output)

4. Click **+** → name the tab `Briefs` → paste into A1, then **Data → Split text to columns**:
   ```
   name,website,type,contact_location,summary,what_they_want,what_we_get,trade,priority,created
   ```

**Outreach** (agent output)

5. Click **+** → name the tab `Outreach` → paste into A1, then **Data → Split text to columns**:
   ```
   company,contact_name,title,location,hunter_location,linkedin_url,email,email_status,contact_check,direct_phone,office_phone,office_address,email_subject,email_body,linkedin_connection_note,linkedin_message,match_reason,status,created
   ```

6. Copy the sheet's URL from the address bar.

Tab names and headers must match exactly.

---

## Part 3: Get free API keys

Keep keys private. Never put them in the sheet, the workflow file or a public repository.

| Service | Steps |
| --- | --- |
| **Gemini** | aistudio.google.com → **Get API key → Create API key** |
| **Serper** | serper.dev → **Sign up** (no card) → copy the API key from the dashboard |
| **Hunter.io** | hunter.io → **Sign up** (no card) → **API** in the menu → copy the key |

---

## Part 4: Set up Google Cloud so n8n can use your sheet

1. **console.cloud.google.com** → project dropdown → **New Project** → name it, e.g. `n8n Agent` → **Create** → select it.
2. Search **Google Sheets API** → **Enable**. Search **Google Drive API** → **Enable**.
3. **APIs & Services → OAuth consent screen** (or **Google Auth Platform**) → **Get started**:
   - App name and support email → **Next**
   - Audience **External** → **Next**
   - Contact email → **Next** → agree → **Create**
4. **Audience → Test users → + Add users** → your Gmail → **Save**.

You'll create the OAuth client in Part 6.

---

## Part 5: Import the workflow and edit Config

1. In n8n: **Create Workflow** → **⋯** (top right) → **Import from File** → `workflow/partner_research_outreach_agent.json`.
2. Press **Shift+1** to fit the canvas. Red ⚠ marks are expected until Part 6 is done.
3. Double-click **Config** and edit the values between the quote marks:

| Setting | Meaning |
| --- | --- |
| `sender_name` | Your name; signs the drafts |
| `sender_role` | Your role, e.g. `Partnerships` |
| `sender_company` | Your organisation |
| `company_pitch` | 1–2 sentences on what your organisation does and wants from partners |
| `contacts_per_company` | People per company in Outreach (number, e.g. `2`) |
| `default_location` | Used when `contact_location` is blank |
| `default_titles` | Role keywords used when `contact_titles` is blank |
| `hunter_enrich` | `true` checks each contact's location with Hunter (0.2 credits each); `false` saves credits |

4. Close the panel and save (**Cmd+S**).

---

## Part 6: Connect every node

**6a. Google Sheets (4 nodes)**

Create the credential once:
1. Open **Read Targets** → **Credential to connect with** → **+ Create new credential** → **Google Sheets OAuth2 API**.
2. Copy the **OAuth Redirect URL** (`http://localhost:5678/rest/oauth2-credential/callback`).
3. In Google Cloud: **Clients → + Create client** (or **Credentials → Create credentials → OAuth client ID**) → **Web application** → **Authorized redirect URIs → + Add URI** → paste → **Create**.
4. Copy the **Client ID** and **Client secret** into n8n → **Sign in with Google** → continue past "Google hasn't verified this app" (it's your own app) → allow → **Save**.

Then, in each node: select the credential, set **Document** to **By URL** and paste your sheet URL, and check the tab.

| Node | Tab |
| --- | --- |
| Read Targets | `Targets` |
| Read Briefs | `Briefs` |
| Save to Briefs Sheet | `Briefs` |
| Save to Outreach Sheet | `Outreach` |

**6b. Gemini (3 nodes)**

In **Gemini Model (Brief)**, **Gemini Model (Pick)** and **Gemini Model (Outreach)**: create or select your Gemini credential and pick a model. The workflow ships with three different Flash-Lite models. Each Gemini model has its own free daily limit, so using a different one in each node lets you process more companies per day. If a model isn't listed for your key, choose any available Flash or Flash-Lite model.

**6c. Serper (1 node)**

Open **Serper Search** → **Credential for Header Auth** → **+ Create new credential**:
- **Name:** `X-API-KEY`
- **Value:** your Serper key

Rename the credential `Serper` and save.

**6d. Hunter.io (2 nodes)**

Open **Find Email (Hunter)** → **+ Create new credential**:
- **Name:** `X-API-KEY`
- **Value:** your Hunter key

Rename it `Hunter` and save. In **Enrich Person (Hunter)**, select the same **Hunter** credential.

Serper and Hunter use the same header name but **different keys**. Make sure each node uses the right credential.

Save the workflow. All red ⚠ marks should be gone.

---

## Part 7: Test on one company, then run the list

1. Keep only one target in **Targets** (move the rest to a spare tab).
2. Click **Execute workflow**. One company takes about 30–60 seconds.
3. Check **Briefs** and **Outreach**. To see what happened at each step, click a node: **Collect Results** shows what Google found, **Pick People** shows who the AI chose and why.
4. Move the other targets back and run again. Companies already in **Briefs** are skipped automatically.

To re-run a company, delete its row in **Briefs** and its rows in **Outreach**.

**Free usage per company:** Serper 3 searches; Hunter about 1 credit per email found plus 0.2 per location check; Gemini 2 calls plus 1 per contact.

---

## Part 8: Review the results

**Tidy the Outreach tab:** wrap the draft columns (**Format → Wrapping → Wrap**), freeze the header row (**View → Freeze → 1 row**), and add a dropdown to `status` (**Data → Data validation → Dropdown**: Draft, Sent, Replied, Meeting booked, Not a fit).

**Check each contact before reaching out:**
1. Open the `linkedin_url` and confirm the person, role and location are current.
2. Read `match_reason` (why the AI chose them) and `contact_check`:

| `contact_check` | Meaning | Action |
| --- | --- | --- |
| Verified: LinkedIn (+ location) match | Hunter links the email to the same LinkedIn profile | Safe to email |
| Likely: location / title matches | Other details agree | Glance at the profile |
| Check: different LinkedIn / location / title | The email may belong to someone else | Verify first, or use LinkedIn |
| Unverified: no Hunter data to compare | Nothing to cross-check | Look at the profile before emailing |
| No email found | Hunter had no address | Use the LinkedIn drafts |

3. Edit the drafts before sending. AI drafts can repeat phrases or invent specifics such as dates.
4. Send LinkedIn messages by hand; never use automation tools on LinkedIn.

---

## Part 9: Troubleshooting

Click any red node to see its error.

| What you see | Fix |
| --- | --- |
| "Sheet with name … not found" | Tab names must be exactly `Targets`, `Briefs`, `Outreach`. |
| Rows appear with blank or extra columns | Header row must match Part 2 exactly. |
| Gemini 404 "model not found / no longer available" | Choose another Flash or Flash-Lite model in that node. |
| Gemini 429 "quota exceeded" | That model's free daily limit is used up: choose a different model or wait for the daily reset. Finished companies are skipped on the next run. |
| Briefs show "Could not parse AI output" | Delete those Briefs rows and run again. |
| Serper or Hunter 401 / 403 | The node is using the wrong credential, or the key is wrong. Header name must be `X-API-KEY`. |
| Hunter 429 / usage limit | Monthly free credits used: set `hunter_enrich` to `false` or `contacts_per_company` to `1`, or wait for the reset. |
| Brief written but no Outreach row | No current employee in that location appeared in Google's LinkedIn results. Check **Collect Results**; try broader `contact_titles`, check `linkedin_company`, or use a city in `contact_location`. |
| Email blank, `contact_check` "No email found" | Check `domain` is the company's email domain (e.g. `amazon.com` for AWS). |
| `office_phone` blank | No Google Maps listing for that company in that location. |
| Run stops after the first company | **Pause (rate limit)** must connect back into the left side of **Loop Over Items**, with its unit set to seconds. |
| "404 Could not load the workflow" in Terminal | An old browser tab is pointing at a deleted workflow. Close old n8n tabs; it's harmless. |
