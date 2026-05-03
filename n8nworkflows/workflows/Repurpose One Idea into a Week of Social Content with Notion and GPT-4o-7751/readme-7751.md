Repurpose One Idea into a Week of Social Content with Notion and GPT-4o

https://n8nworkflows.xyz/workflows/repurpose-one-idea-into-a-week-of-social-content-with-notion-and-gpt-4o-7751


# Repurpose One Idea into a Week of Social Content with Notion and GPT-4o

### 1. Workflow Overview

This workflow automates the repurposing of a single content idea into a full week’s worth of social media posts using Notion and OpenAI’s GPT-4o. It is designed to receive an initial idea or URL trigger, process and enrich the content using AI, store the generated social posts back into Notion, and optionally schedule posts to Tailwind and Pinterest.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures new content ideas or URLs via webhook or a Notion trigger.
- **1.2 Data Preparation:** Conditionally fetches and extracts text from a source URL or uses raw input text.
- **1.3 AI Content Generation:** Sends content to OpenAI GPT-4o for repurposing into structured social media posts.
- **1.4 Data Storage:** Parses AI output and saves posts into Notion.
- **1.5 Optional Scheduling:** Conditionally posts generated content to Tailwind and Pinterest via HTTP API calls.

This modular design supports flexible input methods, robust content extraction, AI-powered content generation, and integration with social media scheduling platforms.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block listens for new content ideas either submitted via an HTTP webhook or triggered from a Notion database update.

**Nodes Involved:**  
- Webhook: Receive Idea/URL  
- Notion Trigger: On Ready Status  
- Merge: Both Triggers  
- Wait: Jitter 7–22s  

**Node Details:**  

- **Webhook: Receive Idea/URL**  
  - Type: Webhook  
  - Role: Receives incoming HTTP requests containing either a text idea or a URL.  
  - Configuration: Default HTTP POST webhook endpoint (parameters not explicitly configured here).  
  - Input: External HTTP request.  
  - Output: Passes data to Merge node.  
  - Edge Cases: Missing or malformed payload; security/authentication not specified (may need to be added).  
   
- **Notion Trigger: On Ready Status**  
  - Type: Notion Trigger  
  - Role: Listens for status changes in Notion content indicating readiness to process.  
  - Configuration: Uses Notion credentials and configured to trigger on specific status changes (exact filters not shown).  
  - Input: Notion event stream.  
  - Output: Passes data to Merge node.  
  - Edge Cases: Notion API rate limits; trigger misconfiguration.  

- **Merge: Both Triggers**  
  - Type: Merge  
  - Role: Combines data from Webhook and Notion trigger into a single stream.  
  - Configuration: Default merge mode (likely "Wait for All" or "Merge By Index").  
  - Inputs: Two inputs from webhook and Notion trigger.  
  - Output: Forwards merged data to Wait node.  
  - Edge Cases: If one trigger does not fire, workflow could hang or delay.  

- **Wait: Jitter 7–22s**  
  - Type: Wait  
  - Role: Introduces a randomized delay between 7 and 22 seconds to reduce rate limits or avoid burst API calls.  
  - Configuration: Wait time randomized within specified bounds.  
  - Input: Merged data.  
  - Output: Forwards to next processing node.  
  - Edge Cases: Excessive delay might cause timeout in upstream systems; no error expected.

---

#### 2.2 Data Preparation

**Overview:**  
Determines whether the input contains a source URL, fetches the page if so, extracts plain text from HTML, or uses raw idea text directly.

**Nodes Involved:**  
- Set: Gather Inputs  
- IF: Has Source URL?  
- HTTP: Fetch Source Page  
- Code: Strip HTML to Text  
- Set: Use Raw Idea  
- Merge: Choose Source  

**Node Details:**  

- **Set: Gather Inputs**  
  - Type: Set  
  - Role: Normalizes and prepares input data fields for subsequent logic.  
  - Configuration: Likely sets variables such as `sourceUrl`, `ideaText`, and other metadata (exact parameters not shown).  
  - Input: From Wait node.  
  - Output: To IF node.  

- **IF: Has Source URL?**  
  - Type: If  
  - Role: Branches workflow based on presence of a source URL.  
  - Configuration: Condition checks if `sourceUrl` field is set and non-empty.  
  - Input: From Set: Gather Inputs.  
  - Output:  
    - True: HTTP: Fetch Source Page  
    - False: Set: Use Raw Idea  

