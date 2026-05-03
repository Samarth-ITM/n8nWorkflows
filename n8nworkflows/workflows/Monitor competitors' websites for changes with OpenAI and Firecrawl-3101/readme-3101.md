Monitor competitors' websites for changes with OpenAI and Firecrawl

https://n8nworkflows.xyz/workflows/monitor-competitors--websites-for-changes-with-openai-and-firecrawl-3101


# Monitor competitors' websites for changes with OpenAI and Firecrawl

### 1. Workflow Overview

This workflow automates monitoring of competitors' or any specified websites for relevant changes using AI and web scraping. It is designed for users who want to receive email alerts when specific changes occur on a webpage, such as price drops, new blog posts, job listings, or terms updates. The workflow leverages Firecrawl for scraping, OpenAI for intelligent analysis and decision-making, and Gmail for notifications.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures user espionage assignment instructions via a form trigger.
- **1.2 Instruction Parsing & Prompt Generation:** Converts user instructions into a structured prompt and extracts the target website URL.
- **1.3 Initial Webpage Scraping:** Scrapes the target webpage content for the first time.
- **1.4 Delay:** Waits one day before the next scrape to enable comparison.
- **1.5 Second Webpage Scraping:** Scrapes the webpage again after the delay.
- **1.6 AI Decision & Notification:** Uses AI to compare the two scrapes, decide if an email notification is warranted, and sends the email if needed.
- **1.7 Loop Back:** Returns to the initial scrape to continue daily monitoring indefinitely.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Captures espionage assignment instructions from the user via a web form to start the monitoring process.

- **Nodes Involved:**  
  - New espionage assignment

- **Node Details:**  
  - **New espionage assignment**  
    - Type: Form Trigger  
    - Role: Entry point for user input; collects a single field named `assignment_instructions`.  
    - Configuration: Form titled "New espionage assignment" with one text field for instructions.  
    - Input: User-submitted form data.  
    - Output: JSON object containing `assignment_instructions`.  
    - Edge Cases: User submits incomplete or ambiguous instructions; no validation beyond form presence.  
    - Notes: Triggers the workflow on submission.

#### 1.2 Instruction Parsing & Prompt Generation

- **Overview:**  
  Converts the raw user instructions into a structured JSON containing the website URL and a verbose AI prompt for later analysis.

- **Nodes Involved:**  
  - convert message to website url & instruction  
  - parse results

- **Node Details:**  
  - **convert message to website url & instruction**  
    - Type: HTTP Request (OpenAI Chat Completion API)  
    - Role: Sends user instructions to OpenAI GPT-4o-2024-08-06 model to extract a plain website URL and generate a detailed AI prompt.  
    - Configuration: POST request to OpenAI chat completions endpoint with a user message instructing conversion of the assignment instructions. Uses OpenAI API key credential.  
    - Key Expressions: Uses expression to insert user instructions dynamically: `{{ $('New espionage assignment').first().json.assignment_instructions }}`  
    - Output: JSON schema with `website_url` and `prompt`.  
    - Edge Cases: API errors, malformed JSON response, or incomplete extraction.  
    - Version: Uses OpenAI API version supporting structured outputs.  
  - **parse results**  
    - Type: Code Node  
    - Role: Parses the JSON string response from the previous node into usable JSON object for downstream nodes.  
    - Configuration: JavaScript code parsing the first choice content as JSON.  
    - Input: Raw JSON string from OpenAI response.  
    - Output: Parsed JSON object with `website_url` and `prompt`.  
    - Edge Cases: JSON parse errors if response malformed.

#### 1.3 Initial Webpage Scraping

- **Overview:**  
  Scrapes the target website's main content in markdown format using Firecrawl API.

- **Nodes Involved:**  
  - scrape page - 1

- **Node Details:**  
  - **scrape page - 1**  
    - Type: HTTP Request  
    - Role: Sends POST request to Firecrawl API to scrape the website URL extracted earlier.  
    - Configuration:  
      - URL: `https://api.firecrawl.dev/v1/scrape`  
      - Body: JSON with `url` from parsed results, requesting markdown format, only main content, with 5 seconds wait.  
      - Authentication: Firecrawl API key via HTTP Header Auth.  
      - Retry enabled on failure.  
    - Input: Website URL from `parse results`.  
    - Output: Scraped page content in markdown.  
    - Edge Cases: API rate limits, network errors, invalid URL, or empty content.  
    - Sticky Note: Reminder to connect Firecrawl account.  

