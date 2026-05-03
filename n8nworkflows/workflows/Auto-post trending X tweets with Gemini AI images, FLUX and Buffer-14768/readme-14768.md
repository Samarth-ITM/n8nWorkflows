Auto-post trending X tweets with Gemini AI images, FLUX and Buffer

https://n8nworkflows.xyz/workflows/auto-post-trending-x-tweets-with-gemini-ai-images--flux-and-buffer-14768


# Auto-post trending X tweets with Gemini AI images, FLUX and Buffer

# 1. Workflow Overview

This workflow automatically finds currently trending topics on X for Nigeria, avoids reusing trends posted in the last 24 hours, generates a short trend-based tweet in pidgin/casual style with Gemini, creates a matching AI image with FLUX, uploads the image to Dropbox to obtain a shareable link, and then publishes the tweet plus image to X via Buffer.

Primary use cases:

- Automated social posting based on live/trending topics
- Country-specific trend harvesting with fallback logic
- AI-assisted short-form content generation
- Image-backed social scheduling through Buffer
- Preventing duplicate topical posts within a rolling 24-hour window

The workflow is organized into the following logical blocks.

## 1.1 Scheduled Execution

The workflow starts on a fixed cron schedule, running three times per day.

## 1.2 Primary Trend Retrieval via Apify

It first attempts to scrape Nigeria X trends using an Apify actor.

## 1.3 Fallback Trend Retrieval

If the primary Apify path fails or returns invalid data, it retries with a second Apify node and then falls back to Gemini search-based trend extraction.

## 1.4 Trend Normalization and Deduplication

Trends from either successful source are normalized into one-item-per-trend records, compared against a Data Table of recently used trends, and filtered so anything used in the last 24 hours is removed.

## 1.5 Trend Selection

The workflow randomly selects up to two unused trends for posting.

## 1.6 Per-Trend Content and Image Generation

For each selected trend, Gemini generates a short tweet and FLUX generates a matching image.

## 1.7 Media Hosting and Post Publishing

The generated image is uploaded to Dropbox, a temporary public link is obtained, and Buffer is used to publish the tweet and image to the X channel.

## 1.8 Trend Persistence

After posting, the trend and usage timestamp are written to the Data Table to prevent reuse within the next 24 hours.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Execution

### Overview
This block defines when the workflow runs. It acts as the only true entry point and triggers the whole automation three times per day.

### Nodes Involved
- Schedule Trigger

### Node Details

#### Schedule Trigger
- **Type and role:** `n8n-nodes-base.scheduleTrigger`; workflow entry point based on cron schedule.
- **Configuration choices:** Configured with cron expression `30 8,12,16 * * *`, which means every day at 08:30, 12:30, and 16:30 server time.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: none
  - Output: `Run an Actor and get dataset`
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**
  - Runs according to the n8n instance timezone/server timezone, which may differ from the intended local posting timezone.
  - If the workflow is inactive, no scheduled runs occur.
- **Sub-workflow reference:** None.

---

## 2.2 Primary Trend Retrieval via Apify

### Overview
This block uses an Apify actor to scrape trending topics for Nigeria from X-related sources and converts the returned dataset into a normalized internal structure. It then checks whether the extraction succeeded.

### Nodes Involved
- Run an Actor and get dataset
- Extract Trends1
- Apify Success? (Primary)

### Node Details

#### Run an Actor and get dataset
- **Type and role:** `@apify/n8n-nodes-apify.apify`; runs an Apify actor and returns its dataset output.
- **Configuration choices:**
  - Operation: `Run actor and get dataset`
  - Actor: `Twitter Trending Topics Scraper 🌎 (easyapi/twitter-trending-topics-scraper)`
  - Custom input body specifies:
    - `country: "nigeria"`
    - `proxyConfiguration.useApifyProxy: false`
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: `Schedule Trigger`
  - Output: `Extract Trends1`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - Invalid or expired Apify credential
  - Actor unavailability or quota exhaustion
  - No data returned
  - Disabling Apify proxy may fail depending on actor/network requirements
- **Sub-workflow reference:** None.

#### Extract Trends1
- **Type and role:** `n8n-nodes-base.code`; parses the Apify dataset and produces a standardized object containing `success` and `trends`.
- **Configuration choices:**
  - The code:
    - Reads all input items
    - Checks the first item for missing data or error presence
    - Extracts `title` fields
    - Trims and deduplicates titles using a `Set`
    - Returns a single item:
      - success case: `{ success: true, trends: [...] }`
      - failure case: `{ success: false, error: ... }`
- **Key expressions or variables used:**
  - `items[0]?.json`
  - `record.title?.trim()`
- **Input and output connections:**
  - Input: `Run an Actor and get dataset`
  - Output: `Apify Success? (Primary)`
- **Version-specific requirements:** Code node type version `2`.
- **Edge cases or potential failure types:**
  - Dataset items may not contain a `title` field
  - Unexpected item structure may produce no valid trends
  - Because `onError` is set to continue, downstream logic must tolerate malformed output
- **Sub-workflow reference:** None.

#### Apify Success? (Primary)
- **Type and role:** `n8n-nodes-base.if`; decides whether the primary Apify extraction succeeded.
- **Configuration choices:**
  - Condition checks `={{ $json.success }}` is boolean true.
  - True branch goes to `Split Trend1`
  - False branch goes to `Run an Actor and get dataset1`
- **Key expressions or variables used:**
  - `{{$json.success}}`
- **Input and output connections:**
  - Input: `Extract Trends1`
  - Outputs:
    - True: `Split Trend1`
    - False: `Run an Actor and get dataset1`
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**
  - If upstream output does not include `success`, condition evaluates false and fallback path is used.
- **Sub-workflow reference:** None.

---

## 2.3 Fallback Trend Retrieval

### Overview
This block provides resilience if the primary source fails. It first retries with a second Apify node and, if that also fails, asks Gemini to retrieve current Nigeria-specific trends via search.

### Nodes Involved
- Run an Actor and get dataset1
- Extract Trends2
- Apify Success? (Fallback)
- Get Daily x trends
- Extract Trends
- Split Trend

### Node Details

#### Run an Actor and get dataset1
- **Type and role:** `@apify/n8n-nodes-apify.apify`; second attempt to retrieve trends via Apify.
- **Configuration choices:**
  - Same actor and same country input as the primary node
  - Different stored credential (`DEMO`)
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: `Apify Success? (Primary)` false branch
  - Output: `Extract Trends2`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - Same as primary Apify node
  - Using a different credential may succeed or fail independently
- **Sub-workflow reference:** None.

#### Extract Trends2
- **Type and role:** `n8n-nodes-base.code`; same normalization logic as `Extract Trends1`, for the secondary Apify path.
- **Configuration choices:**
  - Extracts deduplicated `title` values and returns `{ success, trends }`
- **Key expressions or variables used:**
  - `record.title?.trim()`
