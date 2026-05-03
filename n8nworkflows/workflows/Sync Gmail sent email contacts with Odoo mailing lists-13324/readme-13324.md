Sync Gmail sent email contacts with Odoo mailing lists

https://n8nworkflows.xyz/workflows/sync-gmail-sent-email-contacts-with-odoo-mailing-lists-13324


# Sync Gmail sent email contacts with Odoo mailing lists

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Sync Gmail sent email contacts with Odoo mailing lists  
**Workflow name (in JSON):** Email Contacts Sync from Sent Mailbox to Odoo

**Purpose:**  
Every day at 07:00, the workflow queries Gmail for sent emails in a defined date window (default: *10 days ago*), analyzes each email thread to classify recipients (deliverable, replied, bounced, auto replies, no response), then:
1) Syncs **deliverable** recipients into **Odoo Mailing Contacts**, ensuring they belong to the **“020.Good-to-send”** list.  
2) Syncs **bounced** recipients into **Odoo Blacklisted Email Addresses** with reason **“bounced”**.

**Key logical blocks (by dependency):**
- **1.1 Scheduling & runtime variables**: Trigger + constants (days_ago, Odoo endpoints, list id).
- **1.2 Date window calculation**: Compute `after` and `before` strings for Gmail query.
- **1.3 Gmail Sent query & gating**: Fetch sent messages, stop if none.
- **1.4 Thread analysis & categorization**: For each sent message, load thread, parse headers, infer replies/bounces/auto replies, then merge results.
- **1.5 Odoo Mailing Contacts sync (deliverableEmails)**: For each deliverable email, search Odoo contact; create or update list subscription.
- **1.6 Odoo Blacklist sync (bounceEmails)**: Two parallel blacklist processing paths exist (one after step 19, and another from step 11 false-branch); both search/create/update bounced entries.
- **1.7 Termination**: NoOp end nodes.

---

## 2. Block-by-Block Analysis

### 2.1 Scheduling & runtime variables
**Overview:** Triggers daily execution and defines shared constants (time window, endpoints, Odoo list id).  
**Nodes involved:**  
- Step 1: Schedule Trigger every day at 7 AM  
- Step 2: Set Variables

#### Node: Step 1: Schedule Trigger every day at 7 AM
- **Type / role:** `Schedule Trigger` — entry point; runs workflow daily.
- **Configuration (interpreted):** Runs at **07:00** (hour = 7, minute = 0).
- **Outputs:** → Step 2.
- **Version notes:** typeVersion **1.3**.
- **Failure/edge cases:** Instance timezone impacts what “7 AM” means; ensure n8n timezone is intended.

#### Node: Step 2: Set Variables
- **Type / role:** `Set` — initializes constants.
- **Configuration choices:**
  - Raw JSON output, execute once.
  - Variables:
    - `limit`: 100 (not used downstream in this JSON; possibly legacy)
    - `days_ago`: 10 (used to compute Gmail query date range)
    - `mailing_contact_web_search_read`: `"end_point"` (must be replaced with real Odoo JSON-RPC endpoint)
    - `mailing_contact_web_save`: `"end_point"`
    - `020_Good_to_send`: `991` (Odoo mailing list ID)
    - `mail_blacklist_web_search_read`: `"end_point"`
    - `mail_blacklist_web_save`: `"end_point"`
- **Outputs:** → Step 3.
- **Version notes:** typeVersion **3.4**.
- **Failure/edge cases:** Leaving endpoints as `"end_point"` will cause HTTP failures; list id must exist in Odoo.

**Sticky note covering this block:**  
From “Sticky Note1”:  
- “## 1. Set Variables and Get datetime … In Step 6, Split Out is used to split the ‘messages’ array …”

---

### 2.2 Date window calculation
**Overview:** Computes Gmail query boundaries (`after`, `before`) based on `days_ago`.  
**Nodes involved:**  
- Step 3: Get date time

#### Node: Step 3: Get date time
- **Type / role:** `Code` — compute two date strings.
- **Logic (interpreted):**
  - `today = now - days_ago`
  - `after = YYYY/MM/DD` (target day)
  - `before = YYYY/MM/DD` for next day (exclusive upper bound)
- **Key variables/expressions:** uses `$input.first().json.days_ago`.
- **Outputs:** → Step 4.
- **Version notes:** typeVersion **2**.
- **Failure/edge cases:**
  - Date formatting uses `/` which matches Gmail search query style; OK.
  - Timezone: JavaScript `Date()` uses server timezone; could shift day boundary vs Gmail user timezone.

---

### 2.3 Gmail Sent query & gating
**Overview:** Fetches sent messages for the computed date range; stops execution if no results.  
**Nodes involved:**  
- Step 4:Get the list of emails sent 10 days ago.  
- Step 5: If resultSizeEstimate has a record ==> true  
- End

#### Node: Step 4:Get the list of emails sent 10 days ago.
- **Type / role:** `HTTP Request` — calls Gmail API `users.messages.list`.
- **Configuration choices:**
  - **GET** `https://gmail.googleapis.com/gmail/v1/users/me/messages`
  - Query parameters:
    - `q = in:sent after:{{$json.after}} before:{{$json.before}}`
    - `maxResults = 1000`
  - Authentication: **predefinedCredentialType** → `gmailOAuth2`
  - `alwaysOutputData: true` (even on errors it may output something depending on n8n behavior)
- **Inputs:** from Step 3 (`after`, `before`).
- **Outputs:** → Step 5.
- **Credentials:** Gmail OAuth2 credential required.
- **Failure/edge cases:**
  - Gmail API pagination: only first page unless `nextPageToken` handled (not implemented). Large volumes may be truncated.
  - Gmail API rate limits / 429.
  - Auth scope must allow reading mailbox metadata (Gmail read scopes).
  - If response lacks `resultSizeEstimate`, Step 5 condition might fail.

#### Node: Step 5: If resultSizeEstimate has a record ==> true
- **Type / role:** `If` — checks if any results were returned.
- **Condition:** `$json.resultSizeEstimate != 0`.
- **Outputs:**
  - **True** → Step 6
  - **False** → End
