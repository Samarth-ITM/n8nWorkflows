Generate Upwork proposals with GPT-4o-mini, Airtable and Slack

https://n8nworkflows.xyz/workflows/generate-upwork-proposals-with-gpt-4o-mini--airtable-and-slack-13870


# Generate Upwork proposals with GPT-4o-mini, Airtable and Slack

# 1. Workflow Overview

This workflow monitors an Upwork job feed delivered through a Vollna RSS URL, filters jobs based on your skills and client quality, avoids duplicates via Airtable, generates a tailored proposal with GPT-4o-mini, stores the result in Airtable, and sends a Slack notification.

It is designed for freelancers or agencies who want to identify relevant Upwork opportunities quickly and prepare draft proposals automatically before manual review and submission.

## 1.1 Input Reception and RSS Parsing
The workflow starts from an RSS trigger that polls a Vollna feed every minute. It then parses each feed item into structured job data such as title, description, budget, rating, client spend, derived job ID, and decoded Upwork URL.

## 1.2 Relevance Filtering and Deduplication
Parsed jobs are scored against a hardcoded skill list, filtered by minimum skill match count, filtered again by client rating, then checked in Airtable to see whether the job was already processed.

## 1.3 AI Proposal Generation
For jobs that pass filtering and deduplication, the workflow constructs a structured OpenAI prompt, sends it to GPT-4o-mini, and extracts the final proposal text from the model response.

## 1.4 Persistence and Notification
The generated proposal plus job metadata are saved into Airtable, then formatted into a Slack message and posted to a target Slack channel.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and RSS Parsing

### Overview
This block ingests RSS entries from Vollna and transforms semi-structured feed content into normalized fields the rest of the workflow can use. It is critical because downstream filters and AI prompting depend on accurate extraction.

### Nodes Involved
- RSS Feed - n8n & Automation
- Extract Details from RSS

### Node Details

#### RSS Feed - n8n & Automation
- **Type and role:** `n8n-nodes-base.rssFeedReadTrigger`  
  Trigger node that polls an RSS feed and emits new items.
- **Configuration choices:**
  - Feed URL is set to `YOUR_VOLLNA_RSS_FEED_URL`
  - Polling is configured for every minute
- **Key expressions or variables used:**
  - None
- **Input and output connections:**
  - No input, as this is a trigger
  - Outputs to `Extract Details from RSS`
- **Version-specific requirements:**
  - Uses `typeVersion: 1`
- **Edge cases / failures:**
  - Invalid or unreachable RSS URL
  - Vollna feed authentication or availability issues
  - Duplicate or delayed RSS items depending on source feed behavior
  - Polling every minute may be excessive for some environments if rate-limited
- **Sub-workflow reference:**
  - None

#### Extract Details from RSS
- **Type and role:** `n8n-nodes-base.code`  
  Parses the RSS item and extracts normalized job fields.
- **Configuration choices:**
  - Reads the current item with `$input.item.json`
  - Uses helper functions to:
    - decode the Upwork URL from the Vollna tracking link
    - extract budget
    - extract skills
    - extract client rating
    - extract client total spent
    - derive a job ID
  - Builds a clean output object for later stages
- **Key expressions or variables used:**
  - `item.contentSnippet || item.content || item.summary || ''`
  - `item.guid || item.link || ''`
  - `generateJobId(item.link || item.url || guid)`
  - `extractUpworkUrl(guid)`
- **Output fields produced:**
  - `jobTitle`
  - `jobUrl`
  - `jobDescription`
  - `postedAt`
  - `budget`
  - `skillsRequired`
  - `clientRating`
  - `clientSpent`
  - `jobId`
  - `rawGuid`
  - `upworkUrl`
- **Input and output connections:**
  - Input from `RSS Feed - n8n & Automation`
  - Output to `Filter: Skills Match`
- **Version-specific requirements:**
  - Uses Code node `typeVersion: 2`
- **Edge cases / failures:**
  - RSS content format may differ from expected regex structure
  - URL decoding may fail, in which case a string like `decode error: ...` is returned
  - If the Vollna GUID format changes, `jobId` extraction may become unreliable
  - Missing `contentSnippet`, `content`, `summary`, `guid`, or `link` fields can reduce parsing quality
  - Rating and spend can return `null`, which is handled later