- **Input and output connections:**
  - Input: `Run an Actor and get dataset1`
  - Output: `Apify Success? (Fallback)`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Same structural risks as `Extract Trends1`
- **Sub-workflow reference:** None.

#### Apify Success? (Fallback)
- **Type and role:** `n8n-nodes-base.if`; decides whether the fallback Apify attempt succeeded.
- **Configuration choices:**
  - Checks `={{ $json.success }}` is true
  - True branch goes to `Split Trend1`
  - False branch goes to `Get Daily x trends`
- **Key expressions or variables used:**
  - `{{$json.success}}`
- **Input and output connections:**
  - Input: `Extract Trends2`
  - Outputs:
    - True: `Split Trend1`
    - False: `Get Daily x trends`
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**
  - If output is malformed, Gemini fallback will be used.
- **Sub-workflow reference:** None.

#### Get Daily x trends
- **Type and role:** `@n8n/n8n-nodes-langchain.googleGemini`; uses Gemini with Google Search enabled to retrieve current Nigeria-specific X trends.
- **Configuration choices:**
  - Model: `models/gemini-2.5-flash`
  - Prompt asks Gemini to pull Nigeria-specific trends right now from `https://x.com/explore` based on trend tracking sites
  - Output requested as one trend per line, no descriptions
  - Built-in tools:
    - `googleSearch: true`
    - `urlContext: false`
  - `onError: continueRegularOutput`
  - `alwaysOutputData: true`
- **Key expressions or variables used:** Static prompt, no dynamic interpolation.
- **Input and output connections:**
  - Input: `Apify Success? (Fallback)` false branch
  - Output: `Extract Trends`
- **Version-specific requirements:** Type version `1.1`; requires Gemini/Google AI credential compatible with this node version.
- **Edge cases or potential failure types:**
  - Search tool availability may vary by environment
  - Model may return commentary instead of strict line-only output
  - Auth/rate-limit issues with Gemini credential
- **Sub-workflow reference:** None.

#### Extract Trends
- **Type and role:** `n8n-nodes-base.code`; parses Gemini’s text output into a trends array.
- **Configuration choices:**
  - Reads `content.parts[0].text`
  - Splits by newline
  - Trims values and removes empty entries
  - Returns one item: `{ trends: [...] }`
- **Key expressions or variables used:**
  - `$json["content"]?.parts?.[0]?.text`
- **Input and output connections:**
  - Input: `Get Daily x trends`
  - Output: `Split Trend`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Throws if AI output is missing
  - Response schema can change between Gemini node versions
- **Sub-workflow reference:** None.

#### Split Trend
- **Type and role:** `n8n-nodes-base.code`; converts a single item containing `trends` array into one item per trend.
- **Configuration choices:**
  - Expects `$json.trends`
  - Returns `[{ json: { trend } }, ...]`
- **Key expressions or variables used:**
  - `const trends = $json.trends;`
- **Input and output connections:**
  - Input: `Extract Trends`
  - Output: `Create a data table`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - If `trends` is undefined or not an array, code fails
- **Sub-workflow reference:** None.

---

## 2.4 Trend Normalization and Deduplication

### Overview
This block loads previously used trends from an n8n Data Table, filters them down to only trends used in the last 24 hours, and routes both the Apify and Gemini-derived trend streams through a common deduplication stage.

### Nodes Involved
- Split Trend1
- Create a data table
- Get Used Trends
- Filter Last 24h
- Has Apify or Fallback Trends?
- Remove Used Trends
- Remove Used Trends2

### Node Details

#### Split Trend1
- **Type and role:** `n8n-nodes-base.code`; converts Apify success output from a `trends` array into one item per trend.
- **Configuration choices:**
  - Expects `$json.trends`
  - Returns one item per trend under `json.trend`
- **Key expressions or variables used:**
  - `const trends = $json.trends;`
- **Input and output connections:**
  - Input: true branch of `Apify Success? (Primary)` and true branch of `Apify Success? (Fallback)`
  - Output: `Create a data table`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Same array-structure dependency as `Split Trend`
- **Sub-workflow reference:** None.

#### Create a data table
- **Type and role:** `n8n-nodes-base.dataTable`; one-time setup node to create the Data Table schema.
- **Configuration choices:**
  - Resource: table
  - Operation: create
  - Columns:
    - `trend`
    - `date`
  - Table name: `table_name`
  - Disabled in the workflow
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: `Split Trend` and `Split Trend1`
  - Output: `Get Used Trends`
- **Version-specific requirements:** Type version `1.1`; Data Tables require n8n support for that feature.
- **Edge cases or potential failure types:**
  - Because it is disabled, it does not execute during normal runs
  - If the actual required Data Table does not already exist, `Get Used Trends` and `Save Used Trend` will fail
  - Table name here does not match the later referenced existing table name; this node appears intended for manual initial creation only
- **Sub-workflow reference:** None.

#### Get Used Trends
- **Type and role:** `n8n-nodes-base.dataTable`; reads all records from the existing Data Table used for deduplication.
- **Configuration choices:**
  - Operation: `get`
  - Return all: true
  - Data Table ID points to `trend_table`
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: `Create a data table`
  - Output: `Filter Last 24h`
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**
  - Invalid Data Table ID
  - Missing permissions/project mismatch
  - Empty table is valid and should produce no used trends
- **Sub-workflow reference:** None.

#### Filter Last 24h
- **Type and role:** `n8n-nodes-base.code`; reads current trends and removes those that appear in the Data Table within the last 24 hours.
- **Configuration choices:**
  - Reads all input items as current trends
  - Reads all rows from `Get Used Trends`
  - Computes `oneDay = 24 * 60 * 60 * 1000`
  - Filters used records where `item.json.date` is within the last 24 hours
  - Returns only current trends not present in that recent-used list
  - `onError: continueRegularOutput`
  - `alwaysOutputData: true`
- **Key expressions or variables used:**
  - `$input.all()`
  - `$node["Get Used Trends"].all()`
  - `item.json.date`
  - `item.json.trend`
- **Input and output connections:**
  - Input: `Get Used Trends`
  - Output: `Has Apify or Fallback Trends?`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Date parsing can fail for malformed dates
  - If `Get Used Trends` is unavailable, code falls back to an empty array
  - If no available trends remain, returns an empty array
- **Sub-workflow reference:** None.

#### Has Apify or Fallback Trends?
- **Type and role:** `n8n-nodes-base.if`; checks whether either Apify extraction path succeeded and then fans out to the matching deduplication branch.
- **Configuration choices:**
  - OR condition:
    - `Extract Trends1.first().json.success` if executed
    - `Extract Trends2.first().json.success` if executed
  - If true: outputs to both `Remove Used Trends` and `Remove Used Trends2`
- **Key expressions or variables used:**
  - `{{ $if($('Extract Trends1').isExecuted, $('Extract Trends1').first().json.success, false) }}`
  - `{{ $if($('Extract Trends2').isExecuted, $('Extract Trends2').first().json.success, false) }}`
