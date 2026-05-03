Create LinkedIn Content with Perplexity Research, GPT-4 & Google Sheets

https://n8nworkflows.xyz/workflows/create-linkedin-content-with-perplexity-research--gpt-4---google-sheets-9185


# Create LinkedIn Content with Perplexity Research, GPT-4 & Google Sheets

### 1. Workflow Overview

This workflow automates the generation of LinkedIn content ideas by leveraging AI research, language models, and data management tools. It is designed to run daily and produce fresh, engaging LinkedIn post ideas focused on emerging tech topics such as Generative AI, AI agents, vibe coding, automation, no-code tools, and product management. The workflow consists of the following logical blocks:

- **1.1 Scheduled Trigger & Historical Context Retrieval:** The workflow initiates on a daily schedule, fetching the most recent LinkedIn post ideas from a Google Sheet to avoid content repetition.
- **1.2 AI-Driven Idea Research:** It uses Perplexity AI to perform a broad content search and generate three novel LinkedIn content ideas, explicitly excluding recent ideas to maintain freshness.
- **1.3 Structured Parsing & Idea Refinement:** The raw AI output is parsed into a structured JSON format for consistency and then refined through a GPT-4 based language model to produce polished LinkedIn posts.
- **1.4 Data Storage & Notification:** The generated content, including post text and metadata, is appended to a Google Sheet for record-keeping, and a notification is sent via Telegram to inform the user of the update.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Historical Context Retrieval

- **Overview:**  
  This block triggers the workflow daily at 6 AM and retrieves the last five LinkedIn post ideas from Google Sheets to provide context and avoid duplicate ideas in the new generation process.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get row(s) in sheet  
  - Limit  
  - Sticky Note (comment node)

- **Node Details:**

  - **Schedule Trigger**  
    - *Type & Role:* n8n schedule trigger node initiating workflow execution daily at 6:00 AM.  
    - *Configuration:* Trigger set to hourly 6 AM daily interval.  
    - *Inputs:* None (start node).  
    - *Outputs:* Connects to "Get row(s) in sheet".  
    - *Edge Cases:* Missed triggers if n8n instance downtime; time zone set to Europe/Lisbon.  
    - *Sticky Note:* Describes daily trigger and retrieval of last 3 posts.

  - **Get row(s) in sheet**  
    - *Type & Role:* Google Sheets node fetching rows from a specified sheet storing past LinkedIn ideas.  
    - *Configuration:* Uses Google Sheets OAuth2 credentials; retrieves all rows (no filters).  
    - *Inputs:* From Schedule Trigger.  
    - *Outputs:* Connects to "Limit".  
    - *Edge Cases:* Auth failures, API rate limits, empty sheet.

  - **Limit**  
    - *Type & Role:* Limits the number of items passed forward to last 5 rows (to avoid repetition).  
    - *Configuration:* Keeps last 5 items only.  
    - *Inputs:* From Get row(s) in sheet.  
    - *Outputs:* Connects to "Message a model".  
    - *Edge Cases:* If fewer than 5 rows exist, passes all available.

  - **Sticky Note**  
    - *Role:* Documentation only, no execution effect.  
    - *Content:* "Trigger daily. The workflow triggers on a daily basis and gets the last 3 posts that were generated."  
    - *Position:* Near the trigger block for clarity.

#### 1.2 AI-Driven Idea Research

- **Overview:**  
  This block uses Perplexity AI to research and generate three fresh, non-obvious LinkedIn content ideas, explicitly excluding recent ideas from the "Limit" node to prevent duplication.

- **Nodes Involved:**  
  - Message a model (Perplexity)  
  - Sticky Note (comment node)  
  - Structured Output Parser

