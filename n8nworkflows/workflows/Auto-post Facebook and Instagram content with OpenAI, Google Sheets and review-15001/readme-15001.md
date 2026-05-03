Auto-post Facebook and Instagram content with OpenAI, Google Sheets and review

https://n8nworkflows.xyz/workflows/auto-post-facebook-and-instagram-content-with-openai--google-sheets-and-review-15001


# Auto-post Facebook and Instagram content with OpenAI, Google Sheets and review

Good, I have a complete understanding. Let me now write the comprehensive document. 1. Workflow Overview

This workflow is a **semi-autonomous social media marketing agent** for Facebook and Instagram. It is triggered on demand via a web form, routes the execution to either promote a product from a Google Sheets inventory or broadcast a custom agency pitch, uses OpenAI to draft professional copy, pauses for human approval through an interactive form, then publishes approved content to Meta platforms and updates the tracking spreadsheet.

**Logical Blocks:**

- **Block 1 — Input & Routing (Dispatcher):** A form trigger captures the user's choice (Product vs. Custom Pitch) and an IF node routes execution accordingly.
- **Block 2 — Data Acquisition & Selection:** The Product branch reads all rows from a Google Sheet and selects the least-promoted item. The Pitch branch generates a rotating text-only pitch from a hardcoded list.
- **Block 3 — Content Preparation:** Both branches converge on an Image URL Converter, then OpenAI rewrites the product description or pitch into a platform-optimized caption.
- **Block 4 — Human Review Gate:** A Wait node pauses the execution and renders an approval form. Upon form submission, an IF node verifies the decision.
- **Block 5 — Payload Construction:** A Code node assembles separate Facebook and Threads-ready payloads, preserving the pitch/product distinction and row tracking metadata.
- **Block 6 — Publishing & Tracking:** An IF node separates pitch (text-only Facebook post) from product (Facebook photo + Instagram media publish). After product publishing, the Google Sheet "Times Posted" counter is incremented.

---

### 2. Block-by-Block Analysis

#### Block 1 — Input & Routing (Dispatcher)

**Overview:** This block captures the user's intent via a web form dropdown, converts it into a boolean flag, and splits the workflow into two parallel branches.

**Nodes Involved:** Pixril Dispatcher, Manual Dispatcher, Product vs Pitch Router

---

**Pixril Dispatcher**
- **Type:** `n8n-nodes-base.formTrigger` (v2.5)
- **Role:** Entry point. Renders a web form with a dropdown labelled "Post Type" offering two choices: "Product" and "Custom Pitch".
- **Configuration:** Form title: *PIXRIL Marketing Agent*. Form description: *Select the type of AI Agent campaign you want to dispatch to the PIXRIL social channels right now.* Single required dropdown field named `Post Type`.
- **Key expressions:** None; the dropdown value is passed downstream as `$json["Post Type"]`.
- **Input connections:** None (trigger).
- **Output connections:** → Manual Dispatcher.
- **Edge cases:** If the form is submitted without a selection (should not happen due to `requiredField: true`), the dropdown value will be undefined and downstream routing will default to the Product branch (false).

---

**Manual Dispatcher**
- **Type:** `n8n-nodes-base.code` (v2)
- **Role:** Converts the user's dropdown selection into a boolean flag for routing.
- **Configuration:** Runs once per item. Checks `$json["Post Type"]`:
  - "Custom Pitch" → `{ isCustomPitchSlot: true }`
  - Anything else (i.e., "Product") → `{ isCustomPitchSlot: false }`
- **Key expressions:** `$json["Post Type"]`
- **Input connections:** ← Pixril Dispatcher
- **Output connections:** → Product vs Pitch Router
- **Edge cases:** If a value other than "Product" or "Custom Pitch" is somehow injected, it defaults to the Product route (false).

---

**Product vs Pitch Router**
- **Type:** `n8n-nodes-base.if` (v2.3)
- **Role:** Binary router based on the boolean `isCustomPitchSlot`.
- **Configuration:** Condition: `isCustomPitchSlot` equals `true` (boolean comparison, strict type validation).
- **Key expressions:** `{{ $json.isCustomPitchSlot }}`
- **Input connections:** ← Manual Dispatcher
- **Output connections:**
  - TRUE (isCustomPitchSlot = true) → Generate Custom Pitch
  - FALSE (isCustomPitchSlot = false) → Fetch All Agents
- **Edge cases:** If the upstream Code node fails or outputs an unexpected structure, both branches will receive no data and the workflow will stall.

---

#### Block 2 — Data Acquisition & Selection

**Overview:** This block handles the two parallel data sources. The Product branch pulls inventory from Google Sheets and picks the least-promoted item. The Pitch branch selects a rotating text pitch from a hardcoded list.

**Nodes Involved:** Fetch All Agents, Smart Product Selector, Generate Custom Pitch

---

**Fetch All Agents**
- **Type:** `n8n-nodes-base.googleSheets` (v4.7)
- **Role:** Reads all rows from the configured Google Sheet (the product inventory).
- **Configuration:** Operation: Read. Document ID and Sheet Name are hardcoded to a specific spreadsheet (`1gr8u-Bx58-NCbdtArGSL6ytg6IzJeTrFuWNoHCwRJvs`, sheet `pixril_agents.csv`, gid `961906395`). No filters or ranges specified — retrieves all rows.
- **Credential:** `googleSheetsOAuth2Api` (must be configured with a Google account that has read access to the target spreadsheet).
- **Key expressions:** None; uses hardcoded spreadsheet references.
- **Input connections:** ← Product vs Pitch Router (FALSE branch)
- **Output connections:** → Smart Product Selector
- **Edge cases:** If the sheet is empty, the downstream Code node will receive an empty array and `selectedProduct` will be undefined. If the credential lacks access, the node throws an authentication error. If column headers change, the downstream expressions referencing `Times Posted`, `Agent Name`, `Drive Image ID`, etc., will fail.

