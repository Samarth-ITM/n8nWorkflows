Publish AI-written newsletters and LinkedIn posts from WordPress with Gemini, Google Sheets and Gmail

https://n8nworkflows.xyz/workflows/publish-ai-written-newsletters-and-linkedin-posts-from-wordpress-with-gemini--google-sheets-and-gmail-14525


# Publish AI-written newsletters and LinkedIn posts from WordPress with Gemini, Google Sheets and Gmail

# 1. Workflow Overview

This workflow automatically turns the latest WordPress blog post into two distribution formats:

- an AI-written email newsletter sent to a subscriber list via Gmail
- an AI-written LinkedIn post published for an organization account

It also prevents duplicate processing by storing the last processed WordPress post ID in Google Sheets.

## 1.1 Trigger and Blog Retrieval

The workflow starts on a schedule, then calls the WordPress REST API to fetch the latest post.

## 1.2 Duplicate Detection and Tracking

The latest post ID is compared against a stored `LastProcessedID` value in Google Sheets. If the post is newer, the sheet is updated immediately so the same article is not processed again on the next run.

## 1.3 Content Preparation and AI Generation

The blog content is cleaned in a Code node, then passed to a LangChain AI Agent using Google Gemini. The AI is expected to generate structured output containing:

- `subject_line`
- `newsletter_body`
- `linkedin_teaser`

## 1.4 Output Formatting

The AI response is parsed into usable JSON and enriched with source metadata such as the original article link and featured image URL.

## 1.5 Distribution

The final content is distributed in two paths:

- one LinkedIn post is published
- the subscriber list is read from Google Sheets, then each contact receives an HTML email through Gmail using a loop

---

# 2. Block-by-Block Analysis

## 2.1 Trigger and Blog Retrieval

### Overview

This block launches the workflow on a recurring schedule and fetches the newest WordPress post using the WordPress REST API. It is the primary entry point of the workflow.

### Nodes Involved

- Schedule Trigger
- HTTP Request

### Node Details

#### Schedule Trigger

- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Starts the workflow automatically based on a configured interval.
- **Configuration choices:**  
  The node uses a simple interval rule. In the exported JSON, the interval object is present but not detailed, so the exact frequency should be verified in the editor after import.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  No input. Outputs to `HTTP Request`.
- **Version-specific requirements:** `typeVersion: 1.3`
- **Edge cases or potential failure types:**  
  - Trigger may not run if workflow is inactive.
  - Misconfigured schedule interval may produce unexpected frequency.
- **Sub-workflow reference:** None.

#### HTTP Request

- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the WordPress REST API to fetch the most recent blog post.
- **Configuration choices:**  
  - URL: `https://vaguefoundation.com/blog/wp-json/wp/v2/posts?_embed&per_page=1`
  - Response format is JSON
  - `_embed` is used so featured media can be accessed later
  - `per_page=1` restricts the result to the latest post
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  Input from `Schedule Trigger`; output to `Last ID`.
- **Version-specific requirements:** `typeVersion: 4.3`
- **Edge cases or potential failure types:**  
  - WordPress API unavailable or rate-limited
  - Non-JSON response
  - Empty response if no posts exist
  - Array response shape may require validation depending on WordPress API behavior
- **Sub-workflow reference:** None.

---

## 2.2 Duplicate Detection and Tracking

### Overview

This block reads the stored last processed post ID from Google Sheets, compares it with the newly fetched WordPress post ID, and updates the sheet if the post is new.

### Nodes Involved

- Last ID
- If
- Update Last ID

### Node Details

#### Last ID

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads tracking data from Google Sheets.
- **Configuration choices:**  
  - Document: `YOUR_GOOGLE_SHEETS_DOCUMENT_ID`
  - Sheet: `Sheet2`
  - Intended to hold tracking metadata, including `LastProcessedID`
- **Key expressions or variables used:** None directly in node parameters.
- **Input and output connections:**  
  Input from `HTTP Request`; output to `If`.
