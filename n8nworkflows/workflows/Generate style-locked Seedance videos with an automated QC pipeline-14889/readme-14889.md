Generate style-locked Seedance videos with an automated QC pipeline

https://n8nworkflows.xyz/workflows/generate-style-locked-seedance-videos-with-an-automated-qc-pipeline-14889


# Generate style-locked Seedance videos with an automated QC pipeline

Now I'll write the complete document.

### 1. Workflow Overview

This workflow implements a **cinematic style-locked video generation and automated quality-control pipeline** for VFX and editorial teams. Its purpose is to take a new shot description alongside a director-approved hero reference image, generate three style-locked video variants via the **Seedance AI model** (BytePlus/Doubao), run automated QC checks against a per-show visual style profile (contrast, colour match, brightness variance), and then route approved and rejected results simultaneously to **Slack**, **Gmail**, and **Jira** — with **Telegram** alerts for any rejected variant.

The workflow is organized into five logical blocks:

| Block | Name | Purpose |
|-------|------|---------|
| 1.1 | **Intake & Style Profile Extraction** | Receives a POST webhook with shot metadata and hero reference URL, validates required fields, and extracts or defaults the show's visual style profile. |
| 1.2 | **Variant Generation & Polling** | Builds three style-locked prompts (Primary Comp, Alt Framing, Style Stress Test), sends each to the Seedance API with the hero reference image attached, and polls every 20 seconds until each job reports `succeeded`. |
| 1.3 | **Automated Style QC** | Scores each completed variant against the show's configured thresholds (contrast, colour match, brightness variance), assigns a QC grade (A/B/C/F), and splits variants into approved and rejected paths. |
| 1.4 | **Delivery & Notifications** | Aggregates all QC results and fans out approved and rejected variants to three channels in parallel: a Slack QC report, a styled HTML email to the editorial contact, and a Jira review task. Rejected variants are never silently dropped. |
| 1.5 | **Error Handling** | Catches any uncaught workflow error and immediately posts a Slack alert with the error message and timestamp. |

**Key design principle:** Every generated variant is anchored to a director-approved reference frame. QC scores are measured against per-show numeric thresholds — not subjective guesswork. Only variants that pass all three QC checks are cleared for editorial delivery.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1 — Intake & Style Profile Extraction

**Overview:** This block serves as the workflow's entry point. An HTTP webhook receives a POST request containing shot metadata and a hero reference URL. A Code node validates the required fields and constructs the show's visual style profile, applying sensible defaults for any optional parameters.

**Nodes Involved:**
- Webhook: Style Transfer Request2
- Validate & Extract Show Style Profile2

---

**Webhook: Style Transfer Request2**

| Property | Detail |
|----------|--------|
| **Type** | Webhook (n8n-nodes-base.webhook, v2) |
| **Technical Role** | Entry-point trigger; listens for POST requests on the `/style-look-transfer` endpoint |
| **Configuration** | HTTP method: POST; Path: `style-look-transfer`; No authentication on the webhook itself; Webhook ID: `style-look-transfer-001` |
| **Key Expressions** | None — passes the raw request body downstream |
| **Input Connections** | None (trigger node) |
| **Output Connections** | → Validate & Extract Show Style Profile2 |
| **Edge Cases / Failures** | Requests with non-POST methods are rejected by the webhook. If the body is empty or malformed JSON, downstream validation will catch missing required fields. Network-level timeout is governed by the n8n instance's webhook timeout settings. |

---

**Validate & Extract Show Style Profile2**

| Property | Detail |
|----------|--------|
| **Type** | Code (n8n-nodes-base.code, v2) — JavaScript |
| **Technical Role** | Validates required inbound fields and assembles the show style profile with defaults |
| **Configuration** | Reads `$input.first().json.body`. Throws descriptive `Error` objects for missing `newShotDescription`, `shotCode`, or `heroReferenceUrl`. For all other fields, applies defaults: `colorGrade` → `"desaturated teal-orange blockbuster"`, `contrastLevel` → `"high contrast deep blacks"`, `grainTexture` → `"fine 35mm film grain"`, `lightingMood` → `"cinematic directional key light"`, `atmosphereStyle` → `"subtle atmospheric haze"`, `colorTemp` → `"cool shadows warm highlights"`, `minContrastScore` → `0.70`, `minColorMatchScore` → `0.75`, `maxBrightnessVar` → `0.30`, `sequenceCode` → extracted from first segment of `shotCode` split by `_`, `episodeId` → `"EP001"`, `showName` → `"Production"`, `vendorName` → `"Internal"`, `deliveryEmail` → `null` |
| **Key Expressions** | `body.shotCode.split('_')[0]` for default `sequenceCode` |
| **Input Connections** | ← Webhook: Style Transfer Request2 |
| **Output Connections** | → Build Style-Locked Variants2 |
| **Edge Cases / Failures** | Missing any of the three required fields throws a synchronous error that propagates to the error-handling block. If `deliveryEmail` is null, the downstream Gmail node will fail at send time — this is intentional (no email sent when no recipient is provided). `shotCode` values without an underscore will result in `sequenceCode` equalling the entire `shotCode`. |

---

#### Block 1.2 — Variant Generation & Polling

**Overview:** This block constructs three style-locked prompt variants, formats each into a Seedance API request body with the hero reference image, submits them, and then enters a polling loop that checks job status every 20 seconds until the model reports `succeeded`.

**Nodes Involved:**
- Build Style-Locked Variants2
- Build Style-Anchored Request2
- Seedance: Generate Style-Locked Variant2
- Merge Variant Job + Style Data2
- Poll: Check Variant Generation Status1
- Variant Ready?2
- Wait 20s Before Retry1

---

**Build Style-Locked Variants2**

| Property | Detail |
|----------|--------|
| **Type** | Code (n8n-nodes-base.code, v2) — JavaScript |
| **Technical Role** | Creates three variant objects, each containing a variant ID, name, icon, role description, and a `safePrompt` string that embeds the full show style fingerprint |
| **Configuration** | Reads the incoming shot description and style profile. Constructs a `styleLock` string concatenating all six style attributes plus the lock clause `"SHOW STYLE LOCKED — must match approved hero reference"`. Three variants are defined: **STYLE-V1** (Primary Comp, 🎬), **STYLE-V2** (Alt Framing, 🔄), **STYLE-V3** (Style Stress Test, 🔍). Each `safePrompt` wraps the description + style lock + variant-specific direction + `--duration 5 --camerafixed true` parameters. The `JSON.stringify(...).slice(1,-1)` pattern is used to safely escape the prompt string. |
| **Key Expressions** | `JSON.stringify(...)` used as an ad-hoc string escaper by slicing off the enclosing quotes |
| **Input Connections** | ← Validate & Extract Show Style Profile2 |
| **Output Connections** | → Build Style-Anchored Request2 (outputs 3 items, one per variant) |
| **Edge Cases / Failures** | If the style profile contains special characters that break JSON serialization, the `slice(1,-1)` escape trick may produce malformed strings. Extremely long `newShotDescription` values could exceed the Seedance model's context window. |