#### 1.4 Delay

- **Overview:**  
  Waits for one day to allow meaningful changes to occur on the monitored webpage before the next scrape.

- **Nodes Involved:**  
  - wait 1 day

- **Node Details:**  
  - **wait 1 day**  
    - Type: Wait Node  
    - Role: Pauses workflow execution for 1 day.  
    - Configuration: Unit set to days, amount set to 1.  
    - Input: Triggered after initial scrape.  
    - Output: Triggers second scrape after delay.  
    - Edge Cases: Workflow pause failures or interruptions.

#### 1.5 Second Webpage Scraping

- **Overview:**  
  Performs a second scrape of the same webpage to capture updated content for comparison.

- **Nodes Involved:**  
  - scrape page - 2

- **Node Details:**  
  - **scrape page - 2**  
    - Type: HTTP Request  
    - Role: Same as scrape page - 1, performs a second scrape after delay.  
    - Configuration: Identical to scrape page - 1, uses Firecrawl API with same parameters and credentials.  
    - Input: Website URL from parsed results.  
    - Output: Scraped page content in markdown.  
    - Edge Cases: Same as scrape page - 1.  

#### 1.6 AI Decision & Notification

- **Overview:**  
  Uses an AI agent to compare the two scraped versions of the webpage and decide if an email notification should be sent. If yes, sends an email summarizing relevant changes.

- **Nodes Involved:**  
  - send e-mail?  
  - OpenAI Chat Model  
  - Gmail

- **Node Details:**  
  - **send e-mail?**  
    - Type: Langchain Agent Node  
    - Role: Receives the AI prompt and both versions of the scraped page content to decide if an email should be sent.  
    - Configuration:  
      - Text input: Uses prompt from parsed results.  
      - System message: Includes JSON stringified old and new page markdown content for comparison.  
      - Output parser enabled to interpret AI response.  
      - Instruction: Only send email if conditions in prompt are met.  
    - Input: Scraped page - 2 content, prompt from parsed results, and old scrape content.  
    - Output: Triggers Gmail node if email is to be sent.  
    - Edge Cases: AI misinterpretation, API errors, or no output.  
  - **OpenAI Chat Model**  
    - Type: Langchain Chat Model Node  
    - Role: Provides the language model (GPT-4o) for the agent node.  
    - Configuration: Model set to GPT-4o, connected with OpenAI API credentials.  
    - Input: Receives prompt from send e-mail? node.  
    - Output: AI-generated decision and email content.  
    - Edge Cases: API limits, auth errors.  
  - **Gmail**  
    - Type: Gmail Tool Node  
    - Role: Sends an email notification if the AI agent decides it is necessary.  
    - Configuration:  
      - Recipient: tom@sleak.chat (example email)  
      - Subject: Includes website URL dynamically from parsed results.  
      - Message: Uses AI-generated summary of relevant changes.  
      - Email type: Text, no attribution appended.  
      - Credential: Gmail OAuth2 account connected.  
    - Input: Triggered by AI agent node.  
    - Output: Email sent confirmation.  
    - Edge Cases: Email sending failures, auth errors, invalid recipient.  
    - Sticky Note: Reminder to connect Gmail account.

#### 1.7 Loop Back

- **Overview:**  
  After sending the email or deciding not to, the workflow loops back to the initial scrape to continue daily monitoring indefinitely.

- **Nodes Involved:**  
  - scrape page - 1 (triggered again from send e-mail? node)

- **Node Details:**  
  - The output of the send e-mail? node connects back to scrape page - 1, restarting the cycle.  
  - Ensures continuous daily monitoring until manually stopped.

---

### 3. Summary Table

