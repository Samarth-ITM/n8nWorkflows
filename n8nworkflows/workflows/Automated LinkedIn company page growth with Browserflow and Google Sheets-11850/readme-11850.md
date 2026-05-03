Automated LinkedIn company page growth with Browserflow and Google Sheets

https://n8nworkflows.xyz/workflows/automated-linkedin-company-page-growth-with-browserflow-and-google-sheets-11850


# Automated LinkedIn company page growth with Browserflow and Google Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow title:** Automated LinkedIn company page growth with Browserflow and Google Sheets  
**Workflow name (in JSON):** Grow Linkedin Company Page

**Purpose:**  
Automate LinkedIn Company Page growth by (1) scraping engaged users from selected LinkedIn posts (comments), (2) inviting them as connections, (3) tracking acceptance, and (4) inviting accepted connections to follow a Company Page—using **Browserflow** automation and **Google Sheets** as a campaign database.

**Primary use cases:**
- Build a lead list from LinkedIn post engagement (commenters) at scale.
- Run an outreach pipeline where Google Sheets stores progression states.
- Periodically reconcile statuses (pending/accepted) and trigger the next step.

### 1.1 Lead Scraping & Storage (manual run)
- Reads unscraped post URLs from Google Sheets.
- Scrapes commenters’ profiles using Browserflow.
- Filters out non-person profiles (e.g., company pages).
- Deduplicates against an existing “Profiles” sheet.
- Appends new leads and marks posts as scraped.

### 1.2 Connection Invites (scheduled)
- Pulls leads not yet invited / not accepted from the “Profiles” sheet.
- Checks connection status on LinkedIn via Browserflow.
- Sends connection invites when appropriate.
- Updates the sheet with “pending” or “accepted” signals.

### 1.3 Acceptance Tracking (scheduled)
- Retrieves your latest LinkedIn connections via Browserflow.
- Normalizes profile URLs.
- Updates the “accepted” status in Google Sheets.

### 1.4 Invite Connected Leads to Follow Company Page (scheduled)
- Pulls accepted leads not yet invited to the page.
- Invites them to follow a specified LinkedIn Company Page.
- Marks “invited_to_page” in the sheet.

---

## 2. Block-by-Block Analysis

### Block 1 — Lead scraping from post comments (Manual stage)

**Overview:**  
This block is manually triggered to scrape commenters from LinkedIn posts listed in Google Sheets and store new profiles in the “Profiles” sheet while marking processed posts to avoid re-scraping.

**Nodes involved:**
- When clicking ‘Execute workflow’
- Fetch Posts to Scrape
- Loop Over Items
- Scrape comments from Post
- Split Out
- Filter
- Loop Over Items1
- Check if already collected
- If not in sheet
- Collect leads in sheet
- Mark Post as scraped

#### Node: When clicking ‘Execute workflow’
- **Type / role:** Manual Trigger; entry point for scraping stage.
- **Config:** No parameters.
- **Outputs:** To **Fetch Posts to Scrape**.
- **Edge cases:** None (user-driven).

#### Node: Fetch Posts to Scrape
- **Type / role:** Google Sheets node; fetch posts that have not been scraped yet.
- **Config choices:**
  - **Operation:** (implicit) “Read/Get many” based on filters.
  - **Sheet:** “Posts” (gid=0) in document “Demo - Invite To Follow”.
  - **Filter:** `scraped_at =` (empty) → returns rows where scraped_at is blank.
- **Key fields used downstream:** `url` (LinkedIn post URL).
- **Credentials:** Google Sheets OAuth2.
- **Failure modes:** OAuth expiry, wrong spreadsheet ID, missing columns (`url`, `scraped_at`).

#### Node: Loop Over Items
- **Type / role:** Split In Batches; iterates through post rows.
- **Config:** Default options (batch size not explicitly set; n8n defaults apply).
- **Connections:**
  - Input: **Fetch Posts to Scrape**
  - Output (index 1): **Scrape comments from Post**
  - Output (index 0): unused in this workflow branch
- **Edge cases:** Empty input → nothing to scrape.

#### Node: Scrape comments from Post
- **Type / role:** Browserflow community node; scrapes profiles from post comments.
- **Operation:** `scrapeProfilesFromPostComments`
- **Config choices:**
  - `postUrl = {{ $('Fetch Posts to Scrape').item.json.url }}`
  - `addComments: true`
  - `commentsLimit: 20`