- **Version-specific requirements:** `typeVersion: 4.7`
- **Edge cases or potential failure types:**  
  - Missing Google Sheets credentials
  - Sheet name or document ID mismatch
  - Empty sheet or multiple rows returned unexpectedly
  - `LastProcessedID` missing or stored as text that does not compare cleanly as a number
- **Sub-workflow reference:** None.

#### If

- **Type and technical role:** `n8n-nodes-base.if`  
  Compares the stored ID and the latest WordPress ID to determine whether the post is new.
- **Configuration choices:**  
  - Condition: `{{$json.LastProcessedID}} < {{ $('Fetch Blog').item.json.id }}`
- **Important note:**  
  The expression references a node named `Fetch Blog`, but no such node exists in the workflow. The actual WordPress node is named `HTTP Request`. This is a configuration error and must be corrected.
- **Key expressions or variables used:**  
  - `{{$json.LastProcessedID}}`
  - `{{ $('Fetch Blog').item.json.id }}`
- **Input and output connections:**  
  Input from `Last ID`; true output to `Update Last ID`. No false branch is connected.
- **Version-specific requirements:** `typeVersion: 2.3`
- **Edge cases or potential failure types:**  
  - Expression failure due to nonexistent node reference
  - Numeric comparison issues if values are strings
  - If no tracking row exists, the condition may fail
- **Sub-workflow reference:** None.

#### Update Last ID

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Updates the tracking row so the current post will not be processed again.
- **Configuration choices:**  
  - Operation: `update`
  - Matching column: `row_number`
  - Updates:
    - `row_number = {{$json.row_number}}`
    - `LastProcessedID = {{ $('Fetch Blog').item.json.id }}`
- **Important note:**  
  This node also references `Fetch Blog`, which does not exist. It should reference the WordPress fetch node instead.
- **Key expressions or variables used:**  
  - `{{$json.row_number}}`
  - `{{ $('Fetch Blog').item.json.id }}`
- **Input and output connections:**  
  Input from `If`; output to `data cleanse`.
- **Version-specific requirements:** `typeVersion: 4.7`
- **Edge cases or potential failure types:**  
  - Expression failure due to wrong node name
  - Update fails if `row_number` is absent
  - Sheet permissions or schema mismatch
- **Sub-workflow reference:** None.

---

## 2.3 Content Preparation and AI Generation

### Overview

This block cleans the WordPress content, extracts relevant article metadata, and sends the result to a Gemini-backed AI Agent to generate marketing-ready content.

### Nodes Involved

- data cleanse
- Google Gemini Chat Model
- AI Agent2

### Node Details

#### data cleanse

- **Type and technical role:** `n8n-nodes-base.code`  
  Transforms raw WordPress post data into a simplified payload for AI processing.
- **Configuration choices:**  
  The JavaScript:
  - reads from `$node["Fetch Blog"].json`
  - strips HTML tags from `content.rendered`
  - replaces HTML entities in the title
  - extracts the featured image URL from `_embedded['wp:featuredmedia'][0].source_url`
  - returns:
    - `id`
    - `title`
    - `content`
    - `link`
    - `image_url`
- **Important note:**  
  This code also references `Fetch Blog`, but the workflow contains `HTTP Request`. This mismatch must be fixed.
- **Key expressions or variables used:**  
  - `$node["Fetch Blog"].json`
- **Input and output connections:**  
  Input from `Update Last ID`; output to `AI Agent2`.
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases or potential failure types:**  
  - Undefined node reference
  - `content.rendered` missing
  - `_embedded` absent if WordPress does not return embedded media
  - HTML stripping may remove desired formatting
- **Sub-workflow reference:** None.

#### Google Gemini Chat Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
  Provides the language model used by the LangChain agent.
- **Configuration choices:**  
  No advanced options are set in the export. Default model behavior is used via connected credentials.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  AI model connection to `AI Agent2`.
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:**  
  - Missing Gemini credentials
  - API quota issues
  - Model output not following the expected JSON structure
- **Sub-workflow reference:** None.

