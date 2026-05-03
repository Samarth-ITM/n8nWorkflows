Improve AI support email drafts with Gmail, OpenAI and PostgreSQL

https://n8nworkflows.xyz/workflows/improve-ai-support-email-drafts-with-gmail--openai-and-postgresql-13978


# Improve AI support email drafts with Gmail, OpenAI and PostgreSQL

# 1. Workflow Overview

This workflow implements a feedback loop for AI-assisted customer support emails. Every 3 hours, it checks Gmail sent messages, finds sent replies that correspond to previously generated AI drafts, compares the AI draft with the human-edited final message, and stores the result in PostgreSQL so future draft generation can improve through retrieval of past corrections. If the human edit reveals missing knowledge base information, the workflow also updates the relevant KB entry.

Its main use cases are:

- Learning from human edits to AI-generated support drafts
- Tracking which AI drafts were accepted without modification
- Storing edited final replies as reusable training/retrieval examples
- Detecting missing knowledge and automatically updating the KB

## 1.1 Scheduled Run Initialization

The workflow starts on a timer, retrieves the watermark from the last successful run, computes a fallback value for the first run, and creates a new run log entry.

## 1.2 Sent Email Retrieval

Using the computed watermark, the workflow fetches Gmail sent messages created after the last processed timestamp.

## 1.3 Per-Email Loop and Thread Matching

Each sent email is processed one at a time. The workflow fetches the full Gmail message body, extracts the usable plain-text reply, and matches the Gmail thread ID to a stored AI draft in PostgreSQL.

## 1.4 Comparison and Approval Decision

If a matching draft exists and has not already been processed, OpenAI compares the AI draft with the actual sent email and returns structured feedback. The workflow then branches depending on whether the draft was approved unchanged.

## 1.5 Correction Storage and Embedding

If the sent email differs from the AI draft, the workflow generates an embedding for the final human-sent response and stores the correction in PostgreSQL for later similarity search.

## 1.6 KB Update Branch

If the AI comparison indicates that the human edit introduced knowledge that should be reflected in the KB, the workflow fetches the relevant KB entry, rewrites the answer with OpenAI, updates the KB record, and marks the related correction as KB-updated.

## 1.7 Run Completion

The workflow updates the run log with completion metadata and stores the new watermark when the loop finishes.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Run Initialization

### Overview

This block starts the workflow every 3 hours, determines the last successfully processed sent-email timestamp, and initializes a new feedback run. It ensures incremental processing rather than re-reading the full Gmail sent history each time.

### Nodes Involved

- ⏰ Schedule - Every 3 Hours
- 🗄️ DB - Get Last Watermark
- ⚙️ Set Watermark
- 🗄️ DB - Start Run Log
- ⚙️ Carry Run Context

### Node Details

#### 1) ⏰ Schedule - Every 3 Hours
- **Type and role:** `n8n-nodes-base.scheduleTrigger`; entry-point trigger that launches the workflow periodically.
- **Configuration choices:** Runs every 3 hours using an interval rule.
- **Key expressions or variables used:** None.
- **Input / output connections:** No input; outputs to **🗄️ DB - Get Last Watermark**.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases / failures:**
  - Workflow must be activated for schedule execution.
  - Time-based execution depends on the instance scheduler functioning correctly.
- **Sub-workflow reference:** None.

#### 2) 🗄️ DB - Get Last Watermark
- **Type and role:** PostgreSQL node; fetches the last completed run watermark.
- **Configuration choices:** Executes:
  - `SELECT last_processed_sent_at FROM feedback_run_log WHERE status = 'completed' ORDER BY run_completed_at DESC LIMIT 1;`
- **Key expressions or variables used:** None.
- **Input / output connections:** Input from **⏰ Schedule - Every 3 Hours**; output to **⚙️ Set Watermark**.
- **Version-specific requirements:** Type version `2.6`.
- **Edge cases / failures:**
  - Missing table `feedback_run_log`
  - PostgreSQL credential/authentication issues
  - Empty result set on first-ever run
- **Sub-workflow reference:** None.

#### 3) ⚙️ Set Watermark
- **Type and role:** Code node; converts the DB result into a usable watermark and Gmail query timestamp.
- **Configuration choices:** JavaScript:
  - Reads `last_processed_sent_at`
  - If absent, defaults to 7 days ago
  - Converts watermark to Unix seconds for Gmail search
- **Key expressions or variables used:**
  - Outputs `watermark`
  - Outputs `unixTs`
- **Input / output connections:** Input from **🗄️ DB - Get Last Watermark**; output to **🗄️ DB - Start Run Log**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - Invalid timestamp format from DB
  - JavaScript runtime errors if unexpected input shape appears
- **Sub-workflow reference:** None.

#### 4) 🗄️ DB - Start Run Log
- **Type and role:** PostgreSQL node; creates a new run log row with status `running`.
- **Configuration choices:** Executes:
  - `INSERT INTO feedback_run_log (run_started_at, status) VALUES (NOW(), 'running') RETURNING id;`
- **Key expressions or variables used:** None.
- **Input / output connections:** Input from **⚙️ Set Watermark**; output to **⚙️ Carry Run Context**.
- **Version-specific requirements:** Type version `2.6`.
- **Edge cases / failures:**
  - Missing `feedback_run_log` table
  - Insert permission issues
- **Sub-workflow reference:** None.

#### 5) ⚙️ Carry Run Context
- **Type and role:** Code node; carries the run log ID together with watermark values into downstream nodes.
- **Configuration choices:** Uses data from:
  - current input item for `runLogId`
  - `$('⚙️ Set Watermark').first()` for `watermark` and `unixTs`
- **Key expressions or variables used:**
  - `runLogId`
  - `watermark`
  - `unixTs`
- **Input / output connections:** Input from **🗄️ DB - Start Run Log**; output to **📧 Gmail - Fetch Sent Emails**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - Fails if `⚙️ Set Watermark` did not return data
  - Cross-node reference errors if node names change
- **Sub-workflow reference:** None.

---

## 2.2 Sent Email Retrieval

### Overview

This block queries the Gmail Sent folder for messages sent after the stored watermark. It returns all matching items and passes them into a batch loop.

### Nodes Involved

- 📧 Gmail - Fetch Sent Emails
- 🔄 Loop - Sent Emails

### Node Details

#### 6) 📧 Gmail - Fetch Sent Emails
- **Type and role:** Gmail node; retrieves sent messages after a specified Unix timestamp.
- **Configuration choices:**
  - Operation: `getAll`
  - Return all: enabled
  - Search query: `in:sent after:<unix timestamp>`
  - Implemented with expression: `=in:sent after:{{ $json.unixTs }}`
  - `alwaysOutputData` enabled so downstream logic can still run if no emails are found
- **Key expressions or variables used:**
  - `$json.unixTs`
- **Input / output connections:** Input from **⚙️ Carry Run Context**; output to **🔄 Loop - Sent Emails**.
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases / failures:**
  - Gmail OAuth2 credential errors
  - Gmail API quota/rate limits
  - Search syntax issues if the query is malformed
  - Empty result set is possible and expected
- **Sub-workflow reference:** None.

#### 7) 🔄 Loop - Sent Emails
- **Type and role:** Split In Batches node; iterates through sent emails one at a time.
- **Configuration choices:**
  - `reset: false`