- **Version notes:** typeVersion **2.3**.
- **Failure/edge cases:** If Gmail changes response shape or returns error object, `resultSizeEstimate` may be undefined (typeValidation strict).

#### Node: End
- **Type / role:** `NoOp` — terminates “no messages” path.
- **Inputs:** from Step 5 (false branch).

---

### 2.4 Thread analysis & categorization (deliverable / replied / bounced / auto / noResponse)
**Overview:** Iterates through each sent message, loads the full Gmail thread, parses headers to identify original recipients and classify responses/bounces, then merges all message-level results into one consolidated set.  
**Nodes involved:**  
- Step 6: Split Out emails sent  
- Step 7: Loop Over email sent  
- Step 8: Get email information about thread_id  
- Step 9: Handling mail sorting from headers  
- Step 10: Merge the categorized mailing lists

#### Node: Step 6: Split Out emails sent
- **Type / role:** `Split Out` — converts `messages[]` into items.
- **Configuration:** `fieldToSplitOut = messages`
- **Outputs:** → Step 7
- **Notable flags:** `executeOnce: true` (unusual for Split Out; may indicate intended single execution behavior), `alwaysOutputData: true`.
- **Failure/edge cases:** If `messages` is missing/empty, output may be empty; downstream batch loop may not run.

#### Node: Step 7: Loop Over email sent
- **Type / role:** `Split In Batches` — loops item-by-item (or default batch size) over sent messages.
- **Connections:**
  - Main output 0 → Step 10 (to allow merge to gather results from loop)
  - Main output 1 → Step 8 (per-batch processing)
- **Version notes:** typeVersion **3**.
- **Failure/edge cases:** Without explicit batch size, defaults apply; large lists could be slow.

#### Node: Step 8: Get email information about thread_id
- **Type / role:** `HTTP Request` — calls Gmail API `users.threads.get`.
- **Configuration:**
  - URL: `https://gmail.googleapis.com/gmail/v1/users/me/threads/{{ $json.threadId }}`
  - Auth: Gmail OAuth2
  - `retryOnFail: true`, `alwaysOutputData: true`
- **Inputs:** current message item from Step 7 (must contain `threadId`).
- **Outputs:** → Step 9
- **Failure/edge cases:** Some messages may not have `threadId` (rare); API 404 if thread removed; rate limits.

#### Node: Step 9: Handling mail sorting from headers
- **Type / role:** `Code` — parses thread messages and categorizes recipients.
- **Logic (interpreted):**
  - Reads `messages` from Step 8 thread response.
  - Extracts **original recipients** from the *first message(s)* in thread (those without `In-Reply-To`) using `To` header only (no `Cc` processing in code).
  - Parses `From`, `Subject`, `Auto-Submitted`, `In-Reply-To` for each thread message:
    - **Bounce detection:** `from` contains `mailer-daemon` OR subject contains “delivery status notification” OR “failure notice”
      - If bounce found, it adds *originalRecipients* to `bounced`.
    - **Auto reply detection:** presence of `Auto-Submitted` header → `autoReplies`
    - **Real reply:** if `In-Reply-To` exists → `replied` gets sender.
  - Deduplicates by email.
  - Computes:
    - `deliverableEmails = originalRecipients - bounced`
    - `noResponse = deliverableEmails - replied`
- **Outputs:** → back into Step 7 loop (then Step 10 merge collects across loop).
- **Failure/edge cases:**
  - If recipients were in `Cc` only, they are ignored.
  - Bounce heuristics are simplistic; may misclassify some system emails.
  - Some “reply” messages from yourself could be included if thread includes your own replies.
  - Header parsing assumes standard RFC formatting; edge cases with encoded names may not parse cleanly.

#### Node: Step 10: Merge the categorized mailing lists
- **Type / role:** `Code` — consolidates arrays across all loop iterations.
- **Logic:** concatenates and deduplicates `deliverableEmails`, `repliedEmails`, `bounceEmails`, `autoReplies`, `noResponse` by `email`.
- **Outputs:** → Step 11
- **Failure/edge cases:** If Step 9 produces unexpected shapes, merge may output empty arrays.

**Sticky note covering this block:**  
From “Sticky Note”:  
- “## 2. Loop Over the list of emails sent 10 days ago… categorized into lists … merged …”

---

### 2.5 Odoo Mailing Contacts sync (deliverableEmails → “020.Good-to-send”)
**Overview:** If there are deliverable emails, iterate each email, search for an existing Odoo `mailing.contact`, then either create it with subscription to list 991 or update it to add that subscription if missing.  
**Nodes involved:**  
- Step 11: If has record deliverableEmails  
- Step 12: Split Out deliverableEmails list  
- Step 13: Loop Over deliverableEmails  
- Step 14: Search email in Mailing List Contacts  
- Step 15: If number of records is not 0 = true  
- Step 16.1" Check if list_ids contains 020.Good-to-send  
- Step 17: If list_ids = true  
- Step 16.2: Add new email in Mailing List Contacts  
- Step 18: Update email in Mailing List Contacts

#### Node: Step 11: If has record deliverableEmails
- **Type / role:** `If` — gates deliverable sync.
- **Condition:** `deliverableEmails` array **not empty**.
- **Outputs:**
  - True → Step 12
  - False → Step 27 (bounce handling path)
- **Failure/edge cases:** If `deliverableEmails` undefined, condition may fail under strict validation.

#### Node: Step 12: Split Out deliverableEmails list
- **Type / role:** `Split Out` — one item per email object `{name, email}`.
- **Field:** `deliverableEmails`
- **Outputs:** → Step 13

#### Node: Step 13: Loop Over deliverableEmails
- **Type / role:** `Split In Batches` — iterates deliverable emails.
- **Connections:**
  - Output 0 → Step 19 (after loop completes / pass-through)
  - Output 1 → Step 14 (per-item Odoo processing)
- **Failure/edge cases:** Default batch sizing; large lists may be slow.