- **Output structure (assumed by following nodes):** includes a `comments` array where each element contains `name`, `linkedin_url`, etc.
- **Credentials:** Browserflow API.
- **Failure modes:** LinkedIn automation failures, Browserflow API errors, rate limits, invalid post URL, changed LinkedIn UI.

#### Node: Split Out
- **Type / role:** Split Out; converts `comments[]` array into one item per comment/profile.
- **Config:** `fieldToSplitOut: comments`
- **Connections:** to **Filter**
- **Edge cases:** Missing `comments` field or empty array → produces no items.

#### Node: Filter
- **Type / role:** Filter node; excludes company profiles.
- **Condition:** `linkedin_url` **does not contain** `"company"`.
- **Rationale:** Avoid storing company pages as leads.
- **Edge cases:** If `linkedin_url` is missing/null, filter evaluation could behave unexpectedly depending on n8n’s strict validation; ensure scraper always outputs it.

#### Node: Loop Over Items1
- **Type / role:** Split In Batches; iterates through each scraped profile item.
- **Connections:**
  - Output (index 0): **Mark Post as scraped**
  - Output (index 1): **Check if already collected**
- **Important nuance:** Because it also routes to “Mark Post as scraped”, this block marks the post as scraped during profile iteration (not just once at end). It still “works” but creates redundant updates.
- **Edge cases:** Large lists can trigger many “mark scraped” calls.

#### Node: Check if already collected
- **Type / role:** Google Sheets; lookup lead by LinkedIn URL.
- **Config:**
  - Sheet: “Profiles”
  - Filter: `linkedin_url = {{ $('Loop Over Items1').item.json.linkedin_url }}`
  - **alwaysOutputData: true** (so the workflow continues even if no row is found)
- **Expected behavior:** If found, node outputs existing row(s); if not found, likely outputs empty but continues.
- **Failure modes:** Column mismatch (`linkedin_url`), credential issues.

#### Node: If not in sheet
- **Type / role:** IF; decides whether to append new lead.
- **Condition:** String operator `notExists` on `{{ $json.linkedin_url }}`.
  - **Interpretation:** This relies on what the previous Google Sheets node returns when no match is found.
  - **Risk:** Many Google Sheets “Read” operations return *zero items* when not found; with `alwaysOutputData`, output may still be a single item without `linkedin_url`, which makes `notExists` true. If the node returns found rows, `linkedin_url` exists and condition fails.
- **True path:** **Collect leads in sheet**
- **False path:** back to **Loop Over Items1** (continue)
- **Edge cases:** If Google Sheets returns multiple rows for same URL, dedupe logic may behave inconsistently.

#### Node: Collect leads in sheet
- **Type / role:** Google Sheets; append a new lead row.
- **Operation:** Append
- **Mapped columns:**
  - `name = {{ $('Loop Over Items1').item.json.name }}`
  - `linkedin_url = {{ $('Loop Over Items1').item.json.linkedin_url }}`
- **Failure modes:** Missing name/url, sheet permissions, quota limits.

#### Node: Mark Post as scraped
- **Type / role:** Google Sheets; update the Posts sheet to prevent re-scraping.
- **Operation:** Update (match on `url`)
- **Mapped columns:**
  - `url = {{ $('Loop Over Items').item.json.url }}`
  - `scraped_at = {{ $now.toISO() }}`
- **Matching columns:** `url`
- **Connections:** loops back into **Loop Over Items** to fetch the next post batch iteration.
- **Failure modes:** If URLs differ by trailing slash / tracking params, match may fail and not update.

---

### Block 2 — Invite leads as LinkedIn connections (Scheduled stage)

**Overview:**  
Every 2 days, this block fetches leads that are not yet invited and not yet accepted, checks their LinkedIn connection status, sends invites when possible, and updates Google Sheets accordingly.

**Nodes involved:**
- Schedule Trigger
- Fetch Leads that have not been invited
- Loop over leads
- Check if a person is a connection
- If
- Send a linked in connection invite
- Invite Sent
- If1
- Pending
- Already connected

#### Node: Schedule Trigger
- **Type / role:** Schedule Trigger; entry point for connection-invite stage.
- **Config:** Every `2 days`.
- **Edge cases:** Timezone considerations (n8n instance timezone), missed runs during downtime.