- **Key expressions or variables used:** None directly.
- **Input / output connections:**
  - Input from **📧 Gmail - Fetch Sent Emails**
  - One branch to **📧 Gmail - Fetch Full Message**
  - Another branch to **🗄️ DB - Complete Run Log**
  - Multiple later nodes loop back into this node to continue iteration
- **Version-specific requirements:** Type version `3`.
- **Edge cases / failures:**
  - If no Gmail items exist, loop termination behavior depends on n8n batch execution semantics
  - Misconfigured loop-back can cause stalled or repeated processing
- **Sub-workflow reference:** None.

---

## 2.3 Per-Email Loop and Thread Matching

### Overview

This block enriches each sent-message item by fetching the full Gmail message via HTTP, extracting the plain-text body, removing quoted content, and matching the thread to a stored AI draft in PostgreSQL.

### Nodes Involved

- 📧 Gmail - Fetch Full Message
- ⚙️ Parse Full Message Body
- 🗄️ DB - Match Thread ID
- ❓ IF - Draft Match Found?
- ❓ IF - Already Processed?

### Node Details

#### 8) 📧 Gmail - Fetch Full Message
- **Type and role:** HTTP Request node; calls the Gmail REST API directly to retrieve full message payload details.
- **Configuration choices:**
  - Method: GET
  - URL: `https://www.googleapis.com/gmail/v1/users/me/messages/{{ $json.id }}?format=full`
  - Authentication: predefined credential type using Gmail OAuth2
- **Key expressions or variables used:**
  - `$json.id`
- **Input / output connections:** Input from **🔄 Loop - Sent Emails**; output to **⚙️ Parse Full Message Body**.
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases / failures:**
  - Gmail OAuth2 scopes must allow API access
  - Missing or expired token
  - Message ID absent from prior Gmail list item
  - Gmail API transient failures
- **Sub-workflow reference:** None.

#### 9) ⚙️ Parse Full Message Body
- **Type and role:** Code node; extracts the usable plain-text reply from the full Gmail message.
- **Configuration choices:**
  - Decodes Gmail base64url content
  - Recursively searches MIME parts for `text/plain`
  - Strips quoted reply chains and forwarded-message sections
  - Falls back to `snippet` if no body is extracted
  - Carries forward original loop item fields
- **Key expressions or variables used:**
  - References `$('🔄 Loop - Sent Emails').item.json`
  - Outputs:
    - `sentBody`
    - `threadId`
    - `messageId`
- **Input / output connections:** Input from **📧 Gmail - Fetch Full Message**; output to **🗄️ DB - Match Thread ID**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - HTML-only emails may not contain a `text/plain` part
  - Deeply nested MIME structures may not match all variants
  - Quoted-reply stripping patterns may remove too much or too little
  - Buffer usage assumes Node.js-compatible n8n runtime
- **Sub-workflow reference:** None.

#### 10) 🗄️ DB - Match Thread ID
- **Type and role:** PostgreSQL node; looks up the AI draft associated with the Gmail thread.
- **Configuration choices:** Executes:
  - `SELECT id, gmail_thread_id, gmail_message_id, original_email_body, classification, ai_draft_text, feedback_processed_at, was_approved_as_is FROM ai_drafts WHERE gmail_thread_id = '{{ $json.threadId }}' LIMIT 1;`
  - `alwaysOutputData` enabled
- **Key expressions or variables used:**
  - `$json.threadId`
- **Input / output connections:** Input from **⚙️ Parse Full Message Body**; output to **❓ IF - Draft Match Found?**.
- **Version-specific requirements:** Type version `2.6`.
- **Edge cases / failures:**
  - SQL string interpolation could be fragile if `threadId` contains unexpected characters
  - No match found is expected for some sent emails
  - Database auth/schema issues
- **Sub-workflow reference:** None.

#### 11) ❓ IF - Draft Match Found?
- **Type and role:** IF node; determines whether the previous DB query returned a matching draft row.
- **Configuration choices:**
  - Checks whether `$json.id` exists
- **Key expressions or variables used:**
  - `={{ $json.id }}`
- **Input / output connections:**
  - True branch to **❓ IF - Already Processed?**
  - False branch back to **🔄 Loop - Sent Emails**
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases / failures:**
  - If DB shape changes and `id` is renamed, the condition will fail
- **Sub-workflow reference:** None.

#### 12) ❓ IF - Already Processed?
- **Type and role:** IF node; skips drafts already marked with `feedback_processed_at`.
- **Configuration choices:**
  - Condition checks whether `feedback_processed_at` is empty
  - True branch means “not yet processed”
- **Key expressions or variables used:**
  - `={{ $json.feedback_processed_at }}`
- **Input / output connections:**
  - True branch to **🤖 AI - Compare Draft vs Sent**
  - False branch back to **🔄 Loop - Sent Emails**
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases / failures:**
  - If DB stores null-like values inconsistently, condition behavior may vary
- **Sub-workflow reference:** None.

---

## 2.4 Comparison and Approval Decision

### Overview

This block sends the original customer message, the AI-generated draft, and the actual sent response to OpenAI for comparison. The response is parsed into structured JSON and used to determine whether the draft was accepted unchanged or needs to be stored as a correction.

### Nodes Involved

- 🤖 AI - Compare Draft vs Sent
- OpenAI Chat Model - Compare
- ⚙️ Parse AI Comparison
- ❓ IF - Approved As-Is?
- 🗄️ DB - Mark Approved As-Is

### Node Details

#### 13) 🤖 AI - Compare Draft vs Sent
- **Type and role:** LangChain Agent node; performs the comparison task with a tightly constrained output format.
- **Configuration choices:**
  - Prompt includes:
    - original customer email
    - AI draft text
    - human-sent final email
    - classification
  - Requires exact JSON-only response
  - `maxIterations: 3`
- **Key expressions or variables used:**
  - `$('🔄 Loop - Sent Emails').item.json.text || $('🔄 Loop - Sent Emails').item.json.body || ''`
  - `$('🗄️ DB - Match Thread ID').item.json.ai_draft_text`
  - `$json.text || $json.body || ''`
  - `$('🗄️ DB - Match Thread ID').item.json.classification`
- **Input / output connections:**
  - Main input from **❓ IF - Already Processed?**
  - AI language model input from **OpenAI Chat Model - Compare**
  - Output to **⚙️ Parse AI Comparison**
- **Version-specific requirements:** Type version `3.1`; depends on LangChain-compatible n8n package.
- **Edge cases / failures:**
  - LLM may return malformed JSON despite instructions
  - Context fields may be empty if upstream message extraction fails
  - OpenAI quota/rate limit issues
- **Sub-workflow reference:** None.

#### 14) OpenAI Chat Model - Compare
- **Type and role:** LangChain OpenAI chat model; provides the underlying model used by the comparison agent.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - Max tokens: `800`
  - Temperature: `0`
  - Presence penalty: `0`
  - Frequency penalty: `0`
- **Key expressions or variables used:** None.
- **Input / output connections:** Connects to **🤖 AI - Compare Draft vs Sent** through the AI language model port.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases / failures:**
  - Model availability may differ by account/region/version
  - Credential or API billing issues
- **Sub-workflow reference:** None.

#### 15) ⚙️ Parse AI Comparison
- **Type and role:** Code node; sanitizes and parses the LLM response into structured fields with fallback defaults.
- **Configuration choices:**
  - Removes markdown fences
  - Attempts `JSON.parse`
  - On failure, emits a safe fallback object with `edit_type: 'parse_error'`
  - Enriches parsed result with:
    - `sentBody`
    - `draftId`
    - `originalBody`
    - `classification`
    - `aiDraftText`