#### Node: Step 14: Search email in Mailing List Contacts
- **Type / role:** `HTTP Request` — Odoo JSON-RPC `mailing.contact.web_search_read`.
- **Configuration choices:**
  - Method: **POST**
  - URL from variable: `mailing_contact_web_search_read`
  - Auth: `httpHeaderAuth` (API key in headers)
  - Domain filters:
    - `is_blacklisted = false`
    - match email against `name`, `company_name`, or `email_normalized` using `ilike {{ $json.email }}`
  - Limit: 80
  - Context: lang `en_US`, tz `Asia/Saigon`, company id 1, action 763.
  - `retryOnFail: true`
- **Inputs:** current deliverable email item from Step 13.
- **Outputs:** → Step 15
- **Failure/edge cases:**
  - Endpoint must be a valid Odoo JSON-RPC route (often `/web/dataset/call_kw` or similar).
  - `executeOnce: true` here is risky: it may cause the request to run only once for the first item, not per email (depends on n8n semantics for this node setting). If so, deliverable sync will be incorrect.
  - `alwaysOutputData: false`: errors may stop execution unless handled.

#### Node: Step 15: If number of records is not 0 = true
- **Type / role:** `If` — checks if Odoo search returned results.
- **Condition:** `$json.result.length != 0`
- **Outputs:**
  - True → Step 16.1 (existing contact path)
  - False → Step 16.2 (create new contact path)
- **Failure/edge cases:** Odoo response shape could differ; often records are in `$json.result.records` while `$json.result.length` may not exist unless Odoo returns a list wrapper. This is a potential bug source.

#### Node: Step 16.1" Check if list_ids contains 020.Good-to-send
- **Type / role:** `Code` — determines whether the first found contact already belongs to list 991.
- **Logic:**
  - Reads `result.records[0].list_ids`
  - Sets `islist_ids = list_ids.some(item.id === 991)`
  - Returns `{ islist_ids, id }` (contact id)
- **Outputs:** → Step 17

#### Node: Step 17: If list_ids = true
- **Type / role:** `If` — chooses update path.
- **Condition:** `$json.islist_ids` is **true**
- **Outputs (as wired):**
  - **True branch (already in list)** → Step 13 (loop continues)
  - **False branch (not in list)** → Step 18 (update contact to add subscription)
- **Important note:** The label “If list_ids = true” is misleading given the wiring: *true* means “do nothing”, *false* means “update”.
- **Failure/edge cases:** If Step 16.1 cannot find records, `id` may be undefined and Step 18 will fail.

#### Node: Step 16.2: Add new email in Mailing List Contacts
- **Type / role:** `HTTP Request` — Odoo JSON-RPC `mailing.contact.web_save` (create).
- **Configuration choices:**
  - Creates a `mailing.contact` with `email` set to the current loop item’s email:
    - `{{ $('Step 13: Loop Over deliverableEmails').item.json.email }}`
  - Adds `subscription_ids` entry with `list_id = 991` and `opt_out=false`.
- **Outputs:** → Step 13 (continue loop)
- **Failure/edge cases:** If the Odoo model enforces unique email, duplicates could error; ensure normalization.

#### Node: Step 18: Update email in Mailing List Contacts
- **Type / role:** `HTTP Request` — Odoo JSON-RPC `mailing.contact.web_save` (update).
- **Configuration:** Updates contact `[id]` to add a new `subscription_ids` record for list 991.
- **Input dependency:** uses `{{ $node['Step 17: If list_ids = true'].json.id }}`.
- **Outputs:** → Step 13 (continue loop)
- **Failure/edge cases:** If contact already has that subscription, Odoo might create duplicates depending on model behavior; ideally should update existing subscription instead.

**Sticky note covering this block:**  
From “Sticky Note3”:  
- “## 3. Push the list of deliverable emails to the internal Odoo system… create or ensure tag/list 020.Good-to-send …”

---

### 2.6 Consolidation handoff to bounce processing (and parallel bounce pipelines)
**Overview:** After deliverable loop, the workflow re-emits the merged categorization result and then processes bounced emails into Odoo blacklist. There are *two* implementations for blacklist processing:
- **Pipeline A:** Step 19 → Step 20 → Step 21 → Steps 22–26  
- **Pipeline B:** Step 27 → Step 28 → Step 29 → Steps 30–34  
They are functionally similar and may cause duplicate work if both are reachable in some execution paths.

**Nodes involved:**  
- Step 19: Return all data from step 10.  
- (Pipeline A) Step 20–26 + End3  
- (Pipeline B) Step 27–34 + End1/End2  
- End1, End2, End3

#### Node: Step 19: Return all data from step 10.
- **Type / role:** `Code` — fetches items from Step 10 and returns them.
- **Logic:** `const data = $items("Step 10: Merge the categorized mailing lists"); return data`
- **Outputs:** → Step 20
- **Failure/edge cases:** If Step 10 produced no items, bounce processing won’t run.

---

### 2.7 Odoo Blacklist sync – Pipeline A (Step 20–26)
**Overview:** Iterates `bounceEmails`, searches Odoo blacklist; if not found creates a bounced record; if found and reason not “bounced”, updates to “bounced”.  
**Nodes involved:**  
- Step 20: Split Out bounceEmails  
- Step 21: Loop Over bounceEmails  
- Step 22: Search email in Blacklisted Email Addresses  
- Step 23: If number of records is not 0 = true  
- Step 24.1" Check if reason_type contains Bounced  
- Step 25: If list_ids = true  
- Step 24.2: Add new email in Blacklisted Email Addresses  
- Step 26: Update Reason Type of email in Blacklisted Email Addresses  
- End3

#### Node: Step 20: Split Out bounceEmails
- **Type / role:** `Split Out`
- **Field:** `bounceEmails`
- **Outputs:** → Step 21
- **Failure/edge cases:** Missing `bounceEmails` yields no items.

#### Node: Step 21: Loop Over bounceEmails
- **Type / role:** `Split In Batches`
- **Connections:**
  - Output 0 → End3
  - Output 1 → Step 22
- **Behavior:** Standard loop pattern; end node receives completion path.
  