- **Input and output connections:**
  - Input: `Filter Last 24h`
  - Output: `Remove Used Trends`, `Remove Used Trends2`
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**
  - This node only has an explicit true-route usage in the workflow
  - Both downstream nodes are triggered on true, even though only one source path may be relevant
- **Sub-workflow reference:** None.

#### Remove Used Trends
- **Type and role:** `n8n-nodes-base.code`; for the Apify branch, removes trends found in the recent-used result from the `Split Trend1` stream.
- **Configuration choices:**
  - Collects used trends from current input items
  - Reads full current trend list from `$('Split Trend1').all()`
  - Returns only items whose trend is not in the used list
  - `onError: continueRegularOutput`
  - `alwaysOutputData: true`
- **Key expressions or variables used:**
  - `$input.all().map(i => i.json.trend || '')`
  - `$('Split Trend1').all()`
- **Input and output connections:**
  - Input: `Has Apify or Fallback Trends?`
  - Output: `Pick Trend`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - If `Split Trend1` was not executed, expression access may fail depending on runtime context
  - Returns empty array if nothing remains
- **Sub-workflow reference:** None.

#### Remove Used Trends2
- **Type and role:** `n8n-nodes-base.code`; equivalent of the prior node, but for the Gemini fallback trend stream.
- **Configuration choices:**
  - Collects used trends from input
  - Reads full trend list from `$('Split Trend').all()`
  - Returns only items not already used
  - `onError: continueRegularOutput`
  - `alwaysOutputData: true`
- **Key expressions or variables used:**
  - `$('Split Trend').all()`
- **Input and output connections:**
  - Input: `Has Apify or Fallback Trends?`
  - Output: `Pick Trend`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - If `Split Trend` did not execute, this may fail or produce empty results
  - Because both removal nodes feed the same `Pick Trend`, duplicates or multiple incoming streams should be considered during modifications
- **Sub-workflow reference:** None.

---

## 2.5 Trend Selection

### Overview
This block randomizes the available unused trends and selects up to two of them. It then iterates over the selected trends one at a time.

### Nodes Involved
- Pick Trend
- Loop Over Items
- Loop Done (End of Batch)

### Node Details

#### Pick Trend
- **Type and role:** `n8n-nodes-base.code`; shuffles available trends and keeps at most two.
- **Configuration choices:**
  - Uses Fisher-Yates shuffle
  - Selects `items.slice(0, 2)`
  - Normalizes trend extraction with:
    - `item.json.trend?.json?.trend || item.json.trend`
  - `onError: continueRegularOutput`
  - `alwaysOutputData: true`
- **Key expressions or variables used:**
  - `Math.random()`
  - `item.json.trend?.json?.trend || item.json.trend`
- **Input and output connections:**
  - Input: `Remove Used Trends` and `Remove Used Trends2`
  - Output: `Loop Over Items`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - If both upstream branches emit items, the node may receive a merged set
  - If no items arrive, downstream loop receives nothing
- **Sub-workflow reference:** None.

#### Loop Over Items
- **Type and role:** `n8n-nodes-base.splitInBatches`; processes selected trends one by one.
- **Configuration choices:**
  - Default options, effectively batch iteration over incoming items
- **Key expressions or variables used:** Downstream nodes access `$('Loop Over Items').item.json.trend`.
- **Input and output connections:**
  - Input: `Pick Trend`
  - Main output 0: `Generate Tweet with Gemini`
  - Main output 1 / loop completion path: `Loop Done (End of Batch)`
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - If no items are received, per-item generation block will not run
- **Sub-workflow reference:** None.

#### Loop Done (End of Batch)
- **Type and role:** `n8n-nodes-base.noOp`; closes the batch loop by reconnecting to `Loop Over Items`.
- **Configuration choices:** Placeholder node only.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: `Loop Over Items`
  - Output: `Loop Over Items`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - None directly; serves as loop control.
- **Sub-workflow reference:** None.

---

## 2.6 Per-Trend Content and Image Generation

### Overview
For each chosen trend, the workflow asks Gemini to write a short live-event-style tweet in pidgin/casual tone, then uses that tweet text as the basis for generating an image through FLUX on Hugging Face.

### Nodes Involved
- Generate Tweet with Gemini
- Generate Image with FLUX

### Node Details

#### Generate Tweet with Gemini
- **Type and role:** `@n8n/n8n-nodes-langchain.googleGemini`; generates tweet copy for the current trend.
- **Configuration choices:**
  - Model: `models/gemini-2.5-flash`
  - Prompt uses current trend:
    - `generate a short tweet base on this x.com trend: {{ $json.trend }} ...`
  - Prompt instructs:
    - base on live events/context
    - pidgin + casual tone
    - only one option
    - no markdown `**`
    - max 275 characters
  - Google Search built-in tool enabled
- **Key expressions or variables used:**
  - `{{ $json.trend }}`
- **Input and output connections:**
  - Input: `Loop Over Items`
  - Output: `Generate Image with FLUX`
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**
  - Prompt wording is somewhat unstructured and may yield inconsistent style
  - Search-assisted generation may increase latency
  - Model may still exceed desired length without downstream validation
- **Sub-workflow reference:** None.

#### Generate Image with FLUX
- **Type and role:** `n8n-nodes-base.httpRequest`; calls Hugging Face Inference to generate an image file.
- **Configuration choices:**
  - POST to `https://router.huggingface.co/hf-inference/models/black-forest-labs/FLUX.1-schnell`
  - Sends JSON body with `inputs`
  - Prompt is based on generated tweet text:
    - `photorealistic image of {{ $json.content.parts[0].text }}, ultra realistic, cinematic lighting, 8k, highly detailed, professional photography, few or no text`
  - Accept header `image/png`
  - Response format set to file
  - Uses predefined credential type `huggingFaceApi`
- **Key expressions or variables used:**
  - `{{ $json.content.parts[0].text }}`
- **Input and output connections:**
  - Input: `Generate Tweet with Gemini`
  - Output: `Upload a file`
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases or potential failure types:**
  - Missing/invalid Hugging Face token
  - Inference timeout or model queue delays
  - Generated content may not accurately reflect the trend
  - If Gemini response schema changes, prompt interpolation may break
- **Sub-workflow reference:** None.

---

## 2.7 Media Hosting and Post Publishing

### Overview
This block stores the generated image in Dropbox, requests a temporary public download link, and submits the tweet plus image payload to Buffer MCP for posting to X.

### Nodes Involved
- Upload a file
- Generate Download link
- Extract Download Link
- Post to X via Buffer

### Node Details

#### Upload a file
- **Type and role:** `n8n-nodes-base.dropbox`; uploads the generated image binary to Dropbox.
- **Configuration choices:**
  - Path expression:
    - `/n8n/{{ $('Loop Over Items').item.json.trend }}_{{ $itemIndex + 1 }}.png`
  - Binary data upload enabled
  - OAuth2 authentication