- **Key expressions or variables used:**
  - `$input.first().json.output`
  - `$('⚙️ Parse Full Message Body').item.json.sentBody`
  - `$('🗄️ DB - Match Thread ID').item.json.id`
  - Other cross-node references to matched draft fields
- **Input / output connections:** Input from **🤖 AI - Compare Draft vs Sent**; output to **❓ IF - Approved As-Is?**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - Cross-node references break if names change
  - If AI output schema changes, parsing fallback will activate
- **Sub-workflow reference:** None.

#### 16) ❓ IF - Approved As-Is?
- **Type and role:** IF node; branches based on whether the AI draft was sent unchanged.
- **Configuration choices:**
  - Boolean equality check against `approved_as_is === true`
- **Key expressions or variables used:**
  - `={{ $json.approved_as_is }}`
- **Input / output connections:**
  - True branch to **🗄️ DB - Mark Approved As-Is**
  - False branch to **🔢 Generate Embedding - Human Sent**
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases / failures:**
  - If parsing failed, fallback object causes this branch to evaluate false
- **Sub-workflow reference:** None.

#### 17) 🗄️ DB - Mark Approved As-Is
- **Type and role:** PostgreSQL node; marks the AI draft as approved unchanged and processed.
- **Configuration choices:** Executes:
  - `UPDATE ai_drafts SET was_approved_as_is = TRUE, feedback_processed_at = NOW() WHERE id = {{ $json.draftId }};`
- **Key expressions or variables used:**
  - `$json.draftId`
- **Input / output connections:** Input from **❓ IF - Approved As-Is?** true branch; output back to **🔄 Loop - Sent Emails**.
- **Version-specific requirements:** Type version `2.6`.
- **Edge cases / failures:**
  - `draftId` missing if upstream parsing/context failed
  - Database update permission issues
- **Sub-workflow reference:** None.

---

## 2.5 Correction Storage and Embedding

### Overview

This block handles human-edited responses. It embeds the final sent message, stores the correction pair in PostgreSQL, and marks the original AI draft as processed for feedback purposes.

### Nodes Involved

- 🔢 Generate Embedding - Human Sent
- ⚙️ Extract Embedding
- 🗄️ DB - Save Correction
- 🗄️ DB - Mark Draft Processed

### Node Details

#### 18) 🔢 Generate Embedding - Human Sent
- **Type and role:** HTTP Request node; calls OpenAI embeddings API for the human-sent final email text.
- **Configuration choices:**
  - POST `https://api.openai.com/v1/embeddings`
  - Auth: predefined OpenAI credential
  - Body:
    - `model = text-embedding-3-small`
    - `input = {{ $json.sentBody }}`
- **Key expressions or variables used:**
  - `={{ $json.sentBody }}`
- **Input / output connections:** Input from **❓ IF - Approved As-Is?** false branch; output to **⚙️ Extract Embedding**.
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases / failures:**
  - Empty `sentBody`
  - OpenAI auth/rate limit/model availability issues
  - Large email body exceeding token/input limits
- **Sub-workflow reference:** None.

#### 19) ⚙️ Extract Embedding
- **Type and role:** Code node; converts the embedding array into a PostgreSQL vector literal and restores prior context.
- **Configuration choices:**
  - Reads `item.data[0].embedding`
  - Converts to string form: `[x,y,z,...]`
  - Merges fields from **⚙️ Parse AI Comparison**
- **Key expressions or variables used:**
  - `$input.first().json.data[0].embedding`
  - `$('⚙️ Parse AI Comparison').first().json`
- **Input / output connections:** Input from **🔢 Generate Embedding - Human Sent**; output to **🗄️ DB - Save Correction**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases / failures:**
  - OpenAI API response shape changes
  - Missing embedding array
  - Very large vector string if using different embedding models
- **Sub-workflow reference:** None.

#### 20) 🗄️ DB - Save Correction
- **Type and role:** PostgreSQL node; stores the corrected example into the `corrections` table.
- **Configuration choices:**
  - Inserts:
    - `ai_draft_id`
    - `original_email_body`
    - `classification`
    - `ai_draft_text`
    - `human_sent_text`
    - `diff_summary`
    - `embedding`
    - `source = 'feedback_loop'`
    - `kb_updated = FALSE`
  - Uses query replacements for most text fields and casts embedding as `vector`
- **Key expressions or variables used:**
  - `{{ $json.draftId }}`
  - queryReplacement array from JSON fields
- **Input / output connections:** Input from **⚙️ Extract Embedding**; output to **🗄️ DB - Mark Draft Processed**.
- **Version-specific requirements:** Type version `2.6`; assumes PostgreSQL has vector extension/compatible column type.
- **Edge cases / failures:**
  - Missing `corrections` table
  - `vector` type not installed
  - Embedding dimension mismatch with DB schema
  - Large text or encoding issues
- **Sub-workflow reference:** None.

#### 21) 🗄️ DB - Mark Draft Processed
- **Type and role:** PostgreSQL node; marks the original draft as feedback-processed after saving the correction.
- **Configuration choices:** Executes:
  - `UPDATE ai_drafts SET feedback_processed_at = NOW() WHERE id = {{ $('⚙️ Extract Embedding').item.json.draftId }};`
- **Key expressions or variables used:**
  - `$('⚙️ Extract Embedding').item.json.draftId`
- **Input / output connections:** Input from **🗄️ DB - Save Correction**; output to **❓ IF - KB Update Needed?**.
- **Version-specific requirements:** Type version `2.6`.
- **Edge cases / failures:**
  - Cross-node reference breaks if node renamed
  - Update can succeed even if correction insert failed upstream is not handled here because failure would stop execution
- **Sub-workflow reference:** None.

---

## 2.6 KB Update Branch

### Overview

This block is only executed when the comparison step concludes that the human edit contains information missing from the knowledge base. It fetches the most recent KB entry for the detected category, rewrites the answer, updates the DB record, and marks the related correction as KB-updated.

### Nodes Involved

- ❓ IF - KB Update Needed?
- 🗄️ DB - Fetch KB Entry to Update
- 🤖 AI - Rewrite KB Answer
- 🗄️ DB - Update KB Entry
- 🗄️ DB - Mark KB Updated

### Node Details

#### 22) ❓ IF - KB Update Needed?
- **Type and role:** IF node; decides whether to invoke the KB update branch.
- **Configuration choices:**
  - Boolean equality check against `kb_update_needed === true`
  - Reads value from **⚙️ Parse AI Comparison**
- **Key expressions or variables used:**
  - `={{ $('⚙️ Parse AI Comparison').item.json.kb_update_needed }}`
- **Input / output connections:**
  - True branch to **🗄️ DB - Fetch KB Entry to Update**
  - False branch back to **🔄 Loop - Sent Emails**
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases / failures:**
  - Cross-node reference sensitivity
  - If parsing fallback occurred, this will normally be false
- **Sub-workflow reference:** None.

#### 23) 🗄️ DB - Fetch KB Entry to Update
- **Type and role:** PostgreSQL node; retrieves the latest KB entry for the category identified by the AI comparison.
- **Configuration choices:** Executes:
  - `SELECT id, category, question, answer FROM kb_data WHERE category = $1 ORDER BY updated_at DESC LIMIT 1;`
  - Query replacement supplies `kb_category`
