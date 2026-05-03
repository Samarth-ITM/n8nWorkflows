Create and post Instagram visuals from Google Sheets with Blotato

https://n8nworkflows.xyz/workflows/create-and-post-instagram-visuals-from-google-sheets-with-blotato-13295


# Create and post Instagram visuals from Google Sheets with Blotato

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Create and post Instagram visuals from Google Sheets with Blotato  
**Workflow name (internal):** Auto Create & Post Instagram Visuals

**Purpose:**  
Automatically picks the next “Pending” content idea from Google Sheets, generates an Instagram-ready visual using Blotato based on a selected viral format, waits until rendering is complete, then publishes the media to Instagram (via Blotato’s publishing capability) and finally updates the Google Sheet status to **Done** or **Error**.

**Primary use cases:**
- Automated Instagram content pipeline driven by a spreadsheet backlog
- Format-based creative generation (carousel / single text / slideshow / infographic video wall)
- Lightweight status tracking and retry loop while rendering completes

### 1.1 Idea Intake & Scheduling
Runs at specific hours daily, pulls one pending row from Google Sheets, and routes by the “Visual” format.

### 1.2 Visual Creation (By Viral Format)
Depending on the selected format, calls Blotato with a prompt (the “Ideas” field) and a specific template + template inputs.

### 1.3 Rendering & Asset Check (Polling Loop)
Waits, fetches render status from Blotato, loops until status is `done`.

### 1.4 Instagram Auto Publishing
Posts generated media URLs to a configured Instagram account via Blotato.

### 1.5 Status Tracking & Logging
Updates Google Sheets row status to **Done** on success or **Error** on publish error.

---

## 2. Block-by-Block Analysis

### Block 1 — Idea Intake & Scheduling
**Overview:**  
Triggers on a schedule, fetches the next “Pending” content idea from Google Sheets, and routes execution to the correct visual-generation template based on the `Visual` column.

**Nodes involved:**
- Content Schedule Trigger
- Fetch Content Ideas & Visual
- Route by Viral Content Format

#### Node: Content Schedule Trigger
- **Type / role:** Schedule Trigger — time-based entry point.
- **Configuration choices:** Runs multiple times per day at **06:00**, **11:00**, and **20:00** (server timezone as configured in n8n instance).
- **Inputs / outputs:** Entry node → outputs to Google Sheets fetch.
- **Edge cases / failures:**
  - Timezone mismatch may cause unexpected run times.
  - Overlapping executions if previous run is still active (depending on n8n concurrency settings).

#### Node: Fetch Content Ideas &  Visual
- **Type / role:** Google Sheets node — fetch one row matching filters.
- **Configuration choices (interpreted):**
  - Spreadsheet: “Personal_Content_Ideas”
  - Sheet/tab: “Personal development”
  - Filter: `Status` equals **Pending**
  - Option: **Return first match** (so it processes one pending idea per trigger execution)
- **Key fields used later:** `Ideas`, `Visual`, `row_number`
- **Credentials:** Google Sheets OAuth2 credential named **GiangXAI**
- **Inputs / outputs:** Receives trigger → outputs a single matched row to the Switch.
- **Edge cases / failures:**
  - No matching rows: node may output **0 items**; downstream nodes may not run or may error depending on execution path.
  - Google auth expired / insufficient permissions.
  - Column name mismatch (must have `Status`, `Ideas`, `Visual`, and `row_number` available as used by expressions).

#### Node: Route by Viral Content Format
- **Type / role:** Switch — routes by `Visual` value.
- **Configuration choices:**
  - Rules compare `{{$json.Visual}}` to one of:
    - “Whiteboard infographic”
    - “Tutorial Carousel”
    - “Single Centered Text”
    - “Image Slide Show”
  - Each rule has a dedicated output branch.
- **Inputs / outputs:**
  - Input: row from Google Sheets
  - Outputs:
    - Whiteboard infographic → Generate Whiteboard Infographic
    - Tutorial Carousel → Generate Tutorial Carousel
    - Single Centered Text → Generate Single Text Visual
    - Image Slide Show → Generate an Image Slideshow
- **Edge cases / failures:**
  - If `Visual` is empty/typo/unexpected value: no rule matches → no downstream generation occurs.
  - Case sensitivity is enabled; exact string match required.