- **Sub-workflow reference:**
  - None

---

## 2.2 Relevance Filtering and Deduplication

### Overview
This block keeps only jobs that are likely worth acting on. It filters by skill alignment, filters out low-rated clients, and checks Airtable to avoid reprocessing previously captured jobs.

### Nodes Involved
- Filter: Skills Match
- Filter: Client Rating
- Airtable: Check Duplicate
- Is New Job?

### Node Details

#### Filter: Skills Match
- **Type and role:** `n8n-nodes-base.code`  
  Compares each job against a hardcoded list of your target skills and computes a simple match score.
- **Configuration choices:**
  - Defines `YOUR_SKILLS` as a JavaScript array inside the node
  - Concatenates title, description, and skills into one lowercase text blob
  - Counts how many configured skills appear in that text
  - Drops the item if fewer than 2 skills match
- **Key expressions or variables used:**
  - `YOUR_SKILLS`
  - `const text = \`${item.jobTitle} ${item.jobDescription} ${item.skillsRequired}\`.toLowerCase();`
  - `matchedSkills`
  - `matchScore`
- **Input and output connections:**
  - Input from `Extract Details from RSS`
  - Output to `Filter: Client Rating`
- **Version-specific requirements:**
  - Code node `typeVersion: 2`
- **Edge cases / failures:**
  - Skill matching is substring-based, so false positives are possible
  - Case normalization is handled, but synonyms are not
  - If `YOUR_SKILLS` is not customized, filtering may be too broad or too narrow
  - If text fields are empty, match score will be artificially low
- **Sub-workflow reference:**
  - None

#### Filter: Client Rating
- **Type and role:** `n8n-nodes-base.code`  
  Filters out jobs from clients with poor ratings while allowing unrated clients through.
- **Configuration choices:**
  - Reads `clientRating`
  - Accepts the job if:
    - `clientRating` is `null`, or
    - `clientRating >= 4.5`
  - Drops the item otherwise
- **Key expressions or variables used:**
  - `const rating = item.clientRating;`
  - `const ratingOk = rating === null || rating >= 4.5;`
- **Input and output connections:**
  - Input from `Filter: Skills Match`
  - Output to `Airtable: Check Duplicate`
- **Version-specific requirements:**
  - Code node `typeVersion: 2`
- **Edge cases / failures:**
  - Missing or unparsable ratings become `null` and pass the filter
  - This may allow low-quality clients if parsing fails upstream
- **Sub-workflow reference:**
  - None

#### Airtable: Check Duplicate
- **Type and role:** `n8n-nodes-base.airtable`  
  Searches Airtable for an existing record with the same Job ID.
- **Configuration choices:**
  - Operation: `search`
  - Base: `YOUR_AIRTABLE_BASE_ID`
  - Table: `YOUR_AIRTABLE_TABLE_ID`
  - Filter formula: `={Job ID}="{{ $json.jobId }}"`
  - `alwaysOutputData: true` is enabled so the workflow continues even when no record is found
- **Key expressions or variables used:**
  - `{{ $json.jobId }}`
- **Input and output connections:**
  - Input from `Filter: Client Rating`
  - Output to `Is New Job?`
- **Version-specific requirements:**
  - Airtable node `typeVersion: 2`
  - Requires Airtable personal access token or supported API credential in n8n
- **Edge cases / failures:**
  - Invalid base/table IDs
  - Airtable authentication errors
  - Formula issues if `jobId` contains special characters
  - Search behavior depends on exact field naming: `Job ID` must exist
  - Because search returns Airtable-style records, downstream logic assumes absent `id` means no duplicate
- **Sub-workflow reference:**
  - None

#### Is New Job?
- **Type and role:** `n8n-nodes-base.if`  
  Branches only when the Airtable search did not return an existing record.
- **Configuration choices:**
  - Condition checks that `{{ $json.id }}` does **not exist**
  - In practice:
    - If Airtable found a record, `id` exists and the condition is false
    - If Airtable found nothing, `id` is absent and the condition is true
- **Key expressions or variables used:**
  - `={{ $json.id }}`