- **Key expressions or variables used:**
  - `={{ [$('⚙️ Parse AI Comparison').item.json.kb_category] }}`
- **Input / output connections:** Input from **❓ IF - KB Update Needed?**; output to **🤖 AI - Rewrite KB Answer**.
- **Version-specific requirements:** Type version `2.6`.
- **Edge cases / failures:**
  - No KB row found for the category
  - Missing `kb_data` table
  - Null `kb_category`
- **Sub-workflow reference:** None.

#### 24) 🤖 AI - Rewrite KB Answer
- **Type and role:** OpenAI node; rewrites the KB answer by merging current content with newly discovered information.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - Temperature: `0.2`
  - Max tokens: `500`
  - System prompt instructs output to be only the rewritten answer text
  - User prompt includes existing category, question, answer, new information, and update reason
  - `simplify: false`
- **Key expressions or variables used:**
  - `$json.category`
  - `$json.question`
  - `$json.answer`
  - `$('⚙️ Parse AI Comparison').item.json.kb_new_info`
  - `$('⚙️ Parse AI Comparison').item.json.kb_update_reason`
- **Input / output connections:** Input from **🗄️ DB - Fetch KB Entry to Update**; output to **🗄️ DB - Update KB Entry**.
- **Version-specific requirements:** Type version `2.1`.
- **Edge cases / failures:**
  - No existing KB answer available
  - LLM may produce empty or poor-quality rewritten text
  - API auth/model issues
- **Sub-workflow reference:** None.

#### 25) 🗄️ DB - Update KB Entry
- **Type and role:** PostgreSQL node; writes the rewritten answer back into `kb_data`.
- **Configuration choices:**
  - Preserves old answer in `previous_answer`
  - Sets:
    - new `answer`
    - `updated_by = 'ai_feedback_loop'`
    - `updated_at = NOW()`
  - Uses `id` from fetched KB entry
  - Replacement value accepts either OpenAI response content or output fallback
- **Key expressions or variables used:**
  - `$('🗄️ DB - Fetch KB Entry to Update').item.json.id`
  - `={{ [$json.message?.content?.[0]?.text || $json.output || ''] }}`
- **Input / output connections:** Input from **🤖 AI - Rewrite KB Answer**; output to **🗄️ DB - Mark KB Updated**.
- **Version-specific requirements:** Type version `2.6`.
- **Edge cases / failures:**
  - If OpenAI response shape differs, fallback may still produce empty answer
  - Update permission issues
- **Sub-workflow reference:** None.

#### 26) 🗄️ DB - Mark KB Updated
- **Type and role:** PostgreSQL node; marks the most recent related correction as KB-updated.
- **Configuration choices:** Executes:
  - `UPDATE corrections SET kb_updated = TRUE WHERE ai_draft_id = {{ $('⚙️ Extract Embedding').item.json.draftId }} AND source = 'feedback_loop' ORDER BY created_at DESC LIMIT 1;`
- **Key expressions or variables used:**
  - `$('⚙️ Extract Embedding').item.json.draftId`
- **Input / output connections:** Input from **🗄️ DB - Update KB Entry**; output back to **🔄 Loop - Sent Emails**.
- **Version-specific requirements:** Type version `2.6`.
- **Edge cases / failures:**
  - SQL syntax can depend on PostgreSQL compatibility for `ORDER BY ... LIMIT` in UPDATE statements
  - No matching correction record found
- **Sub-workflow reference:** None.

---

## 2.7 Run Completion

### Overview

This block updates the run log once processing is complete and records the new watermark as the current time. It represents the closure of a polling cycle.

### Nodes Involved

- 🗄️ DB - Complete Run Log

### Node Details

#### 27) 🗄️ DB - Complete Run Log
- **Type and role:** PostgreSQL node; finalizes the feedback run.
- **Configuration choices:** Executes:
  - `UPDATE feedback_run_log SET run_completed_at = NOW(), last_processed_sent_at = NOW(), status = 'completed' WHERE id = {{ $('🗄️ DB - Start Run Log').first().json.id }};`
- **Key expressions or variables used:**
  - `$('🗄️ DB - Start Run Log').first().json.id`
- **Input / output connections:** Receives from **🔄 Loop - Sent Emails**.
- **Version-specific requirements:** Type version `2.6`.
- **Edge cases / failures:**
  - If processing errors stop execution earlier, this node may not run
  - Cross-node reference depends on stable node naming
  - Watermark is set to completion time, not the max actual email sent timestamp
- **Sub-workflow reference:** None.

---

## 2.8 Sticky Notes / Documentation Nodes

### Overview

These nodes contain operational explanations and setup guidance directly inside the workflow canvas. They do not participate in execution but are important for understanding the design.

### Nodes Involved

- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4

### Node Details

#### 28) Sticky Note
- **Type and role:** Sticky Note; high-level description and setup instructions.
- **Configuration choices:** Documents workflow purpose, behavior, and prerequisites.
- **Input / output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:** None.
- **Sub-workflow reference:** Mentions “Workflow 1 (email draft automation)” as an external dependency, but no executable sub-workflow node is present.

#### 29) Sticky Note1
- **Type and role:** Sticky Note; explains schedule and watermark logic.
- **Configuration choices:** Notes first-run fallback to 7 days ago.
- **Input / output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:** None.
- **Sub-workflow reference:** None.

#### 30) Sticky Note2
- **Type and role:** Sticky Note; explains loop and thread matching.
- **Configuration choices:** Notes Gmail list/snippet limitations and skip logic.
- **Input / output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:** None.
- **Sub-workflow reference:** None.

#### 31) Sticky Note3
- **Type and role:** Sticky Note; explains AI comparison branching.
- **Configuration choices:** Documents compare/approve/store behavior.
- **Input / output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:** None.
- **Sub-workflow reference:** None.

