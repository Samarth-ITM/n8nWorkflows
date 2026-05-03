Turn top Instagram reels into 7 new scripts using Apify, OpenAI, Claude and Google Sheets

https://n8nworkflows.xyz/workflows/turn-top-instagram-reels-into-7-new-scripts-using-apify--openai--claude-and-google-sheets-14825


# Turn top Instagram reels into 7 new scripts using Apify, OpenAI, Claude and Google Sheets

# 1. Workflow Overview

This workflow takes high-performing Instagram reels from a target profile, extracts the best-performing posts, transcribes their audio, and uses AI to generate 7 new short-form script ideas. The final scripts are appended to a Google Sheet in a structured format.

Primary use cases:
- Content research based on proven Instagram reel performance
- Script ideation for short-form video teams
- Building reusable content systems from competitor or niche account analysis

Logical blocks:

## 1.1 Input Reception and Instagram Data Collection
The workflow starts manually, launches an Apify Instagram scraping actor, and retrieves the resulting dataset items.

## 1.2 Ranking and Filtering Source Content
The scraped posts are sorted by video play count, limited to the top 10, and filtered so only posts with a usable audio URL continue.

## 1.3 Audio Download and Transcription
For each qualifying reel, the workflow downloads the audio file and sends it to OpenAI transcription. The resulting transcript texts are merged into one combined prompt input.

## 1.4 AI Script Generation
Claude receives the combined transcripts and generates 7 new script ideas in a strict JSON structure.

## 1.5 Output Formatting and Persistence
The Claude response is cleaned, parsed into structured items, and each generated script is appended as a new row in Google Sheets.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Instagram Data Collection

### Overview
This block initializes the workflow and retrieves Instagram post data from a target account using Apify’s Instagram Scraper actor. It provides the raw source dataset used by all downstream processing.

### Nodes Involved
- When clicking ‘Execute workflow’
- Run an Actor1
- Get dataset items1

### Node Details

#### 1) When clicking ‘Execute workflow’
- **Type and technical role:** `n8n-nodes-base.manualTrigger`; manual entry point for testing or ad hoc execution.
- **Configuration choices:** No custom parameters. The workflow starts only when manually executed from the editor.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input. Outputs to `Run an Actor1`.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:**
  - No automation trigger exists, so the workflow will not run on a schedule unless replaced or extended.
- **Sub-workflow reference:** None.

#### 2) Run an Actor1
- **Type and technical role:** `@apify/n8n-nodes-apify.apify`; launches an Apify actor run.
- **Configuration choices:**
  - Uses actor: **Instagram Scraper (apify/instagram-scraper)**
  - Actor ID corresponds to Apify’s Instagram scraping actor.
  - The request body specifies:
    - `directUrls`: a list containing one Instagram profile URL placeholder
    - `resultsLimit`: 30
    - `resultsType`: `posts`
- **Key expressions or variables used:** None in the body itself; the URL is hardcoded and must be edited manually.
- **Input and output connections:** Input from `When clicking ‘Execute workflow’`; output to `Get dataset items1`.
- **Version-specific requirements:** Type version 1 of the Apify node.
- **Edge cases or potential failure types:**
  - Invalid Apify credentials
  - Actor quota exhaustion
  - Bad or placeholder Instagram profile URL not replaced
  - Instagram scraping limitations or temporary bans
  - Empty dataset if profile is inaccessible, private, or yields no posts
- **Sub-workflow reference:** None.

#### 3) Get dataset items1
- **Type and technical role:** `@apify/n8n-nodes-apify.apify`; reads items from the dataset created by the previous actor run.
- **Configuration choices:**
  - Resource: `Datasets`
  - Dataset ID is dynamically taken from the previous node: `{{$json.defaultDatasetId}}`
- **Key expressions or variables used:**
  - `{{$json.defaultDatasetId}}`
- **Input and output connections:** Input from `Run an Actor1`; output to `Sort1`.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:**
  - Missing `defaultDatasetId` if the actor run failed or returned an unexpected payload
  - Empty dataset
  - API access issues to the dataset
- **Sub-workflow reference:** None.

---

## 2.2 Ranking and Filtering Source Content

### Overview
This block narrows the scraped Instagram posts to the most relevant source material. It prioritizes videos with the highest play counts and removes items that cannot be transcribed because they lack an audio URL.

### Nodes Involved
- Sort1
- Limit1
- Filter

### Node Details

#### 4) Sort1
- **Type and technical role:** `n8n-nodes-base.sort`; sorts incoming Instagram posts.
- **Configuration choices:**
  - Sorts by `videoPlayCount`
  - Order: descending
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from `Get dataset items1`; output to `Limit1`.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:**
  - Some dataset items may not contain `videoPlayCount`
  - Non-numeric or null values may affect ordering
  - Image posts or unsupported post formats may still appear before filtering