#### AI Agent2

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Uses the cleaned article input and the Gemini model to generate newsletter and LinkedIn copy.
- **Configuration choices:**  
  No prompt or structured instruction is visible in the exported parameters. That means important behavior may depend on hidden editor configuration or may be incomplete in this export.
- **Expected output shape:**  
  The downstream parser expects JSON with:
  - `subject_line`
  - `newsletter_body`
  - `linkedin_teaser`
- **Key expressions or variables used:** None visible in export.
- **Input and output connections:**  
  Main input from `data cleanse`; main output to `Format Response`; AI language model input from `Google Gemini Chat Model`.
- **Version-specific requirements:** `typeVersion: 3`
- **Edge cases or potential failure types:**  
  - No explicit prompt may cause unusable output
  - Agent may return markdown code fences or plain text instead of JSON
  - Token/context limits if article content is too long
- **Sub-workflow reference:** None.

---

## 2.4 Output Formatting

### Overview

This block parses the AI response into a structured object that downstream email and LinkedIn nodes can use.

### Nodes Involved

- Format Response

### Node Details

#### Format Response

- **Type and technical role:** `n8n-nodes-base.code`  
  Parses the AI output and merges it with article metadata from the cleaned source node.
- **Configuration choices:**  
  The JavaScript:
  - reads AI output from `$input.item.json.output` or `$input.item.json.text`
  - removes markdown code fences such as ```json
  - parses the cleaned string as JSON
  - retrieves source data from `$node["data cleanse"].json`
  - returns:
    - `subject_line`
    - `newsletter_body`
    - `linkedin_teaser`
    - `image_url`
    - `link`
- **Bug in error handling:**  
  In the catch block, it returns `ai_raw: rawData`, but `rawData` is not defined. The intended variable is likely `rawAI`. If parsing fails, the node itself may throw a reference error instead of returning a clean error object.
- **Key expressions or variables used:**  
  - `$input.item.json.output`
  - `$input.item.json.text`
  - `$node["data cleanse"].json`
- **Input and output connections:**  
  Input from `AI Agent2`; outputs to `Email List` and `Create a post`.
- **Version-specific requirements:** `typeVersion: 2`
- **Edge cases or potential failure types:**  
  - Invalid AI JSON output
  - Missing `output` or `text`
  - Catch block bug due to undefined `rawData`
  - Missing `image_url` or `link`
- **Sub-workflow reference:** None.

---

## 2.5 LinkedIn Distribution

### Overview

This block publishes the AI-generated LinkedIn teaser as an organization post.

### Nodes Involved

- Create a post

### Node Details

#### Create a post

- **Type and technical role:** `n8n-nodes-base.linkedIn`  
  Publishes a LinkedIn post.
- **Configuration choices:**  
  - Text: `{{ $('Format Response').item.json.linkedin_teaser }}`
  - Posts as: `organization`
  - Share media category: `ARTICLE`
  - `onError` is set to continue to error output
- **Important limitation:**  
  Although the share media category is `ARTICLE`, no article URL/title/media fields are configured in the exported parameters. Depending on the LinkedIn node behavior, this may produce a plain text post or fail validation.
- **Key expressions or variables used:**  
  - `{{ $('Format Response').item.json.linkedin_teaser }}`
- **Input and output connections:**  
  Input from `Format Response`; no connected downstream node.
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:**  
  - Missing LinkedIn credentials
  - Organization posting permissions not granted
  - Required organization ID not configured in credentials or node settings
  - Post may fail if ARTICLE media requires URL metadata
- **Sub-workflow reference:** None.

---

## 2.6 Email Distribution

### Overview

This block reads subscriber emails from Google Sheets, loops over them one by one, and sends an HTML newsletter through Gmail.

### Nodes Involved

- Email List
- Loop Over Items
- Send Email

### Node Details

#### Email List

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Retrieves the list of email recipients.
- **Configuration choices:**  
  - Document: `YOUR_GOOGLE_SHEETS_DOCUMENT_ID`
  - Sheet: `gid=0` / `Sheet1`
  - Expected column includes something named `email list`
- **Key expressions or variables used:** None directly in parameters.
- **Input and output connections:**  
  Input from `Format Response`; output to `Loop Over Items`.
- **Version-specific requirements:** `typeVersion: 4.7`
- **Edge cases or potential failure types:**  
  - Wrong column name
  - Empty sheet
  - Large recipient list causing long workflow runtime
  - Credential or permission issues
- **Sub-workflow reference:** None.

#### Loop Over Items

- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Iterates over recipients one item at a time.
- **Configuration choices:**  
  Default batch behavior appears to be used.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  Input from `Email List` and from `Send Email` back into the loop. The second output goes to `Send Email`, creating the iteration cycle.
- **Version-specific requirements:** `typeVersion: 3`
- **Edge cases or potential failure types:**  
  - Infinite or broken loop if batch logic is misconfigured
  - Empty input list results in no emails sent
- **Sub-workflow reference:** None.

#### Send Email

- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends HTML newsletter emails via Gmail.
- **Configuration choices:**  
  - Recipient: `{{$json['email list']}}`
  - Subject: `Daily Newsletter : {{ $('Format Response').item.json.subject_line }}`
  - HTML body includes:
    - hidden preview text
    - featured image
    - subject line
    - newsletter body
    - CTA button linking to the original article
  - Attribution disabled
- **Key expressions or variables used:**  
  - `{{$json['email list']}}`
  - `{{ $('Format Response').item.json.subject_line }}`
  - `{{ $('Format Response').item.json.image_url }}`
  - `{{ $('Format Response').item.json.newsletter_body }}`
  - `{{ $('Format Response').item.json.link }}`
- **Input and output connections:**  
  Input from `Loop Over Items`; output back to `Loop Over Items`.
- **Version-specific requirements:** `typeVersion: 2.2`
- **Edge cases or potential failure types:**  
  - Gmail OAuth not configured
  - Invalid or empty recipient email address
  - HTML body may break visually if AI returns malformed HTML/text
  - Gmail sending quota limits
  - Embedded expressions fail if `Format Response` did not return expected fields
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Scheduled workflow entry point |  | HTTP Request | ## Step 1: Detect & Filter<br>Fetch latest blog post and compare ID with Google Sheets to avoid duplicate processing. |
| HTTP Request | n8n-nodes-base.httpRequest | Fetch latest WordPress post via REST API | Schedule Trigger | Last ID | ## Step 1: Detect & Filter<br>Fetch latest blog post and compare ID with Google Sheets to avoid duplicate processing. |
| Last ID | n8n-nodes-base.googleSheets | Read tracking row containing last processed post ID | HTTP Request | If | ## Step 1: Detect & Filter<br>Fetch latest blog post and compare ID with Google Sheets to avoid duplicate processing. |
| If | n8n-nodes-base.if | Check whether fetched post is newer than stored post ID | Last ID | Update Last ID | ## Step 1: Detect & Filter<br>Fetch latest blog post and compare ID with Google Sheets to avoid duplicate processing. |
| Update Last ID | n8n-nodes-base.googleSheets | Update stored last processed post ID in tracker sheet | If | data cleanse | ## Step 1: Detect & Filter<br>Fetch latest blog post and compare ID with Google Sheets to avoid duplicate processing. |
| data cleanse | n8n-nodes-base.code | Clean WordPress HTML content and extract metadata | Update Last ID | AI Agent2 | ## Step 2: AI Content Generation<br>Clean blog data and use Gemini AI to generate newsletter content and LinkedIn teaser. |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Language model used by the AI agent |  | AI Agent2 | ## Step 2: AI Content Generation<br>Clean blog data and use Gemini AI to generate newsletter content and LinkedIn teaser. |
| AI Agent2 | @n8n/n8n-nodes-langchain.agent | Generate newsletter and LinkedIn content from blog post | data cleanse; Google Gemini Chat Model | Format Response | ## Step 2: AI Content Generation<br>Clean blog data and use Gemini AI to generate newsletter content and LinkedIn teaser. |
| Format Response | n8n-nodes-base.code | Parse AI JSON output and prepare distribution payload | AI Agent2 | Email List; Create a post | ## Step 3: Distribution<br>Post to LinkedIn and send personalized emails to subscribers via Gmail. |
| Create a post | n8n-nodes-base.linkedIn | Publish LinkedIn organization post | Format Response |  | ## Step 3: Distribution<br>Post to LinkedIn and send personalized emails to subscribers via Gmail. |
| Email List | n8n-nodes-base.googleSheets | Read newsletter subscriber list | Format Response | Loop Over Items | ## Step 3: Distribution<br>Post to LinkedIn and send personalized emails to subscribers via Gmail. |
| Loop Over Items | n8n-nodes-base.splitInBatches | Iterate through recipients one at a time | Email List; Send Email | Send Email | ## Step 3: Distribution<br>Post to LinkedIn and send personalized emails to subscribers via Gmail. |
| Send Email | n8n-nodes-base.gmail | Send HTML newsletter email to each subscriber | Loop Over Items | Loop Over Items | ## Step 3: Distribution<br>Post to LinkedIn and send personalized emails to subscribers via Gmail. |
| Sticky Note | n8n-nodes-base.stickyNote | Workspace documentation and setup guidance |  |  | ## Blog to Social & Newsletter Automator<br><br>### How it works<br>This workflow automatically fetches the latest blog post from a WordPress API and checks whether it has already been processed using a Google Sheets tracker. If the post is new, the workflow cleans and formats the blog data, then sends it to an AI agent powered by Google Gemini to generate a newsletter and LinkedIn teaser.<br><br>The formatted output is then distributed across channels. A LinkedIn post is published automatically, and a list of subscribers is retrieved from Google Sheets. The workflow loops through each recipient and sends a personalized HTML email newsletter using Gmail.<br><br>### Setup steps<br>1. Connect your Google Sheets account for both tracking (LastProcessedID) and email subscriber list.<br>2. Add your Google Gemini API credentials to enable AI content generation.<br>3. Connect your LinkedIn account for automated posting.<br>4. Connect your Gmail account for sending newsletters.<br>5. Update the WordPress API endpoint in the HTTP Request node. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Section label for detection/filter block |  |  | ## Step 1: Detect & Filter<br>Fetch latest blog post and compare ID with Google Sheets to avoid duplicate processing. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Section label for AI generation block |  |  | ## Step 2: AI Content Generation<br>Clean blog data and use Gemini AI to generate newsletter content and LinkedIn teaser. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Section label for distribution block |  |  | ## Step 3: Distribution<br>Post to LinkedIn and send personalized emails to subscribers via Gmail. |

---

# 4. Reproducing the Workflow from Scratch

Below is the full rebuild sequence in n8n, including the corrections required to make the workflow functional.

## 1. Create the schedule trigger

1. Add a **Schedule Trigger** node.
2. Configure the run frequency as desired, such as:
   - every hour
   - every day
   - every 15 minutes
3. Activate the node as the workflow entry point.

## 2. Create the WordPress fetch node

4. Add an **HTTP Request** node named `HTTP Request`.
5. Connect `Schedule Trigger -> HTTP Request`.
6. Set:
   - **Method:** `GET`
   - **URL:** your WordPress REST endpoint, for example:  
     `https://your-site.com/wp-json/wp/v2/posts?_embed&per_page=1`