---

### Block 2 — Visual Creation (By Viral Format)
**Overview:**  
Generates a visual (Blotato “video” resource) using the idea text as the prompt and a specific template per format.

**Nodes involved:**
- Generate Whiteboard Infographic
- Generate Tutorial Carousel
- Generate Single Text Visual
- Generate an Image Slideshow

#### Node: Generate Whiteboard Infographic
- **Type / role:** Blotato node (`@blotato/n8n-nodes-blotato.blotato`) — create video from template.
- **Configuration choices:**
  - Resource: **video**
  - Prompt: `{{$json.Ideas}}`
  - Template: “Generate an infographic displayed across a massive 32x32 grid of TV screens, creating a stunning video wall installation effect.”
  - Template input override: `footerText = "Follow us to learn more great tips!"`
- **Credentials:** Blotato API credential **Blotato GiangxAI**
- **Inputs / outputs:** From Switch branch → outputs a Blotato job/object (includes an `item.id` used later).
- **Edge cases / failures:**
  - Template ID not accessible in the Blotato account.
  - Prompt too short/empty leads to poor or failed generations.
  - API rate limits or rendering queue delays.

#### Node: Generate Tutorial Carousel
- **Type / role:** Blotato node — create video carousel using template.
- **Configuration choices:**
  - Resource: **video**
  - Prompt: `{{ $('Fetch Content Ideas &  Visual').item.json.Ideas }}`
  - Template: “Tutorial Carousel with Monocolor Background”
  - Template inputs include fixed branding/CTA fields:
    - `hashtag: #phattrienbanthan`
    - `authorName: GiangxAI`
    - `companyName: giangxAI`
    - `aspectRatio: 1:1`
    - `profileImage`: public image URL (must be publicly accessible)
    - `ctaButtons`: JSON string list (e.g. `["Flollow", "Share"]`)
    - `ctaGreeting` / `ctaDescription`
- **Credentials:** Blotato GiangxAI
- **Inputs / outputs:** From Switch → outputs job/object with `item.id`.
- **Edge cases / failures:**
  - `profileImage` URL inaccessible or blocked → template may fail.
  - `ctaButtons` must be valid JSON string format expected by template.
  - Uses a cross-node expression referencing Google Sheets node; if the node name changes, expression breaks.

#### Node: Generate Single Text Visual
- **Type / role:** Blotato node — create a simple quote/text visual.
- **Configuration choices:**
  - Resource: **video**
  - Prompt: `{{ $('Fetch Content Ideas &  Visual').item.json.Ideas }}`
  - Template: “A simple slideshow with a single centered text quote on a solid background.”
  - Template input mapping mode: auto-map to `quotes` (but schema shows `quotes` removed); effectively relies on prompt and/or template defaults.
- **Credentials:** Blotato GiangxAI
- **Inputs / outputs:** From Switch → outputs job/object with `item.id`.
- **Edge cases / failures:**
  - Potential mismatch: template expects `quotes` but schema indicates removed; may still work if Blotato template ignores it, but could fail if template strictly requires it.
  - Node expression depends on the Google Sheets node name.

#### Node: Generate an Image Slideshow
- **Type / role:** Blotato node — create image slideshow with prominent text.
- **Configuration choices:**
  - Resource: **video**
  - Prompt: `{{$json.Ideas}}`
  - Template: “Image Slideshow with Prominent Text”
  - Template inputs (fixed defaults):
    - `textStyle: modern`
    - `transition: zoom`
    - `aspectRatio: 1:1`
    - `aiImageModel: replicate/luma/photon`
    - `textPosition: center`
    - `customTextPositionPercent: 30` (required)
- **Credentials:** Blotato GiangxAI
- **Inputs / outputs:** From Switch → outputs job/object with `item.id`.
- **Edge cases / failures:**
  - Selected AI image model may be unavailable or slower/expensive depending on Blotato plan.
  - Required field `customTextPositionPercent` must remain set.

---

### Block 3 — Rendering & Asset Check (Polling Loop)
**Overview:**  
Implements a wait-and-poll loop: after creation, the workflow waits, fetches the render status from Blotato, checks if it is done, and repeats until ready.

**Nodes involved:**
- Wait for Visual Rendering
- Fetch Generated Visual
- Visual Ready?