- **Sub-workflow reference:** None.

#### 5) Limit1
- **Type and technical role:** `n8n-nodes-base.limit`; restricts the number of items passed downstream.
- **Configuration choices:**
  - Maximum items: 10
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from `Sort1`; output to `Filter`.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:**
  - If fewer than 10 items are available, all are passed through
  - If the sorted data is poor quality, only those top 10 still move forward
- **Sub-workflow reference:** None.

#### 6) Filter
- **Type and technical role:** `n8n-nodes-base.code`; custom JavaScript filter to keep only posts with a non-empty audio URL.
- **Configuration choices:**
  - JavaScript:
    - Reads `item.json.audioUrl`
    - Keeps items only if `audioUrl` exists and is not blank after trimming
- **Key expressions or variables used:**
  - `item.json.audioUrl`
- **Input and output connections:** Input from `Limit1`; output to `HTTP Request`.
- **Version-specific requirements:** Code node type version 2.
- **Edge cases or potential failure types:**
  - Posts without `audioUrl` are silently discarded
  - If all items are filtered out, downstream nodes receive no data
  - If `audioUrl` is malformed but non-empty, it passes this filter and may fail later during download
- **Sub-workflow reference:** None.

---

## 2.3 Audio Download and Transcription

### Overview
This block downloads the reel audio files for the filtered posts, transcribes them with OpenAI, and combines all transcript text into one consolidated payload for script generation.

### Nodes Involved
- HTTP Request
- Transcribe a recording
- Combine Transcripts

### Node Details

#### 7) HTTP Request
- **Type and technical role:** `n8n-nodes-base.httpRequest`; downloads the audio file from the reel’s `audioUrl`.
- **Configuration choices:**
  - URL: `{{$json.audioUrl}}`
  - Response format: file
- **Key expressions or variables used:**
  - `{{$json.audioUrl}}`
- **Input and output connections:** Input from `Filter`; output to `Transcribe a recording`.
- **Version-specific requirements:** Type version 4.4.
- **Edge cases or potential failure types:**
  - Invalid or expired audio URL
  - File download timeout
  - HTTP 403/404 from remote host
  - Binary handling issues if the server returns unexpected content type
- **Sub-workflow reference:** None.

#### 8) Transcribe a recording
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`; submits audio binary to OpenAI transcription.
- **Configuration choices:**
  - Resource: `audio`
  - Operation: `transcribe`
  - Minimal options configured; defaults are relied on
- **Key expressions or variables used:** No visible custom expression in the node configuration.
- **Input and output connections:** Input from `HTTP Request`; output to `Combine Transcripts`.
- **Version-specific requirements:** Type version 2.1.
- **Edge cases or potential failure types:**
  - OpenAI credential issues
  - Unsupported audio format or corrupted binary
  - Rate limits or transcription API errors
  - Large audio files may exceed supported limits
  - If the binary property name expected by the node differs from the downloaded binary name, transcription can fail depending on node defaults
- **Sub-workflow reference:** None.

#### 9) Combine Transcripts
- **Type and technical role:** `n8n-nodes-base.code`; aggregates multiple transcription results into one text block.
- **Configuration choices:**
  - Maps over all incoming items
  - Extracts `item.json.text`
  - Removes falsy values
  - Joins transcripts with double line breaks
  - Returns a single item:
    - `{ transcripts: "..." }`
- **Key expressions or variables used:**
  - `item.json.text`
- **Input and output connections:** Input from `Transcribe a recording`; output to `Message a model1`.
- **Version-specific requirements:** Code node type version 2.
- **Edge cases or potential failure types:**
  - If transcription output uses a different field name than `text`, the combined result will be empty
  - Empty transcript list still produces one item with an empty `transcripts` string
  - Very large combined transcript text may increase token usage or exceed model limits downstream
- **Sub-workflow reference:** None.

---

## 2.4 AI Script Generation

### Overview
This block uses Anthropic Claude to analyze the combined transcript corpus and produce 7 new video scripts. The prompt strongly constrains the output format to a JSON array so it can be machine-parsed later.

### Nodes Involved
- Message a model1

### Node Details

#### 10) Message a model1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.anthropic`; sends a prompt to Claude and retrieves generated content.
- **Configuration choices:**
  - Model: `claude-sonnet-4-5-20250929`
  - `maxTokens`: 3000
  - Single message prompt includes:
    - Role framing as TikTok content strategist and copywriter
    - Combined transcripts inserted via `{{$json.transcripts}}`
    - Requirement to generate 7 scripts
    - Required fields: `hook`, `problem`, `solution`, `how_to_implement`
    - Strict instruction to return only valid JSON array