- **Input and output connections:**
  - Input from `Airtable: Check Duplicate`
  - True output goes to `Build OpenAI Payload`
  - False branch is not connected, so duplicates are silently dropped
- **Version-specific requirements:**
  - IF node `typeVersion: 2`
- **Edge cases / failures:**
  - If Airtable returns a structure different from expected, duplicate detection may fail
  - If `alwaysOutputData` behavior changes or search returns empty objects with `id`, this condition would need revisiting
- **Sub-workflow reference:**
  - None

---

## 2.3 AI Proposal Generation

### Overview
This block constructs a high-context prompt using job metadata and your freelancer profile, sends it to GPT-4o-mini, and normalizes the AI response into a plain proposal field.

### Nodes Involved
- Build OpenAI Payload
- AI: Generate Proposal
- Extract Proposal Text

### Node Details

#### Build OpenAI Payload
- **Type and role:** `n8n-nodes-base.code`  
  Builds the prompt payload for OpenAI using the filtered job data and a hardcoded personal profile section.
- **Configuration choices:**
  - Pulls the current approved job from `$('Filter: Client Rating').first().json`
  - Creates a user prompt containing:
    - job title
    - description
    - budget
    - required skills
    - matched skills
    - a placeholder freelancer profile
  - Creates a system prompt instructing concise, personalized proposal writing
  - Packages both into `openAiPayload`
- **Key expressions or variables used:**
  - `$('Filter: Client Rating').first().json`
  - `openAiPayload.messages[0].content`
  - `openAiPayload.messages[1].content`
  - Placeholder fields:
    - `[YOUR NAME]`
    - `[YOUR SKILLS]`
    - `[YOUR EXPERIENCE]`
- **Input and output connections:**
  - Input from `Is New Job?`
  - Output to `AI: Generate Proposal`
- **Version-specific requirements:**
  - Code node `typeVersion: 2`
- **Edge cases / failures:**
  - If the referenced node name changes, the expression breaks
  - If multiple items are processed concurrently, using `.first()` from another node can create data-coupling issues
  - If placeholders are not replaced, proposal quality will be poor
- **Sub-workflow reference:**
  - None

#### AI: Generate Proposal
- **Type and role:** `@n8n/n8n-nodes-langchain.openAi`  
  Sends the built prompt to OpenAI using GPT-4o-mini.
- **Configuration choices:**
  - Model: `gpt-4o-mini`
  - `maxTokens: 600`
  - Responses are defined explicitly:
    - system role from `openAiPayload.messages[0].content`
    - user role from `openAiPayload.messages[1].content`
  - No built-in tools are used
- **Key expressions or variables used:**
  - `={{ $json.openAiPayload.messages[0].content }}`
  - `={{ $json.openAiPayload.messages[1].content }}`
- **Input and output connections:**
  - Input from `Build OpenAI Payload`
  - Output to `Extract Proposal Text`
- **Version-specific requirements:**
  - OpenAI LangChain node `typeVersion: 2.1`
  - Requires compatible OpenAI credentials configured in n8n
- **Edge cases / failures:**
  - Missing or invalid OpenAI credentials
  - Model availability or quota issues
  - Token limit problems if source descriptions are too large
  - Response structure may differ across node versions or provider updates
- **Sub-workflow reference:**
  - None

#### Extract Proposal Text
- **Type and role:** `n8n-nodes-base.code`  
  Extracts plain proposal content from the OpenAI node output and merges it back with the job data.
- **Configuration choices:**
  - Attempts several response paths:
    - `input.message?.content`
    - `input.text`
    - `input.choices?.[0]?.message?.content`
    - fallback to `JSON.stringify(input)`
  - Retrieves original job data again from `$('Filter: Client Rating').first().json`
  - Writes proposal into `generatedProposal`
- **Key expressions or variables used:**
  - `input.message?.content`
  - `input.text`
  - `input.choices?.[0]?.message?.content`
  - `$('Filter: Client Rating').first().json`
- **Input and output connections:**
  - Input from `AI: Generate Proposal`
  - Output to `Airtable: Save Proposal`
- **Version-specific requirements:**
  - Code node `typeVersion: 2`