- **Node Details:**

  - **Message a model**  
    - *Type & Role:* Perplexity AI node querying the "sonar-pro" model for content research and idea generation.  
    - *Configuration:* System prompt instructs the AI to find fresh LinkedIn post ideas in a niche (Generative AI, vibe coding, etc.) while excluding last 5 ideas retrieved from Google Sheets. It requests 3 post ideas with detailed JSON structure including idea, details, style, and image prompt.  
    - *Inputs:* From Limit node (provides recent ideas for exclusion).  
    - *Outputs:* Connects to Structured Output Parser.  
    - *Edge Cases:* API rate limit or auth failure, empty or malformed response, expression failures in prompt interpolation.  
    - *Sticky Note:* Describes search for ideas using Perplexity and structured output parsing.

  - **Structured Output Parser**  
    - *Type & Role:* Langchain structured output parser to convert Perplexity AI free text into structured JSON array matching the specified schema (idea, details, style, image).  
    - *Configuration:* Uses an example JSON schema with fields for idea, details, style, and image prompt.  
    - *Inputs:* From Message a model.  
    - *Outputs:* Connects to Idea Parser (LLM chain).  
    - *Edge Cases:* Parsing failures if output format deviates, JSON syntax errors.

  - **Sticky Note1**  
    - *Role:* Documentation only.  
    - *Content:* "Search for ideas. It then uses Perplexity to find ideas based on the niche selected and parses them into a structured output."

#### 1.3 Structured Parsing & Idea Refinement

- **Overview:**  
  This block refines the initial AI-generated ideas by parsing the structured JSON data and generating polished LinkedIn posts using GPT-4. The posts are formatted in a human, informal tone with engaging hooks.

- **Nodes Involved:**  
  - Idea Parser (LLM chain)  
  - Code  
  - Post Generator (LLM chain)  
  - Structured Output Parser1

- **Node Details:**

  - **Idea Parser**  
    - *Type & Role:* Langchain LLM chain (GPT-4) that parses the JSON content extracted from Perplexity's output, essentially normalizing and preparing data for further processing.  
    - *Configuration:* Takes the raw message content from Perplexity AI as input text for parsing.  
    - *Inputs:* From Structured Output Parser.  
    - *Outputs:* Connects to Code node.  
    - *Edge Cases:* Parsing errors, response timeouts.

  - **Code**  
    - *Type & Role:* JavaScript code node to flatten the parsed array of ideas into a simple list for batch processing.  
    - *Configuration:* Returns first JSON input's output property flattened to depth 1.  
    - *Inputs:* From Idea Parser.  
    - *Outputs:* Connects to Post Generator.  
    - *Edge Cases:* If input data structure changes, may cause errors.

  - **Post Generator**  
    - *Type & Role:* Langchain LLM chain (GPT-4 mini) that takes each idea and generates a full LinkedIn post draft in JSON format with a natural, informal, and engaging tone.  
    - *Configuration:* The prompt instructs the AI to write posts with bold hooks, short paragraphs, emotional tone, and a soft call to action. Output is strictly JSON with a "post" field. Uses temperature=1, topP=0.9, presencePenalty=0.8, frequencyPenalty=0.3 for creativity and variation.  
    - *Inputs:* From Code node.  
    - *Outputs:* Connects to Structured Output Parser1 and then to Append row in sheet.  
    - *Edge Cases:* Model errors, rate limits, output parsing errors.

  - **Structured Output Parser1**  
    - *Type & Role:* Parses Post Generator output to extract the final "post" JSON field.  
    - *Configuration:* Schema expects JSON with single "post" string property.  
    - *Inputs:* From Post Generator.  
    - *Outputs:* Connects back to Post Generator node's ai_outputParser input (this connection is for internal handling, actual output goes downstream).  
    - *Edge Cases:* Parsing failures if output format deviates.

#### 1.4 Data Storage & Notification

- **Overview:**  
  This block appends the finalized LinkedIn post ideas, including metadata and image prompts, to a Google Sheet for archival and triggers a Telegram notification to inform the user that new content is ready.

- **Nodes Involved:**  
  - Append row in sheet (Google Sheets)  
  - Send a text message (Telegram)  
  - Sticky Note (comment node)

- **Node Details:**

  - **Append row in sheet**  
    - *Type & Role:* Google Sheets node appending new rows with generated ideas and posts.  
    - *Configuration:* Maps fields: idea, details, style, output post text, status ("New"), image prompt, and empty image URL. Uses row ID as `==ROW()-1` to generate sequential IDs.  
    - *Inputs:* From Post Generator.  
    - *Outputs:* Connects to Send a text message.  
    - *Edge Cases:* API limits, auth failures, schema mismatch.

  - **Send a text message**  
    - *Type & Role:* Telegram node sending a notification message to a specified chat ID.  
    - *Configuration:* Sends the static message "New ideas for LinkedIn have been generated!" to chat ID 1516391222.  
    - *Inputs:* From Append row in sheet.  
    - *Outputs:* None (end node).  
    - *Edge Cases:* Telegram API issues, invalid chat ID.

  - **Sticky Note2**  
    - *Role:* Documentation only.  
    - *Content:* "Create and save ideas. Finally, the workflow creates the ideas with an image prompt to generate later, saves them to Google Sheets and sends a notification via Telegram."