#### Node: Wait for Visual Rendering
- **Type / role:** Wait node — pauses execution before polling.
- **Configuration choices:** Uses n8n Wait (no explicit duration shown; default behavior depends on how it’s configured in UI—commonly “Wait for” time interval, or “Resume webhook”. The node contains a `webhookId`, so it may be configured for webhook resume style in some setups.)
- **Inputs / outputs:** Receives the Blotato generation output → passes through to Fetch Generated Visual after waiting.
- **Edge cases / failures:**
  - If configured as “resume via webhook” but no webhook call occurs, execution may stall indefinitely.
  - If configured as time-based, too short → excessive polling; too long → delays posting.

#### Node: Fetch Generated Visual
- **Type / role:** Blotato node — retrieve video job status/details.
- **Configuration choices:**
  - Operation: **get**
  - Resource: **video**
  - Video ID: `{{$json.item.id}}` (expects the upstream Blotato create node output to have `item.id`)
- **Credentials:** Blotato GiangxAI
- **Inputs / outputs:** From Wait → outputs full job object (`item.status`, plus media URLs when done).
- **Edge cases / failures:**
  - If upstream output shape differs (no `item.id`), this expression fails.
  - Temporary API errors while render is processing.

#### Node: Visual Ready?
- **Type / role:** IF node — determines whether render is complete.
- **Configuration choices:**
  - Condition expression checks whether:
    - single item: `$json.item.status === 'done'`
    - multiple items: every item has `.json.item.status === 'done'`
  - If true → proceed to publish
  - If false → loop back to wait
- **Inputs / outputs:**
  - Input from Fetch Generated Visual
  - **True** output → Instagram Auto Publishing
  - **False** output → Wait for Visual Rendering (loop)
- **Edge cases / failures:**
  - If node receives **0 items**, expression logic may evaluate unexpectedly.
  - If Blotato uses different status strings (e.g., `completed`), loop will never end.
  - If multiple items arrive unexpectedly, the expression attempts to handle it, but downstream publish node expects a single set of URLs.

---

### Block 4 — Instagram Auto Publishing
**Overview:**  
Publishes the generated media to Instagram via Blotato, using the idea text as the caption and media URLs from the fetched render output.

**Nodes involved:**
- Instagram Auto Publishing

#### Node: Instagram Auto Publishing
- **Type / role:** Blotato node — publish post to a social account (Instagram) through Blotato.
- **Configuration choices:**
  - Account: `giangxai.aff` (accountId 25299 in Blotato)
  - Caption/text: `{{ $('Fetch Content Ideas &  Visual').item.json.Ideas }}`
  - Media URLs: `{{ $('Fetch Generated Visual').item.json.item?.mediaUrl || $('Fetch Generated Visual').item.json.item?.imageUrls || '' }}`
  - **onError:** `continueErrorOutput` (critical: node will not stop the workflow; it will route errors to a secondary output)
- **Credentials:** Blotato GiangxAI
- **Inputs / outputs:**
  - Main success output → Mark Content as Published
  - Error output → Log Publishing Error
- **Edge cases / failures:**
  - If `mediaUrl` and `imageUrls` are both missing, it posts empty media → likely API error.
  - Instagram publishing constraints (format, aspect ratio, video length, carousel limits) can cause publish failures.
  - Cross-node references by name: renaming nodes breaks expressions.

---

### Block 5 — Status Tracking & Logging
**Overview:**  
Updates the Google Sheet row status to indicate completion or error for the specific processed `row_number`.

**Nodes involved:**
- Mark Content as Published
- Log Publishing Error

#### Node: Mark Content as Published
- **Type / role:** Google Sheets node — update operation for status tracking.
- **Configuration choices:**
  - Operation: **update**
  - Match column: `row_number` (uses mapping/matching columns)
  - Sets:
    - `Status = "Done"`
    - `row_number = {{ $('Fetch Content Ideas &  Visual').item.json.row_number }}`
- **Credentials:** Google Sheets OAuth2 **GiangXAI**
- **Inputs / outputs:** From Instagram Auto Publishing success output → updates sheet.
- **Edge cases / failures:**
  - If `row_number` is missing/not numeric, update may target wrong row or fail.
  - Sheet permissions / protected ranges can block update.