- **Key expressions or variables used:**
  - `{{ $('Loop Over Items').item.json.trend }}`
  - `{{ $itemIndex + 1 }}`
- **Input and output connections:**
  - Input: `Generate Image with FLUX`
  - Output: `Generate Download link`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - Invalid Dropbox credential
  - Path issues from trend text containing forbidden filename characters
  - Binary property mismatches if upstream file response is not attached as expected
- **Sub-workflow reference:** None.

#### Generate Download link
- **Type and role:** `n8n-nodes-base.httpRequest`; requests a temporary Dropbox download URL for the uploaded file.
- **Configuration choices:**
  - POST to `https://api.dropboxapi.com/2/files/get_temporary_link`
  - JSON body:
    - `path: {{ $json.path_lower }}`
  - Uses Dropbox OAuth2 predefined credential type
- **Key expressions or variables used:**
  - `{{ $json.path_lower }}`
- **Input and output connections:**
  - Input: `Upload a file`
  - Output: `Extract Download Link`
- **Version-specific requirements:** Type version `4.4`.
- **Edge cases or potential failure types:**
  - Uploaded file path missing from prior node output
  - Temporary links expire, so delayed posting could fail in later redesigns
- **Sub-workflow reference:** None.

#### Extract Download Link
- **Type and role:** `n8n-nodes-base.set`; isolates the Dropbox temporary URL as `file_url`.
- **Configuration choices:**
  - Assigns `file_url = {{ $json.link }}`
- **Key expressions or variables used:**
  - `{{ $json.link }}`
- **Input and output connections:**
  - Input: `Generate Download link`
  - Output: `Post to X via Buffer`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - If Dropbox API returns no `link`, downstream Buffer post will contain invalid asset URL
- **Sub-workflow reference:** None.

#### Post to X via Buffer
- **Type and role:** `@n8n/n8n-nodes-langchain.mcpClient`; calls Buffer MCP tool `create_post` to publish the tweet and image.
- **Configuration choices:**
  - Endpoint URL: `https://mcp.buffer.com/mcp`
  - Authentication: bearer auth
  - Tool: `create_post`
  - Mapping mode: define below
  - Parameters:
    - `text`: generated tweet text from Gemini
    - `assets`: object with image URL and thumbnail URL both set to Dropbox temporary link
    - `metadata.altText`: tweet text
    - `channelId`: `69ce9bddaf47dacb6980757d`
    - `schedulingType`: `automatic`
  - Node note: `Twiiter Auto post`
- **Key expressions or variables used:**
  - `{{ $('Generate Tweet with Gemini').item.json.content.parts[0].text }}`
  - `{{ $json.file_url }}`
- **Input and output connections:**
  - Input: `Extract Download Link`
  - Output: `Prepare Trend Record`
- **Version-specific requirements:** Type version `1`; requires MCP client support and working Buffer MCP endpoint access.
- **Edge cases or potential failure types:**
  - Invalid Buffer bearer token
  - Wrong `channelId`
  - Buffer may reject malformed assets or expired URLs
  - Tweet text may exceed channel/platform limits
  - The sticky note says replace placeholder channel ID, but the JSON currently contains a concrete ID
- **Sub-workflow reference:** None.

---

## 2.8 Trend Persistence

### Overview
After a post is submitted, this block creates a usage record and stores the trend plus timestamp in the Data Table so the same trend can be excluded for the next 24 hours.

### Nodes Involved
- Prepare Trend Record
- Save Used Trend

### Node Details

#### Prepare Trend Record
- **Type and role:** `n8n-nodes-base.code`; builds the Data Table row for the just-used trend.
- **Configuration choices:**
  - Reads trend from `Loop Over Items`
  - Tries to read tweet text from one of two Gemini response shapes:
    - `content[0].text.text`
    - `content.parts[0].text`
  - Sets:
    - `trend`
    - `tweet`
    - `usedAt` as current ISO timestamp
  - Throws if no trend is found
- **Key expressions or variables used:**
  - `$('Loop Over Items').item.json.trend`
  - `$input.item.json?.content?.[0]?.text?.text`
  - `$input.item.json?.content?.parts?.[0]?.text`
  - `new Date().toISOString()`
- **Input and output connections:**
  - Input: `Post to X via Buffer`
  - Output: `Save Used Trend`
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - Buffer output likely does not contain tweet text, so `tweet` may be empty
  - If loop item context is lost, the node throws `No trend found from Loop Over Items`
- **Sub-workflow reference:** None.

#### Save Used Trend
- **Type and role:** `n8n-nodes-base.dataTable`; stores the used trend with its timestamp in the existing Data Table.
- **Configuration choices:**
  - Data Table ID points to `trend_table`
  - Writes:
    - `trend = {{ $json.trend }}`
    - `date = {{ $json.usedAt }}`
  - Schema mapping defined explicitly
- **Key expressions or variables used:**
  - `{{ $json.usedAt }}`
  - `{{ $json.trend }}`
- **Input and output connections:**
  - Input: `Prepare Trend Record`
  - Output: none
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**
  - Missing Data Table
  - Invalid field mapping
  - Table write permission issues
- **Sub-workflow reference:** None.

---

## 2.9 Documentation and In-Canvas Notes

### Overview
These nodes do not execute logic. They provide setup, usage, and cautionary guidance directly inside the n8n canvas.

### Nodes Involved
- 📋 Workflow Overview
- Section: Trend Fetching
- Section: Deduplication
- Section: Generation & Posting
- ⚠️ HuggingFace Setup
- ⚠️ Buffer Channel ID
- Data Table Setup Tip

### Node Details