---

**Smart Product Selector**
- **Type:** `n8n-nodes-base.code` (v2)
- **Role:** Sorts all fetched products and selects the one with the lowest "Times Posted" count. In case of a tie, sorts alphabetically by "Agent Name".
- **Configuration:** Runs once (not per-item). Input is all items from the Google Sheets node. Outputs an object with:
  - `selectedProduct`: the full row object for the winning product
  - `rowIndexToUpdate`: the row number (used later for the sheet update)
  - `isPitch`: always `false` for this branch
- **Key expressions:** Uses `a["Times Posted"] - b["Times Posted"]` for primary sort and `a["Agent Name"].localeCompare(b["Agent Name"])` as tiebreaker.
- **Input connections:** ← Fetch All Agents
- **Output connections:** → Image URL Converter
- **Edge cases:** If all products have been posted equally, the alphabetical tiebreaker ensures deterministic rotation. If the sheet has zero rows, `sortedProducts[0]` will be undefined and subsequent nodes will fail.

---

**Generate Custom Pitch**
- **Type:** `n8n-nodes-base.code` (v2)
- **Role:** Produces a rotating agency pitch from a hardcoded list of three pitch templates. Uses the current date to rotate (`new Date().getDate() % pitches.length`).
- **Configuration:** Runs once. Outputs an object with:
  - `selectedProduct`: object with keys `Agent Name` ("Custom Agency Pitch"), `Base Description` (the selected pitch text), `Etsy Link` ("https://pixril.com"), `Drive Image ID` (empty string), `Times Posted` (-1)
  - `rowIndexToUpdate`: -1 (signals that no sheet row should be updated)
  - `isPitch`: `true`
- **Key expressions:** `new Date().getDate() % pitches.length` for rotation.
- **Input connections:** ← Product vs Pitch Router (TRUE branch)
- **Output connections:** → Image URL Converter
- **Edge cases:** The pitch text contains a placeholder email (`user@example.com`) that should be replaced with a real address. The rotation only cycles through 3 pitches; if more variety is needed, the array must be extended. The fallback image logic in the downstream node will apply since `Drive Image ID` is empty.

---

#### Block 3 — Content Preparation

**Overview:** Both branches converge here. The Image URL Converter transforms Google Drive file IDs into direct image URLs. The AI Copywriter then rewrites the raw description or pitch into a platform-optimized social media caption.

**Nodes Involved:** Image URL Converter, AI Copywriter

---

**Image URL Converter**
- **Type:** `n8n-nodes-base.code` (v2)
- **Role:** Converts a Google Drive file ID into a direct image URL using the `lh3.googleusercontent.com` pattern. Falls back to a placeholder logo if no Drive Image ID is present.
- **Configuration:** Runs once per item. Checks `$json.selectedProduct["Drive Image ID"]`. If present and non-empty, builds `https://lh3.googleusercontent.com/d/{Drive Image ID}`. Otherwise, uses `REPLACE_WITH_YOUR_PIXRIL_LOGO_DRIVE_ID` as fallback.
- **Key expressions:** `$json.selectedProduct["Drive Image ID"]`
- **Input connections:** ← Smart Product Selector OR ← Generate Custom Pitch
- **Output connections:** → AI Copywriter
- **Output data:** Passes through all previous fields plus `imageUrl`.
- **Edge cases:** The fallback constant `REPLACE_WITH_YOUR_PIXRIL_LOGO_DRIVE_ID` must be replaced with a real Drive ID before deployment, or pitch posts will have a broken image URL. If the Drive file ID is invalid or the file is not publicly shared, the URL will return an error when Meta tries to fetch it.

---

**AI Copywriter**
- **Type:** `@n8n/n8n-nodes-langchain.openAi` (v2.1)
- **Role:** Calls OpenAI GPT-4o to rewrite the product description or pitch into a concise, professional social media caption.
- **Configuration:** Model: GPT-4o. Single response message with a detailed system-style prompt embedded as user content. The prompt instructs the model to:
  1. Keep under 480 characters (Threads limit)
  2. Never use emojis
  3. Lead with the benefit, not the feature
  4. Include the exact Etsy link at the end: `{{ $json.selectedProduct["Etsy Link"] }}`
  5. Preserve accurate time/cost saved statistics
  6. Sound authentic
  7. Output only the rewritten copy, no preamble
- **Credential:** `openAiApi` (must be a valid OpenAI API key with GPT-4o access).
- **Key expressions:** `{{ $json.selectedProduct["Agent Name"] }}`, `{{ $json.selectedProduct["Base Description"] }}`, `{{ $json.selectedProduct["Etsy Link"] }}`
- **Input connections:** ← Image URL Converter
- **Output connections:** → Approval Gate
- **Output data:** The AI response is available as `$json.output[0].content[0].text`.
- **Edge cases:** If the OpenAI API key is invalid, expired, or lacks GPT-4o access, the node will throw an authentication or model-not-found error. If the prompt produces output exceeding 480 characters, the downstream platform may truncate it. Rate limits on the OpenAI API could cause failures during high-volume usage.