#### Node: Step 22: Search email in Blacklisted Email Addresses
- **Type / role:** `HTTP Request` — Odoo JSON-RPC `mail.blacklist.web_search_read`
- **Domain:** `email ilike {{ $json.email }}`
- **Outputs:** → Step 23
- **Risk:** `executeOnce: true` again may cause “only first item” execution, breaking per-email behavior.

#### Node: Step 23: If number of records is not 0 = true
- **Type / role:** `If`
- **Condition:** `$json.result.length != 0`
- **Outputs:**
  - True → Step 24.1 (exists)
  - False → Step 24.2 (create)
- **Potential bug:** same response-shape risk as Step 15.

#### Node: Step 24.1" Check if reason_type contains Bounced
- **Type / role:** `Code`
- **Logic:** reads `result.records[0]`, returns:
  - `id`
  - `isBounced = (reason_type).toLowerCase() === "bounced"`
- **Outputs:** → Step 25

#### Node: Step 25: If list_ids = true
- **Type / role:** `If` (label misleading; it checks `isBounced`)
- **Condition:** `isBounced` is **true**
- **Outputs (as wired):**
  - True → Step 21 (loop continues; do nothing)
  - False → Step 26 (update reason_type to bounced)

#### Node: Step 24.2: Add new email in Blacklisted Email Addresses
- **Type / role:** `HTTP Request` — Odoo `mail.blacklist.web_save` (create).
- **Body:** creates blacklist record with:
  - `email` = `{{ $('Step 21: Loop Over bounceEmails').item.json.email }}`
  - `reason_type` = `"bounced"`
- **Outputs:** → Step 21 (continue loop)

#### Node: Step 26: Update Reason Type of email in Blacklisted Email Addresses
- **Type / role:** `HTTP Request` — Odoo `mail.blacklist.web_save` (update).
- **Updates:** sets `reason_type = "bounced"` for record id from Step 25.
- **Outputs:** → Step 21

#### Node: End3
- **Type / role:** `NoOp` — end of pipeline A loop completion.

**Sticky note covering this block:**  
From “Sticky Note4”:  
- “## 4. Add emails from the bounce email list to the ‘Blacklisted Email Addresses’ …”

---

### 2.8 Odoo Blacklist sync – Pipeline B (Step 27–34)
**Overview:** A second, redundant bounce processing path triggered directly after Step 11 (deliverableEmails empty). It performs the same search/create/update as pipeline A, but with different node names.  
**Nodes involved:**  
- Step 27: If has record bounceEmails  
- Step 28: Split Out bounceEmails  
- Step 29: Loop Over bounceEmails  
- Step 30: Search email in Blacklisted Email Addresses  
- Step 31: If number of records is not 0 = true  
- Step 32.1: Check if reason_type contains Bounced  
- Step 33: If list_ids = true  
- Step 32.2: Add new email in Blacklisted Email Addresses  
- Step 34: Update Reason Type of email in Blacklisted Email Addresses  
- End1, End2

#### Node: Step 27: If has record bounceEmails
- **Type / role:** `If`
- **Condition:** `bounceEmails` array **not empty**
- **Outputs:**
  - True → Step 28
  - False → End1

#### Node: Step 28: Split Out bounceEmails
- **Type / role:** `Split Out` → Step 29

#### Node: Step 29: Loop Over bounceEmails
- **Type / role:** `Split In Batches`
- **Connections:**
  - Output 0 → End2
  - Output 1 → Step 30

#### Node: Step 30: Search email in Blacklisted Email Addresses
- **Type / role:** `HTTP Request` — same as Step 22.
- **Outputs:** → Step 31

#### Node: Step 31: If number of records is not 0 = true
- **Type / role:** `If` — same as Step 23.
- **Outputs:**
  - True → Step 32.1
  - False → Step 32.2

#### Node: Step 32.1: Check if reason_type contains Bounced
- **Type / role:** `Code` — same as Step 24.1.
- **Outputs:** → Step 33

#### Node: Step 33: If list_ids = true
- **Type / role:** `If` (misnamed)
- **Condition:** `isBounced` is **true**
- **Outputs:**
  - True → Step 29 (continue loop; do nothing)
  - False → Step 34 (update)

#### Node: Step 32.2: Add new email in Blacklisted Email Addresses
- **Type / role:** `HTTP Request` — create bounced blacklist record.
- **Email source:** `{{ $('Step 29: Loop Over bounceEmails').item.json.email }}`
- **Outputs:** → Step 29

#### Node: Step 34: Update Reason Type of email in Blacklisted Email Addresses
- **Type / role:** `HTTP Request` — update record reason_type to bounced.
- **Outputs:** → Step 29

#### Nodes: End1, End2
- **Type / role:** `NoOp` — end nodes for “no bounce emails” and “loop completed”.