#### 📋 Workflow Overview
- **Type and role:** `n8n-nodes-base.stickyNote`; high-level description and setup summary.
- **Configuration choices:** Large note covering purpose, steps, setup requirements, and customization ideas.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Section: Trend Fetching
- **Type and role:** Sticky note labeling the trend acquisition area.
- **Configuration choices:** Explains primary Apify and Gemini fallback flow.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Section: Deduplication
- **Type and role:** Sticky note labeling the deduplication area.
- **Configuration choices:** Explains Data Table-based 24-hour filtering and random trend picking.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Section: Generation & Posting
- **Type and role:** Sticky note labeling the generation/posting area.
- **Configuration choices:** Explains looping behavior and publish flow.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### ⚠️ HuggingFace Setup
- **Type and role:** Sticky note with credential guidance.
- **Configuration choices:** Instructs users to use HTTP Bearer Token credential for Hugging Face.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### ⚠️ Buffer Channel ID
- **Type and role:** Sticky note with Buffer setup guidance.
- **Configuration choices:** Advises replacing placeholder with actual Buffer channel ID.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Data Table Setup Tip
- **Type and role:** Sticky note with one-time Data Table setup guidance.
- **Configuration choices:** Explains manually running `Create a data table` once, then disabling it.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Scheduled entry point |  | Run an Actor and get dataset | ### 1️⃣ Fetch Nigeria Trends<br>Apify scrapes X trending topics for Nigeria.<br>If Apify fails or returns no data, falls back to Gemini AI search.<br>Both paths merge at `Has Apify or Fallback Trends?` |
| Run an Actor and get dataset | @apify/n8n-nodes-apify.apify | Primary Apify trend scrape | Schedule Trigger | Extract Trends1 | ### 1️⃣ Fetch Nigeria Trends<br>Apify scrapes X trending topics for Nigeria.<br>If Apify fails or returns no data, falls back to Gemini AI search.<br>Both paths merge at `Has Apify or Fallback Trends?` |
| Extract Trends1 | n8n-nodes-base.code | Normalize primary Apify output | Run an Actor and get dataset | Apify Success? (Primary) | ### 1️⃣ Fetch Nigeria Trends<br>Apify scrapes X trending topics for Nigeria.<br>If Apify fails or returns no data, falls back to Gemini AI search.<br>Both paths merge at `Has Apify or Fallback Trends?` |
| Apify Success? (Primary) | n8n-nodes-base.if | Decide whether to use primary Apify results | Extract Trends1 | Split Trend1; Run an Actor and get dataset1 | ### 1️⃣ Fetch Nigeria Trends<br>Apify scrapes X trending topics for Nigeria.<br>If Apify fails or returns no data, falls back to Gemini AI search.<br>Both paths merge at `Has Apify or Fallback Trends?` |
| Run an Actor and get dataset1 | @apify/n8n-nodes-apify.apify | Secondary Apify retry | Apify Success? (Primary) | Extract Trends2 | ### 1️⃣ Fetch Nigeria Trends<br>Apify scrapes X trending topics for Nigeria.<br>If Apify fails or returns no data, falls back to Gemini AI search.<br>Both paths merge at `Has Apify or Fallback Trends?` |
| Extract Trends2 | n8n-nodes-base.code | Normalize secondary Apify output | Run an Actor and get dataset1 | Apify Success? (Fallback) | ### 1️⃣ Fetch Nigeria Trends<br>Apify scrapes X trending topics for Nigeria.<br>If Apify fails or returns no data, falls back to Gemini AI search.<br>Both paths merge at `Has Apify or Fallback Trends?` |
| Apify Success? (Fallback) | n8n-nodes-base.if | Decide between secondary Apify and Gemini fallback | Extract Trends2 | Split Trend1; Get Daily x trends | ### 1️⃣ Fetch Nigeria Trends<br>Apify scrapes X trending topics for Nigeria.<br>If Apify fails or returns no data, falls back to Gemini AI search.<br>Both paths merge at `Has Apify or Fallback Trends?` |
| Get Daily x trends | @n8n/n8n-nodes-langchain.googleGemini | Gemini fallback trend retrieval | Apify Success? (Fallback) | Extract Trends | ### 1️⃣ Fetch Nigeria Trends<br>Apify scrapes X trending topics for Nigeria.<br>If Apify fails or returns no data, falls back to Gemini AI search.<br>Both paths merge at `Has Apify or Fallback Trends?` |
| Extract Trends | n8n-nodes-base.code | Parse Gemini trend list into array | Get Daily x trends | Split Trend | ### 1️⃣ Fetch Nigeria Trends<br>Apify scrapes X trending topics for Nigeria.<br>If Apify fails or returns no data, falls back to Gemini AI search.<br>Both paths merge at `Has Apify or Fallback Trends?` |
| Split Trend | n8n-nodes-base.code | Split Gemini trends into one item per trend | Extract Trends | Create a data table | ### 1️⃣ Fetch Nigeria Trends<br>Apify scrapes X trending topics for Nigeria.<br>If Apify fails or returns no data, falls back to Gemini AI search.<br>Both paths merge at `Has Apify or Fallback Trends?` |
| Split Trend1 | n8n-nodes-base.code | Split Apify trends into one item per trend | Apify Success? (Primary); Apify Success? (Fallback) | Create a data table | ### 1️⃣ Fetch Nigeria Trends<br>Apify scrapes X trending topics for Nigeria.<br>If Apify fails or returns no data, falls back to Gemini AI search.<br>Both paths merge at `Has Apify or Fallback Trends?` |
| Create a data table | n8n-nodes-base.dataTable | One-time Data Table creation | Split Trend; Split Trend1 | Get Used Trends | ### 2️⃣ Deduplication<br>Reads already-used trends from the `trend_table` Data Table.<br>Filters out any trend used in the **last 24 hours**.<br>Then picks 2 random trends from what's left.<br>🛠️ **One-time setup:**<br>Run `Create a data table` manually once to create the `trend_table`.<br>Then **disable** that node — it only needs to run once. |
| Get Used Trends | n8n-nodes-base.dataTable | Read used trends history | Create a data table | Filter Last 24h | ### 2️⃣ Deduplication<br>Reads already-used trends from the `trend_table` Data Table.<br>Filters out any trend used in the **last 24 hours**.<br>Then picks 2 random trends from what's left. |
| Filter Last 24h | n8n-nodes-base.code | Remove trends used within the last day | Get Used Trends | Has Apify or Fallback Trends? | ### 2️⃣ Deduplication<br>Reads already-used trends from the `trend_table` Data Table.<br>Filters out any trend used in the **last 24 hours**.<br>Then picks 2 random trends from what's left. |
| Has Apify or Fallback Trends? | n8n-nodes-base.if | Route to proper deduped trend source handling | Filter Last 24h | Remove Used Trends; Remove Used Trends2 | ### 2️⃣ Deduplication<br>Reads already-used trends from the `trend_table` Data Table.<br>Filters out any trend used in the **last 24 hours**.<br>Then picks 2 random trends from what's left. |
| Remove Used Trends | n8n-nodes-base.code | Filter used trends from Apify stream | Has Apify or Fallback Trends? | Pick Trend | ### 2️⃣ Deduplication<br>Reads already-used trends from the `trend_table` Data Table.<br>Filters out any trend used in the **last 24 hours**.<br>Then picks 2 random trends from what's left. |
| Remove Used Trends2 | n8n-nodes-base.code | Filter used trends from Gemini fallback stream | Has Apify or Fallback Trends? | Pick Trend | ### 2️⃣ Deduplication<br>Reads already-used trends from the `trend_table` Data Table.<br>Filters out any trend used in the **last 24 hours**.<br>Then picks 2 random trends from what's left. |
| Pick Trend | n8n-nodes-base.code | Shuffle and select up to 2 trends | Remove Used Trends; Remove Used Trends2 | Loop Over Items | ### 2️⃣ Deduplication<br>Reads already-used trends from the `trend_table` Data Table.<br>Filters out any trend used in the **last 24 hours**.<br>Then picks 2 random trends from what's left. |
| Loop Over Items | n8n-nodes-base.splitInBatches | Iterate over selected trends | Pick Trend; Loop Done (End of Batch) | Generate Tweet with Gemini; Loop Done (End of Batch) | ### 3️⃣ Generate Tweet & Image (loops per trend)<br>For each picked trend:<br>- Gemini generates a pidgin-English tweet (≤275 chars)<br>- FLUX.1-schnell generates a matching photorealistic image<br>- Image uploaded to Dropbox → temporary public URL extracted<br>- Tweet + image posted to X via Buffer MCP<br>- Trend saved to `trend_table` to prevent reuse |
| Loop Done (End of Batch) | n8n-nodes-base.noOp | Batch loop closure | Loop Over Items | Loop Over Items | ### 3️⃣ Generate Tweet & Image (loops per trend)<br>For each picked trend:<br>- Gemini generates a pidgin-English tweet (≤275 chars)<br>- FLUX.1-schnell generates a matching photorealistic image<br>- Image uploaded to Dropbox → temporary public URL extracted<br>- Tweet + image posted to X via Buffer MCP<br>- Trend saved to `trend_table` to prevent reuse |
| Generate Tweet with Gemini | @n8n/n8n-nodes-langchain.googleGemini | Generate tweet copy for each trend | Loop Over Items | Generate Image with FLUX | ### 3️⃣ Generate Tweet & Image (loops per trend)<br>For each picked trend:<br>- Gemini generates a pidgin-English tweet (≤275 chars)<br>- FLUX.1-schnell generates a matching photorealistic image<br>- Image uploaded to Dropbox → temporary public URL extracted<br>- Tweet + image posted to X via Buffer MCP<br>- Trend saved to `trend_table` to prevent reuse |
| Generate Image with FLUX | n8n-nodes-base.httpRequest | Generate image from tweet text | Generate Tweet with Gemini | Upload a file | ### 3️⃣ Generate Tweet & Image (loops per trend)<br>For each picked trend:<br>- Gemini generates a pidgin-English tweet (≤275 chars)<br>- FLUX.1-schnell generates a matching photorealistic image<br>- Image uploaded to Dropbox → temporary public URL extracted<br>- Tweet + image posted to X via Buffer MCP<br>- Trend saved to `trend_table` to prevent reuse<br>⚠️ **HuggingFace Credential Required**<br>Create a new **HTTP Bearer Token** credential in n8n with your HuggingFace API token and select it in this node.<br>Do NOT paste the token directly in node parameters. |
| Upload a file | n8n-nodes-base.dropbox | Upload generated image to Dropbox | Generate Image with FLUX | Generate Download link | ### 3️⃣ Generate Tweet & Image (loops per trend)<br>For each picked trend:<br>- Gemini generates a pidgin-English tweet (≤275 chars)<br>- FLUX.1-schnell generates a matching photorealistic image<br>- Image uploaded to Dropbox → temporary public URL extracted<br>- Tweet + image posted to X via Buffer MCP<br>- Trend saved to `trend_table` to prevent reuse |
| Generate Download link | n8n-nodes-base.httpRequest | Request temporary Dropbox file link | Upload a file | Extract Download Link | ### 3️⃣ Generate Tweet & Image (loops per trend)<br>For each picked trend:<br>- Gemini generates a pidgin-English tweet (≤275 chars)<br>- FLUX.1-schnell generates a matching photorealistic image<br>- Image uploaded to Dropbox → temporary public URL extracted<br>- Tweet + image posted to X via Buffer MCP<br>- Trend saved to `trend_table` to prevent reuse |
| Extract Download Link | n8n-nodes-base.set | Store Dropbox temporary link as file_url | Generate Download link | Post to X via Buffer | ### 3️⃣ Generate Tweet & Image (loops per trend)<br>For each picked trend:<br>- Gemini generates a pidgin-English tweet (≤275 chars)<br>- FLUX.1-schnell generates a matching photorealistic image<br>- Image uploaded to Dropbox → temporary public URL extracted<br>- Tweet + image posted to X via Buffer MCP<br>- Trend saved to `trend_table` to prevent reuse |
| Post to X via Buffer | @n8n/n8n-nodes-langchain.mcpClient | Publish tweet and image through Buffer | Extract Download Link | Prepare Trend Record | ### 3️⃣ Generate Tweet & Image (loops per trend)<br>For each picked trend:<br>- Gemini generates a pidgin-English tweet (≤275 chars)<br>- FLUX.1-schnell generates a matching photorealistic image<br>- Image uploaded to Dropbox → temporary public URL extracted<br>- Tweet + image posted to X via Buffer MCP<br>- Trend saved to `trend_table` to prevent reuse<br>📌 **Buffer Setup**<br>Replace `YOUR_BUFFER_CHANNEL_ID` in this node with your actual Buffer channel ID for your X (Twitter) account.<br>Find it in your Buffer dashboard → Channel Settings. |
| Prepare Trend Record | n8n-nodes-base.code | Build row for used trend logging | Post to X via Buffer | Save Used Trend | ### 3️⃣ Generate Tweet & Image (loops per trend)<br>For each picked trend:<br>- Gemini generates a pidgin-English tweet (≤275 chars)<br>- FLUX.1-schnell generates a matching photorealistic image<br>- Image uploaded to Dropbox → temporary public URL extracted<br>- Tweet + image posted to X via Buffer MCP<br>- Trend saved to `trend_table` to prevent reuse |
| Save Used Trend | n8n-nodes-base.dataTable | Persist trend usage to Data Table | Prepare Trend Record |  | ### 3️⃣ Generate Tweet & Image (loops per trend)<br>For each picked trend:<br>- Gemini generates a pidgin-English tweet (≤275 chars)<br>- FLUX.1-schnell generates a matching photorealistic image<br>- Image uploaded to Dropbox → temporary public URL extracted<br>- Tweet + image posted to X via Buffer MCP<br>- Trend saved to `trend_table` to prevent reuse |
| 📋 Workflow Overview | n8n-nodes-base.stickyNote | Canvas documentation |  |  | ## 🐦Generate & Post AI Tweets with Images from X Trending Topics (Gemini + FLUX + Buffer)<br><br>**What this workflow does:**<br>Every day at 8:30am, 12:30pm, and 4:30pm it:<br>1. Scrapes Nigeria's trending X topics via Apify<br>2. Falls back to Gemini AI search if Apify fails<br>3. Filters out trends used in the last 24h (deduplication)<br>4. Picks 2 random unused trends<br>5. Generates a pidgin-language tweet per trend using Gemini 2.5 Flash<br>6. Generates a matching image using FLUX.1-schnell on HuggingFace<br>7. Uploads image to Dropbox → gets a public link<br>8. Posts tweet + image to X via Buffer MCP<br><br>---<br>### 🔧 Setup Required<br>- **Apify** – Connect your Apify API credential<br>- **Google Gemini** – Connect your Google AI (Gemini) credential<br>- **HuggingFace** – Create an HTTP Bearer Token credential with your HF token<br>- **Dropbox** – Connect your Dropbox OAuth2 credential<br>- **Buffer** – Connect your Buffer Bearer Token credential and set your Channel ID in the `Post to X via Buffer` node<br>- **Data Table** – The `trend_table` Data Table must exist in your n8n project (run `Create a data table` once manually, then disable it)<br><br>---<br>### ⚙️ Customization<br>- Change the `country` in the Apify nodes to scrape trends for a different country<br>- Edit the tweet prompt in `Generate Tweet with Gemini` to change tone/language/style<br>- Edit the FLUX prompt in `Generate Image with FLUX` to change image style<br>- Adjust the schedule in `Schedule Trigger` (currently 3x/day) |
| Section: Trend Fetching | n8n-nodes-base.stickyNote | Canvas section note |  |  | ### 1️⃣ Fetch Nigeria Trends<br>Apify scrapes X trending topics for Nigeria.<br>If Apify fails or returns no data, falls back to Gemini AI search.<br>Both paths merge at `Has Apify or Fallback Trends?` |
| Section: Deduplication | n8n-nodes-base.stickyNote | Canvas section note |  |  | ### 2️⃣ Deduplication<br>Reads already-used trends from the `trend_table` Data Table.<br>Filters out any trend used in the **last 24 hours**.<br>Then picks 2 random trends from what's left. |
| Section: Generation & Posting | n8n-nodes-base.stickyNote | Canvas section note |  |  | ### 3️⃣ Generate Tweet & Image (loops per trend)<br>For each picked trend:<br>- Gemini generates a pidgin-English tweet (≤275 chars)<br>- FLUX.1-schnell generates a matching photorealistic image<br>- Image uploaded to Dropbox → temporary public URL extracted<br>- Tweet + image posted to X via Buffer MCP<br>- Trend saved to `trend_table` to prevent reuse |
| ⚠️ HuggingFace Setup | n8n-nodes-base.stickyNote | Canvas credential note |  |  | ⚠️ **HuggingFace Credential Required**<br>Create a new **HTTP Bearer Token** credential in n8n with your HuggingFace API token and select it in this node.<br>Do NOT paste the token directly in node parameters. |
| ⚠️ Buffer Channel ID | n8n-nodes-base.stickyNote | Canvas setup note |  |  | 📌 **Buffer Setup**<br>Replace `YOUR_BUFFER_CHANNEL_ID` in this node with your actual Buffer channel ID for your X (Twitter) account.<br>Find it in your Buffer dashboard → Channel Settings. |
| Data Table Setup Tip | n8n-nodes-base.stickyNote | Canvas setup note |  |  | 🛠️ **One-time setup:**<br>Run `Create a data table` manually once to create the `trend_table`.<br>Then **disable** that node — it only needs to run once. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   `Auto-Post Trending Tweets with AI Images using Gemini, FLUX & Buffer`.