#### Node: Fetch Leads that have not been invited
- **Type / role:** Google Sheets; selects candidates for inviting.
- **Config:**
  - Sheet: “Profiles”
  - Filters specify `invited_as_connection` and `accepted` with no explicit lookupValue → effectively “empty” filtering (depends on node behavior/UI).
  - Intended: rows where `invited_as_connection` is blank AND `accepted` is blank.
- **Risk:** If filter semantics aren’t “is empty”, it may return wrong rows. Validate in n8n UI after import.
- **Outputs:** to **Loop over leads**.

#### Node: Loop over leads
- **Type / role:** Split In Batches; iterates lead rows for inviting.
- **Connections:** Output (index 1) → **Check if a person is a connection**; output (index 0) unused.
- **Edge cases:** Batch sizing / rate limiting not set; may hit LinkedIn limits if too many.

#### Node: Check if a person is a connection
- **Type / role:** Browserflow; checks relationship status.
- **Config:** `linkedinUrl = {{ $json.linkedin_url }}`
- **Expected output fields:** `is_connection` (boolean), `is_pending` (boolean)
- **Failure modes:** UI automation errors, invalid profile URL, LinkedIn rate limits.

#### Node: If
- **Type / role:** IF; determines whether to send invite.
- **Conditions (AND):**
  - `is_connection == false`
  - `is_pending == false`
- **True path:** **Send a linked in connection invite**
- **False path:** **If1** (handles pending/connected outcomes)

#### Node: Send a linked in connection invite
- **Type / role:** Browserflow; sends a connection invite.
- **Operation:** `sendConnectionInvite`
- **Config:** `linkedinUrl = {{ $('Loop over leads').item.json.linkedin_url }}`
- **Failure modes:** invite limit reached, restricted accounts, UI flow changes.

#### Node: Invite Sent
- **Type / role:** Google Sheets; update the lead to indicate invite was sent.
- **Operation:** Update by `linkedin_url`
- **Fields:**
  - `invited_as_connection = {{ $now }}`
  - `linkedin_url = {{ $('Loop over leads').item.json.linkedin_url }}`
- **Connections:** back to **Loop over leads** to continue.
- **Failure modes:** update match failure due to URL normalization differences (with/without trailing slash).

#### Node: If1
- **Type / role:** IF; handles case where the person is pending (based on status check).
- **Condition:** `{{ $('Check if a person is a connection').item.json.is_pending }} == true`
- **True path:** **Pending**
- **False path:** **Already connected**
- **Important nuance:** “False” branch assumes not pending implies already connected, but it could also mean “unknown” or scraper failure. Consider adding error handling.

#### Node: Pending
- **Type / role:** Google Sheets; record “pending” invite state.
- **Operation:** Update by `linkedin_url`
- **Fields:** `invited_as_connection = {{ $now }}`
- **Connections:** back to **Loop over leads**
- **Edge cases:** Could overwrite previous timestamp even if it was set earlier.

#### Node: Already connected
- **Type / role:** Google Sheets; record “accepted/already connected”.
- **Operation:** Update by `linkedin_url`
- **Fields:**
  - `accepted = ✅`
  - `invited_as_connection = =` (literal equals sign; likely a mistake—intended to be blank)
- **Connections:** back to **Loop over leads**
- **Edge cases:** The `invited_as_connection` value `"="` can pollute filtering logic later.

---

### Block 3 — Update connection acceptance status (Scheduled stage)

**Overview:**  
Every 2 days, the workflow fetches your recent LinkedIn connections, normalizes profile URLs, then updates Google Sheets to mark matching leads as accepted.

**Nodes involved:**
- Schedule Trigger1
- List your linkedin connections
- Split out over connections
- Code in JavaScript
- Update connection status

#### Node: Schedule Trigger1
- **Type / role:** Schedule Trigger; entry point for acceptance-tracking stage.
- **Config:** Every 2 days.

#### Node: List your linkedin connections
- **Type / role:** Browserflow; fetches a list of your connections.
- **Operation:** `listConnections`
- **Config:** `limit: 20`
- **Output expected:** object containing `data` array with each connection including `linkedin_url`.
- **Failure modes:** automation/rate limit; limit too low may miss acceptances.

#### Node: Split out over connections
- **Type / role:** Split Out; expands `data[]` to items.
- **Config:** `fieldToSplitOut: data`
- **Edge cases:** If Browserflow returns a different field name, downstream breaks.