- **HTTP: Fetch Source Page**  
  - Type: HTTP Request  
  - Role: Retrieves HTML content from the provided source URL.  
  - Configuration: HTTP GET, URL set dynamically from `sourceUrl`.  
  - Input: From IF node (true branch).  
  - Output: To code node for extraction.  
  - Edge Cases: HTTP errors (404, timeout, redirects), invalid URLs, non-HTML content.  

- **Code: Strip HTML to Text**  
  - Type: Function (JavaScript)  
  - Role: Converts HTML response to clean plain text by stripping tags.  
  - Configuration: Uses DOM parsing or regex to remove HTML elements.  
  - Input: Raw HTML from HTTP node.  
  - Output: Plain text to Merge node.  
  - Edge Cases: Malformed HTML, large content size, potential injection risks.  

- **Set: Use Raw Idea**  
  - Type: Set  
  - Role: Passes raw idea text forward when no URL is provided.  
  - Configuration: Sets plain text from initial input payload.  
  - Input: From IF node (false branch).  
  - Output: To Merge node.  

- **Merge: Choose Source**  
  - Type: Merge  
  - Role: Consolidates text input from either extracted URL content or raw idea text into one unified payload.  
  - Inputs: From Strip HTML and Use Raw Idea nodes.  
  - Output: To user config node.  

---

#### 2.3 AI Content Generation

**Overview:**  
Uses OpenAI GPT-4o to repurpose the gathered content into a structured JSON representing a week of social media posts.

**Nodes Involved:**  
- Set: User Config (Edit Me!)  
- OpenAI: Repurpose → JSON  
- Code: Parse AI Output  

**Node Details:**  

- **Set: User Config (Edit Me!)**  
  - Type: Set  
  - Role: Holds editable user parameters, such as prompt templates, output settings, or API options.  
  - Configuration: User must customize this node with their OpenAI prompt details and other preferences.  
  - Input: From Merge: Choose Source.  
  - Output: To OpenAI node.  

- **OpenAI: Repurpose → JSON**  
  - Type: OpenAI (Chat or Completion)  
  - Role: Sends prompt and content to GPT-4o model to generate repurposed social post content in JSON format.  
  - Configuration: Uses OpenAI credentials, model set to GPT-4o, prompt dynamically constructed from user config and input text.  
  - Input: From User Config node.  
  - Output: Raw AI response JSON to parsing node.  
  - Edge Cases: API rate limits, authentication failures, malformed prompts, incomplete or unexpected AI responses.  

- **Code: Parse AI Output**  
  - Type: Function  
  - Role: Parses the AI-generated JSON string into structured JSON objects for further processing.  
  - Configuration: JSON.parse or equivalent with error handling.  
  - Input: Raw AI output string.  
  - Output: Parsed JSON to Notion save and scheduling logic.  
  - Edge Cases: Malformed JSON, empty or partial AI responses, exception handling required.  

---

#### 2.4 Data Storage

**Overview:**  
Saves the structured social media content into a designated Notion page or database.

**Nodes Involved:**  
- Notion: Save Content Page  

**Node Details:**  

- **Notion: Save Content Page**  
  - Type: Notion node (create/update)  
  - Role: Writes the repurposed social content into Notion for record keeping and further editing.  
  - Configuration: Requires Notion credentials; configured to target specific database or page; maps JSON fields to Notion properties.  
  - Input: Parsed JSON from AI output.  
  - Output: To conditional scheduling IF nodes.  
  - Edge Cases: Notion API rate limits, permission errors, schema mismatch.  

---

#### 2.5 Optional Scheduling

**Overview:**  
Conditionally schedules or posts generated social content to Tailwind and Pinterest via their respective APIs.

**Nodes Involved:**  
- IF: Schedule to Tailwind?  
- HTTP: Post to Tailwind  
- IF: Post to Pinterest?  
- HTTP: Create Pinterest Pin  

**Node Details:**  

- **IF: Schedule to Tailwind?**  
  - Type: If  
  - Role: Checks if the user wants to schedule content on Tailwind.  
  - Configuration: Checks a boolean flag or parameter from parsed JSON or user config.  
  - Input: From Notion save node.  
  - Output: True → HTTP Post to Tailwind; False → Skip.  

- **HTTP: Post to Tailwind**  
  - Type: HTTP Request  
  - Role: Sends scheduling data to Tailwind API to queue social posts.  
  - Configuration: POST request with auth headers, body constructed from social post data.  
  - Input: From IF node.  
  - Output: End of Tailwind branch.  
  - Edge Cases: Tailwind API errors, auth failures, invalid payload.  