---

**Build Style-Anchored Request2**

| Property | Detail |
|----------|--------|
| **Type** | Code (n8n-nodes-base.code, v2) — JavaScript |
| **Technical Role** | Constructs the exact JSON request body expected by the Seedance API, attaching the hero reference image as an `image_url` content block |
| **Configuration** | Hardcoded model: `seedance-1-5-pro-251215`. Request body structure: `{ model, content: [{ type: "text", text: safePrompt }, { type: "image_url", image_url: { url: heroReferenceUrl } }], generate_audio: false, ratio: "adaptive", duration: 5, watermark: false }`. The body is stored as a stringified `requestBody` field on the item for downstream parsing. |
| **Key Expressions** | `JSON.stringify(body)` to serialize the request body |
| **Input Connections** | ← Build Style-Locked Variants2 |
| **Output Connections** | → Seedance: Generate Style-Locked Variant2 |
| **Edge Cases / Failures** | If `heroReferenceUrl` is not a publicly accessible URL, the Seedance API will reject the request. The `adaptive` ratio means the model decides aspect ratio — if a specific ratio is required, it must be hardcoded here. |

---

**Seedance: Generate Style-Locked Variant2**

| Property | Detail |
|----------|--------|
| **Type** | HTTP Request (n8n-nodes-base.httpRequest, v4.3) |
| **Technical Role** | Submits the generation request to the Seedance/BytePlus API and returns the task ID |
| **Configuration** | Method: POST; URL: `https://ark.ap-southeast.bytepluses.com/api/v3/contents/generations/tasks`; Body: parsed from `$json.requestBody` via `JSON.parse()`; Headers: `Authorization: Bearer YOUR_TOKEN_HERE`, `Content-Type: application/json`; Body specification: JSON |
| **Key Expressions** | `={{ JSON.parse($json.requestBody) }}` for the JSON body parameter |
| **Input Connections** | ← Build Style-Anchored Request2 |
| **Output Connections** | → Merge Variant Job + Style Data2 |
| **Edge Cases / Failures** | The `Authorization` header currently contains a placeholder `Bearer YOUR_TOKEN_HERE` — this **must** be replaced with an n8n HTTP Header Auth credential. Invalid or expired tokens return 401/403. Rate limiting on the Seedance API may cause 429 responses. Network timeouts default to n8n's HTTP request timeout (typically 30s). If the API response does not contain an `id` field, the downstream merge node will propagate `undefined`. |

---

**Merge Variant Job + Style Data2**

| Property | Detail |
|----------|--------|
| **Type** | Code (n8n-nodes-base.code, v2) — JavaScript |
| **Technical Role** | Combines the HTTP response (containing the task `id`) with the full variant/style data from the Build Style-Anchored Request node |
| **Configuration** | Reads `$input.first().json` (the HTTP response) and `$('Build Style-Anchored Request2').first().json` (the variant metadata). Merges them into a single object: `{ ...variantData, id: httpResult.id }`. |
| **Key Expressions** | `$('Build Style-Anchored Request2').first().json` — cross-node reference by name |
| **Input Connections** | ← Seedance: Generate Style-Locked Variant2 |
| **Output Connections** | → Poll: Check Variant Generation Status1 |
| **Edge Cases / Failures** | If the HTTP response is unexpected (e.g., error object instead of success), `httpResult.id` may be undefined. Cross-node references by name are fragile — renaming the referenced node will break this expression. |

---

**Poll: Check Variant Generation Status1**

| Property | Detail |
|----------|--------|
| **Type** | HTTP Request (n8n-nodes-base.httpRequest, v4.3) |
| **Technical Role** | Checks the Seedance task status via a GET request using the task ID from the merge node |
| **Configuration** | Method: GET (default); URL: `https://ark.ap-southeast.bytepluses.com/api/v3/contents/generations/tasks/{{ $json.id }}`; Headers: `Authorization: Bearer YOUR_TOKEN_HERE` |
| **Key Expressions** | `={{ $json.id }}` interpolated in the URL path |
| **Input Connections** | ← Merge Variant Job + Style Data2; also ← Wait 20s Before Retry1 (loop-back) |
| **Output Connections** | → Variant Ready?2 |
| **Edge Cases / Failures** | Same credential placeholder issue as the generation node. If the task ID is invalid or the task has been deleted, the API may return 404. The `status` field in the response drives the IF condition — any unexpected status string (e.g., `"failed"`, `"error"`) will route to the "false" branch, causing an infinite polling loop unless a maximum retry count is added externally. |

---

**Variant Ready?2**

| Property | Detail |
|----------|--------|
| **Type** | IF (n8n-nodes-base.if, v2) |
| **Technical Role** | Branches based on whether the Seedance task status equals `"succeeded"` |
| **Configuration** | Combinator: AND; Single condition: `$json.status` equals (string, case-insensitive) `"succeeded"`. Type validation: strict. |
| **Key Expressions** | `={{ $json.status }}` |
| **Input Connections** | ← Poll: Check Variant Generation Status1 |
| **Output Connections** | True (index 0) → Run Style QC Check2; False (index 1) → Wait 20s Before Retry1 |
| **Edge Cases / Failures** | If the Seedance API returns `status: "failed"`, the workflow will keep polling indefinitely (the "false" branch only waits and retries). There is no max-retry or timeout guard. A permanently failed job creates an infinite loop that will eventually exhaust the execution timeout. |

---

**Wait 20s Before Retry1**

| Property | Detail |
|----------|--------|
| **Type** | Wait (n8n-nodes-base.wait, v1.1) |
| **Technical Role** | Pauses execution for 20 seconds before re-polling the Seedance task status |
| **Configuration** | Amount: 20; Unit: seconds; Webhook ID: `slt-wait-001` (used for the resumption webhook in production mode) |
| **Key Expressions** | None |
| **Input Connections** | ← Variant Ready?2 (false branch) |
| **Output Connections** | → Poll: Check Variant Generation Status1 |
| **Edge Cases / Failures** | In n8n production mode, the Wait node uses a webhook to resume execution after the delay. If the n8n instance is restarted during a wait, the execution may not resume. In test/debug mode, the wait may behave differently (blocking vs. webhook). The 20-second interval is hardcoded — if Seedance jobs typically take longer, this creates unnecessary polling overhead. |

---

#### Block 1.3 — Automated Style QC

**Overview:** Once a variant generation job completes, this block scores the resulting video against the show's style thresholds for contrast, colour match, and brightness variance. It assigns a QC grade (A, B, C, or F) and splits variants into approved and rejected paths.

**Nodes Involved:**
- Run Style QC Check2
- QC Gate: Style Approved?2

---

**Run Style QC Check2**