- **Edge cases / failures:**
  - Same cross-node `.first()` coupling risk as above
  - If OpenAI returns a different structure, fallback becomes raw JSON text
  - That fallback can still be saved and later displayed awkwardly in Slack
- **Sub-workflow reference:**
  - None

---

## 2.4 Persistence and Notification

### Overview
This block writes the enriched job and generated proposal to Airtable, formats a Slack-ready summary, and posts it to a channel for immediate review.

### Nodes Involved
- Airtable: Save Proposal
- Build Slack Message
- Slack Notification

### Node Details

#### Airtable: Save Proposal
- **Type and role:** `n8n-nodes-base.airtable`  
  Creates a new Airtable record containing the job metadata and AI-generated proposal.
- **Configuration choices:**
  - Operation: `create`
  - Base: `YOUR_AIRTABLE_BASE_ID`
  - Table: `YOUR_AIRTABLE_TABLE_ID`
  - Explicit field mapping is used
  - Mapped fields include:
    - Budget
    - Job ID
    - Job URL
    - Job Title
    - Posted At
    - upwork_url
    - AI Proposal
    - Match Score
    - Client Rating
    - Matched Skills
    - Skills Required
    - Client Total Spent
  - `Matched Skills` is converted to a comma-separated string
- **Key expressions or variables used:**
  - `={{ $json.matchedSkills.join(', ') }}`
  - Standard field expressions such as `={{ $json.jobTitle }}`
- **Input and output connections:**
  - Input from `Extract Proposal Text`
  - Output to `Build Slack Message`
- **Version-specific requirements:**
  - Airtable node `typeVersion: 2`
  - Airtable table schema should match configured field names
- **Edge cases / failures:**
  - Schema mismatch between configured fields and actual Airtable columns
  - Type conversion issues, especially for dates and numbers
  - `matchedSkills.join(', ')` fails if `matchedSkills` is not an array
  - Record creation can fail on permission issues or invalid field types
- **Sub-workflow reference:**
  - None

#### Build Slack Message
- **Type and role:** `n8n-nodes-base.code`  
  Converts the Airtable create result into a formatted Slack message.
- **Configuration choices:**
  - Reads created record fields from `input.fields`
  - Tries to parse `AI Proposal` as JSON and extract `parsed.output[0].content[0].text`
  - Falls back to raw `AI Proposal` text if parsing fails
  - Builds a rich multiline Slack message with job summary and proposal text
- **Key expressions or variables used:**
  - `const job = input.fields;`
  - `JSON.parse(job['AI Proposal'])`
  - `job['Job Title']`, `job['Budget']`, `job['Match Score']`, etc.
- **Input and output connections:**
  - Input from `Airtable: Save Proposal`
  - Output to `Slack Notification`
- **Version-specific requirements:**
  - Code node `typeVersion: 2`
- **Edge cases / failures:**
  - Airtable output must contain `fields`
  - If `AI Proposal` is large, Slack message may become too long
  - Markdown rendering may not behave as expected if values contain special characters
- **Sub-workflow reference:**
  - None

#### Slack Notification
- **Type and role:** `n8n-nodes-base.slack`  
  Posts the final message to a Slack channel.
- **Configuration choices:**
  - Sends `={{ $json.slackMessage }}`
  - Uses `select: channel`
  - Channel ID is `YOUR_SLACK_CHANNEL_ID`
- **Key expressions or variables used:**
  - `={{ $json.slackMessage }}`
- **Input and output connections:**
  - Input from `Build Slack Message`
  - No downstream node
- **Version-specific requirements:**
  - Slack node `typeVersion: 2.4`
  - Requires valid Slack credentials with permission to post to the target channel
- **Edge cases / failures:**
  - Bot not invited to the channel
  - Missing `chat:write` scope
  - Invalid channel ID
  - Slack message length limits
- **Sub-workflow reference:**
  - None

---

## 2.5 Documentation and In-Canvas Notes

### Overview
These nodes are informational only. They document setup expectations and visually label the workflow blocks in the editor.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3

### Node Details

#### Sticky Note
- **Type and role:** `n8n-nodes-base.stickyNote`  
  General workflow explanation and setup guidance.
- **Configuration choices:**
  - Large note covering the overall workflow purpose and setup steps
- **Input and output connections:**
  - None