#### Node: Log Publishing Error
- **Type / role:** Google Sheets node — update operation for error logging.
- **Configuration choices:**
  - Operation: **update**
  - Match: `row_number`
  - Sets:
    - `Status = "Error"`
    - `row_number = {{ $('Fetch Content Ideas &  Visual').item.json.row_number }}`
- **Credentials:** Google Sheets OAuth2 **GiangXAI**
- **Inputs / outputs:** From Instagram Auto Publishing error output → updates sheet.
- **Edge cases / failures:**
  - Only logs “Error” status; does not capture error message/details (consider adding an “ErrorMessage” column if needed).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Content Schedule Trigger | Schedule Trigger | Scheduled entry point (06:00, 11:00, 20:00) | — | Fetch Content Ideas &  Visual | ## Idea Intake & Scheduling<br>Automatically pull content ideas and decide which viral format to generate. |
| Fetch Content Ideas &  Visual | Google Sheets | Fetch first row with Status=Pending | Content Schedule Trigger | Route by Viral Content Format | ## Idea Intake & Scheduling<br>Automatically pull content ideas and decide which viral format to generate. |
| Route by Viral Content Format | Switch | Route by `Visual` string | Fetch Content Ideas &  Visual | Generate Whiteboard Infographic; Generate Tutorial Carousel; Generate Single Text Visual; Generate an Image Slideshow | ## Idea Intake & Scheduling<br>Automatically pull content ideas and decide which viral format to generate. |
| Generate Whiteboard Infographic | Blotato | Create video via infographic video wall template | Route by Viral Content Format | Wait for Visual Rendering | ## Visual Creation (By Viral Format)<br>Generate different viral-ready visuals based on the selected content format. |
| Generate Tutorial Carousel | Blotato | Create video via tutorial carousel template | Route by Viral Content Format | Wait for Visual Rendering | ## Visual Creation (By Viral Format)<br>Generate different viral-ready visuals based on the selected content format. |
| Generate Single Text Visual | Blotato | Create video via single centered text template | Route by Viral Content Format | Wait for Visual Rendering | ## Visual Creation (By Viral Format)<br>Generate different viral-ready visuals based on the selected content format. |
| Generate an Image Slideshow | Blotato | Create video via image slideshow template | Route by Viral Content Format | Wait for Visual Rendering | ## Visual Creation (By Viral Format)<br>Generate different viral-ready visuals based on the selected content format. |
| Wait for Visual Rendering | Wait | Pause before polling render status (loop) | Any Generate* node; Visual Ready? (false branch) | Fetch Generated Visual | ## Rendering & Asset Check<br>Wait for tool Blotato rendering to complete and validate visual output |
| Fetch Generated Visual | Blotato | Poll/get render job by videoId | Wait for Visual Rendering | Visual Ready? | ## Rendering & Asset Check<br>Wait for tool Blotato rendering to complete and validate visual output |
| Visual Ready? | IF | Check if render status is `done`; loop if not | Fetch Generated Visual | Instagram Auto Publishing (true); Wait for Visual Rendering (false) | ## Rendering & Asset Check<br>Wait for tool Blotato rendering to complete and validate visual output |
| Instagram Auto Publishing | Blotato | Publish media to Instagram account | Visual Ready? (true) | Mark Content as Published (success); Log Publishing Error (error output) | ## Instagram Auto Publishing<br>Automatically publish the generated content to Instagram. |
| Mark Content as Published | Google Sheets | Update row Status=Done by row_number | Instagram Auto Publishing (success) | — | ## Status Tracking & Logging<br>Update content status for tracking, retry, and analytics. |
| Log Publishing Error | Google Sheets | Update row Status=Error by row_number | Instagram Auto Publishing (error) | — | ## Status Tracking & Logging<br>Update content status for tracking, retry, and analytics. |
| Sticky Note | Sticky Note | Comment block | — | — | ## Idea Intake & Scheduling<br>Automatically pull content ideas and decide which viral format to generate. |
| Sticky Note1 | Sticky Note | Comment block | — | — | ## Visual Creation (By Viral Format)<br>Generate different viral-ready visuals based on the selected content format. |
| Sticky Note2 | Sticky Note | Comment block | — | — | ## Rendering & Asset Check<br>Wait for tool Blotato rendering to complete and validate visual output |
| Sticky Note3 | Sticky Note | Comment block | — | — | ## Instagram Auto Publishing<br>Automatically publish the generated content to Instagram. |
| Sticky Note4 | Sticky Note | Comment block | — | — | ## Status Tracking & Logging<br>Update content status for tracking, retry, and analytics. |
| Sticky Note5 | Sticky Note | Setup/credits note | — | — | # 🛠️ Workflow Setup Guide<br><br>Author: [GiangxAI](https://www.youtube.com/@giangxai.official)<br><br>## How it works<br>- Content ideas and visual formats are loaded automatically from Google Sheets on a schedule<br>- Each idea is routed by viral content format (carousel, slideshow, single text, whiteboard)<br>- Blotato generates Instagram-ready visuals based on the selected format<br>- The workflow waits for AI rendering to complete<br>- Generated visual assets are retrieved and validated<br>- Visuals are published to Instagram automatically<br>- Google Sheets is updated with publishing status or error logs<br><br>The entire workflow runs end-to-end without manual design, editing, or posting once configured.<br><br>---<br><br>## Setup guide [n8n](https://n8n.partnerlinks.io/giangxai)<br>- Connect [Google Sheets](https://docs.google.com/spreadsheets/d/1Z7BLM6-n18ljill0v2LnMBzWwQUQWp8ONuGBvJroAt4/edit?usp=sharing) to manage content ideas, formats, and status tracking<br>- Add [Blotato](https://blotato.com/?ref=giang9s) API credentials for automated visual creation<br>- Configure Instagram credentials for auto publishing<br>- Review visual format rules and routing logic<br>- Adjust schedule triggers and captions if needed<br><br>Setup is simple and requires no design skills or manual Instagram posting. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named **“Auto Create & Post Instagram Visuals”** (inactive by default while configuring).