| Property | Detail |
|----------|--------|
| **Type** | Code (n8n-nodes-base.code, v2) — JavaScript |
| **Technical Role** | Performs automated QC scoring against the show style profile and determines pass/fail with a letter grade |
| **Configuration** | Reads the poll result (`$input.first().json`) and retrieves variant data via `$('Merge Variant Job + Style Data2').first().json`. Extracts `videoUrl` from `pollResult.content.video_url` (falls back to a not-found message with job ID). Computes scores: `resolutionScore` based on resolution (1080p→1.0, 720p→0.85, else→0.65); `contrastScore` = min(1.0, resolutionScore×0.9 + random×0.1); `colorMatchScore` = min(1.0, resolutionScore×0.85 + random×0.15); `brightnessVar` = max(0, 0.4 − resolutionScore×0.2 + random×0.1). Compares each score against the show thresholds: `contrastScore ≥ minContrastScore`, `colorMatchScore ≥ minColorMatchScore`, `brightnessVar ≤ maxBrightnessVar`. Determines `overallPassed` (all three must pass). Assigns grade: passed → A if contrastScore > 0.90 else B; failed → C if contrastScore > 0.70 else F. Builds `qcNotes` with pass/fail details. |
| **Key Expressions** | `$('Merge Variant Job + Style Data2').first().json` for cross-node data retrieval; `Math.random()` for simulated QC variance |
| **Input Connections** | ← Variant Ready?2 (true branch) |
| **Output Connections** | → QC Gate: Style Approved?2 |
| **Edge Cases / Failures** | The QC scores use `Math.random()` to simulate variance — this is **not** a real visual analysis. In production, this should be replaced with actual image/video analysis (e.g., via a computer vision API). If the Seedance response does not include `content.video_url`, the video URL defaults to a not-found string. The cross-node reference to "Merge Variant Job + Style Data2" will break if that node is renamed. If the poll result lacks a `resolution` field, `resolutionScore` defaults to 0.65. |

---

**QC Gate: Style Approved?2**

| Property | Detail |
|----------|--------|
| **Type** | IF (n8n-nodes-base.if, v2) |
| **Technical Role** | Routes variants based on the `overallPassed` boolean from the QC check |
| **Configuration** | Combinator: AND; Single condition: `$json.qc.overallPassed` equals `true` (boolean, strict type validation). Condition version: 1. |
| **Key Expressions** | `={{ $json.qc.overallPassed }}` |
| **Input Connections** | ← Run Style QC Check2 |
| **Output Connections** | True (index 0) → Aggregate QC Results2; False (index 1) → Telegram: Notify on QC Rejection1 **and** Aggregate QC Results2 |
| **Edge Cases / Failures** | Both branches feed into Aggregate QC Results2 — rejected variants are still aggregated (they appear in the report), but they additionally trigger a Telegram notification. This is intentional: rejected variants are never silently dropped. If `qc.overallPassed` is undefined (e.g., QC node failed), the condition will evaluate to false, routing to rejection — a safe default. |

---

#### Block 1.4 — Delivery & Notifications

**Overview:** This block aggregates QC results from all three variants, formats Slack and email reports, and dispatches them in parallel to Slack, Gmail, and Jira. Rejected variants additionally trigger a Telegram notification before aggregation.

**Nodes Involved:**
- Telegram: Notify on QC Rejection1
- Aggregate QC Results2
- Slack: Post Style QC Report1
- Gmail: Send QC Report to Editorial1
- Jira: Create Style Review Task1

---

**Telegram: Notify on QC Rejection1**

| Property | Detail |
|----------|--------|
| **Type** | Telegram (n8n-nodes-base.telegram, v1.1) |
| **Technical Role** | Sends an immediate Telegram message when a variant fails QC, providing on-the-go visibility for supervisors |
| **Configuration** | Chat ID: `YOUR_TELEGRAM_CHAT_ID` (placeholder); Parse mode: Markdown; Message template includes shot code, sequence code, object type, brief, total passes, generated passes (passLines), folder structure, and QC pass/fail status. Authentication via Telegram Bot credential. |
| **Key Expressions** | `={{ $json.shotCode }}`, `={{ $json.allQcPassed ? '✅ All QC Passed' : '⚠️ Some Passes Need Review' }}`, etc. |
| **Input Connections** | ← QC Gate: Style Approved?2 (false branch) |
| **Output Connections** | None (terminal notification) |
| **Edge Cases / Failures** | The chat ID is a placeholder and **must** be replaced. If the Telegram Bot credential is not connected, the node will fail. The Markdown parse mode will throw if the message contains unescaped special characters. Note: the message template references fields like `objectType`, `removalBrief`, `totalPasses`, `passLines`, `folderStructure`, `allQcPassed` that appear to originate from a different (clean plate) workflow — in the current style-transfer context, several of these fields will be undefined and render as empty strings. |

---

**Aggregate QC Results2**

| Property | Detail |
|----------|--------|
| **Type** | Code (n8n-nodes-base.code, v2) — JavaScript |
| **Technical Role** | Collects all variant QC results (both approved and rejected), formats Slack-friendly summary strings, and produces a single aggregated item for downstream delivery nodes |
| **Configuration** | Reads `$input.all()` (all variant items). Separates approved vs. rejected variants by `qc.overallPassed`. Builds `slackApproved` string: for each approved variant, produces a multi-line block with variant icon, name, ID, grade, video URL link, and score percentages. Builds `slackRejected` string: for each rejected variant, produces a line with ❌ icon, variant name, and QC notes. Outputs a single item with: shot metadata, show style, approved/rejected variant arrays, counts, Slack-formatted strings, `allPassed` boolean, and `generatedAt` timestamp. |
| **Key Expressions** | `(d.qc.contrastScore*100).toFixed(0)` for percentage formatting; `<${d.videoUrl}|Watch Variant>` for Slack link formatting |
| **Input Connections** | ← QC Gate: Style Approved?2 (true branch) **and** ← QC Gate: Style Approved?2 (false branch) **and** ← Telegram: Notify on QC Rejection1 (implicitly, since QC Gate false feeds both Telegram and Aggregate) |
| **Output Connections** | → Slack: Post Style QC Report1, → Gmail: Send QC Report to Editorial1, → Jira: Create Style Review Task1 (all three in parallel) |
| **Edge Cases / Failures** | If no items reach this node (edge case: all three variant generation jobs failed before QC), `items[0]` would be undefined and throw. The Slack link syntax `<url|text>` is Slack-specific — if the `videoUrl` is the not-found fallback string, the link will be broken. If all variants are rejected, `slackApproved` defaults to `'None passed QC'`. |

---

**Slack: Post Style QC Report1**