#### 32) Sticky Note4
- **Type and role:** Sticky Note; explains corrections storage and KB update.
- **Configuration choices:** Documents how saved corrections are reused by another workflow.
- **Input / output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases / failures:** None.
- **Sub-workflow reference:** Mentions “Workflow 1” as an external dependent workflow, but no executable link exists in this JSON.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| ⏰ Schedule - Every 3 Hours | Schedule Trigger | Launch workflow every 3 hours |  | 🗄️ DB - Get Last Watermark | ## Schedule & Fetch Emails<br>Runs every 3 hours. Fetches last_processed_sent_at from the most recent completed run as a watermark — so only new sent emails are fetched each time. First-ever run defaults to 7 days ago. |
| 🗄️ DB - Get Last Watermark | PostgreSQL | Read last completed run watermark | ⏰ Schedule - Every 3 Hours | ⚙️ Set Watermark | ## Schedule & Fetch Emails<br>Runs every 3 hours. Fetches last_processed_sent_at from the most recent completed run as a watermark — so only new sent emails are fetched each time. First-ever run defaults to 7 days ago. |
| ⚙️ Set Watermark | Code | Compute fallback watermark and Unix timestamp | 🗄️ DB - Get Last Watermark | 🗄️ DB - Start Run Log | ## Schedule & Fetch Emails<br>Runs every 3 hours. Fetches last_processed_sent_at from the most recent completed run as a watermark — so only new sent emails are fetched each time. First-ever run defaults to 7 days ago. |
| 🗄️ DB - Start Run Log | PostgreSQL | Insert new run log row | ⚙️ Set Watermark | ⚙️ Carry Run Context | ## Schedule & Fetch Emails<br>Runs every 3 hours. Fetches last_processed_sent_at from the most recent completed run as a watermark — so only new sent emails are fetched each time. First-ever run defaults to 7 days ago. |
| ⚙️ Carry Run Context | Code | Forward run log ID and watermark context | 🗄️ DB - Start Run Log | 📧 Gmail - Fetch Sent Emails | ## Schedule & Fetch Emails<br>Runs every 3 hours. Fetches last_processed_sent_at from the most recent completed run as a watermark — so only new sent emails are fetched each time. First-ever run defaults to 7 days ago. |
| 📧 Gmail - Fetch Sent Emails | Gmail | Fetch sent Gmail messages after watermark | ⚙️ Carry Run Context | 🔄 Loop - Sent Emails | ## Schedule & Fetch Emails<br>Runs every 3 hours. Fetches last_processed_sent_at from the most recent completed run as a watermark — so only new sent emails are fetched each time. First-ever run defaults to 7 days ago. |
| 🔄 Loop - Sent Emails | Split In Batches | Iterate through sent emails and manage loop continuation | 📧 Gmail - Fetch Sent Emails; ❓ IF - Draft Match Found?; ❓ IF - Already Processed?; ❓ IF - KB Update Needed?; 🗄️ DB - Mark Approved As-Is; 🗄️ DB - Mark KB Updated | 🗄️ DB - Complete Run Log; 📧 Gmail - Fetch Full Message | ## Loop & thread matching<br>Processes one sent email at a time. Full message body is fetched via Gmail API separately — the Sent folder list only returns a snippet. Each email is matched to an AI draft by thread ID.<br>No match or already processed = skip. |
| 📧 Gmail - Fetch Full Message | HTTP Request | Fetch full Gmail message payload | 🔄 Loop - Sent Emails | ⚙️ Parse Full Message Body | ## Loop & thread matching<br>Processes one sent email at a time. Full message body is fetched via Gmail API separately — the Sent folder list only returns a snippet. Each email is matched to an AI draft by thread ID.<br>No match or already processed = skip. |
| ⚙️ Parse Full Message Body | Code | Extract clean sent reply text from Gmail payload | 📧 Gmail - Fetch Full Message | 🗄️ DB - Match Thread ID | ## Loop & thread matching<br>Processes one sent email at a time. Full message body is fetched via Gmail API separately — the Sent folder list only returns a snippet. Each email is matched to an AI draft by thread ID.<br>No match or already processed = skip. |
| 🗄️ DB - Match Thread ID | PostgreSQL | Match sent Gmail thread to stored AI draft | ⚙️ Parse Full Message Body | ❓ IF - Draft Match Found? | ## Loop & thread matching<br>Processes one sent email at a time. Full message body is fetched via Gmail API separately — the Sent folder list only returns a snippet. Each email is matched to an AI draft by thread ID.<br>No match or already processed = skip. |
| ❓ IF - Draft Match Found? | IF | Skip unmatched sent emails | 🗄️ DB - Match Thread ID | ❓ IF - Already Processed?; 🔄 Loop - Sent Emails | ## Loop & thread matching<br>Processes one sent email at a time. Full message body is fetched via Gmail API separately — the Sent folder list only returns a snippet. Each email is matched to an AI draft by thread ID.<br>No match or already processed = skip. |
| ❓ IF - Already Processed? | IF | Skip drafts already feedback-processed | ❓ IF - Draft Match Found? | 🤖 AI - Compare Draft vs Sent; 🔄 Loop - Sent Emails | ## Loop & thread matching<br>Processes one sent email at a time. Full message body is fetched via Gmail API separately — the Sent folder list only returns a snippet. Each email is matched to an AI draft by thread ID.<br>No match or already processed = skip. |
| 🤖 AI - Compare Draft vs Sent | LangChain Agent | Compare AI draft with actual human-sent email | ❓ IF - Already Processed?; OpenAI Chat Model - Compare | ⚙️ Parse AI Comparison | ## AI comparison & branching<br>GPT-4o-mini compares the AI draft vs human-sent email and returns edit type, a plain-English diff summary, and whether a KB update is needed.<br><br>Approved as-is → mark draft, continue loop<br>Edited/rewritten → generate embedding, save to corrections table |
| OpenAI Chat Model - Compare | OpenAI Chat Model | Language model backend for comparison agent |  | 🤖 AI - Compare Draft vs Sent | ## AI comparison & branching<br>GPT-4o-mini compares the AI draft vs human-sent email and returns edit type, a plain-English diff summary, and whether a KB update is needed.<br><br>Approved as-is → mark draft, continue loop<br>Edited/rewritten → generate embedding, save to corrections table |
| ⚙️ Parse AI Comparison | Code | Parse and normalize AI comparison output | 🤖 AI - Compare Draft vs Sent | ❓ IF - Approved As-Is? | ## AI comparison & branching<br>GPT-4o-mini compares the AI draft vs human-sent email and returns edit type, a plain-English diff summary, and whether a KB update is needed.<br><br>Approved as-is → mark draft, continue loop<br>Edited/rewritten → generate embedding, save to corrections table |
| ❓ IF - Approved As-Is? | IF | Branch on unchanged vs edited sent email | ⚙️ Parse AI Comparison | 🗄️ DB - Mark Approved As-Is; 🔢 Generate Embedding - Human Sent | ## AI comparison & branching<br>GPT-4o-mini compares the AI draft vs human-sent email and returns edit type, a plain-English diff summary, and whether a KB update is needed.<br><br>Approved as-is → mark draft, continue loop<br>Edited/rewritten → generate embedding, save to corrections table |
| 🗄️ DB - Mark Approved As-Is | PostgreSQL | Mark draft approved without edits | ❓ IF - Approved As-Is? | 🔄 Loop - Sent Emails | ## AI comparison & branching<br>GPT-4o-mini compares the AI draft vs human-sent email and returns edit type, a plain-English diff summary, and whether a KB update is needed.<br><br>Approved as-is → mark draft, continue loop<br>Edited/rewritten → generate embedding, save to corrections table |
| 🔢 Generate Embedding - Human Sent | HTTP Request | Create embedding for final human response | ❓ IF - Approved As-Is? | ⚙️ Extract Embedding | ## Corrections & KB update<br>Human-edited pairs are embedded and saved to the corrections table — this is what Workflow 1 queries for similarity search when drafting future replies.<br><br>If the edit contained new information, the matching KB entry is rewritten by AI. Previous answer is preserved for audit. |
| ⚙️ Extract Embedding | Code | Convert embedding and carry correction context | 🔢 Generate Embedding - Human Sent | 🗄️ DB - Save Correction | ## Corrections & KB update<br>Human-edited pairs are embedded and saved to the corrections table — this is what Workflow 1 queries for similarity search when drafting future replies.<br><br>If the edit contained new information, the matching KB entry is rewritten by AI. Previous answer is preserved for audit. |
| 🗄️ DB - Save Correction | PostgreSQL | Persist edited draft/final email correction record | ⚙️ Extract Embedding | 🗄️ DB - Mark Draft Processed | ## Corrections & KB update<br>Human-edited pairs are embedded and saved to the corrections table — this is what Workflow 1 queries for similarity search when drafting future replies.<br><br>If the edit contained new information, the matching KB entry is rewritten by AI. Previous answer is preserved for audit. |
| 🗄️ DB - Mark Draft Processed | PostgreSQL | Mark edited draft as feedback-processed | 🗄️ DB - Save Correction | ❓ IF - KB Update Needed? | ## Corrections & KB update<br>Human-edited pairs are embedded and saved to the corrections table — this is what Workflow 1 queries for similarity search when drafting future replies.<br><br>If the edit contained new information, the matching KB entry is rewritten by AI. Previous answer is preserved for audit. |
| ❓ IF - KB Update Needed? | IF | Decide whether to update KB | 🗄️ DB - Mark Draft Processed | 🗄️ DB - Fetch KB Entry to Update; 🔄 Loop - Sent Emails | ## Corrections & KB update<br>Human-edited pairs are embedded and saved to the corrections table — this is what Workflow 1 queries for similarity search when drafting future replies.<br><br>If the edit contained new information, the matching KB entry is rewritten by AI. Previous answer is preserved for audit. |
| 🗄️ DB - Fetch KB Entry to Update | PostgreSQL | Load latest KB entry for detected category | ❓ IF - KB Update Needed? | 🤖 AI - Rewrite KB Answer | ## Corrections & KB update<br>Human-edited pairs are embedded and saved to the corrections table — this is what Workflow 1 queries for similarity search when drafting future replies.<br><br>If the edit contained new information, the matching KB entry is rewritten by AI. Previous answer is preserved for audit. |
| 🤖 AI - Rewrite KB Answer | OpenAI | Rewrite KB answer with new support knowledge | 🗄️ DB - Fetch KB Entry to Update | 🗄️ DB - Update KB Entry | ## Corrections & KB update<br>Human-edited pairs are embedded and saved to the corrections table — this is what Workflow 1 queries for similarity search when drafting future replies.<br><br>If the edit contained new information, the matching KB entry is rewritten by AI. Previous answer is preserved for audit. |
| 🗄️ DB - Update KB Entry | PostgreSQL | Save rewritten KB answer and preserve previous answer | 🤖 AI - Rewrite KB Answer | 🗄️ DB - Mark KB Updated | ## Corrections & KB update<br>Human-edited pairs are embedded and saved to the corrections table — this is what Workflow 1 queries for similarity search when drafting future replies.<br><br>If the edit contained new information, the matching KB entry is rewritten by AI. Previous answer is preserved for audit. |
| 🗄️ DB - Mark KB Updated | PostgreSQL | Mark latest correction as reflected in KB | 🗄️ DB - Update KB Entry | 🔄 Loop - Sent Emails | ## Corrections & KB update<br>Human-edited pairs are embedded and saved to the corrections table — this is what Workflow 1 queries for similarity search when drafting future replies.<br><br>If the edit contained new information, the matching KB entry is rewritten by AI. Previous answer is preserved for audit. |
| 🗄️ DB - Complete Run Log | PostgreSQL | Mark workflow run completed and advance watermark | 🔄 Loop - Sent Emails |  |  |
| Sticky Note | Sticky Note | Embedded workflow description and setup guidance |  |  | ## Self-learning feedback loop for AI email drafts<br><br>This workflow runs every 3 hours and checks which AI-generated email drafts have been reviewed and sent by the support team. It compares the AI draft against what was actually sent, learns from the differences, and stores human-approved responses as training examples for future drafts.<br><br>Over time, the main email workflow surfaces increasingly relevant past responses via vector similarity search — no fine-tuning needed.<br><br>## How it works<br><br>1. Fetches watermark from last completed run — only new sent emails are processed each time<br>2. Gmail Sent folder is fetched since the watermark timestamp<br>3. Each sent email is matched to an AI draft via thread ID<br>4. Already-processed drafts are skipped automatically<br>5. AI compares the draft vs actual sent email and classifies the type of edit made<br>6. If sent unchanged — marked approved-as-is, loop continues<br>7. If edited — embedding generated, correction saved to DB for future similarity search<br>8. If edit reveals missing KB info — KB entry auto-updated<br><br>## Setup steps<br><br>1. Connect Gmail OAuth2 to the Gmail Fetch Sent node<br>2. Connect PostgreSQL credential to all DB nodes<br>3. Connect OpenAI API to the Chat Model and embedding nodes<br>4. Run the DB migration SQL to add feedback columns and create the feedback_run_log table<br>5. Ensure Workflow 1 (email draft automation) is already active and generating drafts<br>6. Activate — runs automatically every 3 hours |
| Sticky Note1 | Sticky Note | Embedded note about scheduling and watermark behavior |  |  | ## Schedule & Fetch Emails<br><br>Runs every 3 hours. Fetches last_processed_sent_at from the most recent completed run as a watermark — so only new sent emails are fetched each time. First-ever run defaults to 7 days ago. |
| Sticky Note2 | Sticky Note | Embedded note about loop and thread matching |  |  | ## Loop & thread matching<br><br>Processes one sent email at a time. Full message body is fetched via Gmail API separately — the Sent folder list only returns a snippet. Each email is matched to an AI draft by thread ID.<br>No match or already processed = skip. |
| Sticky Note3 | Sticky Note | Embedded note about AI comparison branching |  |  | ## AI comparison & branching<br><br>GPT-4o-mini compares the AI draft vs human-sent email and returns edit type, a plain-English diff summary, and whether a KB update is needed.<br><br>Approved as-is → mark draft, continue loop<br>Edited/rewritten → generate embedding, save to corrections table |
| Sticky Note4 | Sticky Note | Embedded note about corrections and KB updates |  |  | ## Corrections & KB update<br><br>Human-edited pairs are embedded and saved to the corrections table — this is what Workflow 1 queries for similarity search when drafting future replies.<br><br>If the edit contained new information, the matching KB entry is rewritten by AI. Previous answer is preserved for audit. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like **Feedback Loop**.