#### Node: Code in JavaScript
- **Type / role:** Code node; URL normalization.
- **Logic:** Ensures each `linkedin_url` ends with a trailing `/` by overwriting the field.
- **Why it matters:** Google Sheets matching later uses `linkedin_url`; inconsistent trailing slashes cause update misses.
- **Failure modes:** If `linkedin_url` missing, sets it to `"/"` (because `"" + "/"`). Consider guarding against empty URLs.

#### Node: Update connection status
- **Type / role:** Google Sheets; mark leads as accepted.
- **Operation:** Update by `linkedin_url`
- **Fields:**
  - `accepted = ✅`
  - `linkedin_url = {{ $json.linkedin_url }}`
- **Edge cases:** Only updates rows already in Profiles. If URLs aren’t normalized consistently across all pipeline steps, updates won’t match.

---

### Block 4 — Invite connected leads to follow the company page (Scheduled stage)

**Overview:**  
Every 2 days, pulls accepted leads not yet invited to the company page, invites them via Browserflow, and marks them as invited in Google Sheets.

**Nodes involved:**
- Schedule Trigger2
- Get connected leads
- Loop over leads2
- Invite connections to follow page
- Update invited to page

#### Node: Schedule Trigger2
- **Type / role:** Schedule Trigger; entry point for page-invite stage.
- **Config:** Every 2 days.

#### Node: Get connected leads
- **Type / role:** Google Sheets; selects leads eligible for company page invite.
- **Config:**
  - Filter: `accepted = ✅`
  - Filter on `invited_to_page` blank (configured with lookupColumn only; validate behavior)
- **Output:** to **Loop over leads2**

#### Node: Loop over leads2
- **Type / role:** Split In Batches; iterates accepted leads.
- **Connections:** Output (index 1) → **Invite connections to follow page**
- **Edge cases:** Potentially invites too many too fast if batches are large; LinkedIn limits.

#### Node: Invite connections to follow page
- **Type / role:** Browserflow; invites a connection to follow a company page.
- **Operation:** `inviteToFollowPage`
- **Config:**
  - `searchTerm = {{ $('Get connected leads').item.json.name }}`
  - `linkedinUrl = https://linkedin.com/company/browserflow-io` (hardcoded; must be replaced)
  - `maxToInvite: 1`
- **Important nuance:** Uses **name search**, not direct profile URL—names can be ambiguous, leading to wrong invites or failures.
- **Failure modes:** Not admin of the page, UI changes, search mismatch, rate limits.

#### Node: Update invited to page
- **Type / role:** Google Sheets; marks lead as invited to page.
- **Operation:** Update by `linkedin_url`
- **Fields:**
  - `invited_to_page = ✅`
  - `accepted = ""` (this clears accepted status—likely unintended)
  - `linkedin_url = {{ $('Loop over leads2').item.json.linkedin_url }}`