7. In response settings, set **Response Format** to `JSON`.
8. Keep `_embed` in the URL so featured image metadata is available.

## 3. Create the tracking sheet reader

9. Add a **Google Sheets** node named `Last ID`.
10. Connect `HTTP Request -> Last ID`.
11. Configure Google Sheets credentials.
12. Select the tracking spreadsheet.
13. Select a worksheet such as `Sheet2`.
14. Configure the node to read the row containing:
   - `row_number`
   - `LastProcessedID`

### Recommended tracking sheet structure

Create a sheet with one row like:

| row_number | LastProcessedID |
|---|---|
| 2 | 0 |

This makes the update step straightforward.

## 4. Create the duplicate check

15. Add an **If** node named `If`.
16. Connect `Last ID -> If`.
17. Set the condition to compare the stored ID against the fetched WordPress post ID.
18. Use a number comparison:
   - **Left Value:** `={{ Number($json.LastProcessedID) }}`
   - **Operator:** `is less than`
   - **Right Value:** `={{ Number($('HTTP Request').item.json.id || $('HTTP Request').first().json.id) }}`

### Important correction

The original workflow references `Fetch Blog`, but the actual node name is `HTTP Request`. Use `HTTP Request` unless you rename the node.

### Extra caution

If your HTTP node returns an array instead of a single item, normalize it first or use a Code node before this step.