| Property | Detail |
|----------|--------|
| **Type** | Slack (n8n-nodes-base.slack, v2.3) |
| **Technical Role** | Posts a formatted QC report to a designated Slack channel |
| **Configuration** | Authentication: OAuth2; Channel: `YOUR_SLACK_CHANNEL_ID` (placeholder, list mode); Message includes: shot code, show name, episode, vendor, shot description, approved/rejected counts, approved variant details with video links and scores, rejected variant details with reasons, hero reference link, and generation timestamp. |
| **Key Expressions** | `={{ $json.shotCode }}`, `={{ $json.slackApproved || 'None passed QC' }}`, `={{ $json.slackRejected }}`, `<{{ $json.heroReferenceUrl }}|View Approved Style Ref>` |
| **Input Connections** | ← Aggregate QC Results2 |
| **Output Connections** | None (terminal notification) |
| **Edge Cases / Failures** | Channel ID is a placeholder — must be replaced with a real channel ID. If the Slack OAuth2 credential is not configured or the bot lacks permission to post in the target channel, the node returns a Slack API error. The `mrkdwn` formatting (bold with `*`, links with `<url|text>`) requires a compatible Slack client. |

---

**Gmail: Send QC Report to Editorial1**

| Property | Detail |
|----------|--------|
| **Type** | Gmail (n8n-nodes-base.gmail, v2.2) |
| **Technical Role** | Sends a styled HTML email with the full QC report to the editorial contact specified in the request |
| **Configuration** | Recipient: `={{ $json.deliveryEmail }}` (from the original webhook payload); Subject: `[Style QC] {shotCode} – {totalApproved} Variants Approved | {showName}`; Message: full HTML with dark-theme styling (background `#0d1117`, text `#e0e0e0`), includes show/episode/vendor info, a summary table (total variants, approved count in green, rejected count in red), approved variants section, rejected variants section, show style profile in a `<pre>` block with monospace green text, and a footer disclaimer. Authentication: Gmail OAuth2 credential. |
| **Key Expressions** | `={{ $json.deliveryEmail }}`, `={{ $json.totalApproved > 0 ? $json.slackApproved : 'No variants passed QC this run.' }}`, `={{ $json.showStyle.colorGrade }}`, etc. |
| **Input Connections** | ← Aggregate QC Results2 |
| **Output Connections** | None (terminal notification) |
| **Edge Cases / Failures** | If `deliveryEmail` is `null` (the default when not provided in the webhook payload), Gmail will throw a validation error — there is no guard for this. The HTML content uses inline styles which may be stripped by some email clients. The `<pre>` block with show style profile may wrap poorly on narrow email clients. |

---

**Jira: Create Style Review Task1**

| Property | Detail |
|----------|--------|
| **Type** | Jira (n8n-nodes-base.jira, v1) |
| **Technical Role** | Creates a Jira review task summarizing the QC results for the shot |
| **Configuration** | Project: `YOUR_JIRA_PROJECT_ID` (placeholder, list mode); Issue Type: `YOUR_JIRA_ISSUE_TYPE_ID` (placeholder, "Task"); Summary: `[Style QC] {shotCode} – {showName} | ✅ {totalApproved} / ❌ {totalRejected}`; No additional fields configured. Authentication: Jira Cloud credential. |
| **Key Expressions** | `={{ $json.shotCode }}`, `={{ $json.totalApproved }}`, `={{ $json.totalRejected }}` |
| **Input Connections** | ← Aggregate QC Results2 |
| **Output Connections** | None (terminal action) |
| **Edge Cases / Failures** | Both Project ID and Issue Type ID are placeholders and must be replaced. If the Jira credential is not connected or the user lacks permission to create issues in the target project, the node returns a Jira API error. The summary field uses emoji characters (✅/❌) which may not render correctly in all Jira configurations. No description field is populated — the Jira task contains only the summary line. |

---

#### Block 1.5 — Error Handling

**Overview:** Any uncaught error anywhere in the workflow triggers this path. An Error Trigger node captures the error, and a Slack node immediately posts an alert with the error message and timestamp so the team can respond without waiting for a scheduled check.

**Nodes Involved:**
- On Workflow Error
- Slack: Error Alert

---

**On Workflow Error**

| Property | Detail |
|----------|--------|
| **Type** | Error Trigger (n8n-nodes-base.errorTrigger, v1) |
| **Technical Role** | Catches any unhandled error in the workflow and passes error details downstream |
| **Configuration** | No parameters — automatically receives the error object |
| **Key Expressions** | None |
| **Input Connections** | None (automatic trigger on any workflow error) |
| **Output Connections** | → Slack: Error Alert |
| **Edge Cases / Failures** | Only catches unhandled errors. Errors inside try/catch blocks in Code nodes will not trigger this path. If the Slack: Error Alert node itself fails (e.g., credential issue), the error alert is lost silently. |

---

**Slack: Error Alert**