---

### 3. Summary Table

| Node Name              | Node Type                              | Functional Role                                         | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                         |
|------------------------|--------------------------------------|--------------------------------------------------------|------------------------|-------------------------|---------------------------------------------------------------------------------------------------|
| Schedule Trigger       | n8n-nodes-base.scheduleTrigger       | Triggers workflow daily at 6 AM                         | None                   | Get row(s) in sheet     | Trigger daily. The workflow triggers on a daily basis and gets the last 3 posts that were generated. |
| Get row(s) in sheet    | n8n-nodes-base.googleSheets           | Retrieves last LinkedIn post ideas from Google Sheets  | Schedule Trigger       | Limit                   | Trigger daily. The workflow triggers on a daily basis and gets the last 3 posts that were generated. |
| Limit                  | n8n-nodes-base.limit                  | Limits to last 5 recent posts to avoid duplication     | Get row(s) in sheet    | Message a model         | Trigger daily. The workflow triggers on a daily basis and gets the last 3 posts that were generated. |
| Message a model        | n8n-nodes-base.perplexity             | Queries Perplexity AI for 3 fresh LinkedIn post ideas  | Limit                  | Structured Output Parser | Search for ideas. It then uses Perplexity to find ideas based on the niche selected and parses them into a structured output. |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Parses Perplexity output into structured JSON           | Message a model         | Idea Parser             | Search for ideas. It then uses Perplexity to find ideas based on the niche selected and parses them into a structured output. |
| Idea Parser            | @n8n/n8n-nodes-langchain.chainLlm    | Parses and normalizes AI content ideas                  | Structured Output Parser | Code                    |                                                                                                   |
| Code                   | n8n-nodes-base.code                   | Flattens parsed JSON array for batch processing         | Idea Parser            | Post Generator          |                                                                                                   |
| Post Generator         | @n8n/n8n-nodes-langchain.chainLlm    | Generates polished LinkedIn posts from ideas            | Code                   | Append row in sheet     |                                                                                                   |
| Structured Output Parser1 | @n8n/n8n-nodes-langchain.outputParserStructured | Parses final post JSON output from Post Generator       | Post Generator          | Post Generator (ai_outputParser input) |                                                                                                   |
| Append row in sheet    | n8n-nodes-base.googleSheets           | Saves generated posts and metadata to Google Sheets    | Post Generator         | Send a text message     | Create and save ideas. Finally, the workflow creates the ideas with an image prompt to generate later, saves them to Google Sheets and sends a notification via Telegram. |
| Send a text message    | n8n-nodes-base.telegram               | Sends Telegram notification of new LinkedIn ideas      | Append row in sheet    | None                    | Create and save ideas. Finally, the workflow creates the ideas with an image prompt to generate later, saves them to Google Sheets and sends a notification via Telegram. |
| Sticky Note            | n8n-nodes-base.stickyNote             | Documentation node                                      | None                   | None                    | Trigger daily. The workflow triggers on a daily basis and gets the last 3 posts that were generated. |
| Sticky Note1           | n8n-nodes-base.stickyNote             | Documentation node                                      | None                   | None                    | Search for ideas. It then uses Perplexity to find ideas based on the niche selected and parses them into a structured output. |
| Sticky Note2           | n8n-nodes-base.stickyNote             | Documentation node                                      | None                   | None                    | Create and save ideas. Finally, the workflow creates the ideas with an image prompt to generate later, saves them to Google Sheets and sends a notification via Telegram. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 6:00 AM (timezone Europe/Lisbon)  
   - Connect output to Google Sheets node "Get row(s) in sheet".

2. **Create Google Sheets Node "Get row(s) in sheet":**  
   - Type: Google Sheets (read operation)  
   - Connect Google Sheets OAuth2 credentials.  
   - Configure to read all rows from the sheet containing prior LinkedIn post ideas.  
   - Connect output to "Limit" node.