| Node Name                          | Node Type                          | Functional Role                          | Input Node(s)                      | Output Node(s)                  | Sticky Note                                                                                     |
|-----------------------------------|----------------------------------|----------------------------------------|----------------------------------|--------------------------------|------------------------------------------------------------------------------------------------|
| New espionage assignment           | Form Trigger                     | Captures user instructions             | -                                | convert message to website url & instruction |                                                                                                |
| convert message to website url & instruction | HTTP Request (OpenAI API)         | Extracts website URL and AI prompt     | New espionage assignment          | parse results                  |                                                                                                |
| parse results                     | Code                             | Parses OpenAI JSON response            | convert message to website url & instruction | scrape page - 1               |                                                                                                |
| scrape page - 1                   | HTTP Request (Firecrawl API)     | Initial webpage scrape                  | parse results                    | wait 1 day                    | Connect your Firecrawl account                                                                 |
| wait 1 day                       | Wait                             | Delays workflow for 1 day               | scrape page - 1                  | scrape page - 2               |                                                                                                |
| scrape page - 2                   | HTTP Request (Firecrawl API)     | Second webpage scrape                   | wait 1 day                      | send e-mail?                 | Connect your Firecrawl account                                                                 |
| send e-mail?                     | Langchain Agent                  | AI decision to send email notification | scrape page - 2                  | scrape page - 1, Gmail        |                                                                                                |
| OpenAI Chat Model                 | Langchain Chat Model             | Provides GPT-4o model for AI agent     | send e-mail? (ai_languageModel) | send e-mail? (ai_tool)        | Connect your own OpenAI account                                                                |
| Gmail                           | Gmail Tool                      | Sends email notification                | send e-mail? (ai_tool)           | -                              | Connect your own Gmail account                                                                 |
| Sticky Note                      | Sticky Note                     | Notes and reminders                     | -                                | -                              | ## Note: almost never works right away. Adjust the prompts in the 'Tools agent' and 'Gmail' node as desired to steer the agent's behavior in the right direction |
| Sticky Note2                     | Sticky Note                     | Reminder to connect Firecrawl account  | -                                | -                              | Connect your Firecrawl account                                                                 |
| Sticky Note3                     | Sticky Note                     | Reminder to connect OpenAI account     | -                                | -                              | Connect your own OpenAI account                                                                |
| Sticky Note4                     | Sticky Note                     | Reminder to connect Gmail account      | -                                | -                              | Connect your own Gmail account                                                                 |
| Sticky Note5                     | Sticky Note                     | Reminder to connect OpenAI account     | -                                | -                              | Connect your own OpenAI account                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Name: `New espionage assignment`  
   - Type: Form Trigger  
   - Configuration:  
     - Form Title: "New espionage assignment"  
     - Add one text field labeled `assignment_instructions`  
   - Purpose: To receive user instructions for monitoring.

2. **Add HTTP Request Node to Convert Instructions**  
   - Name: `convert message to website url & instruction`  
   - Type: HTTP Request  
   - Configuration:  
     - URL: `https://api.openai.com/v1/chat/completions`  
     - Method: POST  
     - Authentication: Use OpenAI API key credential  
     - Body (JSON):  
       ```json
       {
         "model": "gpt-4o-2024-08-06",
         "messages": [
           {
             "role": "user",
             "content": "convert the following message to a website url (just the plain text url, NOT formatted or in markdown) and prompt to AI. Make the prompt as verbose as possible. Message: {{ $('New espionage assignment').first().json.assignment_instructions }}"
           }
         ],
         "response_format": {
           "type": "json_schema",
           "json_schema": {
             "name": "variable_extraction",
             "schema": {
               "type": "object",
               "properties": {
                 "website_url": { "type": "string" },
                 "prompt": { "type": "string" }
               },
               "required": ["website_url", "prompt"],
               "additionalProperties": false
             },
             "strict": true
           }
         }
       }
       ```
     - Send Body: JSON  
   - Connect input from `New espionage assignment`.

3. **Add Code Node to Parse OpenAI Response**  
   - Name: `parse results`  
   - Type: Code  
   - Configuration:  
     - Language: JavaScript  
     - Code:  
       ```javascript
       const parsedObject = JSON.parse($('convert message to website url & instruction').first().json.choices[0].message.content);
       return parsedObject;
       ```
   - Connect input from `convert message to website url & instruction`.