**Sticky note covering this block:**  
From “Sticky Note5”:  
- “## 5. Same…..Add emails from the bounce email list …”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Step 1: Schedule Trigger every day at 7 AM | Schedule Trigger | Daily trigger entrypoint | — | Step 2: Set Variables | ## 1. Set Variables and Get datetime<br>- Set Variables and Get datetime<br>- Get the list of emails sent 10 days ago.<br>- In Step 6, Split Out is used to split the "messages" array into multiple individual items. |
| Step 2: Set Variables | Set | Define constants (days_ago, endpoints, list id) | Step 1 | Step 3 | ## 1. Set Variables and Get datetime<br>- Set Variables and Get datetime<br>- Get the list of emails sent 10 days ago.<br>- In Step 6, Split Out is used to split the "messages" array into multiple individual items. |
| Step 3: Get date time | Code | Compute `after`/`before` for Gmail query | Step 2 | Step 4 | ## 1. Set Variables and Get datetime<br>- Set Variables and Get datetime<br>- Get the list of emails sent 10 days ago.<br>- In Step 6, Split Out is used to split the "messages" array into multiple individual items. |
| Step 4:Get the list of emails sent 10 days ago. | HTTP Request | Gmail messages.list query for sent window | Step 3 | Step 5 | ## 1. Set Variables and Get datetime<br>- Set Variables and Get datetime<br>- Get the list of emails sent 10 days ago.<br>- In Step 6, Split Out is used to split the "messages" array into multiple individual items. |
| Step 5: If resultSizeEstimate has a record ==> true | If | Stop when no Gmail results | Step 4 | Step 6 / End | ## 1. Set Variables and Get datetime<br>- Set Variables and Get datetime<br>- Get the list of emails sent 10 days ago.<br>- In Step 6, Split Out is used to split the "messages" array into multiple individual items. |
| End | NoOp | Terminates when no sent emails | Step 5 (false) | — |  |
| Step 6: Split Out emails sent | Split Out | Split Gmail `messages[]` into items | Step 5 (true) | Step 7 | ## 2. Loop Over the list of emails sent 10 days ago.<br>- We can customize the number of days in "Step 2: Set Variables" using the days_ago field.<br>- Each iteration checks if anyone has responded to the sent emails. After obtaining the results, they are categorized into lists: deliverableEmails, repliedEmails, bounceEmails, autoReplies, and noResponse.<br>- After each iteration, the data is merged into the lists above. |
| Step 7: Loop Over email sent | Split In Batches | Iterate sent messages | Step 6 | Step 8 / Step 10 | ## 2. Loop Over the list of emails sent 10 days ago.<br>- We can customize the number of days in "Step 2: Set Variables" using the days_ago field.<br>- Each iteration checks if anyone has responded to the sent emails. After obtaining the results, they are categorized into lists: deliverableEmails, repliedEmails, bounceEmails, autoReplies, and noResponse.<br>- After each iteration, the data is merged into the lists above. |
| Step 8: Get email information about thread_id | HTTP Request | Gmail threads.get per sent message | Step 7 | Step 9 | ## 2. Loop Over the list of emails sent 10 days ago.<br>- We can customize the number of days in "Step 2: Set Variables" using the days_ago field.<br>- Each iteration checks if anyone has responded to the sent emails. After obtaining the results, they are categorized into lists: deliverableEmails, repliedEmails, bounceEmails, autoReplies, and noResponse.<br>- After each iteration, the data is merged into the lists above. |
| Step 9: Handling mail sorting from headers | Code | Classify recipients (deliver/reply/bounce/auto/noResponse) | Step 8 | Step 7 | ## 2. Loop Over the list of emails sent 10 days ago.<br>- We can customize the number of days in "Step 2: Set Variables" using the days_ago field.<br>- Each iteration checks if anyone has responded to the sent emails. After obtaining the results, they are categorized into lists: deliverableEmails, repliedEmails, bounceEmails, autoReplies, and noResponse.<br>- After each iteration, the data is merged into the lists above. |
| Step 10: Merge the categorized mailing lists | Code | Merge/deduplicate arrays across iterations | Step 7 | Step 11 | ## 2. Loop Over the list of emails sent 10 days ago.<br>- We can customize the number of days in "Step 2: Set Variables" using the days_ago field.<br>- Each iteration checks if anyone has responded to the sent emails. After obtaining the results, they are categorized into lists: deliverableEmails, repliedEmails, bounceEmails, autoReplies, and noResponse.<br>- After each iteration, the data is merged into the lists above. |
| Step 11: If has record deliverableEmails | If | Gate deliverable sync | Step 10 | Step 12 / Step 27 | ## 3. Push the list of deliverable emails to the internal Odoo system.<br>- For each email in the deliverable emails list, we check if it exists in the internal Odoo system.<br>- If not, create a new contact and tag it as "020.Good-to-send".<br>- If yes, ensure it is assigned to "020.Good-to-send" and update when needed. |
| Step 12: Split Out deliverableEmails list | Split Out | One item per deliverable email | Step 11 (true) | Step 13 | ## 3. Push the list of deliverable emails to the internal Odoo system.<br>- For each email in the deliverable emails list, we check if it exists in the internal Odoo system.<br>- If not, create a new contact and tag it as "020.Good-to-send".<br>- If yes, ensure it is assigned to "020.Good-to-send" and update when needed. |
| Step 13: Loop Over deliverableEmails | Split In Batches | Iterate deliverable emails | Step 12 / Step 17 / Step 16.2 / Step 18 | Step 14 / Step 19 | ## 3. Push the list of deliverable emails to the internal Odoo system.<br>- For each email in the deliverable emails list, we check if it exists in the internal Odoo system.<br>- If not, create a new contact and tag it as "020.Good-to-send".<br>- If yes, ensure it is assigned to "020.Good-to-send" and update when needed. |
| Step 14: Search email in Mailing List Contacts | HTTP Request | Odoo mailing.contact search | Step 13 | Step 15 | ## 3. Push the list of deliverable emails to the internal Odoo system.<br>- For each email in the deliverable emails list, we check if it exists in the internal Odoo system.<br>- If not, create a new contact and tag it as "020.Good-to-send".<br>- If yes, ensure it is assigned to "020.Good-to-send" and update when needed. |
| Step 15: If number of records is not 0 = true | If | Exists in Odoo? | Step 14 | Step 16.1 / Step 16.2 | ## 3. Push the list of deliverable emails to the internal Odoo system.<br>- For each email in the deliverable emails list, we check if it exists in the internal Odoo system.<br>- If not, create a new contact and tag it as "020.Good-to-send".<br>- If yes, ensure it is assigned to "020.Good-to-send" and update when needed. |
| Step 16.1" Check if list_ids contains 020.Good-to-send | Code | Determine if contact already in list 991 | Step 15 (true) | Step 17 | ## 3. Push the list of deliverable emails to the internal Odoo system.<br>- For each email in the deliverable emails list, we check if it exists in the internal Odoo system.<br>- If not, create a new contact and tag it as "020.Good-to-send".<br>- If yes, ensure it is assigned to "020.Good-to-send" and update when needed. |
| Step 17: If list_ids = true | If | If already in list, skip; else update | Step 16.1 | Step 13 / Step 18 | ## 3. Push the list of deliverable emails to the internal Odoo system.<br>- For each email in the deliverable emails list, we check if it exists in the internal Odoo system.<br>- If not, create a new contact and tag it as "020.Good-to-send".<br>- If yes, ensure it is assigned to "020.Good-to-send" and update when needed. |
| Step 16.2: Add new email in Mailing List Contacts | HTTP Request | Odoo mailing.contact create + subscribe list 991 | Step 15 (false) | Step 13 | ## 3. Push the list of deliverable emails to the internal Odoo system.<br>- For each email in the deliverable emails list, we check if it exists in the internal Odoo system.<br>- If not, create a new contact and tag it as "020.Good-to-send".<br>- If yes, ensure it is assigned to "020.Good-to-send" and update when needed. |
| Step 18: Update email in Mailing List Contacts | HTTP Request | Odoo mailing.contact update subscription | Step 17 (false) | Step 13 | ## 3. Push the list of deliverable emails to the internal Odoo system.<br>- For each email in the deliverable emails list, we check if it exists in the internal Odoo system.<br>- If not, create a new contact and tag it as "020.Good-to-send".<br>- If yes, ensure it is assigned to "020.Good-to-send" and update when needed. |
| Step 19: Return all data from step 10. | Code | Re-emit merged categorization results | Step 13 | Step 20 | ## 4. Add emails from the bounce email list to the "Blacklisted Email Addresses" in the internal Odoo system.<br>- For each email in the bounce email list, we check if it exists in the internal Odoo system.<br>- If not, create a new contact in the "Blacklisted Email Addresses" and tag it as "bounced".<br>- If yes, ensure it is assigned to "bounced" and update when needed. |
| Step 20: Split Out bounceEmails | Split Out | Split bounce list for Pipeline A | Step 19 | Step 21 | ## 4. Add emails from the bounce email list to the "Blacklisted Email Addresses" in the internal Odoo system.<br>- For each email in the bounce email list, we check if it exists in the internal Odoo system.<br>- If not, create a new contact in the "Blacklisted Email Addresses" and tag it as "bounced".<br>- If yes, ensure it is assigned to "bounced" and update when needed. |
| Step 21: Loop Over bounceEmails | Split In Batches | Iterate bounce emails (Pipeline A) | Step 20 / Step 24.2 / Step 26 / Step 25 | Step 22 / End3 | ## 4. Add emails from the bounce email list to the "Blacklisted Email Addresses" in the internal Odoo system.<br>- For each email in the bounce email list, we check if it exists in the internal Odoo system.<br>- If not, create a new contact in the "Blacklisted Email Addresses" and tag it as "bounced".<br>- If yes, ensure it is assigned to "bounced" and update when needed. |
| Step 22: Search email in Blacklisted Email Addresses | HTTP Request | Odoo mail.blacklist search (Pipeline A) | Step 21 | Step 23 | ## 4. Add emails from the bounce email list to the "Blacklisted Email Addresses" in the internal Odoo system.<br>- For each email in the bounce email list, we check if it exists in the internal Odoo system.<br>- If not, create a new contact in the "Blacklisted Email Addresses" and tag it as "bounced".<br>- If yes, ensure it is assigned to "bounced" and update when needed. |
| Step 23: If number of records is not 0 = true | If | Exists in blacklist? (Pipeline A) | Step 22 | Step 24.1 / Step 24.2 | ## 4. Add emails from the bounce email list to the "Blacklisted Email Addresses" in the internal Odoo system.<br>- For each email in the bounce email list, we check if it exists in the internal Odoo system.<br>- If not, create a new contact in the "Blacklisted Email Addresses" and tag it as "bounced".<br>- If yes, ensure it is assigned to "bounced" and update when needed. |
| Step 24.1" Check if reason_type contains Bounced | Code | Determine if reason already “bounced” | Step 23 (true) | Step 25 | ## 4. Add emails from the bounce email list to the "Blacklisted Email Addresses" in the internal Odoo system.<br>- For each email in the bounce email list, we check if it exists in the internal Odoo system.<br>- If not, create a new contact in the "Blacklisted Email Addresses" and tag it as "bounced".<br>- If yes, ensure it is assigned to "bounced" and update when needed. |
| Step 25: If list_ids = true | If | If already bounced, skip; else update | Step 24.1 | Step 21 / Step 26 | ## 4. Add emails from the bounce email list to the "Blacklisted Email Addresses" in the internal Odoo system.<br>- For each email in the bounce email list, we check if it exists in the internal Odoo system.<br>- If not, create a new contact in the "Blacklisted Email Addresses" and tag it as "bounced".<br>- If yes, ensure it is assigned to "bounced" and update when needed. |
| Step 24.2: Add new email in Blacklisted Email Addresses | HTTP Request | Create bounced blacklist entry (Pipeline A) | Step 23 (false) | Step 21 | ## 4. Add emails from the bounce email list to the "Blacklisted Email Addresses" in the internal Odoo system.<br>- For each email in the bounce email list, we check if it exists in the internal Odoo system.<br>- If not, create a new contact in the "Blacklisted Email Addresses" and tag it as "bounced".<br>- If yes, ensure it is assigned to "bounced" and update when needed. |
| Step 26: Update Reason Type of email in Blacklisted Email Addresses | HTTP Request | Update reason_type to bounced (Pipeline A) | Step 25 (false) | Step 21 | ## 4. Add emails from the bounce email list to the "Blacklisted Email Addresses" in the internal Odoo system.<br>- For each email in the bounce email list, we check if it exists in the internal Odoo system.<br>- If not, create a new contact in the "Blacklisted Email Addresses" and tag it as "bounced".<br>- If yes, ensure it is assigned to "bounced" and update when needed. |
| End3 | NoOp | Terminates Pipeline A loop completion | Step 21 (done) | — |  |
| Step 27: If has record bounceEmails | If | Gate Pipeline B bounce sync | Step 11 (false) | Step 28 / End1 | ## 5. Same.....Add emails from the bounce email list to the "Blacklisted Email Addresses" in the internal Odoo system.<br>- For each email in the bounce email list, we check if it exists in the internal Odoo system.<br>- If not, create a new contact in the "Blacklisted Email Addresses" and tag it as "bounced".<br>- If yes, ensure it is assigned to "bounced" and update when needed. |
| Step 28: Split Out bounceEmails | Split Out | Split bounce list for Pipeline B | Step 27 (true) | Step 29 | ## 5. Same.....Add emails from the bounce email list to the "Blacklisted Email Addresses" in the internal Odoo system.<br>- For each email in the bounce email list, we check if it exists in the internal Odoo system.<br>- If not, create a new contact in the "Blacklisted Email Addresses" and tag it as "bounced".<br>- If yes, ensure it is assigned to "bounced" and update when needed. |
| Step 29: Loop Over bounceEmails | Split In Batches | Iterate bounce emails (Pipeline B) | Step 28 / Step 32.2 / Step 34 / Step 33 | Step 30 / End2 | ## 5. Same.....Add emails from the bounce email list to the "Blacklisted Email Addresses" in the internal Odoo system.<br>- For each email in the bounce email list, we check if it exists in the internal Odoo system.<br>- If not, create a new contact in the "Blacklisted Email Addresses" and tag it as "bounced".<br>- If yes, ensure it is assigned to "bounced" and update when needed. |
| Step 30: Search email in Blacklisted Email Addresses | HTTP Request | Odoo mail.blacklist search (Pipeline B) | Step 29 | Step 31 | ## 5. Same.....Add emails from the bounce email list to the "Blacklisted Email Addresses" in the internal Odoo system.<br>- For each email in the bounce email list, we check if it exists in the internal Odoo system.<br>- If not, create a new contact in the "Blacklisted Email Addresses" and tag it as "bounced".<br>- If yes, ensure it is assigned to "bounced" and update when needed. |
| Step 31: If number of records is not 0 = true | If | Exists in blacklist? (Pipeline B) | Step 30 | Step 32.1 / Step 32.2 | ## 5. Same.....Add emails from the bounce email list to the "Blacklisted Email Addresses" in the internal Odoo system.<br>- For each email in the bounce email list, we check if it exists in the internal Odoo system.<br>- If not, create a new contact in the "Blacklisted Email Addresses" and tag it as "bounced".<br>- If yes, ensure it is assigned to "bounced" and update when needed. |
| Step 32.1: Check if reason_type contains Bounced | Code | Determine if reason already “bounced” | Step 31 (true) | Step 33 | ## 5. Same.....Add emails from the bounce email list to the "Blacklisted Email Addresses" in the internal Odoo system.<br>- For each email in the bounce email list, we check if it exists in the internal Odoo system.<br>- If not, create a new contact in the "Blacklisted Email Addresses" and tag it as "bounced".<br>- If yes, ensure it is assigned to "bounced" and update when needed. |
| Step 33: If list_ids = true | If | If already bounced, skip; else update | Step 32.1 | Step 29 / Step 34 | ## 5. Same.....Add emails from the bounce email list to the "Blacklisted Email Addresses" in the internal Odoo system.<br>- For each email in the bounce email list, we check if it exists in the internal Odoo system.<br>- If not, create a new contact in the "Blacklisted Email Addresses" and tag it as "bounced".<br>- If yes, ensure it is assigned to "bounced" and update when needed. |
| Step 32.2: Add new email in Blacklisted Email Addresses | HTTP Request | Create bounced blacklist entry (Pipeline B) | Step 31 (false) | Step 29 | ## 5. Same.....Add emails from the bounce email list to the "Blacklisted Email Addresses" in the internal Odoo system.<br>- For each email in the bounce email list, we check if it exists in the internal Odoo system.<br>- If not, create a new contact in the "Blacklisted Email Addresses" and tag it as "bounced".<br>- If yes, ensure it is assigned to "bounced" and update when needed. |
| Step 34: Update Reason Type of email in Blacklisted Email Addresses | HTTP Request | Update reason_type to bounced (Pipeline B) | Step 33 (false) | Step 29 | ## 5. Same.....Add emails from the bounce email list to the "Blacklisted Email Addresses" in the internal Odoo system.<br>- For each email in the bounce email list, we check if it exists in the internal Odoo system.<br>- If not, create a new contact in the "Blacklisted Email Addresses" and tag it as "bounced".<br>- If yes, ensure it is assigned to "bounced" and update when needed. |
| End1 | NoOp | Terminates when no bounceEmails (Pipeline B) | Step 27 (false) | — |  |
| End2 | NoOp | Terminates Pipeline B loop completion | Step 29 (done) | — |  |
| Sticky Note | Sticky Note | Comment block | — | — |  |
| Sticky Note1 | Sticky Note | Comment block | — | — |  |
| Sticky Note3 | Sticky Note | Comment block | — | — |  |
| Sticky Note4 | Sticky Note | Comment block | — | — |  |
| Sticky Note5 | Sticky Note | Comment block | — | — |  |
| Sticky Note10 | Sticky Note | Overall description / links | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name it: **Email Contacts Sync from Sent Mailbox to Odoo** (or your preferred name).
   - Ensure n8n is v2.x compatible with node versions used (sticky note mentions **n8n 2.4.6**).