| Property | Detail |
|----------|--------|
| **Type** | Slack (n8n-nodes-base.slack, v2.3) |
| **Technical Role** | Posts an error alert to a Slack channel |
| **Configuration** | Authentication: OAuth2; Channel: `C0ANFAL4WJ2` (channel name: "social"); Message: `❌ *Style Look Transfer Workflow Failed*` followed by the error message and timestamp. |
| **Key Expressions** | `={{ $json.message }}`, `={{ new Date().toISOString() }}` |
| **Input Connections** | ← On Workflow Error |
| **Output Connections** | None (terminal notification) |
| **Edge Cases / Failures** | If the Slack OAuth2 credential is not configured, this alert will itself fail — creating a silent failure. The channel ID appears to be a real channel — confirm it is the intended alert channel. |

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|-----------|-----------|-----------------|----------------|-----------------|-------------|
| Overview: Style Look Transfer Pipeline1 | Sticky Note | Documentation — workflow overview, how it works, and setup steps | — | — | ## 🎨 Style Look Transfer — AI Video QC Pipeline. How it works: This workflow automates cinematic style transfer for VFX and editorial pipelines. It takes a new shot description and a hero reference image, generates three style-locked video variants via the Seedance AI model, runs automated QC checks against your show's approved style profile, then routes results to Slack, email, and Jira. The key idea: every generated variant is anchored to a director-approved reference frame. QC scores (contrast, colour match, brightness variance) are measured against per-show thresholds — not guesswork. Only variants that pass are sent to editorial. Setup steps: 1. Webhook — use the `/style-look-transfer` endpoint as your trigger. Send `shotCode`, `newShotDescription`, and `heroReferenceUrl` in the POST body. 2. Seedance API — replace the `Authorization` header value with your own Seedance API key stored as an n8n credential. 3. Slack — connect your Slack OAuth2 credential and update the `channelId` to your target channel. 4. Gmail — connect a Gmail OAuth2 credential. The `deliveryEmail` field in the request body controls where the QC report is sent. 5. Jira — connect your Jira Cloud credential and update the `project` and `issueType` IDs to match your board. 6. Telegram (optional) — connect your Telegram Bot credential and confirm the `chatId` is correct. 7. Do a test run with a sample payload — check Slack for the QC report and confirm Jira task creation. |
| Section: Intake & Style Profile1 | Sticky Note | Section label — Intake & Style Profile Extraction | — | — | ## 📡 Intake & Style Profile Extraction. Receives a POST request with shot metadata and hero reference URL. Validates required fields, then extracts the show's visual style profile — colour grade, contrast thresholds, grain, lighting mood — which is used to lock every generated variant to the same approved look. |
| Section: Variant Generation & Polling1 | Sticky Note | Section label — Variant Generation | — | — | ## 🎬 Variant Generation. Builds three style-locked prompts — Primary Comp, Alt Framing, and Style Stress Test. Each prompt embeds the full show style fingerprint. Requests are sent to the Seedance model with the hero reference image attached, then polled every 20 seconds until each job completes. |
| Section: Style QC Engine1 | Sticky Note | Section label — Automated Style QC | — | — | ## 🔍 Automated Style QC. Scores each variant against show thresholds for contrast, colour match, and brightness variance. Variants that fall below the minimum scores are flagged for supervisor review and blocked from delivery. A QC grade (A, B, C, or F) is assigned per variant. |
| Section: Delivery & Notifications1 | Sticky Note | Section label — Editorial Delivery & Notifications | — | — | ## 📤 Editorial Delivery & Notifications. Aggregates QC results and routes approved variants to three channels simultaneously: a Slack QC report, a styled HTML email to the editorial contact, and a Jira review task. Rejected variants are listed with failure reasons — never silently dropped. |
| Security: Credentials Note1 | Sticky Note | Security warning — credential management | — | — | ## 🔐 Credentials & Security. Replace all hardcoded `Authorization` bearer tokens with n8n credentials (HTTP Header Auth). Use OAuth2 for Slack and Gmail. Store your Seedance API key as a named credential — never paste raw tokens into node parameters before sharing this template. |
| Section: Error Handling | Sticky Note | Section label — Error Handling | — | — | ## ⚠️ Error Handling. Any uncaught workflow error triggers this path. A Slack alert is posted immediately with the error message and timestamp so the team can act without waiting for a scheduled check. |
| Webhook: Style Transfer Request2 | Webhook (v2) | Entry-point — receives POST with shot metadata and hero reference URL | — | Validate & Extract Show Style Profile2 | ## 📡 Intake & Style Profile Extraction. Receives a POST request with shot metadata and hero reference URL. Validates required fields, then extracts the show's visual style profile — colour grade, contrast thresholds, grain, lighting mood — which is used to lock every generated variant to the same approved look. |
| Validate & Extract Show Style Profile2 | Code (v2) | Validates required fields and constructs show style profile with defaults | Webhook: Style Transfer Request2 | Build Style-Locked Variants2 | ## 📡 Intake & Style Profile Extraction. Receives a POST request with shot metadata and hero reference URL. Validates required fields, then extracts the show's visual style profile — colour grade, contrast thresholds, grain, lighting mood — which is used to lock every generated variant to the same approved look. |
| Build Style-Locked Variants2 | Code (v2) | Creates three variant prompt objects (Primary Comp, Alt Framing, Style Stress Test) | Validate & Extract Show Style Profile2 | Build Style-Anchored Request2 | ## 🎬 Variant Generation. Builds three style-locked prompts — Primary Comp, Alt Framing, and Style Stress Test. Each prompt embeds the full show style fingerprint. Requests are sent to the Seedance model with the hero reference image attached, then polled every 20 seconds until each job completes. |
| Build Style-Anchored Request2 | Code (v2) | Constructs the Seedance API request body with text prompt and hero reference image | Build Style-Locked Variants2 | Seedance: Generate Style-Locked Variant2 | ## 🎬 Variant Generation. Builds three style-locked prompts — Primary Comp, Alt Framing, and Style Stress Test. Each prompt embeds the full show style fingerprint. Requests are sent to the Seedance model with the hero reference image attached, then polled every 20 seconds until each job completes. |
| Seedance: Generate Style-Locked Variant2 | HTTP Request (v4.3) | Submits video generation request to Seedance API | Build Style-Anchored Request2 | Merge Variant Job + Style Data2 | ## 🎬 Variant Generation. Builds three style-locked prompts — Primary Comp, Alt Framing, and Style Stress Test. Each prompt embeds the full show style fingerprint. Requests are sent to the Seedance model with the hero reference image attached, then polled every 20 seconds until each job completes. ; ## 🔐 Credentials & Security. Replace all hardcoded `Authorization` bearer tokens with n8n credentials (HTTP Header Auth). Use OAuth2 for Slack and Gmail. Store your Seedance API key as a named credential — never paste raw tokens into node parameters before sharing this template. |
| Merge Variant Job + Style Data2 | Code (v2) | Merges Seedance task ID with variant/style metadata | Seedance: Generate Style-Locked Variant2 | Poll: Check Variant Generation Status1 | ## 🎬 Variant Generation. Builds three style-locked prompts — Primary Comp, Alt Framing, and Style Stress Test. Each prompt embeds the full show style fingerprint. Requests are sent to the Seedance model with the hero reference image attached, then polled every 20 seconds until each job completes. |
| Poll: Check Variant Generation Status1 | HTTP Request (v4.3) | GETs Seedance task status using the task ID | Merge Variant Job + Style Data2; Wait 20s Before Retry1 | Variant Ready?2 | ## 🎬 Variant Generation. Builds three style-locked prompts — Primary Comp, Alt Framing, and Style Stress Test. Each prompt embeds the full show style fingerprint. Requests are sent to the Seedance model with the hero reference image attached, then polled every 20 seconds until each job completes. ; ## 🔐 Credentials & Security. Replace all hardcoded `Authorization` bearer tokens with n8n credentials (HTTP Header Auth). Use OAuth2 for Slack and Gmail. Store your Seedance API key as a named credential — never paste raw tokens into node parameters before sharing this template. |
| Variant Ready?2 | IF (v2) | Branches: status=succeeded → QC; otherwise → wait and retry | Poll: Check Variant Generation Status1 | Run Style QC Check2 (true); Wait 20s Before Retry1 (false) | ## 🎬 Variant Generation. Builds three style-locked prompts — Primary Comp, Alt Framing, and Style Stress Test. Each prompt embeds the full show style fingerprint. Requests are sent to the Seedance model with the hero reference image attached, then polled every 20 seconds until each job completes. |
| Wait 20s Before Retry1 | Wait (v1.1) | Pauses 20 seconds before re-polling Seedance task status | Variant Ready?2 (false) | Poll: Check Variant Generation Status1 | ## 🎬 Variant Generation. Builds three style-locked prompts — Primary Comp, Alt Framing, and Style Stress Test. Each prompt embeds the full show style fingerprint. Requests are sent to the Seedance model with the hero reference image attached, then polled every 20 seconds until each job completes. |
| Run Style QC Check2 | Code (v2) | Scores variant against show style thresholds and assigns QC grade | Variant Ready?2 (true) | QC Gate: Style Approved?2 | ## 🔍 Automated Style QC. Scores each variant against show thresholds for contrast, colour match, and brightness variance. Variants that fall below the minimum scores are flagged for supervisor review and blocked from delivery. A QC grade (A, B, C, or F) is assigned per variant. |
| QC Gate: Style Approved?2 | IF (v2) | Routes approved variants to aggregation; rejected variants to Telegram + aggregation | Run Style QC Check2 | Aggregate QC Results2 (true); Telegram: Notify on QC Rejection1 + Aggregate QC Results2 (false) | ## 🔍 Automated Style QC. Scores each variant against show thresholds for contrast, colour match, and brightness variance. Variants that fall below the minimum scores are flagged for supervisor review and blocked from delivery. A QC grade (A, B, C, or F) is assigned per variant. |
| Telegram: Notify on QC Rejection1 | Telegram (v1.1) | Sends Telegram alert when a variant fails QC | QC Gate: Style Approved?2 (false) | — | ## 🔍 Automated Style QC. Scores each variant against show thresholds for contrast, colour match, and brightness variance. Variants that fall below the minimum scores are flagged for supervisor review and blocked from delivery. A QC grade (A, B, C, or F) is assigned per variant. ; ## 🔐 Credentials & Security. Replace all hardcoded `Authorization` bearer tokens with n8n credentials (HTTP Header Auth). Use OAuth2 for Slack and Gmail. Store your Seedance API key as a named credential — never paste raw tokens into node parameters before sharing this template. |
| Aggregate QC Results2 | Code (v2) | Collects all variant QC results into a single aggregated item for delivery | QC Gate: Style Approved?2 (true + false) | Slack: Post Style QC Report1; Gmail: Send QC Report to Editorial1; Jira: Create Style Review Task1 | ## 📤 Editorial Delivery & Notifications. Aggregates QC results and routes approved variants to three channels simultaneously: a Slack QC report, a styled HTML email to the editorial contact, and a Jira review task. Rejected variants are listed with failure reasons — never silently dropped. |
| Slack: Post Style QC Report1 | Slack (v2.3) | Posts formatted QC report to Slack channel | Aggregate QC Results2 | — | ## 📤 Editorial Delivery & Notifications. Aggregates QC results and routes approved variants to three channels simultaneously: a Slack QC report, a styled HTML email to the editorial contact, and a Jira review task. Rejected variants are listed with failure reasons — never silently dropped. ; ## 🔐 Credentials & Security. Replace all hardcoded `Authorization` bearer tokens with n8n credentials (HTTP Header Auth). Use OAuth2 for Slack and Gmail. Store your Seedance API key as a named credential — never paste raw tokens into node parameters before sharing this template. |
| Gmail: Send QC Report to Editorial1 | Gmail (v2.2) | Sends styled HTML QC report email to editorial contact | Aggregate QC Results2 | — | ## 📤 Editorial Delivery & Notifications. Aggregates QC results and routes approved variants to three channels simultaneously: a Slack QC report, a styled HTML email to the editorial contact, and a Jira review task. Rejected variants are listed with failure reasons — never silently dropped. ; ## 🔐 Credentials & Security. Replace all hardcoded `Authorization` bearer tokens with n8n credentials (HTTP Header Auth). Use OAuth2 for Slack and Gmail. Store your Seedance API key as a named credential — never paste raw tokens into node parameters before sharing this template. |
| Jira: Create Style Review Task1 | Jira (v1) | Creates a Jira review task with QC summary | Aggregate QC Results2 | — | ## 📤 Editorial Delivery & Notifications. Aggregates QC results and routes approved variants to three channels simultaneously: a Slack QC report, a styled HTML email to the editorial contact, and a Jira review task. Rejected variants are listed with failure reasons — never silently dropped. ; ## 🔐 Credentials & Security. Replace all hardcoded `Authorization` bearer tokens with n8n credentials (HTTP Header Auth). Use OAuth2 for Slack and Gmail. Store your Seedance API key as a named credential — never paste raw tokens into node parameters before sharing this template. |
| On Workflow Error | Error Trigger (v1) | Catches any unhandled workflow error | — | Slack: Error Alert | ## ⚠️ Error Handling. Any uncaught workflow error triggers this path. A Slack alert is posted immediately with the error message and timestamp so the team can act without waiting for a scheduled check. |
| Slack: Error Alert | Slack (v2.3) | Posts error alert to Slack channel | On Workflow Error | — | ## ⚠️ Error Handling. Any uncaught workflow error triggers this path. A Slack alert is posted immediately with the error message and timestamp so the team can act without waiting for a scheduled check. ; ## 🔐 Credentials & Security. Replace all hardcoded `Authorization` bearer tokens with n8n credentials (HTTP Header Auth). Use OAuth2 for Slack and Gmail. Store your Seedance API key as a named credential — never paste raw tokens into node parameters before sharing this template. |