- **Risk:** Clearing `accepted` will remove the lead from future “accepted” queries and may break reporting. Consider leaving `accepted` unchanged.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point for scraping stage | — | Fetch Posts to Scrape | ## Retrieve Leads from Post comments\nThis workflow gets leads by scraping people that reacted under a post ands stores them in Google Sheets |
| Fetch Posts to Scrape | Google Sheets | Fetch posts where `scraped_at` is empty | When clicking ‘Execute workflow’ | Loop Over Items | ## Retrieve Leads from Post comments\nThis workflow gets leads by scraping people that reacted under a post ands stores them in Google Sheets |
| Loop Over Items | Split In Batches | Iterate posts to scrape | Fetch Posts to Scrape; Mark Post as scraped | Scrape comments from Post | ## Retrieve Leads from Post comments\nThis workflow gets leads by scraping people that reacted under a post ands stores them in Google Sheets |
| Scrape comments from Post | Browserflow (community) | Scrape commenter profiles from a LinkedIn post | Loop Over Items | Split Out | ## Retrieve Leads from Post comments\nThis workflow gets leads by scraping people that reacted under a post ands stores them in Google Sheets |
| Split Out | Split Out | Expand `comments[]` into items | Scrape comments from Post | Filter | ## Retrieve Leads from Post comments\nThis workflow gets leads by scraping people that reacted under a post ands stores them in Google Sheets |
| Filter | Filter | Remove company profiles by URL check | Split Out | Loop Over Items1 | ## Retrieve Leads from Post comments\nThis workflow gets leads by scraping people that reacted under a post ands stores them in Google Sheets |
| Loop Over Items1 | Split In Batches | Iterate scraped profiles | Filter; Collect leads in sheet; If not in sheet | Mark Post as scraped; Check if already collected | ## Retrieve Leads from Post comments\nThis workflow gets leads by scraping people that reacted under a post ands stores them in Google Sheets |
| Check if already collected | Google Sheets | Check if profile already exists in “Profiles” | Loop Over Items1 | If not in sheet | ## Retrieve Leads from Post comments\nThis workflow gets leads by scraping people that reacted under a post ands stores them in Google Sheets |
| If not in sheet | IF | Branch: append lead if not found | Check if already collected | Collect leads in sheet; Loop Over Items1 | ## Retrieve Leads from Post comments\nThis workflow gets leads by scraping people that reacted under a post ands stores them in Google Sheets |
| Collect leads in sheet | Google Sheets | Append new lead to “Profiles” | If not in sheet | Loop Over Items1 | ## Retrieve Leads from Post comments\nThis workflow gets leads by scraping people that reacted under a post ands stores them in Google Sheets |
| Mark Post as scraped | Google Sheets | Update `scraped_at` for the post URL | Loop Over Items1 | Loop Over Items | ## Retrieve Leads from Post comments\nThis workflow gets leads by scraping people that reacted under a post ands stores them in Google Sheets |
| Schedule Trigger | Schedule Trigger | Start connection-invite stage every 2 days | — | Fetch Leads that have not been invited | ## Invite Leads as a LinkedIn Connection\nThis part of the workflow grabs the leads, checks for their connection status and invites them if needed. It also maintains the most up-2-date status in the Sheet. |
| Fetch Leads that have not been invited | Google Sheets | Select leads not invited/not accepted | Schedule Trigger | Loop over leads | ## Invite Leads as a LinkedIn Connection\nThis part of the workflow grabs the leads, checks for their connection status and invites them if needed. It also maintains the most up-2-date status in the Sheet. |
| Loop over leads | Split In Batches | Iterate leads for connection invites | Fetch Leads that have not been invited; Invite Sent; Pending; Already connected | Check if a person is a connection | ## Invite Leads as a LinkedIn Connection\nThis part of the workflow grabs the leads, checks for their connection status and invites them if needed. It also maintains the most up-2-date status in the Sheet. |
| Check if a person is a connection | Browserflow (community) | Check if lead is already connected/pending | Loop over leads | If | ## Invite Leads as a LinkedIn Connection\nThis part of the workflow grabs the leads, checks for their connection status and invites them if needed. It also maintains the most up-2-date status in the Sheet. |
| If | IF | If not connected and not pending → send invite | Check if a person is a connection | Send a linked in connection invite; If1 | ## Invite Leads as a LinkedIn Connection\nThis part of the workflow grabs the leads, checks for their connection status and invites them if needed. It also maintains the most up-2-date status in the Sheet. |
| Send a linked in connection invite | Browserflow (community) | Send LinkedIn connection invite | If | Invite Sent | ## Invite Leads as a LinkedIn Connection\nThis part of the workflow grabs the leads, checks for their connection status and invites them if needed. It also maintains the most up-2-date status in the Sheet. |
| Invite Sent | Google Sheets | Mark invited_as_connection timestamp | Send a linked in connection invite | Loop over leads | ## Invite Leads as a LinkedIn Connection\nThis part of the workflow grabs the leads, checks for their connection status and invites them if needed. It also maintains the most up-2-date status in the Sheet. |
| If1 | IF | If pending → Pending else Already connected | If | Pending; Already connected | ## Invite Leads as a LinkedIn Connection\nThis part of the workflow grabs the leads, checks for their connection status and invites them if needed. It also maintains the most up-2-date status in the Sheet. |
| Pending | Google Sheets | Update lead as “pending” | If1 | Loop over leads | ## Invite Leads as a LinkedIn Connection\nThis part of the workflow grabs the leads, checks for their connection status and invites them if needed. It also maintains the most up-2-date status in the Sheet. |
| Already connected | Google Sheets | Mark accepted for already-connected | If1 | Loop over leads | ## Invite Leads as a LinkedIn Connection\nThis part of the workflow grabs the leads, checks for their connection status and invites them if needed. It also maintains the most up-2-date status in the Sheet. |
| Schedule Trigger1 | Schedule Trigger | Start acceptance-tracking every 2 days | — | List your linkedin connections | ## Update Connection Status\nFetch your recent connections to check which leads have accepted your connection invite. |
| List your linkedin connections | Browserflow (community) | Fetch recent connections | Schedule Trigger1 | Split out over connections | ## Update Connection Status\nFetch your recent connections to check which leads have accepted your connection invite. |
| Split out over connections | Split Out | Expand `data[]` into items | List your linkedin connections | Code in JavaScript | ## Update Connection Status\nFetch your recent connections to check which leads have accepted your connection invite. |
| Code in JavaScript | Code | Normalize linkedin_url with trailing slash | Split out over connections | Update connection status | ## Update Connection Status\nFetch your recent connections to check which leads have accepted your connection invite. |
| Update connection status | Google Sheets | Update accepted=✅ by linkedin_url | Code in JavaScript | — | ## Update Connection Status\nFetch your recent connections to check which leads have accepted your connection invite. |
| Schedule Trigger2 | Schedule Trigger | Start page-invite stage every 2 days | — | Get connected leads | ## Invite connections to follow page\nInvite your LinkedIn Connections to Follow your page. Make sure to set the LinkedIn url of the page you are admin to |
| Get connected leads | Google Sheets | Select accepted leads not invited_to_page | Schedule Trigger2 | Loop over leads2 | ## Invite connections to follow page\nInvite your LinkedIn Connections to Follow your page. Make sure to set the LinkedIn url of the page you are admin to |
| Loop over leads2 | Split In Batches | Iterate accepted leads | Get connected leads; Update invited to page | Invite connections to follow page | ## Invite connections to follow page\nInvite your LinkedIn Connections to Follow your page. Make sure to set the LinkedIn url of the page you are admin to |
| Invite connections to follow page | Browserflow (community) | Invite connection to follow Company Page | Loop over leads2 | Update invited to page | ## Invite connections to follow page\nInvite your LinkedIn Connections to Follow your page. Make sure to set the LinkedIn url of the page you are admin to |
| Update invited to page | Google Sheets | Mark invited_to_page=✅ (and clears accepted) | Invite connections to follow page | Loop over leads2 | ## Invite connections to follow page\nInvite your LinkedIn Connections to Follow your page. Make sure to set the LinkedIn url of the page you are admin to |
| Sticky Note | Sticky Note | Comment block | — | — | ## Retrieve Leads from Post comments\nThis workflow gets leads by scraping people that reacted under a post ands stores them in Google Sheets |
| Sticky Note1 | Sticky Note | Comment block | — | — | ## Invite Leads as a LinkedIn Connection\nThis part of the workflow grabs the leads, checks for their connection status and invites them if needed. It also maintains the most up-2-date status in the Sheet. |
| Sticky Note2 | Sticky Note | Comment block | — | — | ## Update Connection Status\nFetch your recent connections to check which leads have accepted your connection invite. |
| Sticky Note3 | Sticky Note | Comment block | — | — | ## Invite connections to follow page\nInvite your LinkedIn Connections to Follow your page. Make sure to set the LinkedIn url of the page you are admin to |
| Sticky Note4 | Sticky Note | Global workflow description/setup | — | — | ## Fully Automated LinkedIn Company Page Growth\n\n### How it works\n\nThis workflow automates LinkedIn Company Page growth using\n**[Browserflow](https://browserflow.io)** and **Google Sheets**.\n\nIt runs in four scheduled stages:\n\n1. **Lead scraping**  \n   Users who engage with selected LinkedIn posts are scraped (commenters and optionally likers). Profiles are cleaned, deduplicated, and stored in Google Sheets. Posts are marked as scraped to avoid reprocessing.\n\n2. **Connection invites**  \n   New leads are checked for their current connection status. If not connected, a LinkedIn invite is sent. Pending and already-connected profiles are logged.\n\n3. **Acceptance tracking**  \n   Accepted invitations are detected by checking your LinkedIn connections and syncing updates back to the sheet.\n\n4. **Company Page invites**  \n   Once connected, leads are automatically invited to follow your Company Page (requires admin access).\n\nGoogle Sheets is used as a database so you can manage the status of your outreach campaign.\n\n---\n\n### Setup steps\n\n1. Import this template into your worlkflow.\n2. Install the **Browserflow for LinkedIn** community node.\n3. Connect **Browserflow** using your API key (you can sign up for a free trial at\n   **[Browserflow](https://browserflow.io)**).\n4. Make a copy of the provided\n   **[Google Sheets template](https://docs.google.com/spreadsheets/d/1-zak-RUGU4ubw3aZ_9lF9LF1dxEo9ME_-mtzRGtIDFg/edit?gid=0#gid=0)**.\n5. Update all Google Sheets nodes to point to your own copy  \n   *(Pro tip: you can use the n8n AI assistant for this)*.\n6. Enter your LinkedIn Company Page URL in the **Invite Connections to Follow Page** action.\n7. (Optional) Adjust schedule intervals.\n8. Find some posts on LinkedIn and add its urls to your Google Sheets to start scraping!\n\nI would first recommend running each flow independently to see how it works in actiona and finally enable the workflow and let it run automatically. |