2) **Add trigger**
   - Add **Schedule Trigger**
   - Set to run **every day at 07:00**.
   - Connect to the next node.

3) **Add constants**
   - Add a **Set** node named **Step 2: Set Variables**
   - Mode: **Raw JSON**
   - Paste key fields (adapt values):
     - `days_ago`: `10`
     - `020_Good_to_send`: your Odoo mailing list id (e.g., `991`)
     - `mailing_contact_web_search_read`: your Odoo JSON-RPC endpoint URL for searching mailing contacts
     - `mailing_contact_web_save`: your Odoo JSON-RPC endpoint URL for saving mailing contacts
     - `mail_blacklist_web_search_read`: Odoo endpoint URL for blacklist search
     - `mail_blacklist_web_save`: Odoo endpoint URL for blacklist save
   - Connect Schedule Trigger → Set node.

4) **Compute date window**
   - Add a **Code** node named **Step 3: Get date time**
   - Implement logic to output:
     - `after` (YYYY/MM/DD for `now - days_ago`)
     - `before` (next day)
   - Connect Step 2 → Step 3.

5) **Query Gmail sent messages**
   - Add **HTTP Request** node named **Step 4:Get the list of emails sent 10 days ago.**
   - Method: GET
   - URL: `https://gmail.googleapis.com/gmail/v1/users/me/messages`
   - Query params:
     - `q` = `in:sent after:{{$json.after}} before:{{$json.before}}`
     - `maxResults` = `1000`
   - Authentication: **Gmail OAuth2** credential (predefined credential type).
     - Create a Gmail OAuth2 credential with appropriate scopes (Gmail read access).
   - Connect Step 3 → Step 4.