- **IF: Post to Pinterest?**  
  - Type: If  
  - Role: Checks if Pinterest posting is enabled.  
  - Configuration: Boolean flag check on input data.  
  - Input: From Notion save node.  
  - Output: True → HTTP Create Pinterest Pin; False → Skip.  

- **HTTP: Create Pinterest Pin**  
  - Type: HTTP Request  
  - Role: Posts social content as a new Pinterest pin via API.  
  - Configuration: POST with OAuth or token auth, body includes image link, description, board ID.  
  - Input: From IF node.  
  - Output: End of Pinterest branch.  
  - Edge Cases: Pinterest API rate limits, auth failure, missing required fields.  

---

### 3. Summary Table

| Node Name                    | Node Type           | Functional Role                      | Input Node(s)                | Output Node(s)                  | Sticky Note                        |
|------------------------------|---------------------|-----------------------------------|-----------------------------|--------------------------------|----------------------------------|
| Sticky: Overview & Setup      | Sticky Note         | General overview and setup notes  | —                           | —                              |                                  |
| Sticky: Step 1 – Triggers     | Sticky Note         | Notes for trigger step            | —                           | —                              |                                  |
| Sticky: Step 2 – AI & Storage | Sticky Note         | Notes for AI and storage step     | —                           | —                              |                                  |
| Webhook: Receive Idea/URL     | Webhook             | Receives new idea or URL input    | External HTTP request        | Merge: Both Triggers            |                                  |
| Notion Trigger: On Ready Status | Notion Trigger     | Detects content ready status in Notion | Notion event stream          | Merge: Both Triggers            |                                  |
| Merge: Both Triggers          | Merge               | Merges webhook & Notion triggers  | Webhook, Notion Trigger      | Wait: Jitter 7–22s              |                                  |
| Wait: Jitter 7–22s            | Wait                | Adds randomized delay             | Merge: Both Triggers         | Set: Gather Inputs              |                                  |
| Set: Gather Inputs            | Set                 | Normalizes input data             | Wait: Jitter                 | IF: Has Source URL?             |                                  |
| IF: Has Source URL?           | If                  | Checks for presence of source URL | Set: Gather Inputs           | HTTP: Fetch Source Page (true), Set: Use Raw Idea (false) |                                  |
| HTTP: Fetch Source Page       | HTTP Request        | Fetches HTML from source URL      | IF: Has Source URL           | Code: Strip HTML to Text        |                                  |
| Code: Strip HTML to Text      | Function            | Extracts plain text from HTML     | HTTP: Fetch Source Page      | Merge: Choose Source            |                                  |
| Set: Use Raw Idea             | Set                 | Uses raw input idea text          | IF: Has Source URL           | Merge: Choose Source            |                                  |
| Merge: Choose Source          | Merge               | Merges extracted or raw text      | Code: Strip HTML, Set: Use Raw Idea | Set: User Config (Edit Me!) |                                  |
| Set: User Config (Edit Me!)   | Set                 | Holds user-editable parameters    | Merge: Choose Source         | OpenAI: Repurpose → JSON        |                                  |
| OpenAI: Repurpose → JSON      | OpenAI              | Generates social posts JSON       | Set: User Config             | Code: Parse AI Output           |                                  |
| Code: Parse AI Output         | Function            | Parses AI JSON output             | OpenAI node                 | Notion: Save Content Page, IF: Schedule to Tailwind?, IF: Post to Pinterest? |                                  |
| Notion: Save Content Page     | Notion              | Saves social posts in Notion      | Code: Parse AI Output        | IF: Schedule to Tailwind?, IF: Post to Pinterest? |                                  |
| IF: Schedule to Tailwind?     | If                  | Checks if scheduling to Tailwind  | Notion: Save Content Page    | HTTP: Post to Tailwind          |                                  |
| HTTP: Post to Tailwind        | HTTP Request        | Posts content to Tailwind API     | IF: Schedule to Tailwind?    | —                              |                                  |
| IF: Post to Pinterest?        | If                  | Checks if posting to Pinterest    | Notion: Save Content Page    | HTTP: Create Pinterest Pin      |                                  |
| HTTP: Create Pinterest Pin    | HTTP Request        | Posts content to Pinterest API    | IF: Post to Pinterest?       | —                              |                                  |
| Sticky: Troubleshooting & Observability | Sticky Note | Notes on troubleshooting and observability | —                           | —                              |                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: Receive Idea/URL**  
   - Type: Webhook  
   - Setup: HTTP POST endpoint to accept JSON payload containing either a text idea or URL.  
   - No authentication configured by default (consider adding).  