---

### 4. Reproducing the Workflow from Scratch

Below is a step-by-step procedure to recreate this workflow manually in n8n.

**Prerequisites:**
- n8n instance (self-hosted or cloud) with the following credentials pre-configured:
  - Seedance/BytePlus API key (HTTP Header Auth — Bearer token)
  - Slack OAuth2 credential (with `chat:write` and `channels:read` scopes)
  - Gmail OAuth2 credential
  - Jira Cloud OAuth2 credential (with issue creation permissions)
  - Telegram Bot credential (optional, for rejection alerts)

---

**Step 1 — Create the Webhook Entry Point**

1. Add a **Webhook** node. Name it `Webhook: Style Transfer Request2`.
2. Set HTTP Method to `POST`.
3. Set Path to `style-look-transfer`.
4. Leave Options as default (no authentication on the webhook).
5. Note the webhook ID (auto-generated; can optionally set to `style-look-transfer-001`).

**Step 2 — Add the Validation & Style Profile Node**

6. Add a **Code** node. Name it `Validate & Extract Show Style Profile2`.
7. Set the JavaScript code to:
   - Read `$input.first().json.body`
   - Validate that `newShotDescription`, `shotCode`, and `heroReferenceUrl` are present (throw `Error` if missing)
   - Construct the `showStyle` object with all six visual attributes and three QC thresholds, each with a default fallback
   - Extract `sequenceCode` from the first underscore-delimited segment of `shotCode`
   - Default optional fields: `episodeId` → `"EP001"`, `showName` → `"Production"`, `vendorName` → `"Internal"`, `deliveryEmail` → `null`
   - Return a single item with all fields plus `requestTimestamp`
8. Connect: Webhook → Validate & Extract Show Style Profile2

**Step 3 — Build the Three Style-Locked Variants**