---

#### Block 4 — Human Review Gate

**Overview:** The workflow pauses execution and generates an interactive web form pre-populated with the AI-generated caption, product name, and image URL. The reviewer selects "Approve" or "Reject". Upon form submission, execution resumes and a conditional node checks the decision.

**Nodes Involved:** Approval Gate, Check Approval

---

**Approval Gate**
- **Type:** `n8n-nodes-base.wait` (v1.1)
- **Role:** Pauses execution and serves a web form for human review.
- **Configuration:** Resume method: Form. Form title: *Pixril Post Approval*. Form description: *Review the AI-generated copy and image below. Select Approve to dispatch to Meta.* Four form fields:
  1. **Product Name** (text, default = `{{ $('Image URL Converter').item.json.selectedProduct['Agent Name'] }}`, required)
  2. **Image URL** (text, default = `{{ $('Image URL Converter').item.json.imageUrl }}`, required)
  3. **AI Generated Copy** (textarea, default = `{{ $json.output[0].content[0].text }}`, required)
  4. **Decision** (dropdown: "Approve" / "Reject", required)
- **Webhook ID:** `1aa4ea48-bce9-438a-856e-afe36a8253a8`
- **Key expressions:** References the Image URL Converter node and the AI Copywriter output.
- **Input connections:** ← AI Copywriter
- **Output connections:** → Check Approval (upon form submission)
- **Edge cases:** The form URL must be accessible to the reviewer. If the reviewer modifies the AI-generated text in the textarea, the edited version is what flows downstream (this is a feature, not a bug). If the workflow times out before the form is submitted (based on n8n execution timeout settings), the execution will fail. The reviewer can edit the Image URL field, but this won't retroactively change the image used for the pitch/product distinction.

---

**Check Approval**
- **Type:** `n8n-nodes-base.if` (v2.3)
- **Role:** Checks whether the reviewer approved or rejected the post.
- **Configuration:** Condition: `Decision` equals "Approve" (string comparison, case-sensitive, strict type validation).
- **Key expressions:** `{{ $json.Decision }}`
- **Input connections:** ← Approval Gate
- **Output connections:**
  - TRUE (Decision = "Approve") → Build Payloads
  - FALSE (Decision = "Reject") → no connection; execution ends silently
- **Edge cases:** If the dropdown value is somehow modified or a different casing is used, the strict string comparison will fail and treat it as a rejection. No error message is sent to the reviewer upon rejection.

---

#### Block 5 — Payload Construction

**Overview:** Assembles the data payloads for Facebook and Threads publishing. Determines whether this is a pitch (text-only, no image, no sheet update) or a product (photo post with image, sheet update required).

**Nodes Involved:** Build Payloads

---

**Build Payloads**
- **Type:** `n8n-nodes-base.code` (v2)
- **Role:** Constructs separate payload objects for Facebook and Threads, and passes through the row index and pitch/product flag.
- **Configuration:** Runs once per item. Retrieves:
  - `approvedCopy` from `$json["AI Generated Copy"]`
  - `imageUrl` from `$json["Image URL"]`
  - `rowIndex` from `$node["Image URL Converter"].json.rowIndexToUpdate`
  - `isPitch` from `$node["Image URL Converter"].json.isPitch`
  
  If `isPitch === true`, sets `imageUrl` to empty string (pitches are text-only on all platforms).

  Output object:
  ```
  {
    facebook: { message: approvedCopy, url: imageUrl },
    threads: { media_type: isPitch ? "TEXT" : "IMAGE", text: approvedCopy, image_url: imageUrl },
    rowIndex: rowIndex,
    isPitch: isPitch
  }
  ```
- **Key expressions:** `$json["AI Generated Copy"]`, `$json["Image URL"]`, `$node["Image URL Converter"].json.rowIndexToUpdate`, `$node["Image URL Converter"].json.isPitch`
- **Input connections:** ← Check Approval (TRUE branch)
- **Output connections:** → Is this a Pitch?
- **Edge cases:** If the Approval Gate form fields are renamed, the expressions `$json["AI Generated Copy"]` and `$json["Image URL"]` will break. The `threads` payload is built for future use but is not currently sent to any publishing node. The `rowIndex` will be `-1` for pitches, which signals downstream nodes to skip the sheet update.

---

#### Block 6 — Publishing & Tracking

**Overview:** Routes approved content to Meta for publishing. Pitches are posted to Facebook as text-only feed posts. Products are posted to Facebook as photo posts and then published to Instagram via the Content Publishing API. After Instagram publishing, the Google Sheet counter is incremented.

**Nodes Involved:** Is this a Pitch?, FB - Post Pitch, Post to Facebook, IG - Create Container, IG - Publish Post, Is this a Product?, Increment Post Count

---

**Is this a Pitch?**
- **Type:** `n8n-nodes-base.if` (v2.3)
- **Role:** Routes based on whether the content is a pitch or a product.
- **Configuration:** Condition: `isPitch` equals `true` (boolean, strict validation).
- **Key expressions:** `{{ $json.isPitch }}`
- **Input connections:** ← Build Payloads
- **Output connections:**
  - TRUE (isPitch = true) → FB - Post Pitch
  - FALSE (isPitch = false) → Post to Facebook
- **Edge cases:** If `isPitch` is undefined or not a boolean, the strict type validation may cause unexpected routing.

---