## 5. Create the tracking sheet updater

19. Add a **Google Sheets** node named `Update Last ID`.
20. Connect the **true** output of `If -> Update Last ID`.
21. Set the operation to **Update**.
22. Use the same spreadsheet and sheet as `Last ID`.
23. Configure matching on `row_number`.
24. Map:
   - `row_number = {{$json.row_number}}`
   - `LastProcessedID = {{ $('HTTP Request').item.json.id || $('HTTP Request').first().json.id }}`

## 6. Create the content-cleaning code node

25. Add a **Code** node named `data cleanse`.
26. Connect `Update Last ID -> data cleanse`.
27. Use JavaScript similar to this corrected version:

```javascript
const raw = $node["HTTP Request"].json;

const cleanContent = (raw.content?.rendered || '')
  .replace(/<[^>]*>?/gm, '')
  .trim();

const cleanTitle = (raw.title?.rendered || '')
  .replace(/&#8217;/g, "'")
  .replace(/&#8211;/g, "-");

const imageUrl = raw._embedded?.['wp:featuredmedia']?.[0]?.source_url || "";

return {
  id: raw.id,
  title: cleanTitle,
  content: cleanContent,
  link: raw.link || "",
  image_url: imageUrl
};
```

### Important correction

The original code referenced `$node["Fetch Blog"]`. Replace it with `$node["HTTP Request"]` or rename your HTTP node to match.