- **Version-specific requirements:**
  - `typeVersion: 1`
- **Edge cases / failures:**
  - None at runtime; documentation only
- **Sub-workflow reference:**
  - None

#### Sticky Note3
- **Type and role:** `n8n-nodes-base.stickyNote`  
  Labels the ingest and parsing area.
- **Configuration choices:**
  - Describes polling and field extraction
- **Input and output connections:**
  - None
- **Version-specific requirements:**
  - `typeVersion: 1`
- **Edge cases / failures:**
  - None
- **Sub-workflow reference:**
  - None

#### Sticky Note1
- **Type and role:** `n8n-nodes-base.stickyNote`  
  Labels the filtering and deduplication area.
- **Configuration choices:**
  - Describes the 2+ skill match rule, client rating filter, and duplicate skip logic
- **Input and output connections:**
  - None
- **Version-specific requirements:**
  - `typeVersion: 1`
- **Edge cases / failures:**
  - None
- **Sub-workflow reference:**
  - None

#### Sticky Note2
- **Type and role:** `n8n-nodes-base.stickyNote`  
  Labels the proposal generation, storage, and Slack notification area.
- **Configuration choices:**
  - Describes GPT generation, Airtable save, and Slack send
- **Input and output connections:**
  - None
- **Version-specific requirements:**
  - `typeVersion: 1`
- **Edge cases / failures:**
  - None
- **Sub-workflow reference:**
  - None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| RSS Feed - n8n & Automation | RSS Feed Read Trigger | Poll Vollna RSS feed every minute |  | Extract Details from RSS | ## How it works\nThis workflow monitors Upwork job listings every minute via a Vollna RSS feed. Each new job is parsed to extract the title, description, budget, skills, and a clean Upwork URL. Jobs are scored against your skill list — only those matching 2 or more of your skills pass through. Low-rated clients are filtered out. Each job is checked against Airtable to avoid processing duplicates.\nFor every new qualifying job, GPT-4o-mini writes a personalised 150–250 word proposal referencing details from the actual job post. The proposal and all job data are saved to Airtable with a status of "New". A Slack message is sent instantly with the job details, matched skills, and the full proposal ready to copy and submit.\nSetup steps\n1. Vollna — Sign up at [vollna.com](https://vollna.com), create a job filter for your skills, and copy the RSS feed URL into the RSS trigger node\n2. OpenAI — Add your API key as an n8n credential and connect it to the AI node\n3. Airtable — Create a base using the schema in the README, then add your Base ID and Table ID to both Airtable nodes\n4. Slack — Create a Slack app with `chat:write` scope, invite it to your channel, and connect it in the Slack node\n5. Customise — Update YOUR_SKILLS in the Filter node and update MY PROFILE in the Build OpenAI Payload node with your actual experience\n📡 **Ingest & Parse**\nPolls Vollna RSS every minute. Extracts job title, description, budget, skills, and clean Upwork URL from each item. |