9. Add a **Code** node. Name it `Build Style-Locked Variants2`.
10. Set the JavaScript code to:
    - Read shot description and style profile from the incoming item
    - Build a `styleLock` string from all six style attributes plus the lock clause
    - Define three variant objects: STYLE-V1 (Primary Comp), STYLE-V2 (Alt Framing), STYLE-V3 (Style Stress Test)
    - Each variant includes: `variantId`, `variantName`, `variantIcon`, `variantRole`, and `safePrompt`
    - Use `JSON.stringify(...).slice(1,-1)` for prompt string escaping
    - Each prompt appends `--duration 5 --camerafixed true`
    - Return 3 items (one per variant), each spreading all incoming fields
11. Connect: Validate & Extract Show Style Profile2 → Build Style-Locked Variants2

**Step 4 — Build the Seedance Request Body**

12. Add a **Code** node. Name it `Build Style-Anchored Request2`.
13. Set the JavaScript code to:
    - Read the incoming item
    - Construct a request body with: `model: "seedance-1-5-pro-251215"`, `content` array (text prompt + image_url with hero reference), `generate_audio: false`, `ratio: "adaptive"`, `duration: 5`, `watermark: false`
    - Store the body as a stringified `requestBody` field
    - Return a single item with all incoming fields plus `requestBody`
14. Connect: Build Style-Locked Variants2 → Build Style-Anchored Request2

**Step 5 — Submit to Seedance API**

15. Add an **HTTP Request** node. Name it `Seedance: Generate Style-Locked Variant2`.
16. Configure:
    - Method: `POST`
    - URL: `https://ark.ap-southeast.bytepluses.com/api/v3/contents/generations/tasks`
    - Body specification: JSON
    - JSON Body: `={{ JSON.parse($json.requestBody) }}`
    - Send Headers: enabled
    - Header 1: `Authorization` = `Bearer YOUR_TOKEN_HERE` (replace with n8n HTTP Header Auth credential)
    - Header 2: `Content-Type` = `application/json`
17. Connect: Build Style-Anchored Request2 → Seedance: Generate Style-Locked Variant2

**Step 6 — Merge Task ID with Variant Data**

18. Add a **Code** node. Name it `Merge Variant Job + Style Data2`.
19. Set the JavaScript code to:
    - Read `$input.first().json` (HTTP response with task `id`)
    - Read `$('Build Style-Anchored Request2').first().json` (variant metadata)
    - Return `{ ...variantData, id: httpResult.id }`
20. Connect: Seedance: Generate Style-Locked Variant2 → Merge Variant Job + Style Data2

**Step 7 — Poll for Variant Completion**

21. Add an **HTTP Request** node. Name it `Poll: Check Variant Generation Status1`.
22. Configure:
    - Method: `GET` (default)
    - URL: `=https://ark.ap-southeast.bytepluses.com/api/v3/contents/generations/tasks/{{ $json.id }}`
    - Send Headers: enabled
    - Header 1: `Authorization` = `Bearer YOUR_TOKEN_HERE` (replace with same credential as Step 5)
23. Connect: Merge Variant Job + Style Data2 → Poll: Check Variant Generation Status1

**Step 8 — Check If Variant Is Ready**

24. Add an **IF** node. Name it `Variant Ready?2`.
25. Configure:
    - Combinator: AND
    - Condition: `$json.status` equals (string, case-insensitive) `"succeeded"`
    - Type validation: strict
26. Connect: Poll → Variant Ready?2
    - True output (index 0) → Run Style QC Check2
    - False output (index 1) → Wait 20s Before Retry1

**Step 9 — Add the Wait Node for Polling Retry**

27. Add a **Wait** node. Name it `Wait 20s Before Retry1`.
28. Set Amount: `20` (seconds).
29. Connect: Wait 20s Before Retry1 → Poll: Check Variant Generation Status1 (loop-back)

**Step 10 — Run Style QC Check**

30. Add a **Code** node. Name it `Run Style QC Check2`.
31. Set the JavaScript code to:
    - Read the poll result and retrieve variant data via `$('Merge Variant Job + Style Data2').first().json`
    - Extract `videoUrl` from `pollResult.content.video_url` (with fallback)
    - Compute `resolutionScore` based on resolution field (1080p→1.0, 720p→0.85, else→0.65)
    - Compute `contrastScore`, `colorMatchScore`, `brightnessVar` using resolution-based formula plus `Math.random()` variance
    - Compare each against show thresholds (`minContrastScore`, `minColorMatchScore`, `maxBrightnessVar`)
    - Determine `overallPassed` (all three checks must pass)
    - Assign QC grade: A/B if passed (A if contrast > 0.90), C/F if failed
    - Build `qcNotes` with detailed pass/fail explanation
    - Return a single item with all variant metadata plus `videoUrl`, `jobId`, `resolution`, `qc` object, and `generatedAt`
32. Connect: Variant Ready?2 (true) → Run Style QC Check2

**Step 11 — QC Gate: Approve or Reject**

33. Add an **IF** node. Name it `QC Gate: Style Approved?2`.
34. Configure:
    - Combinator: AND
    - Condition: `$json.qc.overallPassed` equals `true` (boolean, strict type validation)
    - Condition version: 1
35. Connect: Run Style QC Check2 → QC Gate: Style Approved?2
    - True output (index 0) → Aggregate QC Results2
    - False output (index 1) → Telegram: Notify on QC Rejection1 **and** Aggregate QC Results2 (two connections)

**Step 12 — Telegram Notification for Rejected Variants**

36. Add a **Telegram** node. Name it `Telegram: Notify on QC Rejection1`.
37. Configure:
    - Chat ID: `YOUR_TELEGRAM_CHAT_ID` (replace with your actual chat ID)
    - Parse Mode: Markdown
    - Text: Template with shot code, sequence code, QC status, pass details, folder structure, and timestamp
38. Connect: QC Gate: Style Approved?2 (false) → Telegram: Notify on QC Rejection1

**Step 13 — Aggregate QC Results**

39. Add a **Code** node. Name it `Aggregate QC Results2`.
40. Set the JavaScript code to:
    - Read all items from `$input.all()`
    - Separate approved and rejected variants
    - Build `slackApproved` string with variant icons, names, grades, video links, and score percentages
    - Build `slackRejected` string with ❌ markers and QC notes
    - Return a single item with shot metadata, approved/rejected arrays, counts, Slack-formatted strings, `allPassed` boolean, and `generatedAt`
41. Connect: QC Gate: Style Approved?2 (true) → Aggregate QC Results2; QC Gate: Style Approved?2 (false) → Aggregate QC Results2

**Step 14 — Slack QC Report**

42. Add a **Slack** node. Name it `Slack: Post Style QC Report1`.
43. Configure:
    - Authentication: OAuth2 (select your Slack credential)
    - Select: Channel
    - Channel ID: `YOUR_SLACK_CHANNEL_ID` (replace with your channel ID)
    - Text: Formatted message with shot code, show name, episode, vendor, shot description, approved/rejected counts, approved variant details with video links and scores, rejected variant details, hero reference link, and timestamp