3. **Create Limit Node:**  
   - Type: Limit  
   - Set "Keep" to "lastItems" with maxItems = 5  
   - Connect output to Perplexity AI node "Message a model".

4. **Create Perplexity Node "Message a model":**  
   - Type: Perplexity AI  
   - Select model: "sonar-pro"  
   - Use OAuth2 credentials for Perplexity API.  
   - Set system prompt to instruct AI to generate 3 fresh LinkedIn post ideas avoiding the last 5 ideas (referenced dynamically from the Limit node).  
   - Request output as a structured JSON array, including fields: idea, details, style, image prompt.  
   - Connect output to Langchain structured output parser.

5. **Create Structured Output Parser Node:**  
   - Type: Langchain output parser (structured)  
   - Provide JSON schema example matching expected Perplexity output (array of objects with idea, details, style, image).  
   - Connect output to Langchain LLM chain "Idea Parser".

6. **Create Langchain Chain Node "Idea Parser":**  
   - Type: Langchain Chain (LLM) with GPT-4 mini model  
   - Configure to parse the JSON content from Perplexity AI output.  
   - Connect output to Code node.

7. **Create Code Node:**  
   - Type: Code  
   - JavaScript: `return $input.first().json.output.flat(1);`  
   - Flattens array for batch processing.  
   - Connect output to Langchain Chain "Post Generator".

8. **Create Langchain Chain Node "Post Generator":**  
   - Type: Langchain Chain (LLM) with GPT-4 mini model  
   - Configure prompt to generate human-like, informal, bold LinkedIn post drafts from idea, details, and style fields.  
   - Set model parameters: temperature=1, topP=0.9, presencePenalty=0.8, frequencyPenalty=0.3.  
   - Enable output parser with JSON schema expecting `{ "post": "" }`.  
   - Connect output to Google Sheets node "Append row in sheet".

9. **Create Structured Output Parser1 Node:**  
   - Type: Langchain Output Parser (structured)  
   - Schema expects JSON with a single "post" string property.  
   - Connect output internally to Post Generator node's ai_outputParser input for processing.

10. **Create Google Sheets Node "Append row in sheet":**  
    - Type: Google Sheets (append row)  
    - Connect Google Sheets OAuth2 credentials.  
    - Map fields:  
      - id: formula `==ROW()-1` for unique sequential ID  
      - idea, details, style: from Code node output JSON fields  
      - output: from Post Generator parsed post text  
      - status: "New" (static)  
      - imagePrompt: from idea's image prompt field  
      - image: empty string (reserved for future use)  
    - Connect output to Telegram node.

11. **Create Telegram Node "Send a text message":**  
    - Type: Telegram  
    - No credentials configured here (assumes existing setup)  
    - Set chat ID to `1516391222`  
    - Message text: "New ideas for LinkedIn have been generated!"  
    - Connect no further nodes (endpoint).

12. **Add sticky notes for clarity:**  
    - Near Schedule Trigger and Get row(s) in sheet: "Trigger daily. The workflow triggers on a daily basis and gets the last 3 posts that were generated."  
    - Near Message a model and Structured Output Parser: "Search for ideas. It then uses Perplexity to find ideas based on the niche selected and parses them into a structured output."  
    - Near Append row in sheet and Send a text message: "Create and save ideas. Finally, the workflow creates the ideas with an image prompt to generate later, saves them to Google Sheets and sends a notification via Telegram."

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow uses GPT-4 mini model for content generation ensuring advanced natural language output quality.        | Model choice impacts cost and response speed; adjust if needed.                                    |
| The workflow is designed to avoid repeated ideas by referencing recent posts stored in Google Sheets.           | Ensures content freshness and relevance.                                                           |
| Telegram notification chat ID must be configured correctly to receive alerts.                                  | Verify Telegram Bot and chat ID setup before deployment.                                           |
| OpenAI and Perplexity credentials must be correctly set up and authorized with sufficient quota.                | Potential failure points include auth errors and rate limits.                                      |
| Google Sheets schema includes fields for post tracking and future image generation prompts.                     | Extensible for future integration with image generation workflows.                                 |
| Sticky notes provide essential documentation inside the workflow for maintainability and onboarding.           | Encouraged best practice for collaborative development.                                           |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.