2. **Add node: Schedule Trigger**
   - Name: **Content Schedule Trigger**
   - Configure three daily trigger times: **06:00**, **11:00**, **20:00**.

3. **Add node: Google Sheets**
   - Name: **Fetch Content Ideas &  Visual**
   - Credentials: create/select **Google Sheets OAuth2** credential (authorize the Google account that owns the sheet).
   - Document: select the spreadsheet that contains your backlog.
   - Sheet/tab: select your content tab (e.g., “Personal development”).
   - Operation: use a “get”/“read” mode that supports filtering (in n8n Google Sheets v4+ this is typically “Read” with filters).
   - Filter: `Status` **equals** `Pending`.
   - Option: **Return first match** enabled.
   - Ensure your sheet includes at minimum these columns:
     - `Ideas` (text prompt / caption source)
     - `Visual` (one of the exact routing strings)
     - `Status` (Pending/Done/Error)
     - `row_number` (for update matching)

4. **Connect:** Content Schedule Trigger → Fetch Content Ideas &  Visual

5. **Add node: Switch**
   - Name: **Route by Viral Content Format**
   - Add rules (string equals) on expression `{{$json.Visual}}`:
     - “Whiteboard infographic”
     - “Tutorial Carousel”
     - “Single Centered Text”
     - “Image Slide Show”
   - Ensure each rule routes to a separate output.

6. **Connect:** Fetch Content Ideas &  Visual → Route by Viral Content Format

7. **Install/enable Blotato node package** (if not present)
   - Requires the community node/package: `@blotato/n8n-nodes-blotato` (availability depends on your n8n deployment method).

8. **Create Blotato API credentials** in n8n
   - Add credential type **Blotato API**
   - Paste your Blotato API key/token.

9. **Add Blotato generation nodes (4 nodes), one per format**
   - **Generate Whiteboard Infographic**
     - Resource: **video**
     - Operation: create/generate (default for the node)
     - Prompt: `{{$json.Ideas}}`
     - Template: select the infographic “video wall” template
     - Template input: set `footerText` to your CTA text
   - **Generate Tutorial Carousel**
     - Prompt: `{{ $('Fetch Content Ideas &  Visual').item.json.Ideas }}`
     - Template: “Tutorial Carousel with Monocolor Background”
     - Provide required template inputs (notably a publicly accessible `profileImage` URL), plus branding fields.
   - **Generate Single Text Visual**
     - Prompt: `{{ $('Fetch Content Ideas &  Visual').item.json.Ideas }}`
     - Template: single centered text quote visual
   - **Generate an Image Slideshow**
     - Prompt: `{{$json.Ideas}}`
     - Template: “Image Slideshow with Prominent Text”
     - Set required inputs (e.g., `customTextPositionPercent`) and preferred `aiImageModel`, aspect ratio, etc.