44. Connect: Aggregate QC Results2 → Slack: Post Style QC Report1

**Step 15 — Gmail QC Report**

45. Add a **Gmail** node. Name it `Gmail: Send QC Report to Editorial1`.
46. Configure:
    - Authentication: OAuth2 (select your Gmail credential)
    - Send To: `={{ $json.deliveryEmail }}`
    - Subject: `=[Style QC] {{ $json.shotCode }} – {{ $json.totalApproved }} Variants Approved | {{ $json.showName }}`
    - Message: Full HTML with dark theme, summary table, approved/rejected sections, show style profile in `<pre>` block, and footer disclaimer
47. Connect: Aggregate QC Results2 → Gmail: Send QC Report to Editorial1

**Step 16 — Jira Review Task**

48. Add a **Jira** node. Name it `Jira: Create Style Review Task1`.
49. Configure:
    - Authentication: Select your Jira Cloud credential
    - Project: `YOUR_JIRA_PROJECT_ID` (replace with your project ID)
    - Issue Type: `YOUR_JIRA_ISSUE_TYPE_ID` (replace with "Task" or your type ID)
    - Summary: `=[Style QC] {{ $json.shotCode }} – {{ $json.showName }} | ✅ {{ $json.totalApproved }} / ❌ {{ $json.totalRejected }}`
50. Connect: Aggregate QC Results2 → Jira: Create Style Review Task1

**Step 17 — Error Handling Path**

51. Add an **Error Trigger** node. Name it `On Workflow Error`.
52. Add a **Slack** node. Name it `Slack: Error Alert`.
53. Configure Slack node:
    - Authentication: OAuth2 (select your Slack credential)
    - Select: Channel
    - Channel ID: your error alert channel ID (e.g., `C0ANFAL4WJ2` with name "social")
    - Text: `❌ *Style Look Transfer Workflow Failed*\n\nError: {{ $json.message }}\nTime: {{ new Date().toISOString() }}`
54. Connect: On Workflow Error → Slack: Error Alert

**Step 18 — Credential Replacement (Critical)**

55. For both HTTP Request nodes (Seedance Generate and Poll), replace the hardcoded `Bearer YOUR_TOKEN_HERE` with an **n8n HTTP Header Auth** credential:
    - Go to Credentials → New Credential → HTTP Header Auth
    - Header Name: `Authorization`
    - Header Value: `Bearer <your-actual-seedance-api-key>`
    - Select this credential in both HTTP Request nodes' header configuration
56. Confirm all OAuth2 credentials are connected for Slack (×2), Gmail, and Jira nodes.
57. Confirm Telegram Bot credential is connected and chat ID is correct.

**Step 19 — Test the Workflow**

58. Activate the workflow.
59. Send a test POST request to the webhook URL `/style-look-transfer` with a JSON body containing at minimum: `shotCode`, `newShotDescription`, and `heroReferenceUrl`.
60. Verify: Slack QC report posted, email received (if `deliveryEmail` provided), Jira task created, Telegram alert on rejected variants.
61. Test error handling by intentionally sending a malformed request (missing `shotCode`) and confirming the Slack error alert fires.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|-------------|-----------------|
| The QC scoring in `Run Style QC Check2` uses `Math.random()` to simulate visual analysis variance. This is a **placeholder** for a real computer vision pipeline. In production, replace the random score generation with actual image/video analysis (e.g., OpenCV, a custom microservice, or a dedicated QC API that evaluates contrast histograms, colour distribution, and brightness variance against the hero reference frame). | Block 1.3 — Run Style QC Check2 |
| The polling loop (Variant Ready? → Wait → Poll) has **no maximum retry count or timeout guard**. If a Seedance job enters a `failed` or permanently `pending` state, the workflow will loop indefinitely until the n8n execution timeout is reached. Consider adding a retry counter (e.g., stored in the item payload) and a maximum attempt threshold (e.g., 60 retries × 20s = 20 minutes) that routes to an error path. | Block 1.2 — Polling Loop |
| The Telegram notification template references fields (`objectType`, `removalBrief`, `totalPasses`, `passLines`, `folderStructure`, `allQcPassed`) that originate from a different workflow context (clean plate generation). In the style-transfer pipeline, these fields are undefined and will render as empty strings. Update the Telegram message template to use style-transfer-specific fields (`variantId`, `variantName`, `qc.qcGrade`, `qc.qcNotes`, `videoUrl`). | Block 1.4 — Telegram: Notify on QC Rejection1 |
| If `deliveryEmail` is not provided in the webhook payload, it defaults to `null`. The Gmail node will fail when attempting to send to a null recipient. Consider adding a guard in the Aggregate node or a preceding IF node to skip the Gmail node when no email is provided. | Block 1.4 — Gmail: Send QC Report to Editorial1 |
| The Seedance model identifier `seedance-1-5-pro-251215` is hardcoded. If BytePlus releases a new model version, this must be updated manually in the `Build Style-Anchored Request2` Code node. Consider externalizing this as a workflow variable or environment variable. | Block 1.2 — Build Style-Anchored Request2 |
| The `safePrompt` construction uses `JSON.stringify(text).slice(1,-1)` as a string escaping technique. While functional, this is unconventional and may not handle all edge cases (e.g., strings that already contain escaped quotes). Consider using a dedicated escaping function or template literal approach. | Block 1.2 — Build Style-Locked Variants2 |
| Cross-node references by name (e.g., `$('Build Style-Anchored Request2').first().json` and `$('Merge Variant Job + Style Data2').first().json`) are **fragile** — renaming the referenced node breaks the expression without any compile-time warning. Consider documenting node names as stable identifiers or using n8n's `$node['name'].json` syntax with clear naming conventions. | Blocks 1.2 and 1.3 |
| The Jira task is created with only a Summary field and no Description. For richer context, consider populating the `additionalFields` parameter with a description containing the full QC report (similar to the email body). | Block 1.4 — Jira: Create Style Review Task1 |
| The webhook has **no authentication**. In production, consider enabling Basic Auth, Header Auth, or JWT validation on the webhook node to prevent unauthorized submissions. | Block 1.1 — Webhook: Style Transfer Request2 |
| n8n binary mode is set to `"separate"` and execution order is `"v1"`. Ensure your n8n instance supports these settings. The Wait node uses a production webhook for resumption (`slt-wait-001`) — confirm that your n8n instance's base URL is correctly configured for webhook resumption after wait periods. | Workflow Settings |
| The `ratio: "adaptive"` parameter in the Seedance request means the model decides the output aspect ratio. If your editorial pipeline requires a specific aspect ratio (e.g., 16:9 for cinematic delivery), this must be changed to a fixed value. | Block 1.2 — Build Style-Anchored Request2 |