---

## 4. Reproducing the Workflow from Scratch

### Prerequisites
1. **Install the Browserflow community node** (Browserflow for LinkedIn) in your n8n instance.
2. Create credentials:
   - **Google Sheets OAuth2** credential with access to your spreadsheet.
   - **Browserflow API** credential (API key from **https://browserflow.io**).
3. Prepare Google Sheets with (at minimum) two tabs:
   - **Posts** sheet with columns: `url`, `scraped_at`
   - **Profiles** sheet with columns: `linkedin_url`, `name`, `invited_as_connection`, `accepted`, `invited_to_page`

### Stage A — Lead scraping (manual)
1. Add **Manual Trigger** node named **When clicking ‘Execute workflow’**.
2. Add **Google Sheets** node named **Fetch Posts to Scrape**:
   - Document: your copied spreadsheet
   - Sheet: **Posts**
   - Filter: `scraped_at` equals empty (configure as “is empty” if available; otherwise `=` with blank).
3. Add **Split In Batches** node named **Loop Over Items** and connect:
   - Manual Trigger → Fetch Posts to Scrape → Loop Over Items
4. Add **Browserflow** node named **Scrape comments from Post**:
   - Operation: `scrapeProfilesFromPostComments`
   - `postUrl`: expression referencing current post row URL (e.g. from “Fetch Posts to Scrape”: `{{$json.url}}` or `{{ $('Fetch Posts to Scrape').item.json.url }}`)
   - `addComments: true`
   - `commentsLimit: 20`
   - Connect Loop Over Items (output 1) → Scrape comments from Post
5. Add **Split Out** node named **Split Out**:
   - `fieldToSplitOut: comments`
   - Connect Scrape comments from Post → Split Out
6. Add **Filter** node named **Filter**:
   - Condition: `linkedin_url` does **not contain** `company`
   - Connect Split Out → Filter
7. Add **Split In Batches** node named **Loop Over Items1**:
   - Connect Filter → Loop Over Items1
8. Add **Google Sheets** node named **Check if already collected**:
   - Document: your spreadsheet
   - Sheet: **Profiles**
   - Filter: `linkedin_url` equals `{{ $json.linkedin_url }}` (from Loop Over Items1 current item)
   - Enable “Always output data” (so the IF node can evaluate even when not found)
   - Connect Loop Over Items1 (output 1) → Check if already collected
9. Add **IF** node named **If not in sheet**:
   - Condition: “linkedin_url does not exist” (string `notExists`) on `{{ $json.linkedin_url }}`
   - Connect Check if already collected → If not in sheet
10. Add **Google Sheets** node named **Collect leads in sheet**:
   - Operation: Append
   - Sheet: **Profiles**
   - Map:
     - `name = {{ $('Loop Over Items1').item.json.name }}`
     - `linkedin_url = {{ $('Loop Over Items1').item.json.linkedin_url }}`
   - Connect If not in sheet (true) → Collect leads in sheet
11. Connect Collect leads in sheet → Loop Over Items1 (to continue iteration).
12. Add **Google Sheets** node named **Mark Post as scraped**:
   - Operation: Update
   - Sheet: **Posts**
   - Matching column: `url`
   - Set:
     - `url = {{ $('Loop Over Items').item.json.url }}`
     - `scraped_at = {{ $now.toISO() }}`
   - Connect Loop Over Items1 (output 0) → Mark Post as scraped
13. Connect Mark Post as scraped → Loop Over Items (to move to next post).

### Stage B — Connection invites (scheduled)
14. Add **Schedule Trigger** named **Schedule Trigger**:
   - Interval: every 2 days
15. Add **Google Sheets** node **Fetch Leads that have not been invited**:
   - Sheet: **Profiles**
   - Filter intended: `invited_as_connection is empty` AND `accepted is empty`
16. Add **Split In Batches** node **Loop over leads**.
17. Add **Browserflow** node **Check if a person is a connection**:
   - Set `linkedinUrl = {{ $json.linkedin_url }}`
18. Add **IF** node **If**:
   - `is_connection == false` AND `is_pending == false`
19. Add **Browserflow** node **Send a linked in connection invite**:
   - Operation: `sendConnectionInvite`
   - `linkedinUrl = {{ $('Loop over leads').item.json.linkedin_url }}`
20. Add **Google Sheets** node **Invite Sent**:
   - Operation: Update by `linkedin_url`
   - Set `invited_as_connection = {{ $now }}`
21. Add **IF** node **If1**:
   - Condition: `is_pending == true`
22. Add **Google Sheets** node **Pending**:
   - Update by `linkedin_url`
   - Set `invited_as_connection = {{ $now }}`
23. Add **Google Sheets** node **Already connected**:
   - Update by `linkedin_url`
   - Set `accepted = ✅`
   - (Recommended fix) leave `invited_as_connection` unchanged or set to blank—not `"="`.
24. Connect nodes:
   - Schedule Trigger → Fetch Leads that have not been invited → Loop over leads
   - Loop over leads (output 1) → Check if a person is a connection → If
   - If (true) → Send invite → Invite Sent → Loop over leads
   - If (false) → If1
   - If1 (true) → Pending → Loop over leads
   - If1 (false) → Already connected → Loop over leads

### Stage C — Acceptance tracking (scheduled)
25. Add **Schedule Trigger** named **Schedule Trigger1** (every 2 days).
26. Add **Browserflow** node **List your linkedin connections**:
   - Operation: `listConnections`
   - `limit: 20` (increase if needed)
27. Add **Split Out** node **Split out over connections**:
   - `fieldToSplitOut: data`
28. Add **Code** node **Code in JavaScript**:
   - Ensure `linkedin_url` ends with `/` (same logic as workflow).
29. Add **Google Sheets** node **Update connection status**:
   - Operation: Update by `linkedin_url`
   - Set `accepted = ✅`
30. Connect:
   - Schedule Trigger1 → List your linkedin connections → Split out over connections → Code in JavaScript → Update connection status

### Stage D — Invite to follow the Company Page (scheduled)
31. Add **Schedule Trigger** named **Schedule Trigger2** (every 2 days).
32. Add **Google Sheets** node **Get connected leads**:
   - Filter: `accepted = ✅` AND `invited_to_page is empty`
33. Add **Split In Batches** node **Loop over leads2**.
34. Add **Browserflow** node **Invite connections to follow page**:
   - Operation: `inviteToFollowPage`
   - **Set your Company Page URL** in `linkedinUrl`
   - `searchTerm = {{ $('Get connected leads').item.json.name }}`
   - `maxToInvite = 1`
35. Add **Google Sheets** node **Update invited to page**:
   - Operation: Update by `linkedin_url`
   - Set `invited_to_page = ✅`
   - (Recommended fix) do **not** clear `accepted`
36. Connect:
   - Schedule Trigger2 → Get connected leads → Loop over leads2
   - Loop over leads2 (output 1) → Invite connections to follow page → Update invited to page → Loop over leads2

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Browserflow product site | https://browserflow.io |
| Google Sheets template referenced by workflow | https://docs.google.com/spreadsheets/d/1-zak-RUGU4ubw3aZ_9lF9LF1dxEo9ME_-mtzRGtIDFg/edit?gid=0#gid=0 |
| Company page URL is hardcoded and must be replaced | In node **Invite connections to follow page** (`linkedinUrl = https://linkedin.com/company/browserflow-io`) |
| Potential logic issues to review after import | Filters for “empty” sheet fields; “Already connected” writes `invited_as_connection` as `"="`; “Update invited to page” clears `accepted` |
| Operational constraints | LinkedIn rate limits / invite caps; Browserflow automation fragility if LinkedIn UI changes |