6) **Gate when Gmail returns nothing**
   - Add **If** node named **Step 5: If resultSizeEstimate has a record ==> true**
   - Condition: Number `{{$json.resultSizeEstimate}}` **not equals** `0`
   - False branch → **NoOp** node “End”
   - True branch → continue.

7) **Split messages & loop**
   - Add **Split Out** node “Step 6: Split Out emails sent”
     - Field to split: `messages`
   - Add **Split In Batches** node “Step 7: Loop Over email sent”
   - Connect: Step 5 (true) → Step 6 → Step 7.

8) **Fetch thread details**
   - Add **HTTP Request** “Step 8: Get email information about thread_id”
   - URL: `https://gmail.googleapis.com/gmail/v1/users/me/threads/{{ $json.threadId }}`
   - Auth: Gmail OAuth2
   - Connect Step 7 (batch item output) → Step 8.

9) **Categorize recipients**
   - Add **Code** node “Step 9: Handling mail sorting from headers”
   - Implement header parsing:
     - Parse original recipients from messages without `In-Reply-To` (at minimum `To`; optionally add `Cc` if you want).
     - Identify bounce/autoreply/replies.
     - Output arrays: `deliverableEmails`, `repliedEmails`, `bounceEmails`, `autoReplies`, `noResponse`.
   - Connect Step 8 → Step 9 → back to Step 7 (to continue loop).