- **Key expressions or variables used:**
  - `{{$json.transcripts}}`
- **Input and output connections:** Input from `Combine Transcripts`; output to `Format AI Output1`.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:**
  - Anthropic credential issues
  - Invalid or unavailable model ID depending on account access or node version
  - Claude may still wrap JSON in markdown fences despite instructions
  - Large transcript input may exceed context/token limits
  - Output may be malformed JSON, causing the next node to fail
- **Sub-workflow reference:** None.

---

## 2.5 Output Formatting and Persistence

### Overview
This block converts Claude’s raw response into normalized items and writes each generated script into Google Sheets. It is the final persistence stage of the workflow.

### Nodes Involved
- Format AI Output1
- Append row in sheet1

### Node Details

#### 11) Format AI Output1
- **Type and technical role:** `n8n-nodes-base.code`; cleans and parses Claude’s response, then splits it into one item per script.
- **Configuration choices:**
  - Reads raw text from `content[0].text`
  - Removes markdown code fences such as ```json
  - Parses the remaining string using `JSON.parse`
  - Returns one item per script with fields:
    - `script_number`
    - `hook`
    - `problem`
    - `solution`
    - `how_to_implement`
  - Missing fields are defaulted to empty strings
- **Key expressions or variables used:**
  - `$json.content[0].text`
  - `JSON.parse(cleaned)`
- **Input and output connections:** Input from `Message a model1`; output to `Append row in sheet1`.
- **Version-specific requirements:** Code node type version 2.
- **Edge cases or potential failure types:**
  - `content[0].text` may not exist if the model response structure changes
  - Invalid JSON causes a hard failure at `JSON.parse`
  - If Claude returns an object instead of an array, mapping fails
  - If the response contains commentary outside the JSON, cleanup may be insufficient
- **Sub-workflow reference:** None.

#### 12) Append row in sheet1
- **Type and technical role:** `n8n-nodes-base.googleSheets`; appends one row per generated script into a specific worksheet.
- **Configuration choices:**
  - Operation: `append`
  - Spreadsheet document selected explicitly
  - Worksheet selected explicitly
  - Column mapping:
    - `Script Number` ← `{{$json.script_number}}`
    - `Hook` ← `{{$json.hook}}`
    - `Problem` ← `{{$json.problem}}`
    - `Solution` ← `{{$json.solution}}`
    - `How To Implement` ← `{{$json.how_to_implement}}`
    - `CreatedAt` ← `{{ new Date().toISOString() }}`
    - `Source` ← `"Instagram"`
  - Mapping mode: define fields manually
- **Key expressions or variables used:**
  - `{{$json.script_number}}`
  - `{{$json.hook}}`
  - `{{$json.problem}}`
  - `{{$json.solution}}`
  - `{{$json.how_to_implement}}`
  - `{{ new Date().toISOString() }}`
- **Input and output connections:** Input from `Format AI Output1`; no downstream node.
- **Version-specific requirements:** Type version 4.7.
- **Edge cases or potential failure types:**
  - Google OAuth credentials invalid or expired
  - Missing worksheet or renamed columns
  - Permission denied on the spreadsheet
  - Data type mismatch is unlikely here because conversion is disabled and values are mostly strings
  - Appends are non-idempotent; rerunning creates duplicate rows
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual workflow start |  | Run an Actor1 | # Instagram Top Reels to Google Sheets Script Engine<br>## HOW IT WORKS:<br>This workflow helps you study what is already working on Instagram and turn those patterns into new reel scripts.<br>It pulls posts from an Instagram profile through Apify, ranks them by video play count, filters for reels with audio, downloads the audio, and transcribes it with OpenAI.<br>It then sends the strongest transcript examples to Claude to generate 7 new short form scripts based on the winning patterns.<br>Finally, it formats the scripts into an existing Google Sheets with clear sections for Hook, Problem, Solution, and How to Implement, so the output is easy to review and reuse.<br>## HOW TO SET UP:<br>Add your Apify token, OpenAI API key, Anthropic API key, and Google Docs credential in the relevant nodes.<br>Replace the example Instagram profile URL in the Apify node with the profile you want to analyze.<br>Create these columns in your Google Sheet before running the workflow:<br>Script Number<br>Hook<br>Problem<br>Solution<br>How To Implement<br>CreatedAt<br>Source<br>In the Google Sheets node, choose the spreadsheet and worksheet where you want the output to go.<br>Run the workflow, and it will append 7 newly generated scripts to your sheet each time. |
| Run an Actor1 | Apify | Run Instagram Scraper actor | When clicking ‘Execute workflow’ | Get dataset items1 | # Instagram Top Reels to Google Sheets Script Engine<br>## HOW IT WORKS:<br>This workflow helps you study what is already working on Instagram and turn those patterns into new reel scripts.<br>It pulls posts from an Instagram profile through Apify, ranks them by video play count, filters for reels with audio, downloads the audio, and transcribes it with OpenAI.<br>It then sends the strongest transcript examples to Claude to generate 7 new short form scripts based on the winning patterns.<br>Finally, it formats the scripts into an existing Google Sheets with clear sections for Hook, Problem, Solution, and How to Implement, so the output is easy to review and reuse.<br>## HOW TO SET UP:<br>Add your Apify token, OpenAI API key, Anthropic API key, and Google Docs credential in the relevant nodes.<br>Replace the example Instagram profile URL in the Apify node with the profile you want to analyze.<br>Create these columns in your Google Sheet before running the workflow:<br>Script Number<br>Hook<br>Problem<br>Solution<br>How To Implement<br>CreatedAt<br>Source<br>In the Google Sheets node, choose the spreadsheet and worksheet where you want the output to go.<br>Run the workflow, and it will append 7 newly generated scripts to your sheet each time.<br>## Input and retrieval |
| Get dataset items1 | Apify | Retrieve scraped Instagram dataset items | Run an Actor1 | Sort1 | # Instagram Top Reels to Google Sheets Script Engine<br>## HOW IT WORKS:<br>This workflow helps you study what is already working on Instagram and turn those patterns into new reel scripts.<br>It pulls posts from an Instagram profile through Apify, ranks them by video play count, filters for reels with audio, downloads the audio, and transcribes it with OpenAI.<br>It then sends the strongest transcript examples to Claude to generate 7 new short form scripts based on the winning patterns.<br>Finally, it formats the scripts into an existing Google Sheets with clear sections for Hook, Problem, Solution, and How to Implement, so the output is easy to review and reuse.<br>## HOW TO SET UP:<br>Add your Apify token, OpenAI API key, Anthropic API key, and Google Docs credential in the relevant nodes.<br>Replace the example Instagram profile URL in the Apify node with the profile you want to analyze.<br>Create these columns in your Google Sheet before running the workflow:<br>Script Number<br>Hook<br>Problem<br>Solution<br>How To Implement<br>CreatedAt<br>Source<br>In the Google Sheets node, choose the spreadsheet and worksheet where you want the output to go.<br>Run the workflow, and it will append 7 newly generated scripts to your sheet each time.<br>## Input and retrieval |
| Sort1 | Sort | Sort posts by play count descending | Get dataset items1 | Limit1 | # Instagram Top Reels to Google Sheets Script Engine<br>## HOW IT WORKS:<br>This workflow helps you study what is already working on Instagram and turn those patterns into new reel scripts.<br>It pulls posts from an Instagram profile through Apify, ranks them by video play count, filters for reels with audio, downloads the audio, and transcribes it with OpenAI.<br>It then sends the strongest transcript examples to Claude to generate 7 new short form scripts based on the winning patterns.<br>Finally, it formats the scripts into an existing Google Sheets with clear sections for Hook, Problem, Solution, and How to Implement, so the output is easy to review and reuse.<br>## HOW TO SET UP:<br>Add your Apify token, OpenAI API key, Anthropic API key, and Google Docs credential in the relevant nodes.<br>Replace the example Instagram profile URL in the Apify node with the profile you want to analyze.<br>Create these columns in your Google Sheet before running the workflow:<br>Script Number<br>Hook<br>Problem<br>Solution<br>How To Implement<br>CreatedAt<br>Source<br>In the Google Sheets node, choose the spreadsheet and worksheet where you want the output to go.<br>Run the workflow, and it will append 7 newly generated scripts to your sheet each time.<br>## Sort and filter input |
| Limit1 | Limit | Keep only top 10 posts | Sort1 | Filter | # Instagram Top Reels to Google Sheets Script Engine<br>## HOW IT WORKS:<br>This workflow helps you study what is already working on Instagram and turn those patterns into new reel scripts.<br>It pulls posts from an Instagram profile through Apify, ranks them by video play count, filters for reels with audio, downloads the audio, and transcribes it with OpenAI.<br>It then sends the strongest transcript examples to Claude to generate 7 new short form scripts based on the winning patterns.<br>Finally, it formats the scripts into an existing Google Sheets with clear sections for Hook, Problem, Solution, and How to Implement, so the output is easy to review and reuse.<br>## HOW TO SET UP:<br>Add your Apify token, OpenAI API key, Anthropic API key, and Google Docs credential in the relevant nodes.<br>Replace the example Instagram profile URL in the Apify node with the profile you want to analyze.<br>Create these columns in your Google Sheet before running the workflow:<br>Script Number<br>Hook<br>Problem<br>Solution<br>How To Implement<br>CreatedAt<br>Source<br>In the Google Sheets node, choose the spreadsheet and worksheet where you want the output to go.<br>Run the workflow, and it will append 7 newly generated scripts to your sheet each time.<br>## Sort and filter input |
| Filter | Code | Remove items without audio URLs | Limit1 | HTTP Request | # Instagram Top Reels to Google Sheets Script Engine<br>## HOW IT WORKS:<br>This workflow helps you study what is already working on Instagram and turn those patterns into new reel scripts.<br>It pulls posts from an Instagram profile through Apify, ranks them by video play count, filters for reels with audio, downloads the audio, and transcribes it with OpenAI.<br>It then sends the strongest transcript examples to Claude to generate 7 new short form scripts based on the winning patterns.<br>Finally, it formats the scripts into an existing Google Sheets with clear sections for Hook, Problem, Solution, and How to Implement, so the output is easy to review and reuse.<br>## HOW TO SET UP:<br>Add your Apify token, OpenAI API key, Anthropic API key, and Google Docs credential in the relevant nodes.<br>Replace the example Instagram profile URL in the Apify node with the profile you want to analyze.<br>Create these columns in your Google Sheet before running the workflow:<br>Script Number<br>Hook<br>Problem<br>Solution<br>How To Implement<br>CreatedAt<br>Source<br>In the Google Sheets node, choose the spreadsheet and worksheet where you want the output to go.<br>Run the workflow, and it will append 7 newly generated scripts to your sheet each time.<br>## Sort and filter input |
| HTTP Request | HTTP Request | Download audio file from Instagram reel | Filter | Transcribe a recording | # Instagram Top Reels to Google Sheets Script Engine<br>## HOW IT WORKS:<br>This workflow helps you study what is already working on Instagram and turn those patterns into new reel scripts.<br>It pulls posts from an Instagram profile through Apify, ranks them by video play count, filters for reels with audio, downloads the audio, and transcribes it with OpenAI.<br>It then sends the strongest transcript examples to Claude to generate 7 new short form scripts based on the winning patterns.<br>Finally, it formats the scripts into an existing Google Sheets with clear sections for Hook, Problem, Solution, and How to Implement, so the output is easy to review and reuse.<br>## HOW TO SET UP:<br>Add your Apify token, OpenAI API key, Anthropic API key, and Google Docs credential in the relevant nodes.<br>Replace the example Instagram profile URL in the Apify node with the profile you want to analyze.<br>Create these columns in your Google Sheet before running the workflow:<br>Script Number<br>Hook<br>Problem<br>Solution<br>How To Implement<br>CreatedAt<br>Source<br>In the Google Sheets node, choose the spreadsheet and worksheet where you want the output to go.<br>Run the workflow, and it will append 7 newly generated scripts to your sheet each time.<br>## Transcribe best performing scripts |
| Transcribe a recording | OpenAI | Transcribe reel audio | HTTP Request | Combine Transcripts | # Instagram Top Reels to Google Sheets Script Engine<br>## HOW IT WORKS:<br>This workflow helps you study what is already working on Instagram and turn those patterns into new reel scripts.<br>It pulls posts from an Instagram profile through Apify, ranks them by video play count, filters for reels with audio, downloads the audio, and transcribes it with OpenAI.<br>It then sends the strongest transcript examples to Claude to generate 7 new short form scripts based on the winning patterns.<br>Finally, it formats the scripts into an existing Google Sheets with clear sections for Hook, Problem, Solution, and How to Implement, so the output is easy to review and reuse.<br>## HOW TO SET UP:<br>Add your Apify token, OpenAI API key, Anthropic API key, and Google Docs credential in the relevant nodes.<br>Replace the example Instagram profile URL in the Apify node with the profile you want to analyze.<br>Create these columns in your Google Sheet before running the workflow:<br>Script Number<br>Hook<br>Problem<br>Solution<br>How To Implement<br>CreatedAt<br>Source<br>In the Google Sheets node, choose the spreadsheet and worksheet where you want the output to go.<br>Run the workflow, and it will append 7 newly generated scripts to your sheet each time.<br>## Transcribe best performing scripts |
| Combine Transcripts | Code | Merge all transcript texts into one prompt input | Transcribe a recording | Message a model1 | # Instagram Top Reels to Google Sheets Script Engine<br>## HOW IT WORKS:<br>This workflow helps you study what is already working on Instagram and turn those patterns into new reel scripts.<br>It pulls posts from an Instagram profile through Apify, ranks them by video play count, filters for reels with audio, downloads the audio, and transcribes it with OpenAI.<br>It then sends the strongest transcript examples to Claude to generate 7 new short form scripts based on the winning patterns.<br>Finally, it formats the scripts into an existing Google Sheets with clear sections for Hook, Problem, Solution, and How to Implement, so the output is easy to review and reuse.<br>## HOW TO SET UP:<br>Add your Apify token, OpenAI API key, Anthropic API key, and Google Docs credential in the relevant nodes.<br>Replace the example Instagram profile URL in the Apify node with the profile you want to analyze.<br>Create these columns in your Google Sheet before running the workflow:<br>Script Number<br>Hook<br>Problem<br>Solution<br>How To Implement<br>CreatedAt<br>Source<br>In the Google Sheets node, choose the spreadsheet and worksheet where you want the output to go.<br>Run the workflow, and it will append 7 newly generated scripts to your sheet each time.<br>## Transcribe best performing scripts |
| Message a model1 | Anthropic | Generate 7 new scripts from transcript patterns | Combine Transcripts | Format AI Output1 | # Instagram Top Reels to Google Sheets Script Engine<br>## HOW IT WORKS:<br>This workflow helps you study what is already working on Instagram and turn those patterns into new reel scripts.<br>It pulls posts from an Instagram profile through Apify, ranks them by video play count, filters for reels with audio, downloads the audio, and transcribes it with OpenAI.<br>It then sends the strongest transcript examples to Claude to generate 7 new short form scripts based on the winning patterns.<br>Finally, it formats the scripts into an existing Google Sheets with clear sections for Hook, Problem, Solution, and How to Implement, so the output is easy to review and reuse.<br>## HOW TO SET UP:<br>Add your Apify token, OpenAI API key, Anthropic API key, and Google Docs credential in the relevant nodes.<br>Replace the example Instagram profile URL in the Apify node with the profile you want to analyze.<br>Create these columns in your Google Sheet before running the workflow:<br>Script Number<br>Hook<br>Problem<br>Solution<br>How To Implement<br>CreatedAt<br>Source<br>In the Google Sheets node, choose the spreadsheet and worksheet where you want the output to go.<br>Run the workflow, and it will append 7 newly generated scripts to your sheet each time.<br>## Generate new scripts and save them in Google Sheets |
| Format AI Output1 | Code | Clean Claude output and split into one item per script | Message a model1 | Append row in sheet1 | # Instagram Top Reels to Google Sheets Script Engine<br>## HOW IT WORKS:<br>This workflow helps you study what is already working on Instagram and turn those patterns into new reel scripts.<br>It pulls posts from an Instagram profile through Apify, ranks them by video play count, filters for reels with audio, downloads the audio, and transcribes it with OpenAI.<br>It then sends the strongest transcript examples to Claude to generate 7 new short form scripts based on the winning patterns.<br>Finally, it formats the scripts into an existing Google Sheets with clear sections for Hook, Problem, Solution, and How to Implement, so the output is easy to review and reuse.<br>## HOW TO SET UP:<br>Add your Apify token, OpenAI API key, Anthropic API key, and Google Docs credential in the relevant nodes.<br>Replace the example Instagram profile URL in the Apify node with the profile you want to analyze.<br>Create these columns in your Google Sheet before running the workflow:<br>Script Number<br>Hook<br>Problem<br>Solution<br>How To Implement<br>CreatedAt<br>Source<br>In the Google Sheets node, choose the spreadsheet and worksheet where you want the output to go.<br>Run the workflow, and it will append 7 newly generated scripts to your sheet each time.<br>## Generate new scripts and save them in Google Sheets |
| Append row in sheet1 | Google Sheets | Append generated scripts to spreadsheet | Format AI Output1 |  | # Instagram Top Reels to Google Sheets Script Engine<br>## HOW IT WORKS:<br>This workflow helps you study what is already working on Instagram and turn those patterns into new reel scripts.<br>It pulls posts from an Instagram profile through Apify, ranks them by video play count, filters for reels with audio, downloads the audio, and transcribes it with OpenAI.<br>It then sends the strongest transcript examples to Claude to generate 7 new short form scripts based on the winning patterns.<br>Finally, it formats the scripts into an existing Google Sheets with clear sections for Hook, Problem, Solution, and How to Implement, so the output is easy to review and reuse.<br>## HOW TO SET UP:<br>Add your Apify token, OpenAI API key, Anthropic API key, and Google Docs credential in the relevant nodes.<br>Replace the example Instagram profile URL in the Apify node with the profile you want to analyze.<br>Create these columns in your Google Sheet before running the workflow:<br>Script Number<br>Hook<br>Problem<br>Solution<br>How To Implement<br>CreatedAt<br>Source<br>In the Google Sheets node, choose the spreadsheet and worksheet where you want the output to go.<br>Run the workflow, and it will append 7 newly generated scripts to your sheet each time.<br>## Generate new scripts and save them in Google Sheets |
| Sticky Note | Sticky Note | Canvas documentation note |  |  | # Instagram Top Reels to Google Sheets Script Engine<br>## HOW IT WORKS:<br>This workflow helps you study what is already working on Instagram and turn those patterns into new reel scripts.<br>It pulls posts from an Instagram profile through Apify, ranks them by video play count, filters for reels with audio, downloads the audio, and transcribes it with OpenAI.<br>It then sends the strongest transcript examples to Claude to generate 7 new short form scripts based on the winning patterns.<br>Finally, it formats the scripts into an existing Google Sheets with clear sections for Hook, Problem, Solution, and How to Implement, so the output is easy to review and reuse.<br>## HOW TO SET UP:<br>Add your Apify token, OpenAI API key, Anthropic API key, and Google Docs credential in the relevant nodes.<br>Replace the example Instagram profile URL in the Apify node with the profile you want to analyze.<br>Create these columns in your Google Sheet before running the workflow:<br>Script Number<br>Hook<br>Problem<br>Solution<br>How To Implement<br>CreatedAt<br>Source<br>In the Google Sheets node, choose the spreadsheet and worksheet where you want the output to go.<br>Run the workflow, and it will append 7 newly generated scripts to your sheet each time. |
| Sticky Note1 | Sticky Note | Section label |  |  | ## Input and retrieval |
| Sticky Note2 | Sticky Note | Section label |  |  | ## Sort and filter input |
| Sticky Note3 | Sticky Note | Section label |  |  | ## Transcribe best performing scripts |
| Sticky Note4 | Sticky Note | Section label |  |  | ## Generate new scripts and save them in Google Sheets |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - In n8n, create a blank workflow.
   - Name it something like: `Top IG Reels to 7 New Scripts`.

2. **Add a Manual Trigger node**
   - Node type: `Manual Trigger`
   - Keep default settings.
   - This will be the entry point for manual execution and testing.

3. **Add an Apify node to run the Instagram scraper**
   - Node type: `Apify`
   - Rename it to `Run an Actor1`
   - Configure credentials with your **Apify API token**
   - Choose the actor: **Instagram Scraper (apify/instagram-scraper)**
   - In the custom input body, define:
     - `directUrls`: array with the Instagram profile URL to analyze
     - `resultsLimit`: `30`
     - `resultsType`: `posts`
   - Replace the placeholder profile URL with a real public Instagram profile URL.
   - Connect:
     - `Manual Trigger` → `Run an Actor1`

4. **Add an Apify node to fetch dataset items**
   - Node type: `Apify`
   - Rename it to `Get dataset items1`
   - Resource: `Datasets`
   - Dataset ID expression:
     - `{{$json.defaultDatasetId}}`
   - Use the same Apify credentials.
   - Connect:
     - `Run an Actor1` → `Get dataset items1`

5. **Add a Sort node**
   - Node type: `Sort`
   - Rename it to `Sort1`
   - Sort field:
     - Field name: `videoPlayCount`
     - Order: `Descending`
   - Connect:
     - `Get dataset items1` → `Sort1`

6. **Add a Limit node**
   - Node type: `Limit`
   - Rename it to `Limit1`
   - Max items: `10`
   - Connect:
     - `Sort1` → `Limit1`

7. **Add a Code node to filter posts with audio**
   - Node type: `Code`
   - Rename it to `Filter`
   - Use this logic:
     - Keep only items where `audioUrl` exists and is not empty
   - Equivalent code:
     ```javascript
     return items.filter(item => {
       const audioUrl = item.json.audioUrl;
       return audioUrl && audioUrl.trim() !== "";
     });
     ```
   - Connect:
     - `Limit1` → `Filter`

8. **Add an HTTP Request node to download audio**
   - Node type: `HTTP Request`
   - Rename it to `HTTP Request`
   - URL expression:
     - `{{$json.audioUrl}}`
   - Response format:
     - `File`
   - Keep binary output enabled through the response-as-file option.
   - Connect:
     - `Filter` → `HTTP Request`

9. **Add an OpenAI node for transcription**
   - Node type: `OpenAI` from the LangChain/OpenAI integration
   - Rename it to `Transcribe a recording`
   - Configure credentials with your **OpenAI API key**
   - Resource: `Audio`
   - Operation: `Transcribe`
   - Leave advanced options at default unless your account or model requires specific settings.
   - Ensure the node consumes the binary file output from the previous HTTP node.
   - Connect:
     - `HTTP Request` → `Transcribe a recording`

10. **Add a Code node to combine transcripts**
    - Node type: `Code`
    - Rename it to `Combine Transcripts`
    - Use code that aggregates `text` fields from all transcription results into one string:
      ```javascript
      const transcripts = items
        .map(item => item.json.text)
        .filter(Boolean)
        .join('\n\n');

      return [
        {
          json: {
            transcripts
          }
        }
      ];
      ```
    - Connect:
      - `Transcribe a recording` → `Combine Transcripts`

11. **Add an Anthropic node for script generation**
    - Node type: `Anthropic`
    - Rename it to `Message a model1`
    - Configure credentials with your **Anthropic API key**
    - Model:
      - Use `claude-sonnet-4-5-20250929` if available in your environment
      - If unavailable, select a current Claude Sonnet model and keep behavior equivalent
    - Set `maxTokens` to `3000`
    - Add one message containing the generation prompt.
    - Insert the transcript variable with:
      - `{{$json.transcripts}}`
    - Prompt requirements:
      - Analyze high-performing transcripts
      - Generate 7 new viral video scripts
      - Use fields:
        - `hook`
        - `problem`
        - `solution`
        - `how_to_implement`
      - Return only a valid JSON array
    - Connect:
      - `Combine Transcripts` → `Message a model1`

12. **Add a Code node to parse AI output**
    - Node type: `Code`
    - Rename it to `Format AI Output1`
    - Use code that:
      - Reads Claude’s text from `content[0].text`
      - Removes markdown fences
      - Parses JSON
      - Emits one item per script
    - Equivalent code:
      ```javascript
      const raw = $json.content[0].text;

      const cleaned = raw
        .replace(/```json/g, '')
        .replace(/```/g, '')
        .trim();

      const parsed = JSON.parse(cleaned);

      return parsed.map((item, index) => ({
        json: {
          script_number: index + 1,
          hook: item.hook || "",
          problem: item.problem || "",
          solution: item.solution || "",
          how_to_implement: item.how_to_implement || ""
        }
      }));
      ```
    - Connect:
      - `Message a model1` → `Format AI Output1`

13. **Prepare the Google Sheet**
    - Create a spreadsheet in Google Sheets.
    - Add a worksheet for output.
    - Create these columns in the header row:
      - `Script Number`
      - `Hook`
      - `Problem`
      - `Solution`
      - `How To Implement`
      - `CreatedAt`
      - `Source`

14. **Add a Google Sheets node**
    - Node type: `Google Sheets`
    - Rename it to `Append row in sheet1`
    - Configure credentials with **Google Sheets OAuth2**
    - Operation: `Append`
    - Select the target spreadsheet and worksheet
    - Use manual field mapping with these values:
      - `Script Number` → `{{$json.script_number}}`
      - `Hook` → `{{$json.hook}}`
      - `Problem` → `{{$json.problem}}`
      - `Solution` → `{{$json.solution}}`
      - `How To Implement` → `{{$json.how_to_implement}}`
      - `CreatedAt` → `{{ new Date().toISOString() }}`
      - `Source` → `Instagram`
    - Connect:
      - `Format AI Output1` → `Append row in sheet1`

15. **Validate end-to-end execution**
    - Run the workflow manually.
    - Confirm:
      - Apify returns Instagram posts
      - Posts contain `videoPlayCount` and `audioUrl`
      - Audio files download successfully
      - OpenAI returns transcript text under `text`
      - Claude returns valid JSON
      - Google Sheets appends 7 rows

16. **Recommended hardening improvements**
    - Add error handling around:
      - Empty Apify dataset
      - No items with `audioUrl`
      - JSON parse failure in Claude output
      - Google Sheets duplicates on reruns
    - Optionally add:
      - A Set node for configurable profile URL
      - A Schedule Trigger for recurring runs
      - A deduplication layer in Sheets or n8n

### Credential Configuration Summary
- **Apify account**
  - Required for both Apify nodes
  - Must have access to run the Instagram Scraper actor and read datasets
- **OpenAI account**
  - Required for transcription
  - Must support audio transcription
- **Anthropic account**
  - Required for Claude generation
  - Must have access to the selected Claude model
- **Google Sheets account**
  - Required to append rows to the spreadsheet
  - Spreadsheet must be accessible by the authenticated Google account

### Sub-workflow Setup
- No sub-workflows are used in this workflow.
- There are no Execute Workflow nodes and no separate child workflow dependencies.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow is designed as a manual content research and script generation pipeline from Instagram reels to Google Sheets. | General project note |
| Before running, replace the placeholder Instagram profile URL in the Apify actor input. | Apify input setup |
| The Google Sheet must contain the columns: Script Number, Hook, Problem, Solution, How To Implement, CreatedAt, Source. | Google Sheets output requirements |
| The workflow appends new rows on every execution and does not prevent duplicates by default. | Operational note |
| The workflow relies on public/legal data and standard API integrations: Apify, OpenAI, Anthropic, and Google Sheets. | Compliance/architecture note |