2. **Add a Schedule Trigger node.**
   - Type: `Schedule Trigger`
   - Set a cron schedule:
     - Expression: `30 8,12,16 * * *`
   - This will run 3 times daily.

3. **Add the primary Apify node.**
   - Type: `Apify`
   - Operation: `Run actor and get dataset`
   - Select actor: `Twitter Trending Topics Scraper 🌎 (easyapi/twitter-trending-topics-scraper)`
   - Custom body:
     ```json
     {
       "country": "nigeria",
       "proxyConfiguration": {
         "useApifyProxy": false
       }
     }
     ```
   - Connect `Schedule Trigger -> Run an Actor and get dataset`.
   - Configure an Apify credential.

4. **Add a Code node named `Extract Trends1`.**
   - Paste logic that:
     - Reads all items
     - Detects errors or missing data
     - Extracts `title`
     - Deduplicates titles
     - Returns one item like:
       - success true with `trends`
       - or success false with `error`
   - Connect `Run an Actor and get dataset -> Extract Trends1`.

5. **Add an IF node named `Apify Success? (Primary)`.**
   - Condition:
     - Boolean true on `{{$json.success}}`
   - Connect `Extract Trends1 -> Apify Success? (Primary)`.

6. **Add a second Apify node named `Run an Actor and get dataset1`.**
   - Same actor and same custom body as the primary Apify node.
   - Connect the **false** output of `Apify Success? (Primary)` to this node.
   - Configure either the same Apify credential or a different one if needed.