2. **Add a Schedule Trigger node**
   - Type: **Schedule Trigger**
   - Name: **⏰ Schedule - Every 3 Hours**
   - Set interval to:
     - Field: `hours`
     - Every: `3`

3. **Add a PostgreSQL node to read the last watermark**
   - Type: **Postgres**
   - Name: **🗄️ DB - Get Last Watermark**
   - Operation: **Execute Query**
   - Query:
     ```sql
     SELECT last_processed_sent_at
     FROM feedback_run_log
     WHERE status = 'completed'
     ORDER BY run_completed_at DESC
     LIMIT 1;
     ```
   - Connect PostgreSQL credentials.
   - Connect **⏰ Schedule - Every 3 Hours → 🗄️ DB - Get Last Watermark**

4. **Add a Code node to compute the watermark**
   - Type: **Code**
   - Name: **⚙️ Set Watermark**
   - Paste logic that:
     - reads the DB row
     - uses `last_processed_sent_at` if present
     - otherwise sets a default of 7 days ago
     - computes Unix timestamp seconds
   - Expected output fields:
     - `watermark`
     - `unixTs`
   - Connect **🗄️ DB - Get Last Watermark → ⚙️ Set Watermark**

5. **Add a PostgreSQL node to log the run start**
   - Type: **Postgres**
   - Name: **🗄️ DB - Start Run Log**
   - Operation: **Execute Query**
   - Query:
     ```sql
     INSERT INTO feedback_run_log (run_started_at, status)
     VALUES (NOW(), 'running')
     RETURNING id;
     ```
   - Connect **⚙️ Set Watermark → 🗄️ DB - Start Run Log**