## 7. Create the Gemini model node

28. Add a **Google Gemini Chat Model** node.
29. Configure Gemini credentials.
30. Leave default options unless you need a specific Gemini model or temperature.

## 8. Create the AI agent

31. Add an **AI Agent** node named `AI Agent2`.
32. Connect `data cleanse -> AI Agent2`.
33. Connect `Google Gemini Chat Model -> AI Agent2` through the AI language model port.
34. Configure the agent prompt so it returns valid JSON only.

### Recommended prompt

Use instructions like:

```text
You are given a blog post title, body, link, and image URL.

Create:
1. a compelling email subject line
2. a concise newsletter body in HTML-friendly format
3. a professional LinkedIn teaser

Return ONLY valid JSON with this structure:
{
  "subject_line": "...",
  "newsletter_body": "...",
  "linkedin_teaser": "..."
}
```

### Input expectation

The agent should receive the cleaned output from `data cleanse`, especially:
- `title`
- `content`
- `link`
- `image_url`

## 9. Create the AI response formatter

35. Add a **Code** node named `Format Response`.
36. Connect `AI Agent2 -> Format Response`.
37. Use a corrected parser such as:

```javascript
let rawAI = $input.item.json.output || $input.item.json.text || "";

try {
  const cleanJSON = rawAI.replace(/```json/g, '').replace(/```/g, '').trim();
  const parsed = JSON.parse(cleanJSON);
  const sourceData = $node["data cleanse"].json;

  return {
    subject_line: parsed.subject_line,
    newsletter_body: parsed.newsletter_body,
    linkedin_teaser: parsed.linkedin_teaser,
    image_url: sourceData.image_url || "",
    link: sourceData.link || ""
  };
} catch (error) {
  return {
    error: "Parsing failed",
    ai_raw: rawAI
  };
}
```

### Important correction

The original catch block used `rawData`, which is undefined. Use `rawAI`.

## 10. Create the LinkedIn publishing node

38. Add a **LinkedIn** node named `Create a post`.
39. Connect `Format Response -> Create a post`.
40. Configure LinkedIn credentials with organization posting permission.
41. Set:
   - **Post As:** `Organization`
   - **Text:** `={{ $('Format Response').item.json.linkedin_teaser }}`
42. If using article posts, also configure article URL and metadata if the node version requires it.
43. Optionally enable continue-on-fail, as in the original workflow.

### Recommended addition

If possible, include the article URL from:
- `={{ $('Format Response').item.json.link }}`

## 11. Create the subscriber list reader

44. Add a **Google Sheets** node named `Email List`.
45. Connect `Format Response -> Email List`.
46. Configure Google Sheets credentials.
47. Select the spreadsheet containing subscribers.
48. Select the worksheet containing email addresses.
49. Ensure there is a column named exactly `email list`, or adjust the Gmail expression to match your real column name.

### Recommended subscriber sheet structure

| email list |
|---|
| person1@example.com |
| person2@example.com |

## 12. Create the loop node

50. Add a **Loop Over Items** node using **Split In Batches**.
51. Connect `Email List -> Loop Over Items`.
52. Use a batch size of `1` for one recipient at a time.

## 13. Create the Gmail sender

53. Add a **Gmail** node named `Send Email`.
54. Connect the loop’s item output to `Send Email`.
55. Configure Gmail OAuth2 credentials.
56. Set:
   - **To:** `={{ $json['email list'] }}`
   - **Subject:** `=Daily Newsletter : {{ $('Format Response').item.json.subject_line }}`
   - **Append Attribution:** Off
57. Paste the HTML message body, using expressions for:
   - `subject_line`
   - `image_url`
   - `newsletter_body`
   - `link`

## 14. Close the loop

58. Connect `Send Email -> Loop Over Items` back into the continue input so iteration proceeds through all recipients.

## 15. Add optional sticky notes

59. Add sticky notes for:
   - overall workflow description
   - detection/filter step
   - AI generation step
   - distribution step

## 16. Configure credentials

60. Create and attach credentials for:
   - **Google Sheets**
   - **Google Gemini**
   - **LinkedIn**
   - **Gmail**

## 17. Validate before activation

61. Test the workflow manually with one sample post.
62. Confirm:
   - the HTTP response contains the expected WordPress fields
   - the tracking sheet updates correctly
   - the AI returns valid JSON
   - LinkedIn publishing succeeds
   - Gmail sends correctly to test recipients
63. Activate the workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Blog to Social & Newsletter Automator | Workspace description |
| This workflow automatically fetches the latest blog post from a WordPress API and checks whether it has already been processed using a Google Sheets tracker. If the post is new, the workflow cleans and formats the blog data, then sends it to an AI agent powered by Google Gemini to generate a newsletter and LinkedIn teaser. | Workflow purpose |
| The formatted output is then distributed across channels. A LinkedIn post is published automatically, and a list of subscribers is retrieved from Google Sheets. The workflow loops through each recipient and sends a personalized HTML email newsletter using Gmail. | Distribution logic |
| Connect your Google Sheets account for both tracking (LastProcessedID) and email subscriber list. | Setup note |
| Add your Google Gemini API credentials to enable AI content generation. | Setup note |
| Connect your LinkedIn account for automated posting. | Setup note |
| Connect your Gmail account for sending newsletters. | Setup note |
| Update the WordPress API endpoint in the HTTP Request node. | Setup note |

## Additional technical observations

- The workflow contains several broken references to a nonexistent node named `Fetch Blog`. These must be replaced with `HTTP Request` or resolved by renaming the HTTP node.
- The `Format Response` node contains a parsing fallback bug due to `rawData` being undefined.
- The AI Agent node export does not include a visible prompt, so the most important generation logic may need to be recreated manually.
- The LinkedIn node appears underconfigured for an `ARTICLE` share type unless additional article fields are added.
- No false-path handling is defined after the `If` node. If no new article is found, the workflow simply stops, which is acceptable but implicit.