7. **Add a Code node named `Extract Trends2`.**
   - Use the same extraction logic as `Extract Trends1`.
   - Connect `Run an Actor and get dataset1 -> Extract Trends2`.

8. **Add an IF node named `Apify Success? (Fallback)`.**
   - Condition:
     - Boolean true on `{{$json.success}}`
   - Connect `Extract Trends2 -> Apify Success? (Fallback)`.

9. **Add a Code node named `Split Trend1`.**
   - This node must transform:
     - input: one item with `trends` array
     - output: one item per trend under `json.trend`
   - Connect:
     - `Apify Success? (Primary)` true -> `Split Trend1`
     - `Apify Success? (Fallback)` true -> `Split Trend1`

10. **Add the Gemini fallback trend node named `Get Daily x trends`.**
    - Type: `Google Gemini`
    - Model: `models/gemini-2.5-flash`
    - Enable built-in `googleSearch`
    - Use a prompt asking Gemini to pull Nigeria-specific current trends and output only one trend per line.
    - Set `onError` to continue if available in your UI.
    - Connect `Apify Success? (Fallback)` false -> `Get Daily x trends`.
    - Configure Google Gemini / Google AI credential.

11. **Add a Code node named `Extract Trends`.**
    - Read Gemini text from `content.parts[0].text`
    - Split by newline
    - Trim each line
    - Remove empty lines
    - Return `{ trends: [...] }`
    - Connect `Get Daily x trends -> Extract Trends`.

12. **Add a Code node named `Split Trend`.**
    - Same concept as `Split Trend1`:
      - input item with `trends`
      - output one item per trend
    - Connect `Extract Trends -> Split Trend`.

13. **Create the Data Table infrastructure.**
    - Add a Data Table node named `Create a data table`.
    - Configure:
      - Resource: table
      - Operation: create
      - Add columns:
        - `trend`
        - `date`
    - This node is only for one-time setup.
    - Connect both:
      - `Split Trend -> Create a data table`
      - `Split Trend1 -> Create a data table`

14. **Run the `Create a data table` node manually once** to create the table, then disable it.
    - Important: ensure the actual table used later is the same table you reference in read/write nodes.
    - In practice, after creation, note the Data Table ID and use it in the next nodes.

15. **Add a Data Table node named `Get Used Trends`.**
    - Operation: `get`
    - Return all: true
    - Select the created Data Table, intended as `trend_table`
    - Connect `Create a data table -> Get Used Trends`.