6. **Add a Code node to carry run context**
   - Type: **Code**
   - Name: **⚙️ Carry Run Context**
   - Build output with:
     - `runLogId` from the insert result
     - `watermark` from **⚙️ Set Watermark**
     - `unixTs` from **⚙️ Set Watermark**
   - Connect **🗄️ DB - Start Run Log → ⚙️ Carry Run Context**

7. **Add a Gmail node to fetch sent emails**
   - Type: **Gmail**
   - Name: **📧 Gmail - Fetch Sent Emails**
   - Operation: **Get Many / Get All**
   - Return All: **true**
   - Filter query:
     ```text
     in:sent after:{{ $json.unixTs }}
     ```
     In n8n expression form:
     ```text
     =in:sent after:{{ $json.unixTs }}
     ```
   - Enable **Always Output Data**
   - Connect Gmail OAuth2 credentials.
   - Connect **⚙️ Carry Run Context → 📧 Gmail - Fetch Sent Emails**

8. **Add a Split In Batches node**
   - Type: **Split In Batches**
   - Name: **🔄 Loop - Sent Emails**
   - Keep reset disabled (`false`)
   - Connect **📧 Gmail - Fetch Sent Emails → 🔄 Loop - Sent Emails**

9. **Add an HTTP Request node to fetch the full Gmail message**
   - Type: **HTTP Request**
   - Name: **📧 Gmail - Fetch Full Message**
   - Method: **GET**
   - URL:
     ```text
     =https://www.googleapis.com/gmail/v1/users/me/messages/{{ $json.id }}?format=full
     ```
   - Authentication: **Predefined Credential Type**
   - Credential type: **gmailOAuth2**
   - Connect Gmail OAuth2 credentials.
   - Connect **🔄 Loop - Sent Emails → 📧 Gmail - Fetch Full Message**

10. **Add a Code node to parse the full message body**
    - Type: **Code**
    - Name: **⚙️ Parse Full Message Body**
    - Implement logic to:
      - decode Gmail base64url body
      - recursively locate `text/plain` payload parts
      - strip quoted replies
      - fallback to `snippet`
      - return:
        - original loop item fields
        - `sentBody`
        - `threadId`
        - `messageId`
    - Connect **📧 Gmail - Fetch Full Message → ⚙️ Parse Full Message Body**

11. **Add a PostgreSQL node to match Gmail thread to AI draft**
    - Type: **Postgres**
    - Name: **🗄️ DB - Match Thread ID**
    - Operation: **Execute Query**
    - Enable **Always Output Data**
    - Query:
      ```sql
      SELECT id, gmail_thread_id, gmail_message_id, original_email_body, classification,
             ai_draft_text, feedback_processed_at, was_approved_as_is
      FROM ai_drafts
      WHERE gmail_thread_id = '{{ $json.threadId }}'
      LIMIT 1;
      ```
    - Connect **⚙️ Parse Full Message Body → 🗄️ DB - Match Thread ID**

12. **Add an IF node to detect whether a draft was found**
    - Type: **IF**
    - Name: **❓ IF - Draft Match Found?**
    - Condition:
      - value: `{{ $json.id }}`
      - operator: **exists**
    - Connect **🗄️ DB - Match Thread ID → ❓ IF - Draft Match Found?**

13. **Add an IF node to skip already processed drafts**
    - Type: **IF**
    - Name: **❓ IF - Already Processed?**
    - Condition:
      - value: `{{ $json.feedback_processed_at }}`
      - operator: **is empty**
    - Connect true path from **❓ IF - Draft Match Found? → ❓ IF - Already Processed?**
    - Connect false path from **❓ IF - Draft Match Found? → 🔄 Loop - Sent Emails**

14. **Add an OpenAI Chat Model node**
    - Type: **OpenAI Chat Model** (LangChain)
    - Name: **OpenAI Chat Model - Compare**
    - Model: **gpt-4o-mini**
    - Temperature: `0`
    - Max tokens: `800`
    - Connect OpenAI API credentials.

15. **Add a LangChain Agent node for comparison**
    - Type: **AI Agent**
    - Name: **🤖 AI - Compare Draft vs Sent**
    - Prompt type: define prompt manually
    - Paste a prompt that includes:
      - original customer email
      - AI draft
      - human-sent final email
      - classification
      - strict JSON-only response format with fields:
        - `approved_as_is`
        - `edit_type`
        - `diff_summary`
        - `tone_shift`
        - `info_added`
        - `info_removed`
        - `kb_update_needed`
        - `kb_update_reason`
        - `kb_category`
        - `kb_new_info`
    - Set `maxIterations` to `3`
    - Connect **OpenAI Chat Model - Compare** to the AI language model port of **🤖 AI - Compare Draft vs Sent**
    - Connect true path from **❓ IF - Already Processed? → 🤖 AI - Compare Draft vs Sent**
    - Connect false path from **❓ IF - Already Processed? → 🔄 Loop - Sent Emails**

16. **Add a Code node to parse the AI output**
    - Type: **Code**
    - Name: **⚙️ Parse AI Comparison**
    - Logic should:
      - remove code fences
      - parse JSON safely
      - return fallback values if parsing fails
      - merge in:
        - `sentBody`
        - `draftId`
        - `originalBody`
        - `classification`
        - `aiDraftText`
    - Connect **🤖 AI - Compare Draft vs Sent → ⚙️ Parse AI Comparison**

17. **Add an IF node for approval status**
    - Type: **IF**
    - Name: **❓ IF - Approved As-Is?**
    - Condition:
      - `{{ $json.approved_as_is }}`
      - equals `true`
    - Connect **⚙️ Parse AI Comparison → ❓ IF - Approved As-Is?**

18. **Add a PostgreSQL node to mark approved drafts**
    - Type: **Postgres**
    - Name: **🗄️ DB - Mark Approved As-Is**
    - Query:
      ```sql
      UPDATE ai_drafts
      SET was_approved_as_is = TRUE,
          feedback_processed_at = NOW()
      WHERE id = {{ $json.draftId }};
      ```
    - Connect true branch of **❓ IF - Approved As-Is? → 🗄️ DB - Mark Approved As-Is**
    - Connect **🗄️ DB - Mark Approved As-Is → 🔄 Loop - Sent Emails**

19. **Add an HTTP Request node for embeddings**
    - Type: **HTTP Request**
    - Name: **🔢 Generate Embedding - Human Sent**
    - Method: **POST**
    - URL: `https://api.openai.com/v1/embeddings`
    - Authentication: **Predefined Credential Type**
    - Credential type: **openAiApi**
    - Send Body: **true**
    - Body parameters:
      - `model = text-embedding-3-small`
      - `input = {{ $json.sentBody }}`
    - Connect false branch of **❓ IF - Approved As-Is? → 🔢 Generate Embedding - Human Sent**

20. **Add a Code node to extract the embedding**
    - Type: **Code**
    - Name: **⚙️ Extract Embedding**
    - Logic should:
      - read `data[0].embedding`
      - convert array to string format `[v1,v2,...]`
      - merge prior comparison fields
    - Connect **🔢 Generate Embedding - Human Sent → ⚙️ Extract Embedding**

21. **Add a PostgreSQL node to save the correction**
    - Type: **Postgres**
    - Name: **🗄️ DB - Save Correction**
    - Query:
      ```sql
      INSERT INTO corrections (
        ai_draft_id,
        original_email_body,
        classification,
        ai_draft_text,
        human_sent_text,
        diff_summary,
        embedding,
        source,
        kb_updated
      ) VALUES (
        {{ $json.draftId }},
        $1,
        $2,
        $3,
        $4,
        $5,
        $6::vector,
        'feedback_loop',
        FALSE
      ) RETURNING id;
      ```
    - Query replacements should supply:
      1. original body
      2. classification
      3. AI draft text
      4. sent body
      5. diff summary
      6. embedding string
    - Connect **⚙️ Extract Embedding → 🗄️ DB - Save Correction**