**FB - Post Pitch**
- **Type:** `n8n-nodes-base.facebookGraphApi` (v1)
- **Role:** Publishes a text-only post to the Facebook Page feed.
- **Configuration:** Node: `785083878011217` (Facebook Page ID). Edge: `feed`. Method: POST. Query parameters: `message` = `{{ $node["Build Payloads"].json.facebook.message }}`. Graph API version: v23.0.
- **Credential:** `facebookGraphApi` (named "Pixril Page (Permanent)" — must be a permanent page access token with `pages_manage_posts` permission).
- **Key expressions:** `{{ $node["Build Payloads"].json.facebook.message }}`
- **Input connections:** ← Is this a Pitch? (TRUE branch)
- **Output connections:** None (end of pitch execution path)
- **Edge cases:** The Page ID is hardcoded and must be replaced for a different Facebook Page. If the token lacks the `pages_manage_posts` permission, the API will return a permission error. If the message exceeds Facebook's character limit, the API may reject it. No image is attached for pitches.

---

**Post to Facebook**
- **Type:** `n8n-nodes-base.facebookGraphApi` (v1)
- **Role:** Publishes a photo post to the Facebook Page.
- **Configuration:** Node: `785083878011217` (Facebook Page ID). Edge: `photos`. Method: POST. Query parameters: `url` = `{{ $node["Build Payloads"].json.facebook.url }}`, `message` = `{{ $node["Build Payloads"].json.facebook.message }}`. Graph API version: v23.0.
- **Credential:** `facebookGraphApi` (same "Pixril Page (Permanent)" credential).
- **Key expressions:** `{{ $node["Build Payloads"].json.facebook.url }}`, `{{ $node["Build Payloads"].json.facebook.message }}`
- **Input connections:** ← Is this a Pitch? (FALSE branch)
- **Output connections:** → IG - Create Container
- **Edge cases:** If the image URL is invalid, inaccessible, or points to a non-public Google Drive file, the Facebook API will fail to download it and return an error. The Page ID is hardcoded.

---

**IG - Create Container**
- **Type:** `n8n-nodes-base.facebookGraphApi` (v1)
- **Role:** Creates an Instagram media container for publishing via the Content Publishing API.
- **Configuration:** Node: `17841472322424687` (Instagram Business Account ID). Edge: `media`. Method: POST. Query parameters: `image_url` = `{{ $node["Build Payloads"].json.facebook.url }}`, `caption` = `{{ $node["Build Payloads"].json.facebook.message }}`. Graph API version: v23.0.
- **Credential:** `facebookGraphApi` (same "Pixril Page (Permanent)" credential).
- **Key expressions:** `{{ $node["Build Payloads"].json.facebook.url }}`, `{{ $node["Build Payloads"].json.facebook.message }}`
- **Input connections:** ← Post to Facebook
- **Output connections:** → IG - Publish Post
- **Edge cases:** The Instagram Business Account ID is hardcoded. The image URL must be publicly accessible. The caption must comply with Instagram's character and hashtag limits. If the container creation fails, the publish step will also fail. There may be a slight delay between container creation and it being ready for publishing; the workflow does not include an explicit wait.

---