4. **Add HTTP Request Node for Initial Scrape**  
   - Name: `scrape page - 1`  
   - Type: HTTP Request  
   - Configuration:  
     - URL: `https://api.firecrawl.dev/v1/scrape`  
     - Method: POST  
     - Authentication: Firecrawl API key via HTTP Header Auth  
     - Body (JSON):  
       ```json
       {
         "url": "{{ $('parse results').item.json.website_url }}",
         "formats": ["markdown"],
         "onlyMainContent": true,
         "waitFor": 5000
       }
       ```
     - Send Body: JSON  
     - Enable retry on failure  
   - Connect input from `parse results`.

5. **Add Wait Node for 1 Day Delay**  
   - Name: `wait 1 day`  
   - Type: Wait  
   - Configuration:  
     - Unit: days  
     - Amount: 1  
   - Connect input from `scrape page - 1`.

6. **Add HTTP Request Node for Second Scrape**  
   - Name: `scrape page - 2`  
   - Type: HTTP Request  
   - Configuration: Same as `scrape page - 1`  
   - Connect input from `wait 1 day`.

7. **Add Langchain Agent Node for AI Decision**  
   - Name: `send e-mail?`  
   - Type: Langchain Agent  
   - Configuration:  
     - Text: `={{ $('parse results').item.json.prompt }}\n\nNOTE: ONLY send an email if the situation meets the above condition. Otherwise, do NOT use the tool\n\nNOTE: this concerns differences between the "old version page" (scrape from yesterday) and "new version page" (scrape from now)`  
     - System Message:  
       ```text
       old version page: 

       {{ JSON.stringify($('scrape page - 1').item.json["data"]["markdown"]) }} 

       /// 

       new version page: 

       {{ JSON.stringify($('scrape page - 2').item.json["data"]["markdown"]) }}
       ```  
     - Enable output parser  
   - Connect input from `scrape page - 2`.

8. **Add Langchain Chat Model Node**  
   - Name: `OpenAI Chat Model`  
   - Type: Langchain Chat Model  
   - Configuration:  
     - Model: GPT-4o  
     - Credentials: OpenAI API key  
   - Connect as AI language model for `send e-mail?`.

9. **Add Gmail Tool Node to Send Email**  
   - Name: `Gmail`  
   - Type: Gmail Tool  
   - Configuration:  
     - Send To: `tom@sleak.chat` (replace with actual recipient)  
     - Subject: `=Relevant changes on {{ $('parse results').item.json.website_url }}`  
     - Message: `={{ $fromAI("change", "What relevant part has changed on the website?") }}`  
     - Email Type: Text  
     - Append Attribution: false  
     - Credential: Gmail OAuth2 account  
   - Connect input from `send e-mail?` AI tool output.

10. **Connect Loop Back**  
    - Connect main output of `send e-mail?` node back to `scrape page - 1` node to restart the monitoring cycle.

11. **Add Sticky Notes (Optional)**  
    - Add notes reminding to connect Firecrawl, OpenAI, and Gmail credentials.  
    - Add note about prompt adjustment for better AI behavior.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Almost never works right away. Adjust the prompts in the 'Tools agent' and 'Gmail' node as desired to steer the agent's behavior in the right direction. | Sticky Note near `send e-mail?` and `Gmail` nodes.                                              |
| Connect your Firecrawl account.                                                                                | Sticky Note near `scrape page - 1` and `scrape page - 2` nodes.                                 |
| Connect your own OpenAI account.                                                                                | Sticky Notes near `OpenAI Chat Model` and `send e-mail?` nodes.                                 |
| Connect your own Gmail account.                                                                                 | Sticky Note near `Gmail` node.                                                                  |
| Example use cases include tracking price changes, new blog posts, job postings, product launches, terms updates, and customer reviews. | Workflow description.                                                                            |
| When testing the workflow, a new browser tab opens to fill in espionage assignment details.                     | Workflow description.                                                                            |
| Cancel espionage assignments anytime in the executions tab.                                                    | Workflow description.                                                                            |

---

This documentation provides a detailed, structured reference to understand, reproduce, and maintain the "Monitor competitors' websites for changes with OpenAI and Firecrawl" workflow. It highlights key configurations, dependencies, and potential failure points to facilitate robust usage and customization.