22. **Add a PostgreSQL node to mark the draft as processed**
    - Type: **Postgres**
    - Name: **🗄️ DB - Mark Draft Processed**
    - Query:
      ```sql
      UPDATE ai_drafts
      SET feedback_processed_at = NOW()
      WHERE id = {{ $('⚙️ Extract Embedding').item.json.draftId }};
      ```
    - Connect **🗄️ DB - Save Correction → 🗄️ DB - Mark Draft Processed**

23. **Add an IF node to decide whether KB update is needed**
    - Type: **IF**
    - Name: **❓ IF - KB Update Needed?**
    - Condition:
      - `{{ $('⚙️ Parse AI Comparison').item.json.kb_update_needed }}`
      - equals `true`
    - Connect **🗄️ DB - Mark Draft Processed → ❓ IF - KB Update Needed?**

24. **Add a PostgreSQL node to fetch the KB entry**
    - Type: **Postgres**
    - Name: **🗄️ DB - Fetch KB Entry to Update**
    - Query:
      ```sql
      SELECT id, category, question, answer
      FROM kb_data
      WHERE category = $1
      ORDER BY updated_at DESC
      LIMIT 1;
      ```
    - Query replacement:
      - `{{ $('⚙️ Parse AI Comparison').item.json.kb_category }}`
    - Connect true branch of **❓ IF - KB Update Needed? → 🗄️ DB - Fetch KB Entry to Update**
    - Connect false branch of **❓ IF - KB Update Needed? → 🔄 Loop - Sent Emails**

25. **Add an OpenAI node to rewrite the KB answer**
    - Type: **OpenAI**
    - Name: **🤖 AI - Rewrite KB Answer**
    - Model: **gpt-4o-mini**
    - Temperature: `0.2`
    - Max tokens: `500`
    - Use messages:
      - **System**: instruct model to update KB answer, preserve existing correct content, and return only the final answer text
      - **User**: include category, question, current answer, `kb_new_info`, and `kb_update_reason`
    - Set `simplify` to `false`
    - Connect **🗄️ DB - Fetch KB Entry to Update → 🤖 AI - Rewrite KB Answer**

26. **Add a PostgreSQL node to update the KB**
    - Type: **Postgres**
    - Name: **🗄️ DB - Update KB Entry**
    - Query:
      ```sql
      UPDATE kb_data
      SET previous_answer = answer,
          answer = $1,
          updated_by = 'ai_feedback_loop',
          updated_at = NOW()
      WHERE id = {{ $('🗄️ DB - Fetch KB Entry to Update').item.json.id }};
      ```
    - Query replacement should use the OpenAI response text, e.g.:
      - `{{ $json.message?.content?.[0]?.text || $json.output || '' }}`
    - Connect **🤖 AI - Rewrite KB Answer → 🗄️ DB - Update KB Entry**

27. **Add a PostgreSQL node to mark the correction as KB-updated**
    - Type: **Postgres**
    - Name: **🗄️ DB - Mark KB Updated**
    - Query:
      ```sql
      UPDATE corrections
      SET kb_updated = TRUE
      WHERE ai_draft_id = {{ $('⚙️ Extract Embedding').item.json.draftId }}
        AND source = 'feedback_loop'
      ORDER BY created_at DESC
      LIMIT 1;
      ```
    - Connect **🗄️ DB - Update KB Entry → 🗄️ DB - Mark KB Updated**
    - Connect **🗄️ DB - Mark KB Updated → 🔄 Loop - Sent Emails**

28. **Add a PostgreSQL node to complete the run**
    - Type: **Postgres**
    - Name: **🗄️ DB - Complete Run Log**
    - Query:
      ```sql
      UPDATE feedback_run_log
      SET run_completed_at = NOW(),
          last_processed_sent_at = NOW(),
          status = 'completed'
      WHERE id = {{ $('🗄️ DB - Start Run Log').first().json.id }};
      ```
    - Connect **🔄 Loop - Sent Emails → 🗄️ DB - Complete Run Log**

29. **Add optional sticky notes** to document:
    - overall workflow purpose
    - schedule/watermark logic
    - loop/thread matching
    - AI comparison branching
    - corrections and KB update logic

30. **Set up credentials**
    - **Gmail OAuth2**
      - Use an account with access to the support mailbox Sent folder
      - Ensure Gmail API scopes cover message read access
    - **PostgreSQL**
      - Ensure connectivity to the database containing:
        - `ai_drafts`
        - `corrections`
        - `feedback_run_log`
        - `kb_data`
      - If using vector storage, install the required PostgreSQL vector extension and match embedding dimensions
    - **OpenAI API**
      - Required for:
        - comparison model
        - embedding generation
        - KB rewrite

31. **Prepare database schema before enabling the workflow**
    - Required tables/columns implied by this workflow:
      - `feedback_run_log` with:
        - `id`
        - `run_started_at`
        - `run_completed_at`
        - `last_processed_sent_at`
        - `status`
      - `ai_drafts` with:
        - `id`
        - `gmail_thread_id`
        - `gmail_message_id`
        - `original_email_body`
        - `classification`
        - `ai_draft_text`
        - `feedback_processed_at`
        - `was_approved_as_is`
      - `corrections` with:
        - `id`
        - `ai_draft_id`
        - `original_email_body`
        - `classification`
        - `ai_draft_text`
        - `human_sent_text`
        - `diff_summary`
        - `embedding` as vector-compatible type
        - `source`
        - `kb_updated`
        - `created_at` if you want the final update query to work reliably
      - `kb_data` with:
        - `id`
        - `category`
        - `question`
        - `answer`
        - `previous_answer`
        - `updated_by`
        - `updated_at`

32. **Test with sample records**
    - Create at least one `ai_drafts` row with a real `gmail_thread_id`
    - Ensure a matching sent email exists in Gmail after the watermark
    - Run the workflow manually once before activation

33. **Activate the workflow**
    - Once credentials and schema are confirmed, activate it so the schedule trigger runs automatically.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow is designed as a self-learning feedback loop for AI-generated support email drafts. | Overall purpose |
| It stores human-approved or human-edited sent replies so another email-drafting workflow can use them through vector similarity search instead of model fine-tuning. | Architectural context |
| The workflow assumes an external “Workflow 1 (email draft automation)” already exists and is actively generating draft records in `ai_drafts`. | External dependency |
| Setup guidance embedded in the workflow notes: connect Gmail OAuth2, PostgreSQL, and OpenAI credentials; run DB migrations; activate after the upstream draft-generation workflow is ready. | Operational setup |
| KB updates preserve the previous answer for audit by copying `answer` into `previous_answer` before writing the new one. | KB governance note |
| The watermark is updated to `NOW()` at run completion, not to the latest actual sent-message timestamp. This may skip edge-case emails if mailbox timestamps and execution timing are misaligned. | Important implementation caveat |
| Thread matching depends entirely on Gmail thread IDs being captured correctly in the upstream draft-generation workflow. | Integration requirement |
| Parsing logic prefers `text/plain` email content and may degrade for HTML-only or unusually structured MIME messages. | Message parsing caveat |