| Extract Details from RSS | Code | Normalize RSS item into structured job data | RSS Feed - n8n & Automation | Filter: Skills Match | ## How it works\nThis workflow monitors Upwork job listings every minute via a Vollna RSS feed. Each new job is parsed to extract the title, description, budget, skills, and a clean Upwork URL. Jobs are scored against your skill list — only those matching 2 or more of your skills pass through. Low-rated clients are filtered out. Each job is checked against Airtable to avoid processing duplicates.\nFor every new qualifying job, GPT-4o-mini writes a personalised 150–250 word proposal referencing details from the actual job post. The proposal and all job data are saved to Airtable with a status of "New". A Slack message is sent instantly with the job details, matched skills, and the full proposal ready to copy and submit.\nSetup steps\n1. Vollna — Sign up at [vollna.com](https://vollna.com), create a job filter for your skills, and copy the RSS feed URL into the RSS trigger node\n2. OpenAI — Add your API key as an n8n credential and connect it to the AI node\n3. Airtable — Create a base using the schema in the README, then add your Base ID and Table ID to both Airtable nodes\n4. Slack — Create a Slack app with `chat:write` scope, invite it to your channel, and connect it in the Slack node\n5. Customise — Update YOUR_SKILLS in the Filter node and update MY PROFILE in the Build OpenAI Payload node with your actual experience\n📡 **Ingest & Parse**\nPolls Vollna RSS every minute. Extracts job title, description, budget, skills, and clean Upwork URL from each item. |
| Filter: Skills Match | Code | Score job relevance by skill keyword matches | Extract Details from RSS | Filter: Client Rating | 🎯 **Filter & Deduplicate**\nNeeds 2+ skill matches. Skips low-rated clients and already-processed jobs. |
| Filter: Client Rating | Code | Reject low-rated clients, allow unrated or high-rated ones | Filter: Skills Match | Airtable: Check Duplicate | 🎯 **Filter & Deduplicate**\nNeeds 2+ skill matches. Skips low-rated clients and already-processed jobs. |
| Airtable: Check Duplicate | Airtable | Search Airtable for existing Job ID | Filter: Client Rating | Is New Job? | 🎯 **Filter & Deduplicate**\nNeeds 2+ skill matches. Skips low-rated clients and already-processed jobs. |
| Is New Job? | If | Continue only when Airtable search found no prior record | Airtable: Check Duplicate | Build OpenAI Payload | 🎯 **Filter & Deduplicate**\nNeeds 2+ skill matches. Skips low-rated clients and already-processed jobs. |
| Build OpenAI Payload | Code | Build model prompt and structured OpenAI request content | Is New Job? | AI: Generate Proposal | 🤖 **Generate, Save & Notify**\nGPT-4o-mini writes a tailored proposal. Saves to Airtable, sends to Slack. |
| AI: Generate Proposal | OpenAI | Generate proposal text with GPT-4o-mini | Build OpenAI Payload | Extract Proposal Text | 🤖 **Generate, Save & Notify**\nGPT-4o-mini writes a tailored proposal. Saves to Airtable, sends to Slack. |
| Extract Proposal Text | Code | Normalize OpenAI response into `generatedProposal` | AI: Generate Proposal | Airtable: Save Proposal | 🤖 **Generate, Save & Notify**\nGPT-4o-mini writes a tailored proposal. Saves to Airtable, sends to Slack. |
| Airtable: Save Proposal | Airtable | Create Airtable record with job data and AI proposal | Extract Proposal Text | Build Slack Message | 🤖 **Generate, Save & Notify**\nGPT-4o-mini writes a tailored proposal. Saves to Airtable, sends to Slack. |
| Build Slack Message | Code | Format Airtable record into Slack-ready text | Airtable: Save Proposal | Slack Notification | 🤖 **Generate, Save & Notify**\nGPT-4o-mini writes a tailored proposal. Saves to Airtable, sends to Slack. |
| Slack Notification | Slack | Post job summary and proposal to Slack | Build Slack Message |  | 🤖 **Generate, Save & Notify**\nGPT-4o-mini writes a tailored proposal. Saves to Airtable, sends to Slack. |
| Sticky Note | Sticky Note | Overall documentation and setup instructions |  |  |  |
| Sticky Note3 | Sticky Note | Visual label for ingest and parsing block |  |  |  |
| Sticky Note1 | Sticky Note | Visual label for filter and deduplication block |  |  |  |
| Sticky Note2 | Sticky Note | Visual label for generation, save, and notify block |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Automate proposals on Upwork with AI, Airtable and Slack`.

2. **Add the RSS trigger**
   - Create node: **RSS Feed Read Trigger**
   - Name it: `RSS Feed - n8n & Automation`
   - Set the feed URL to your Vollna RSS feed
   - Set polling to **every minute**
   - This will be the workflow entry point

3. **Add the parsing code node**
   - Create node: **Code**
   - Name it: `Extract Details from RSS`
   - Connect `RSS Feed - n8n & Automation` → `Extract Details from RSS`
   - Paste logic that:
     - reads the RSS item
     - extracts:
       - title
       - description
       - posting date
       - budget
       - skills
       - client rating
       - client total spent
       - decoded Upwork URL
       - stable job ID
   - Ensure the node outputs these fields:
     - `jobTitle`
     - `jobUrl`
     - `jobDescription`
     - `postedAt`
     - `budget`
     - `skillsRequired`
     - `clientRating`
     - `clientSpent`
     - `jobId`
     - `rawGuid`
     - `upworkUrl`

4. **Add the skill filter**
   - Create node: **Code**
   - Name it: `Filter: Skills Match`
   - Connect `Extract Details from RSS` → `Filter: Skills Match`
   - Add a JavaScript array called `YOUR_SKILLS`
   - Include your target keywords, for example:
     - `n8n`
     - `automation`
     - `workflow`
     - `zapier`
     - `make.com`
     - `openai`
     - `python`
     - `javascript`
   - Build a lowercase searchable text from:
     - `jobTitle`
     - `jobDescription`
     - `skillsRequired`
   - Count matching skills
   - Return no items if match count is below 2
   - Otherwise output:
     - original fields
     - `matchScore`
     - `matchedSkills`

5. **Add the client rating filter**
   - Create node: **Code**
   - Name it: `Filter: Client Rating`
   - Connect `Filter: Skills Match` → `Filter: Client Rating`
   - Logic:
     - accept job if `clientRating` is null
     - or if `clientRating >= 4.5`
     - otherwise return no items

6. **Prepare Airtable**
   - Create an Airtable base, for example `Leads CRM`
   - Create a table, for example `Upwork_jobs`
   - Add these fields:
     - `Job Title` text
     - `Job URL` text
     - `upwork_url` text
     - `Posted At` date/time
     - `Budget` text
     - `Skills Required` text
     - `Matched Skills` text
     - `Match Score` number
     - `Client Rating` text
     - `Client Total Spent` text
     - `AI Proposal` long text
     - `Status` text
     - `Job ID` text
     - `Notes` text

7. **Create Airtable credentials in n8n**
   - Add an Airtable credential, typically with a Personal Access Token
   - Make sure the token can read and write the target base/table

8. **Add duplicate check node**
   - Create node: **Airtable**
   - Name it: `Airtable: Check Duplicate`
   - Connect `Filter: Client Rating` → `Airtable: Check Duplicate`
   - Operation: **Search**
   - Select your Airtable credential
   - Choose your base ID
   - Choose your table ID
   - Set formula:
     - `={Job ID}="{{ $json.jobId }}"`
   - Enable **Always Output Data**

9. **Add IF node for new jobs**
   - Create node: **If**
   - Name it: `Is New Job?`
   - Connect `Airtable: Check Duplicate` → `Is New Job?`
   - Configure condition:
     - left value: `={{ $json.id }}`
     - operator: **does not exist**
   - Use the **true** output for new jobs
   - Leave the false branch unconnected if you want duplicates silently ignored

10. **Add prompt-building node**
    - Create node: **Code**
    - Name it: `Build OpenAI Payload`
    - Connect the **true** branch of `Is New Job?` → `Build OpenAI Payload`
    - Build a prompt object called `openAiPayload`
    - Include:
      - model name `gpt-4o-mini`
      - max tokens `600`
      - system message with proposal-writing instructions
      - user message with:
        - job title
        - job description
        - budget
        - required skills
        - matched skills
        - your freelancer profile
    - Replace placeholders with your real details:
      - `[YOUR NAME]`
      - `[YOUR SKILLS]`
      - `[YOUR EXPERIENCE]`

11. **Create OpenAI credentials in n8n**
    - Add your OpenAI API credential in n8n
    - Ensure your account has access to `gpt-4o-mini`

12. **Add OpenAI generation node**
    - Create node: **OpenAI** from the LangChain/OpenAI integration
    - Name it: `AI: Generate Proposal`
    - Connect `Build OpenAI Payload` → `AI: Generate Proposal`
    - Choose model: `gpt-4o-mini`
    - Set max tokens to `600`
    - Map the response messages:
      - system content from `={{ $json.openAiPayload.messages[0].content }}`
      - user content from `={{ $json.openAiPayload.messages[1].content }}`
    - No tools are needed

13. **Add proposal extraction node**
    - Create node: **Code**
    - Name it: `Extract Proposal Text`
    - Connect `AI: Generate Proposal` → `Extract Proposal Text`
    - Logic should:
      - inspect the OpenAI output
      - try common response locations such as:
        - `message.content`
        - `text`
        - `choices[0].message.content`
      - if none exists, stringify the full response
      - merge back the original job data
      - output `generatedProposal`

14. **Add Airtable create node**
    - Create node: **Airtable**
    - Name it: `Airtable: Save Proposal`
    - Connect `Extract Proposal Text` → `Airtable: Save Proposal`
    - Operation: **Create**
    - Select the same base and table
    - Use explicit field mapping
    - Map:
      - `Budget` ← `budget`
      - `Job ID` ← `jobId`
      - `Job URL` ← `jobUrl`
      - `Job Title` ← `jobTitle`
      - `Posted At` ← `postedAt`
      - `upwork_url` ← `upworkUrl`
      - `AI Proposal` ← `generatedProposal`
      - `Match Score` ← `matchScore`
      - `Client Rating` ← `clientRating`
      - `Matched Skills` ← joined `matchedSkills`
      - `Skills Required` ← `skillsRequired`
      - `Client Total Spent` ← `clientSpent`
   - Optional but recommended:
      - also map `Status` to `"New"`

15. **Create Slack credentials in n8n**
    - Create a Slack app
    - Grant at least `chat:write`
    - Install the app to your workspace
    - Invite the bot/app to the target channel
    - Add Slack credentials in n8n

16. **Add Slack message builder**
    - Create node: **Code**
    - Name it: `Build Slack Message`
    - Connect `Airtable: Save Proposal` → `Build Slack Message`
    - Build a formatted message including:
      - job title
      - budget
      - match score
      - matched skills
      - posted date
      - required skills
      - job link
      - AI-generated proposal
      - Airtable confirmation / job ID
    - Read values from Airtable response `fields`

17. **Add Slack send node**
    - Create node: **Slack**
    - Name it: `Slack Notification`
    - Connect `Build Slack Message` → `Slack Notification`
    - Configure:
      - send message text from `={{ $json.slackMessage }}`
      - choose **channel**
      - set channel ID to your target Slack channel

18. **Add documentation sticky notes**
   - Add a general sticky note explaining:
     - Vollna feed polling
     - filtering
     - Airtable deduplication
     - OpenAI proposal generation
     - Slack notification
   - Add one sticky note over the input area:
     - `📡 Ingest & Parse`
   - Add one over the filtering area:
     - `🎯 Filter & Deduplicate`
   - Add one over the generation area:
     - `🤖 Generate, Save & Notify`

19. **Test each stage**
   - Test RSS parsing with sample feed items
   - Verify `jobId` stability
   - Confirm duplicate detection works by re-running the same item
   - Confirm OpenAI output structure matches your extraction code
   - Confirm Airtable record creation
   - Confirm Slack delivery

20. **Activate the workflow**
   - Once all credentials and IDs are configured, activate the workflow
   - Monitor first runs closely for parsing mismatches or field mapping errors

## Important implementation notes
1. The current design uses cross-node lookups like `$('Filter: Client Rating').first().json` in AI-related code nodes.
2. This works, but it is less robust than passing data item-to-item directly.
3. If you expect batches or concurrent items, prefer carrying job data forward in each node rather than re-reading `.first()` from earlier nodes.

## Required credentials
- **OpenAI credential** for the OpenAI node
- **Airtable credential** for both Airtable nodes
- **Slack credential** for the Slack node

## Required external services
- Vollna account with an RSS job feed
- Airtable base and table
- Slack workspace and target channel

## Expected input/output behavior
- **Input:** RSS items from Vollna containing Upwork job metadata
- **Output:** Airtable record plus Slack message for every new, relevant, non-duplicate job

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Vollna is the RSS source used to monitor Upwork jobs. | [https://vollna.com](https://vollna.com) |
| The workflow description says the proposal and job data are saved with a status of "New", but the current Airtable create node does not actually map the `Status` field. If needed, add a static value in `Airtable: Save Proposal`. | Implementation note |
| The Airtable setup note mentions creating a base using the schema in the README, but no separate document is included in the provided workflow JSON. The schema can be inferred from the Airtable field mappings in the workflow. | Documentation consistency note |
| The workflow is currently inactive (`active: false`). | Runtime status |
| Polling every minute gives fast detection but may create unnecessary executions if your feed is noisy or rate-limited. | Operational note |