**IG - Publish Post**
- **Type:** `n8n-nodes-base.facebookGraphApi` (v1)
- **Role:** Publishes the Instagram media container created in the previous step.
- **Configuration:** Node: `17841472322424687` (Instagram Business Account ID). Edge: `media_publish`. Method: POST. Query parameters: `creation_id` = `{{ $json.id }}` (the container ID returned by the previous node). Graph API version: v23.0.
- **Credential:** `facebookGraphApi` (same "Pixril Page (Permanent)" credential).
- **Key expressions:** `{{ $json.id }}`
- **Input connections:** ← IG - Create Container
- **Output connections:** → Is this a Product?
- **Edge cases:** If the container is not yet ready for publishing (Instagram's processing delay), this call will fail. No retry logic is included.

---

**Is this a Product?**
- **Type:** `n8n-nodes-base.if` (v2.3)
- **Role:** Determines whether to update the Google Sheet (only for products, not pitches).
- **Configuration:** Condition: `isPitch` equals `false` (boolean, using the "false" operator). This is effectively checking "is this a product?" — if isPitch is false, it is a product.
- **Key expressions:** `{{ $node["Build Payloads"].json.isPitch }}`
- **Input connections:** ← IG - Publish Post
- **Output connections:**
  - TRUE (isPitch = false, meaning it IS a product) → Increment Post Count
  - FALSE (isPitch = true, meaning it is a pitch) → no connection; execution ends
- **Edge cases:** This node is only reached via the product path (after Instagram publishing), so `isPitch` should always be `false` here. However, the check exists as a safety guard.

---

**Increment Post Count**
- **Type:** `n8n-nodes-base.googleSheets` (v4.7)
- **Role:** Updates the "Times Posted" column for the selected product row in the Google Sheet, incrementing it by 1.
- **Configuration:** Operation: Update. Document ID and Sheet Name same as Fetch All Agents (`1gr8u-Bx58-NCbdtArGSL6ytg6IzJeTrFuWNoHCwRJvs`, sheet `pixril_agents.csv`). Matching column: `row_number` (used to identify the row to update). Columns to update:
  - `row_number` = `{{ $node["Build Payloads"].json.rowIndex }}`
  - `Times Posted` = `{{ $node["Smart Product Selector"].json.selectedProduct["Times Posted"] + 1 }}`
- **Credential:** `googleSheetsOAuth2Api` (must have edit access to the spreadsheet).
- **Key expressions:** `{{ $node["Build Payloads"].json.rowIndex }}`, `{{ $node["Smart Product Selector"].json.selectedProduct["Times Posted"] + 1 }}`
- **Input connections:** ← Is this a Product? (TRUE branch)
- **Output connections:** None (end of product execution path)
- **Edge cases:** If `row_number` is -1 (pitch scenario, though this path should not be reached), the update will fail or update the wrong row. The `row_number` field must exist in the sheet and be unique. Concurrent executions could cause a race condition if two workflows try to increment the same row simultaneously.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Pixril Dispatcher | formTrigger (v2.5) | Entry point web form for selecting campaign type | None (trigger) | Manual Dispatcher | 1. The Dispatcher — Triggers the workflow via an interactive web form. Routes the execution based on your selection: a specific product promo or a custom agency pitch. |
| Manual Dispatcher | code (v2) | Converts dropdown selection into boolean routing flag | Pixril Dispatcher | Product vs Pitch Router | 1. The Dispatcher — Triggers the workflow via an interactive web form. Routes the execution based on your selection: a specific product promo or a custom agency pitch. |
| Product vs Pitch Router | if (v2.3) | Routes to Pitch or Product branch based on boolean flag | Manual Dispatcher | TRUE → Generate Custom Pitch; FALSE → Fetch All Agents | 2. Data & Selection — Product Route: Pulls inventory from Google Sheets and intelligently selects the least-promoted item. Pitch Route: Generates a rotating, professional agency pitch. |
| Generate Custom Pitch | code (v2) | Generates a rotating agency pitch from hardcoded templates | Product vs Pitch Router (TRUE) | Image URL Converter | 2. Data & Selection — Product Route: Pulls inventory from Google Sheets and intelligently selects the least-promoted item. Pitch Route: Generates a rotating, professional agency pitch. |
| Fetch All Agents | googleSheets (v4.7) | Reads all product rows from the Google Sheet | Product vs Pitch Router (FALSE) | Smart Product Selector | 2. Data & Selection — Product Route: Pulls inventory from Google Sheets and intelligently selects the least-promoted item. Pitch Route: Generates a rotating, professional agency pitch. |
| Smart Product Selector | code (v2) | Selects the product with the lowest Times Posted count | Fetch All Agents | Image URL Converter | 2. Data & Selection — Product Route: Pulls inventory from Google Sheets and intelligently selects the least-promoted item. Pitch Route: Generates a rotating, professional agency pitch. |
| Image URL Converter | code (v2) | Converts Drive Image ID to direct URL; applies fallback logo | Smart Product Selector OR Generate Custom Pitch | AI Copywriter | 3. AI Writing & Human Review — OpenAI drafts professional, platform-specific copy. The workflow pauses and generates a web form for you to review and approve the content before publishing. |
| AI Copywriter | @n8n/n8n-nodes-langchain.openAi (v2.1) | Rewrites product/pitch description into social media caption via GPT-4o | Image URL Converter | Approval Gate | 3. AI Writing & Human Review — OpenAI drafts professional, platform-specific copy. The workflow pauses and generates a web form for you to review and approve the content before publishing. |
| Approval Gate | wait (v1.1) | Pauses execution and serves a web approval form | AI Copywriter | Check Approval | 3. AI Writing & Human Review — OpenAI drafts professional, platform-specific copy. The workflow pauses and generates a web form for you to review and approve the content before publishing. |
| Check Approval | if (v2.3) | Checks if reviewer approved or rejected the post | Approval Gate | TRUE → Build Payloads; FALSE → (end) | 3. AI Writing & Human Review — OpenAI drafts professional, platform-specific copy. The workflow pauses and generates a web form for you to review and approve the content before publishing. |
| Build Payloads | code (v2) | Assembles Facebook and Threads payloads; sets image/row metadata | Check Approval (TRUE) | Is this a Pitch? | 4. Meta Publishing & Tracking — Routes approved content to Meta. Pitches are posted to Facebook as text-only. Products are posted to FB/IG with media, and the Google Sheet is incremented. |
| Is this a Pitch? | if (v2.3) | Routes pitch content to text-only Facebook post, product content to photo post | Build Payloads | TRUE → FB - Post Pitch; FALSE → Post to Facebook | 4. Meta Publishing & Tracking — Routes approved content to Meta. Pitches are posted to Facebook as text-only. Products are posted to FB/IG with media, and the Google Sheet is incremented. |
| FB - Post Pitch | facebookGraphApi (v1) | Publishes text-only pitch to Facebook Page feed | Is this a Pitch? (TRUE) | None (end) | 4. Meta Publishing & Tracking — Routes approved content to Meta. Pitches are posted to Facebook as text-only. Products are posted to FB/IG with media, and the Google Sheet is incremented. |
| Post to Facebook | facebookGraphApi (v1) | Publishes photo post to Facebook Page | Is this a Pitch? (FALSE) | IG - Create Container | 4. Meta Publishing & Tracking — Routes approved content to Meta. Pitches are posted to Facebook as text-only. Products are posted to FB/IG with media, and the Google Sheet is incremented. |
| IG - Create Container | facebookGraphApi (v1) | Creates Instagram media container via Content Publishing API | Post to Facebook | IG - Publish Post | 4. Meta Publishing & Tracking — Routes approved content to Meta. Pitches are posted to Facebook as text-only. Products are posted to FB/IG with media, and the Google Sheet is incremented. |
| IG - Publish Post | facebookGraphApi (v1) | Publishes the Instagram media container | IG - Create Container | Is this a Product? | 4. Meta Publishing & Tracking — Routes approved content to Meta. Pitches are posted to Facebook as text-only. Products are posted to FB/IG with media, and the Google Sheet is incremented. |
| Is this a Product? | if (v2.3) | Guards sheet update so only products (not pitches) increment the counter | IG - Publish Post | TRUE → Increment Post Count; FALSE → (end) | 4. Meta Publishing & Tracking — Routes approved content to Meta. Pitches are posted to Facebook as text-only. Products are posted to FB/IG with media, and the Google Sheet is incremented. |
| Increment Post Count | googleSheets (v4.7) | Increments the Times Posted counter for the selected product row | Is this a Product? (TRUE) | None (end) | 4. Meta Publishing & Tracking — Routes approved content to Meta. Pitches are posted to Facebook as text-only. Products are posted to FB/IG with media, and the Google Sheet is incremented. |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to recreate the entire workflow in a new n8n instance:

**Prerequisites:**
- A Google Sheets OAuth2 credential with read/write access to the target spreadsheet
- An OpenAI API credential with GPT-4o access
- A Facebook Graph API credential (permanent page token) with `pages_manage_posts` and `instagram_content_publish` permissions
- A Google Sheet with columns: `Agent Name`, `Base Description`, `Etsy Link`, `Drive Image ID`, `Times Posted`, `row_number`
- A Google Drive image file ID for your fallback logo

---

**Step 1 — Create the Form Trigger**
1. Add a **Form Trigger** node. Name it `Pixril Dispatcher`.
2. Set Form Title to: `PIXRIL Marketing Agent`
3. Set Form Description to: `Select the type of AI Agent campaign you want to dispatch to the PIXRIL social channels right now.`
4. Add one form field:
   - Type: Dropdown
   - Label: `Post Type`
   - Options: `Product`, `Custom Pitch`
   - Required: Yes

---

**Step 2 — Add the Manual Dispatcher (Code)**
1. Add a **Code** node. Name it `Manual Dispatcher`.
2. Set mode to `Run Once for Each Item`.
3. Insert the following logic:
   - Read `$json["Post Type"]`
   - If "Custom Pitch", return `{ isCustomPitchSlot: true }`
   - Otherwise, return `{ isCustomPitchSlot: false }`
4. Connect: Pixril Dispatcher → Manual Dispatcher

---

**Step 3 — Add the Product vs Pitch Router (IF)**
1. Add an **IF** node. Name it `Product vs Pitch Router`.
2. Configure condition:
   - Combinator: AND
   - Left value: `{{ $json.isCustomPitchSlot }}`
   - Operator: Boolean → Equals
   - Right value: `true`
   - Type validation: Strict
3. Connect: Manual Dispatcher → Product vs Pitch Router

---

**Step 4 — Add the Generate Custom Pitch (Code) — TRUE branch**
1. Add a **Code** node. Name it `Generate Custom Pitch`.
2. Set mode to `Run Once`.
3. Insert logic that:
   - Defines an array of 3 pitch strings (professional, emoji-free agency pitches with a placeholder email)
   - Selects a pitch using `new Date().getDate() % pitches.length`
   - Returns: `{ selectedProduct: { "Agent Name": "Custom Agency Pitch", "Base Description": <selected pitch>, "Etsy Link": "https://pixril.com", "Drive Image ID": "", "Times Posted": -1 }, rowIndexToUpdate: -1, isPitch: true }`
4. Connect: Product vs Pitch Router (TRUE output) → Generate Custom Pitch

---

**Step 5 — Add Fetch All Agents (Google Sheets) — FALSE branch**
1. Add a **Google Sheets** node. Name it `Fetch All Agents`.
2. Operation: Read
3. Select your Google Sheets credential
4. Set Document ID to your spreadsheet ID (e.g., `1gr8u-Bx58-NCbdtArGSL6ytg6IzJeTrFuWNoHCwRJvs`)
5. Set Sheet Name to your sheet (e.g., `pixril_agents.csv`)
6. Leave all filters/default options as-is
7. Connect: Product vs Pitch Router (FALSE output) → Fetch All Agents

---

**Step 6 — Add Smart Product Selector (Code)**
1. Add a **Code** node. Name it `Smart Product Selector`.
2. Set mode to `Run Once`.
3. Insert logic that:
   - Reads `$input.all()` and maps to product array
   - Sorts by `Times Posted` ascending (primary), then `Agent Name` alphabetically (tiebreaker)
   - Returns: `{ selectedProduct: sortedProducts[0], rowIndexToUpdate: sortedProducts[0].row_number, isPitch: false }`
4. Connect: Fetch All Agents → Smart Product Selector

---

**Step 7 — Add Image URL Converter (Code)**
1. Add a **Code** node. Name it `Image URL Converter`.
2. Set mode to `Run Once for Each Item`.
3. Insert logic that:
   - Reads `$json.selectedProduct["Drive Image ID"]`
   - If present and non-empty: `imageUrl = "https://lh3.googleusercontent.com/d/" + Drive Image ID`
   - Otherwise: `imageUrl = "https://lh3.googleusercontent.com/d/" + "REPLACE_WITH_YOUR_PIXRIL_LOGO_DRIVE_ID"` (replace this constant with your actual logo Drive ID)
   - Returns all previous fields plus `imageUrl`
4. Connect: Generate Custom Pitch → Image URL Converter
5. Connect: Smart Product Selector → Image URL Converter

---

**Step 8 — Add AI Copywriter (OpenAI LangChain)**
1. Add an **OpenAI** node (from the LangChain category). Name it `AI Copywriter`.
2. Select your OpenAI credential
3. Model: `gpt-4o`
4. Add a single response message with the following prompt (set as content):
   ```
   You are a professional B2B social media copywriter for Pixril, an AI automation agency.

   REWRITE this product description for maximum engagement on Threads and Facebook.

   Product: {{ $json.selectedProduct["Agent Name"] }}
   Original: {{ $json.selectedProduct["Base Description"] }}

   Guidelines:
   1. Keep under 480 characters (Threads limit).
   2. STRICT RULE: DO NOT USE ANY EMOJIS. Maintain a clean, professional, and authoritative tone.
   3. Lead with the BENEFIT, not the feature.
   4. MUST include this exact link at the end: {{ $json.selectedProduct["Etsy Link"] }}
   5. Maintain accurate "Time Saved" and "Cost Saved" statistics.
   6. Sound authentic—avoid robotic marketing speak.

   Respond ONLY with the rewritten copy. No preamble.
   ```
5. Connect: Image URL Converter → AI Copywriter

---

**Step 9 — Add Approval Gate (Wait)**
1. Add a **Wait** node. Name it `Approval Gate`.
2. Resume method: Form
3. Form Title: `Pixril Post Approval`
4. Form Description: `Review the AI-generated copy and image below. Select Approve to dispatch to Meta.`
5. Add four form fields:
   - **Product Name**: Text input, default `{{ $('Image URL Converter').item.json.selectedProduct['Agent Name'] }}`, required
   - **Image URL**: Text input, default `{{ $('Image URL Converter').item.json.imageUrl }}`, required
   - **AI Generated Copy**: Textarea, default `{{ $json.output[0].content[0].text }}`, required
   - **Decision**: Dropdown with options "Approve" and "Reject", required
6. Connect: AI Copywriter → Approval Gate

---

**Step 10 — Add Check Approval (IF)**
1. Add an **IF** node. Name it `Check Approval`.
2. Configure condition:
   - Combinator: AND
   - Left value: `{{ $json.Decision }}`
   - Operator: String → Equals
   - Right value: `Approve`
   - Type validation: Strict, Case sensitive
3. Connect: Approval Gate → Check Approval (TRUE output continues, FALSE output ends the workflow)

---

**Step 11 — Add Build Payloads (Code)**
1. Add a **Code** node. Name it `Build Payloads`.
2. Set mode to `Run Once for Each Item`.
3. Insert logic that:
   - Reads `$json["AI Generated Copy"]` and `$json["Image URL"]`
   - Reads `$node["Image URL Converter"].json.rowIndexToUpdate` and `$node["Image URL Converter"].json.isPitch`
   - If `isPitch === true`, sets `imageUrl` to empty string
   - Constructs `facebook` payload: `{ message: approvedCopy, url: imageUrl }`
   - Constructs `threads` payload: `{ media_type: isPitch ? "TEXT" : "IMAGE", text: approvedCopy, image_url: imageUrl }`
   - Returns: `{ facebook: facebookPayload, threads: threadsPayload, rowIndex: rowIndex, isPitch: isPitch }`
4. Connect: Check Approval (TRUE) → Build Payloads

---

**Step 12 — Add Is this a Pitch? (IF)**
1. Add an **IF** node. Name it `Is this a Pitch?`
2. Configure condition:
   - Combinator: AND
   - Left value: `{{ $json.isPitch }}`
   - Operator: Boolean → True (single value)
   - Type validation: Strict
3. Connect: Build Payloads → Is this a Pitch?
   - TRUE output → FB - Post Pitch
   - FALSE output → Post to Facebook

---

**Step 13 — Add FB - Post Pitch (Facebook Graph API)**
1. Add a **Facebook Graph API** node. Name it `FB - Post Pitch`.
2. Select your Facebook Graph API credential (permanent page token)
3. Node (Page ID): your Facebook Page ID (e.g., `785083878011217`)
4. Edge: `feed`
5. HTTP Method: POST
6. Graph API Version: `v23.0`
7. Add query parameters:
   - `message` = `{{ $node["Build Payloads"].json.facebook.message }}`
8. Connect: Is this a Pitch? (TRUE) → FB - Post Pitch
9. No further connections from this node.

---

**Step 14 — Add Post to Facebook (Facebook Graph API)**
1. Add a **Facebook Graph API** node. Name it `Post to Facebook`.
2. Select your Facebook Graph API credential
3. Node (Page ID): your Facebook Page ID (e.g., `785083878011217`)
4. Edge: `photos`
5. HTTP Method: POST
6. Graph API Version: `v23.0`
7. Add query parameters:
   - `url` = `{{ $node["Build Payloads"].json.facebook.url }}`
   - `message` = `{{ $node["Build Payloads"].json.facebook.message }}`
8. Connect: Is this a Pitch? (FALSE) → Post to Facebook

---

**Step 15 — Add IG - Create Container (Facebook Graph API)**
1. Add a **Facebook Graph API** node. Name it `IG - Create Container`.
2. Select your Facebook Graph API credential
3. Node (IG Business Account ID): your Instagram Business Account ID (e.g., `17841472322424687`)
4. Edge: `media`
5. HTTP Method: POST
6. Graph API Version: `v23.0`
7. Add query parameters:
   - `image_url` = `{{ $node["Build Payloads"].json.facebook.url }}`
   - `caption` = `{{ $node["Build Payloads"].json.facebook.message }}`
8. Connect: Post to Facebook → IG - Create Container

---

**Step 16 — Add IG - Publish Post (Facebook Graph API)**
1. Add a **Facebook Graph API** node. Name it `IG - Publish Post`.
2. Select your Facebook Graph API credential
3. Node (IG Business Account ID): your Instagram Business Account ID (same as above)
4. Edge: `media_publish`
5. HTTP Method: POST
6. Graph API Version: `v23.0`
7. Add query parameters:
   - `creation_id` = `{{ $json.id }}`
8. Connect: IG - Create Container → IG - Publish Post

---

**Step 17 — Add Is this a Product? (IF)**
1. Add an **IF** node. Name it `Is this a Product?`
2. Configure condition:
   - Combinator: AND
   - Left value: `{{ $node["Build Payloads"].json.isPitch }}`
   - Operator: Boolean → False (single value)
   - Type validation: Strict
3. Connect: IG - Publish Post → Is this a Product?
   - TRUE output (isPitch is false = it IS a product) → Increment Post Count
   - FALSE output → no connection (end)

---

**Step 18 — Add Increment Post Count (Google Sheets)**
1. Add a **Google Sheets** node. Name it `Increment Post Count`.
2. Operation: Update
3. Select your Google Sheets credential (same as Fetch All Agents)
4. Set Document ID to the same spreadsheet ID
5. Set Sheet Name to the same sheet name
6. Matching column: `row_number`
7. Mapping mode: Define below
8. Columns to update:
   - `row_number` = `{{ $node["Build Payloads"].json.rowIndex }}`
   - `Times Posted` = `{{ $node["Smart Product Selector"].json.selectedProduct["Times Posted"] + 1 }}`
9. Disable "Attempt to convert types" and "Convert fields to string"
10. Connect: Is this a Product? (TRUE) → Increment Post Count

---

**Step 19 — Configure Credentials**
1. **Google Sheets OAuth2**: Create and connect a Google OAuth2 credential with scope for Sheets read/write. Ensure the connected Google account has access to the target spreadsheet.
2. **OpenAI API**: Create and connect an OpenAI API credential with a valid key that has GPT-4o access.
3. **Facebook Graph API**: Create and connect a Facebook Graph API credential. Use a permanent page access token with `pages_manage_posts` and `instagram_content_publish` permissions. Verify the Page ID and Instagram Business Account ID match your Meta Business Suite.

---

**Step 20 — Update Hardcoded Values**
1. In the **Image URL Converter** code, replace `REPLACE_WITH_YOUR_PIXRIL_LOGO_DRIVE_ID` with your actual Google Drive file ID for the fallback logo image.
2. In the **Generate Custom Pitch** code, replace `user@example.com` with your actual contact email address.
3. In the **Fetch All Agents** and **Increment Post Count** nodes, update the Document ID and Sheet Name to match your spreadsheet.
4. In the **FB - Post Pitch** and **Post to Facebook** nodes, update the Facebook Page ID.
5. In the **IG - Create Container** and **IG - Publish Post** nodes, update the Instagram Business Account ID.

---

**Step 21 — Test the Workflow**
1. Click "Test URL" on the Pixril Dispatcher form trigger to get the live form URL.
2. Open the form URL in a browser.
3. Select "Product" and submit.
4. Verify the workflow reaches the Approval Gate and renders the review form.
5. Approve the post and verify it publishes to Facebook and Instagram.
6. Check the Google Sheet to confirm the "Times Posted" counter incremented.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Built by Pixril — specialists in production-ready AI workflows and automation templates for n8n. | https://pixril.etsy.com |
| The `threads` payload is constructed in Build Payloads but not currently dispatched to any Threads API node. It is prepared for future integration. | Build Payloads node |
| The hardcoded Facebook Page ID (`785083878011217`) and Instagram Business Account ID (`17841472322424687`) must be replaced with your own before deployment. | FB - Post Pitch, Post to Facebook, IG - Create Container, IG - Publish Post |
| The fallback image constant `REPLACE_WITH_YOUR_PIXRIL_LOGO_DRIVE_ID` in the Image URL Converter must be replaced with a real Google Drive file ID, and that file must be publicly accessible. | Image URL Converter node |
| The placeholder email `user@example.com` in the pitch templates should be replaced with a real contact address. | Generate Custom Pitch node |
| Instagram Content Publishing API has a processing delay between container creation and publish readiness. If `IG - Publish Post` fails with a container-not-ready error, consider adding a Wait node (10–30 seconds) between IG - Create Container and IG - Publish Post. | IG - Create Container → IG - Publish Post |
| No error handling or retry logic is included. If any Meta API call fails, the workflow will terminate without updating the Google Sheet or notifying the user. Consider adding Error Trigger or Set nodes for production use. | All Meta API nodes |
| The Google Sheet must include a `row_number` column (populated automatically by n8n when reading sheets) for the Increment Post Count node to match the correct row. | Fetch All Agents, Increment Post Count |
| Concurrent workflow executions could cause race conditions on the "Times Posted" counter if two instances select the same product simultaneously. Consider adding a locking mechanism or queuing system for production use. | Smart Product Selector, Increment Post Count |
| The workflow is set to timezone `Asia/Karachi`. Adjust this in workflow settings if needed for your locale. | Workflow settings |
| The workflow is inactive by default (`active: false`). Activate it manually after configuration. | Workflow settings |