10) **Merge categorization across all sent items**
   - Add **Code** node “Step 10: Merge the categorized mailing lists”
   - Merge/deduplicate by email.
   - Connect Step 7 completion output → Step 10.

11) **Gate deliverable sync**
   - Add **If** node “Step 11: If has record deliverableEmails”
   - Condition: Array `{{$json.deliverableEmails}}` **not empty**
   - True → deliverable sync steps
   - False → bounce-only path (optional).

12) **Deliverable sync: split and loop**
   - Add **Split Out** “Step 12: Split Out deliverableEmails list” (field: `deliverableEmails`)
   - Add **Split In Batches** “Step 13: Loop Over deliverableEmails”
   - Connect Step 11 (true) → Step 12 → Step 13.

13) **Search mailing.contact in Odoo**
   - Add **HTTP Request** “Step 14: Search email in Mailing List Contacts”
   - Method: POST
   - URL: `{{$node['Step 2: Set Variables'].json.mailing_contact_web_search_read}}`
   - Auth: **HTTP Header Auth** credential (Odoo API key header)
   - Body: JSON-RPC `mailing.contact` → `web_search_read` with a domain matching the email.
   - Connect Step 13 (item output) → Step 14.

14) **If exists → check list membership**
   - Add **If** “Step 15…” checking the response indicates at least 1 record.
   - Add **Code** “Step 16.1…” to check whether `list_ids` contains the list id (`020_Good_to_send`) and return `{islist_ids, id}`.
   - Add **If** “Step 17…”:
     - If already in list → loop continues
     - Else → update contact to add subscription.

15) **Create contact when missing**
   - Add **HTTP Request** “Step 16.2: Add new email in Mailing List Contacts”
   - JSON-RPC `mailing.contact.web_save` create with `email` from the current loop item and `subscription_ids` containing list id.
   - Connect from Step 15 (false) → Step 16.2 → back to Step 13.

16) **Update contact when list missing**
   - Add **HTTP Request** “Step 18: Update email in Mailing List Contacts”
   - JSON-RPC `mailing.contact.web_save` update with the record id and add subscription_ids.
   - Connect Step 17 (false) → Step 18 → back to Step 13.

17) **Bounce processing (choose one pipeline)**
   - Recommended: keep **one** bounce pipeline to avoid duplication.
   - Add a **Code** node “Step 19…” if you want to re-fetch merged output (optional if you carry data forward directly).
   - Add **Split Out** on `bounceEmails`, then **Split In Batches**, then:
     - Odoo blacklist search (`mail.blacklist.web_search_read`)
     - If found: check `reason_type`, update to `"bounced"` if needed
     - If not found: create via `mail.blacklist.web_save` with reason_type `"bounced"`

18) **Credentials setup**
   - **Gmail OAuth2 credential**:
     - Google Cloud project OAuth client
     - Scopes sufficient for Gmail read (e.g., Gmail readonly)
     - Connect credential to both Gmail HTTP nodes (messages list + threads get).
   - **Odoo API key via HTTP Header Auth**:
     - Configure header name/value as required by your Odoo gateway (often `Authorization: Bearer <token>` or `X-API-Key: <key>` depending on your setup).
     - Use this credential on all Odoo HTTP nodes.

19) **Finalize**
   - Add **NoOp** end nodes (optional) for visual termination.
   - Test with manual execution, then activate.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| # Email Contacts Sync from Sent Mailbox to Odoo … (audience, problem, results, take it further) | From Sticky Note10 (workflow description panel) |
| Contact / consulting links | [Linkedin](https://www.linkedin.com/company/bac-ha-software/posts/?feedView=all) / [Website](https://bachasoftware.com/bhsoft-contacts) |
| “Google Sheets will be used to log all notified events.” | Mentioned in Sticky Note10, but **no Google Sheets nodes exist** in this workflow JSON (feature is not implemented here). |
| Reported requirement: “N8n Version 2.4.6; Gmail OAuth2 API; Odoo API-KEY” | From Sticky Note10 |