2. **Create Notion Trigger Node: On Ready Status**  
   - Type: Notion Trigger  
   - Setup: Connect your Notion credentials.  
   - Configure trigger to fire when a page status changes to “Ready” or equivalent.  

3. **Create Merge Node: Both Triggers**  
   - Type: Merge  
   - Inputs: Connect Webhook and Notion Trigger outputs.  
   - Mode: Default (Wait for all or merge by index as needed).  

4. **Add Wait Node: Jitter 7–22s**  
   - Type: Wait  
   - Configure randomized wait between 7 and 22 seconds to avoid API bursts.  
   - Connect output of Merge node as input.  

5. **Create Set Node: Gather Inputs**  
   - Type: Set  
   - Define variables: Extract fields from previous nodes to standardize variables such as `sourceUrl` and `ideaText`.  
   - Connect Wait node output to this node.  

6. **Add IF Node: Has Source URL?**  
   - Type: If  
   - Condition: Check if `sourceUrl` exists and is not empty.  
   - Connect output of Gather Inputs node as input.  

7. **HTTP Request Node: Fetch Source Page**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: Use expression to set `sourceUrl` from input data.  
   - Connect IF node’s true branch to this node.  

8. **Function Node: Strip HTML to Text**  
   - Type: Function  
   - JavaScript code: Parse HTML content and return plain text string.  
   - Input: HTTP fetch node output.  
   - Output: Plain text passed downstream.  

9. **Set Node: Use Raw Idea**  
   - Type: Set  
   - Set plain text variable from original input idea text.  
   - Connect IF node’s false branch here.  

10. **Merge Node: Choose Source**  
    - Type: Merge  
    - Inputs: From Strip HTML node and Use Raw Idea node.  
    - Connect output to next node.  

11. **Set Node: User Config (Edit Me!)**  
    - Type: Set  
    - Define parameters for OpenAI prompt, model, max tokens, temperature, and any additional user preferences.  
    - Connect output from Merge node here.  

12. **OpenAI Node: Repurpose → JSON**  
    - Type: OpenAI  
    - Credentials: Connect your OpenAI API credentials.  
    - Model: GPT-4o.  
    - Prompt: Use input text and user config to prompt AI for repurposing content in JSON format.  
    - Connect User Config node output here.  

13. **Function Node: Parse AI Output**  
    - Type: Function  
    - Parse JSON string from AI response safely.  
    - Output: Structured data for Notion and scheduling.  

14. **Notion Node: Save Content Page**  
    - Type: Notion  
    - Credentials: Connect your Notion API credentials.  
    - Configure to create or update a page with the generated social posts, mapping JSON fields to Notion properties.  
    - Connect output of Parse AI Output node here.  

15. **IF Node: Schedule to Tailwind?**  
    - Type: If  
    - Condition: Check a boolean flag in parsed data indicating if scheduling to Tailwind is required.  
    - Connect from Notion save node.  

16. **HTTP Request Node: Post to Tailwind**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: Tailwind API endpoint.  
    - Auth: Configure Tailwind API key or OAuth as required.  
    - Body: Construct from parsed social posts data.  
    - Connect IF node true branch here.  

17. **IF Node: Post to Pinterest?**  
    - Type: If  
    - Condition: Check a boolean flag indicating if posting to Pinterest is required.  
    - Connect from Notion save node.  

18. **HTTP Request Node: Create Pinterest Pin**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: Pinterest API endpoint for creating pins.  
    - Auth: OAuth or token-based Pinterest credentials.  
    - Body: Include image URL, description, board ID from parsed data.  
    - Connect IF node true branch here.  

19. **Add Sticky Notes**  
    - Add sticky notes for overview, setup instructions, troubleshooting, and step explanations as needed for maintainability.  

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                         |
|--------------------------------------------------------------------------------------------------|---------------------------------------|
| Workflow uses OpenAI GPT-4o model for advanced AI content generation.                             | OpenAI API                             |
| Notion API integration requires proper permissions to read and write pages/databases.           | Notion API documentation               |
| Tailwind and Pinterest integrations rely on HTTP API endpoints and require API keys or OAuth.   | Tailwind: https://tailwindapp.com/api, Pinterest: https://developers.pinterest.com/ |
| To avoid API rate limits, a randomized jitter delay is introduced after the initial triggers.    | Workflow design detail                 |
| User must configure the “Set: User Config (Edit Me!)” node to customize prompts and parameters. | Key customization point                |

---

**Disclaimer:** The provided content is derived exclusively from an automated n8n workflow and complies strictly with applicable content policies. No illegal, offensive, or protected material is included. All data processed is legal and public.