10. **Connect Switch outputs:**
    - Whiteboard infographic → Generate Whiteboard Infographic
    - Tutorial Carousel → Generate Tutorial Carousel
    - Single Centered Text → Generate Single Text Visual
    - Image Slide Show → Generate an Image Slideshow

11. **Add node: Wait**
    - Name: **Wait for Visual Rendering**
    - Configure either:
      - A **fixed wait time** (recommended for polling, e.g., 30–120 seconds), **or**
      - A **resume webhook** pattern if you have an external callback (only if Blotato can call back).
    - The provided workflow behaves like a polling loop, so a fixed wait time is typically appropriate.

12. **Connect each Generate* node → Wait for Visual Rendering**

13. **Add node: Blotato (get video)**
    - Name: **Fetch Generated Visual**
    - Resource: **video**
    - Operation: **get**
    - Video ID: `{{$json.item.id}}`

14. **Connect:** Wait for Visual Rendering → Fetch Generated Visual

15. **Add node: IF**
    - Name: **Visual Ready?**
    - Condition: a boolean expression that evaluates render completion, e.g. status equals `done`.
    - Use the workflow’s logic if you want multi-item safety:
      - If single item: `$json.item.status === 'done'`
      - Else: every item’s `.json.item.status === 'done'`
    - True branch: proceed to publishing
    - False branch: loop back to wait

16. **Connect:**
    - Fetch Generated Visual → Visual Ready?
    - Visual Ready? (false) → Wait for Visual Rendering

17. **Add node: Blotato (publish to Instagram)**
    - Name: **Instagram Auto Publishing**
    - Enable **Continue On Fail** / set error handling to route errors (in this workflow: `continueErrorOutput`).
    - Select the Blotato Instagram account (Account ID) you want to publish to.
    - Post text/caption: `{{ $('Fetch Content Ideas &  Visual').item.json.Ideas }}`
    - Media URLs expression:
      - Use `mediaUrl` for single video output, otherwise `imageUrls` for multiple images/carousel.
      - Keep a fallback to avoid undefined (but ideally add a guard that prevents posting if empty).

18. **Connect:** Visual Ready? (true) → Instagram Auto Publishing

19. **Add node: Google Sheets (update success)**
    - Name: **Mark Content as Published**
    - Credentials: same Google Sheets OAuth2
    - Operation: **update**
    - Match by: `row_number`
    - Set `Status` to **Done**
    - Set `row_number` to `{{ $('Fetch Content Ideas &  Visual').item.json.row_number }}`

20. **Add node: Google Sheets (update error)**
    - Name: **Log Publishing Error**
    - Operation: **update**
    - Match by: `row_number`
    - Set `Status` to **Error**
    - Set `row_number` to `{{ $('Fetch Content Ideas &  Visual').item.json.row_number }}`

21. **Connect Instagram publishing outputs:**
    - Instagram Auto Publishing (success/main) → Mark Content as Published
    - Instagram Auto Publishing (error) → Log Publishing Error

22. **(Optional but recommended) Add safeguards**
    - If no “Pending” row found: stop gracefully (e.g., IF node after fetch: “items exist?”).
    - Add a maximum retry counter for the render polling loop to avoid infinite looping.
    - Add columns like `PublishedAt`, `PostId`, `ErrorMessage` and write them back.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Author: GiangxAI | https://www.youtube.com/@giangxai.official |
| Setup guide n8n link | https://n8n.partnerlinks.io/giangxai |
| Example Google Sheet (content ideas/status tracking) | https://docs.google.com/spreadsheets/d/1Z7BLM6-n18ljill0v2LnMBzWwQUQWp8ONuGBvJroAt4/edit?usp=sharing |
| Blotato (API + templates) | https://blotato.com/?ref=giang9s |
| Key operational summary (from workflow note): scheduled sheet intake → format routing → Blotato generation → wait/poll → publish → status update | Embedded in the workflow sticky note “Workflow Setup Guide” |