16. **Add a Code node named `Filter Last 24h`.**
    - Logic should:
      - Read current incoming trends
      - Read all rows from `Get Used Trends`
      - Keep only used rows where `date` is within the last 24 hours
      - Remove matching trends from the current trend list
      - Return one item per remaining trend
    - Connect `Get Used Trends -> Filter Last 24h`.

17. **Add an IF node named `Has Apify or Fallback Trends?`.**
    - Use an OR condition to test whether either `Extract Trends1` or `Extract Trends2` executed successfully.
    - Connect `Filter Last 24h -> Has Apify or Fallback Trends?`.

18. **Add Code node `Remove Used Trends`.**
    - Purpose: for the Apify path, compare the items from `Filter Last 24h` to the full `Split Trend1` output and keep only unused trends.
    - Connect `Has Apify or Fallback Trends?` true -> `Remove Used Trends`.

19. **Add Code node `Remove Used Trends2`.**
    - Purpose: for the Gemini fallback path, compare the items from `Filter Last 24h` to the full `Split Trend` output and keep only unused trends.
    - Connect `Has Apify or Fallback Trends?` true -> `Remove Used Trends2`.

20. **Add Code node `Pick Trend`.**
    - Implement:
      - Fisher-Yates shuffle
      - Select up to 2 items
      - Normalize trend extraction if nested
    - Connect:
      - `Remove Used Trends -> Pick Trend`
      - `Remove Used Trends2 -> Pick Trend`

21. **Add a Split In Batches node named `Loop Over Items`.**
    - Use default settings.
    - Connect `Pick Trend -> Loop Over Items`.

22. **Add a No Operation node named `Loop Done (End of Batch)`.**
    - Connect:
      - `Loop Over Items -> Loop Done (End of Batch)`
      - `Loop Done (End of Batch) -> Loop Over Items`
    - This closes the iteration cycle.

23. **Add a Gemini node named `Generate Tweet with Gemini`.**
    - Type: `Google Gemini`
    - Model: `models/gemini-2.5-flash`
    - Enable `googleSearch`
    - Prompt should include `{{$json.trend}}`
    - Instruct the model to:
      - generate one short tweet
      - react to live context
      - use pidgin + casual tone
      - stay below 275 characters
      - avoid markdown
    - Connect `Loop Over Items -> Generate Tweet with Gemini`.

24. **Add an HTTP Request node named `Generate Image with FLUX`.**
    - Method: `POST`
    - URL: `https://router.huggingface.co/hf-inference/models/black-forest-labs/FLUX.1-schnell`
    - Authentication: predefined Hugging Face credential
    - Header:
      - `Accept: image/png`
    - Send JSON body:
      - `inputs` should describe a photorealistic image based on the Gemini tweet text
    - Response format: `File`
    - Connect `Generate Tweet with Gemini -> Generate Image with FLUX`.
    - Create a Hugging Face API credential using bearer token authentication.

25. **Add a Dropbox node named `Upload a file`.**
    - Operation: upload file
    - Enable binary data
    - Path expression:
      - `/n8n/{{ $('Loop Over Items').item.json.trend }}_{{ $itemIndex + 1 }}.png`
    - Connect `Generate Image with FLUX -> Upload a file`.
    - Configure Dropbox OAuth2 credential.

26. **Add an HTTP Request node named `Generate Download link`.**
    - Method: `POST`
    - URL: `https://api.dropboxapi.com/2/files/get_temporary_link`
    - Authentication: Dropbox predefined credential
    - JSON body:
      - `path: {{ $json.path_lower }}`
    - Connect `Upload a file -> Generate Download link`.

27. **Add a Set node named `Extract Download Link`.**
    - Add field:
      - `file_url = {{$json.link}}`
    - Connect `Generate Download link -> Extract Download Link`.

28. **Add an MCP Client node named `Post to X via Buffer`.**
    - Endpoint URL: `https://mcp.buffer.com/mcp`
    - Authentication: bearer auth
    - Tool: `create_post`
    - Set parameters:
      - `channelId`: your Buffer channel ID for the connected X account
      - `schedulingType`: `automatic`
      - `text`: generated tweet text from the Gemini tweet node
      - `assets.images[0].url`: `{{$json.file_url}}`
      - `assets.images[0].thumbnailUrl`: `{{$json.file_url}}`
      - `assets.images[0].metadata.altText`: generated tweet text
    - Connect `Extract Download Link -> Post to X via Buffer`.
    - Configure Buffer bearer token credential.

29. **Add a Code node named `Prepare Trend Record`.**
    - Build a record with:
      - `trend` from `Loop Over Items`
      - `tweet` if available
      - `usedAt` as `new Date().toISOString()`
    - Connect `Post to X via Buffer -> Prepare Trend Record`.

30. **Add a Data Table node named `Save Used Trend`.**
    - Use the same Data Table as `Get Used Trends`
    - Map:
      - `trend = {{$json.trend}}`
      - `date = {{$json.usedAt}}`
    - Connect `Prepare Trend Record -> Save Used Trend`.

31. **Disable the `Create a data table` node** after one-time setup.
    - This is necessary to avoid trying to recreate the table at every run.

32. **Add sticky notes if desired** to document:
    - workflow overview
    - trend fetching area
    - deduplication area
    - generation/posting area
    - Hugging Face credential requirement
    - Buffer channel ID reminder
    - Data Table one-time setup note

33. **Review credentials required before activation.**
    - Apify API
    - Google Gemini / Google AI
    - Hugging Face bearer token
    - Dropbox OAuth2
    - Buffer bearer auth

34. **Validate critical assumptions before enabling production use.**
    - The Data Table exists and the ID is correct
    - The Buffer `channelId` is correct
    - Dropbox paths can accept trend text safely
    - Gemini output schema matches expressions used in image and post nodes
    - Hugging Face returns binary image output as expected

35. **Activate the workflow** once all manual tests succeed.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow is designed for Nigeria-specific X trends by default. | Change `country` in both Apify actor nodes if you want another country. |
| The Gemini fallback uses search-enabled prompting rather than a direct X API integration. | Fallback node: `Get Daily x trends` |
| The image generation model is FLUX.1-schnell served via Hugging Face inference router. | `https://router.huggingface.co/hf-inference/models/black-forest-labs/FLUX.1-schnell` |
| Dropbox temporary links are time-limited. If publishing is delayed, asset URLs may expire. | Dropbox temporary link API |
| Buffer posting is performed through MCP rather than a standard dedicated Buffer node. | MCP endpoint: `https://mcp.buffer.com/mcp` |
| The canvas note says to replace `YOUR_BUFFER_CHANNEL_ID`, but the workflow JSON already contains a concrete channel ID. Verify it belongs to the intended X account before running. | Buffer channel configuration |
| The Data Table setup note indicates a one-time manual creation step, then disabling the creation node. | Relevant node: `Create a data table` |
| Main service referenced for trend exploration in the Gemini prompt. | https://x.com/explore |