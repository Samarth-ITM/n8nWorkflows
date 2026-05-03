🗞️ AI-Powered Sustainability Newsletter for Marketing with Gmail, GPT-4o

https://n8nworkflows.xyz/workflows/----ai-powered-sustainability-newsletter-for-marketing-with-gmail--gpt-4o-3684


# 🗞️ AI-Powered Sustainability Newsletter for Marketing with Gmail, GPT-4o

### 1. Workflow Overview

This workflow automates the creation and distribution of a daily sustainability-focused newsletter by scraping the EU Commission News Portal, classifying articles for sustainability relevance using OpenAI GPT-4o, storing results in Google Sheets, generating an HTML email digest, and sending it via Gmail. It is designed for business owners, sustainability professionals, consultants, and media curators who want to stay informed or promote sustainability-related content efficiently.

The workflow is logically divided into the following blocks:

- **1.1 Workflow Trigger**: Scheduled trigger to start the workflow daily at 08:30 AM.
- **1.2 Web Scraping and Parsing**: HTTP request to fetch the EU news page, extract article blocks, and parse article details.
- **1.3 Article Filtering and AI Classification**: Filter articles by date and type, then classify each article’s sustainability relevance using an AI agent with OpenAI GPT-4o.
- **1.4 Data Recording**: Append classified article data to a Google Sheet for tracking.
- **1.5 Email Generation and Sending**: Generate a professional HTML email digest from the stored articles and send it to a mailing list via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Workflow Trigger

- **Overview:**  
  Initiates the workflow automatically every morning at 08:30 AM local time.

- **Nodes Involved:**  
  - Trigger at 08:30 am

- **Node Details:**  
  - **Trigger at 08:30 am**  
    - Type: Schedule Trigger  
    - Configuration: Triggers daily at 08:30 (hour: 8, minute: 30)  
    - Inputs: None (start node)  
    - Outputs: Connects to "Query EU News Website" and "Get Sustainability News" nodes  
    - Edge Cases: Timezone differences may affect trigger time if n8n instance timezone is not set correctly.

---

#### 1.2 Web Scraping and Parsing

- **Overview:**  
  Fetches the EU Commission news webpage HTML, extracts article blocks, and parses key article information such as title, date, link, description, image, and reading time.

- **Nodes Involved:**  
  - Query EU News Website  
  - Extract Articles Blocks  
  - Split Out by Article Block  
  - Parse Article Blocks

- **Node Details:**  
  - **Query EU News Website**  
    - Type: HTTP Request  
    - Configuration: GET request to `https://commission.europa.eu/news-and-media/news_en`  
    - Inputs: Trigger node  
    - Outputs: Raw HTML to "Extract Articles Blocks"  
    - Edge Cases: HTTP errors, site downtime, or changes in website structure may break scraping.

  - **Extract Articles Blocks**  
    - Type: HTML Extract  
    - Configuration: Extracts HTML of each article block using CSS selector `div.ecl-content-item-block__item` returning an array of HTML snippets  
    - Inputs: HTML from HTTP Request  
    - Outputs: Array of article block HTML to "Split Out by Article Block"  
    - Edge Cases: Changes in CSS selectors on source site may cause extraction failure.

  - **Split Out by Article Block**  
    - Type: Split Out  
    - Configuration: Splits array of article blocks into individual items for processing  
    - Inputs: Array of article blocks  
    - Outputs: Individual article blocks to "Parse Article Blocks"  
    - Edge Cases: Empty article list results in no further processing.

  - **Parse Article Blocks**  
    - Type: HTML Extract  
    - Configuration: Extracts article details from each HTML block using multiple CSS selectors:  
      - type: `ul.ecl-content-block__primary-meta-container li:nth-child(1)`  
      - date: `ul.ecl-content-block__primary-meta-container li:nth-child(2) time`  
      - title: `div.ecl-content-block__title a`  
      - link (attribute href): `div.ecl-content-block__title a`  
      - description: `div.ecl-content-block__description p`  
      - image (attribute src): `picture img`  
      - read_time: `ul.ecl-content-block__secondary-meta-container span.ecl-content-block__secondary-meta-label`  
    - Inputs: Single article HTML block  
    - Outputs: Parsed article JSON to "If" node  
    - Edge Cases: Missing fields (e.g., no image or description) handled gracefully; CSS selector changes may break extraction.

---

#### 1.3 Article Filtering and AI Classification

- **Overview:**  
  Filters articles by date (exactly 5 days ago) and type ("News article"), then loops through each article to classify sustainability relevance using an AI agent powered by OpenAI GPT-4o.

- **Nodes Involved:**  
  - If  
  - Loop Over Articles  
  - OpenAI Chat Model3  
  - Agent Classification  
  - Structured Output Parser  
  - Sustainability Flag  
  - Merge Article + Flag

- **Node Details:**  
  - **If**  
    - Type: If Condition  
    - Configuration: Checks if article date equals exactly 5 days ago (formatted as "day month year") and type equals "News article"  
    - Inputs: Parsed article JSON  
    - Outputs: True branch to "Loop Over Articles" (process articles matching criteria)  
    - Edge Cases: Date format mismatch or missing date/type fields may cause false negatives.

  - **Loop Over Articles**  
    - Type: Split In Batches  
    - Configuration: Processes articles one by one for classification  
    - Inputs: Filtered articles from "If" node  
    - Outputs: To "Agent Classification" and "Merge Article + Flag"  
    - Edge Cases: Large article volumes may cause longer processing times or API rate limits.

  - **OpenAI Chat Model3**  
    - Type: LangChain OpenAI Chat Model  
    - Configuration: Uses GPT-4o-mini model for classification  
    - Inputs: Connected as AI language model for "Agent Classification"  
    - Outputs: AI responses to "Agent Classification"  
    - Credential: Requires OpenAI API key with GPT-4o access  
    - Edge Cases: API quota exceeded, network errors, or malformed prompts.

  - **Agent Classification**  
    - Type: LangChain Agent  
    - Configuration:  
      - Prompt includes article title and description, asking if article is about sustainability.  
      - System message instructs to return JSON with `{"answer": true}` or `{"answer": false}` only.  
      - Output parser enabled to parse structured JSON response.  
    - Inputs: Article data and AI model output  
    - Outputs: Classification result to "Sustainability Flag"  
    - Edge Cases: AI misclassification, empty descriptions, or parsing errors.

  - **Structured Output Parser**  
    - Type: LangChain Output Parser Structured  
    - Configuration: JSON schema expects `{"answer": "boolean | null"}`  
    - Inputs: AI output from "Agent Classification"  
    - Outputs: Parsed boolean answer for sustainability flag  
    - Edge Cases: Parsing failures if AI response deviates from expected format.

  - **Sustainability Flag**  
    - Type: Set  
    - Configuration: Sets a new field `sustainability` with the AI classification result (`true` or `false`)  
    - Inputs: Parsed classification result  
    - Outputs: To "Merge Article + Flag"  
    - Edge Cases: Missing or invalid classification data.

  - **Merge Article + Flag**  
    - Type: Merge  
    - Configuration: Combines original article data with sustainability flag using SQL combine mode  
    - Inputs: Article data and sustainability flag  
    - Outputs: To "Record Results"  
    - Edge Cases: Data mismatch or merge conflicts.

---

#### 1.4 Data Recording

- **Overview:**  
  Appends the sustainability-classified articles to a Google Sheet for record-keeping and further analysis.

- **Nodes Involved:**  
  - Record Results

- **Node Details:**  
  - **Record Results**  
    - Type: Google Sheets  
    - Configuration:  
      - Operation: Append rows  
      - Document ID and Sheet Name configured to target the desired Google Sheet and sheet tab  
      - Columns mapped: `date`, `link`, `type`, `image`, `title`, `read_time`, `description`, `sustainability`  
    - Inputs: Merged article data with sustainability flag  
    - Outputs: To "Loop Over Articles" (loop continuation)  
    - Credential: Requires Google Sheets API credentials with write access  
    - Edge Cases: API quota limits, permission errors, or invalid sheet IDs.

---

#### 1.5 Email Generation and Sending

- **Overview:**  
  Retrieves all articles flagged as sustainability-related from the Google Sheet, generates a styled HTML email digest, and sends it to a mailing list via Gmail.

- **Nodes Involved:**  
  - Get Sustainability News  
  - Generate Email HTML  
  - Send to your mailing list

- **Node Details:**  
  - **Get Sustainability News**  
    - Type: Google Sheets  
    - Configuration:  
      - Operation: Read rows with filter where `sustainability` equals `"true"`  
      - Document ID and Sheet Name configured to the same Google Sheet as "Record Results"  
    - Inputs: Trigger node (parallel to HTTP request)  
    - Outputs: To "Generate Email HTML"  
    - Credential: Google Sheets API credentials with read access  
    - Edge Cases: Empty results if no articles flagged true.

  - **Generate Email HTML**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Generates a responsive HTML email body including:  
        - Header with date and branding link  
        - Article blocks with image, title (linked), description, type, date, and read time  
        - Footer with attribution and company logo  
      - Uses inline CSS for styling and safe email rendering  
    - Inputs: Array of sustainability articles from Google Sheets  
    - Outputs: JSON with `email_body` field to "Send to your mailing list"  
    - Edge Cases: Missing images or descriptions handled gracefully; malformed data may affect rendering.

  - **Send to your mailing list**  
    - Type: Gmail  
    - Configuration:  
      - Recipient email(s) configured (e.g., `email@gmail.com`)  
      - Subject: "Your Sustainability News Digest from LogiGreen"  
      - Message body: Uses generated HTML from previous node  
      - Option to disable Gmail attribution appended  
    - Inputs: Email body JSON  
    - Credential: Gmail OAuth2 credentials with send email permission  
    - Edge Cases: Gmail API quota, authentication errors, invalid recipient addresses.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                       | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                      |
|-------------------------|----------------------------------|------------------------------------|-------------------------------|--------------------------------|------------------------------------------------------------------------------------------------|
| Trigger at 08:30 am     | Schedule Trigger                 | Starts workflow daily at 08:30 AM  | None                          | Query EU News Website, Get Sustainability News | ### 1. Workflow Trigger with Cron Job<br>The workflow is triggered every morning at 08:30 am (local time)<br>How to setup? Select the time you want to set it up |
| Query EU News Website   | HTTP Request                    | Fetches EU news webpage HTML       | Trigger at 08:30 am           | Extract Articles Blocks         | ### 2. Scrapping and Parsing of Articles blocks<br>This starts with the HTTP node collecting HTML code that is parsed to extract Article Titles, Link, Image Cover and Reading time.<br>Nothing to do |
| Extract Articles Blocks | HTML Extract                    | Extracts article blocks HTML       | Query EU News Website         | Split Out by Article Block      | See above                                                                                      |
| Split Out by Article Block | Split Out                    | Splits article blocks into items   | Extract Articles Blocks       | Parse Article Blocks            | See above                                                                                      |
| Parse Article Blocks    | HTML Extract                    | Parses article details             | Split Out by Article Block    | If                            | See above                                                                                      |
| If                      | If Condition                   | Filters articles by date and type | Parse Article Blocks          | Loop Over Articles              | ### 3. Classifiy all the articles (Sustainability: true or false)<br>This starts with the If node that filters based on the scope date fixed by you. Through the loop, the AI Agent classify the articles using the title and description.<br>The ones that are flagged as "sustainability" are recorded in a Google Sheet.<br>How to setup? See sticky note content |
| Loop Over Articles      | Split In Batches               | Processes articles one by one      | If                           | Agent Classification, Merge Article + Flag | See above                                                                                      |
| OpenAI Chat Model3      | LangChain OpenAI Chat Model    | Provides AI model for classification | Connected internally to Agent Classification | Agent Classification           | See above                                                                                      |
| Agent Classification    | LangChain Agent                | Classifies sustainability relevance | Loop Over Articles, OpenAI Chat Model3 | Sustainability Flag            | See above                                                                                      |
| Structured Output Parser | LangChain Output Parser Structured | Parses AI JSON response           | Agent Classification          | Agent Classification           | See above                                                                                      |
| Sustainability Flag     | Set                           | Sets sustainability flag field    | Agent Classification          | Merge Article + Flag            | See above                                                                                      |
| Merge Article + Flag    | Merge                         | Combines article data with flag   | Loop Over Articles, Sustainability Flag | Record Results                 | See above                                                                                      |
| Record Results          | Google Sheets                  | Appends article data to sheet     | Merge Article + Flag          | Loop Over Articles             | See above                                                                                      |
| Get Sustainability News | Google Sheets                  | Reads sustainability articles     | Trigger at 08:30 am           | Generate Email HTML             | See above                                                                                      |
| Generate Email HTML     | Code                          | Generates HTML email body          | Get Sustainability News       | Send to your mailing list       | ### 4. Generate HTML page and send by email<br>This block collects all the articles of the day to create a prettified HTML page that is sent using the Gmail node.<br>How to setup? Gmail Node: set up your Gmail API credentials<br>[Learn more about the Gmail Trigger Node] |
| Send to your mailing list | Gmail                        | Sends email digest                 | Generate Email HTML           | None                          | See above                                                                                      |
| Sticky Note1            | Sticky Note                   | Notes on workflow trigger          | None                         | None                          | See content in sticky note                                                                     |
| Sticky Note2            | Sticky Note                   | Notes on scraping and parsing      | None                         | None                          | See content in sticky note                                                                     |
| Sticky Note3            | Sticky Note                   | Notes on classification and recording | None                         | None                          | See content in sticky note                                                                     |
| Sticky Note              | Sticky Note                   | Notes on email generation and sending | None                         | None                          | See content in sticky note                                                                     |
| Sticky Note4            | Sticky Note                   | Link to tutorial video             | None                         | None                          | [Check the Tutorial](https://www.youtube.com/watch?v=q8VCAUbuat8)                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 08:30 AM (local time).

2. **Add an HTTP Request node**  
   - Name: Query EU News Website  
   - Method: GET  
   - URL: `https://commission.europa.eu/news-and-media/news_en`  
   - Connect Trigger node output to this node.

3. **Add an HTML Extract node**  
   - Name: Extract Articles Blocks  
   - Operation: Extract HTML content  
   - Extraction:  
     - Key: `articles`  
     - CSS Selector: `div.ecl-content-item-block__item`  
     - Return Array: true  
     - Return Value: html  
   - Connect HTTP Request output to this node.

4. **Add a Split Out node**  
   - Name: Split Out by Article Block  
   - Field to split out: `articles`  
   - Connect Extract Articles Blocks output to this node.

5. **Add another HTML Extract node**  
   - Name: Parse Article Blocks  
   - Operation: Extract HTML content  
   - Extraction values:  
     - `type`: `ul.ecl-content-block__primary-meta-container li:nth-child(1)`  
     - `date`: `ul.ecl-content-block__primary-meta-container li:nth-child(2) time`  
     - `title`: `div.ecl-content-block__title a`  
     - `link` (attribute href): `div.ecl-content-block__title a`  
     - `description`: `div.ecl-content-block__description p`  
     - `image` (attribute src): `picture img`  
     - `read_time`: `ul.ecl-content-block__secondary-meta-container span.ecl-content-block__secondary-meta-label`  
   - Connect Split Out node output to this node.

6. **Add an If node**  
   - Name: If  
   - Condition:  
     - Left Value: `{{$json.date}}`  
     - Operator: equals  
     - Right Value: `={{ $now.minus(5,"day").day }} {{ $now.minus(5,"day").monthLong }} {{ $now.minus(5,"day").year }}`  
     - AND  
     - Left Value: `{{$json.type}}`  
     - Operator: equals  
     - Right Value: `News article`  
   - Connect Parse Article Blocks output to this node.

7. **Add a Split In Batches node**  
   - Name: Loop Over Articles  
   - Connect If node's true output to this node.

8. **Add a LangChain OpenAI Chat Model node**  
   - Name: OpenAI Chat Model3  
   - Model: `gpt-4o-mini`  
   - Configure OpenAI credentials with GPT-4o access.

9. **Add a LangChain Agent node**  
   - Name: Agent Classification  
   - Text:  
     ```
     Title: {{$json.title}}
     Description: {{$json.description}}

     Is this article about sustainability? Return only: true or false
     ```  
   - System Message:  
     ```
     You are a classification assistant.

     Your role is to analyze the title and description of an article and determine if it is related to sustainability.

     You must only return {"answer": true} if the article is clearly related to sustainability (e.g., environmental protection, renewable energy, sustainable development, climate action, green economy, etc.).

     If it is not clearly related, return {"answer": false}.

     If the description is empty or missing, rely only on the title. Your response must be only one of the two JSON options: {"answer": true} or {"answer": false}. Do not provide explanations.
     ```  
   - Connect OpenAI Chat Model3 as the AI language model input.

10. **Add a LangChain Output Parser Structured node**  
    - Name: Structured Output Parser  
    - JSON Schema Example:  
      ```
      {
        "answer": "boolean | null"
      }
      ```  
    - Connect Agent Classification AI output to this node.

11. **Connect Agent Classification output parser to Agent Classification node’s output parser input.**

12. **Add a Set node**  
    - Name: Sustainability Flag  
    - Set field `sustainability` to `{{$json.output.answer}}` (boolean true/false)  
    - Connect Agent Classification output to this node.

13. **Add a Merge node**  
    - Name: Merge Article + Flag  
    - Mode: Combine by SQL  
    - Connect Loop Over Articles first output and Sustainability Flag output to this node.

14. **Add a Google Sheets node**  
    - Name: Record Results  
    - Operation: Append  
    - Document ID: Your Google Sheet ID  
    - Sheet Name: Your target sheet (e.g., `gid=0`)  
    - Map columns: `date`, `link`, `type`, `image`, `title`, `read_time`, `description`, `sustainability`  
    - Connect Merge Article + Flag output to this node.  
    - Configure Google Sheets API credentials with write access.

15. **Connect Record Results node output back to Loop Over Articles second input to continue batch processing.**

16. **Add another Google Sheets node**  
    - Name: Get Sustainability News  
    - Operation: Read rows with filter  
    - Filter: `sustainability` equals `"true"`  
    - Document ID and Sheet Name: same as Record Results  
    - Connect Trigger at 08:30 am node output to this node (parallel to HTTP Request).

17. **Add a Code node**  
    - Name: Generate Email HTML  
    - JavaScript code:  
      ```js
      const summary = `Welcome to the EU Sustainability News Digest provided by <a href="https://logi-green.com" style="color: #0077cc; text-decoration: none;">LogiGreen Consulting</a>.`;

      const articles = items.map(item => item.json);

      let html = `
      <div style="font-family: Arial, sans-serif; max-width: 700px; margin: auto;">
        <h2 style="color: #2c3e50;">🌍 EU News Digest – ${new Date().toLocaleDateString('en-GB', { day: 'numeric', month: 'long', year: 'numeric' })}</h2>
        <p style="font-size: 16px; color: #333;">${summary}</p>
        <hr style="border: 1px solid #eee;" />
      `;

      for (const article of articles) {
        const link = article.link.startsWith("http") ? article.link : `https://ec.europa.eu${article.link}`;
        html += `
          <div style="display: flex; margin: 20px 0; border-bottom: 1px solid #ddd; padding-bottom: 15px;">
            ${article.image ? `<img src="${article.image}" alt="" width="150" style="margin-right: 15px; border-radius: 6px; object-fit: cover;" />` : ''}
            <div>
              <p style="margin: 0; font-size: 12px; color: #888;">${article.type} | ${article.date}</p>
              <h3 style="margin: 5px 0;">
                <a href="${link}" style="text-decoration: none; color: #0077cc;">${article.title}</a>
              </h3>
              <p style="margin: 5px 0; color: #333;">${article.description || ''}</p>
              ${article.read_time ? `<p style="font-size: 12px; color: gray;">${article.read_time}</p>` : ''}
            </div>
          </div>
        `;
      }

      html += `
        <div style="margin-top: 40px; padding-top: 20px; border-top: 1px solid #eee; text-align: center;">
          <p style="font-size: 12px; color: #999;">You received this email as part of the EU Sustainability News Digest project.</p>
          <a href="https://logi-green.com" target="_blank">
            <img src="https://www.logi-green.com/web/image/website/1/logo/LogiGreen%20Consulting?unique=e2af3c6" alt="LogiGreen Consulting Logo" style="height: 40px; margin-top: 10px;" />
          </a>
        </div>
      </div>
      `;

      return [{ json: { email_body: html } }];
      ```
    - Connect Get Sustainability News output to this node.

18. **Add a Gmail node**  
    - Name: Send to your mailing list  
    - Operation: Send Email  
    - Recipient(s): Your mailing list email(s) (e.g., `email@gmail.com`)  
    - Subject: "Your Sustainability News Digest from LogiGreen"  
    - Message: Use expression to set message body to `{{$json.email_body}}` from previous node  
    - Disable Gmail attribution if desired  
    - Connect Generate Email HTML output to this node.  
    - Configure Gmail OAuth2 credentials with send email permission.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow is triggered every morning at 08:30 AM local time.                                                    | Sticky Note1                                                                                     |
| Scraping uses CSS selectors specific to EU Commission News Portal; changes in website structure may require updates. | Sticky Note2                                                                                     |
| AI classification uses OpenAI GPT-4o-mini with a strict JSON response format to ensure reliable parsing.       | Sticky Note3                                                                                     |
| Gmail node requires OAuth2 credentials; ensure API access is configured correctly to send emails.              | Sticky Note (4)                                                                                  |
| Video tutorial available for detailed setup and deployment: [Watch My Tutorial](https://www.youtube.com/watch?v=q8VCAUbuat8) | Sticky Note4                                                                                     |
| Workflow built with n8n version 1.85.4; compatibility with other versions may vary.                            | Metadata                                                                                        |
| Branding and logo used in email are from LogiGreen Consulting: https://logi-green.com                          | Email generation node and general branding                                                     |
| Google Sheets node documentation: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets | Sticky Note3                                                                                     |

---

This document provides a detailed, structured reference to understand, reproduce, and maintain the "🗞️ AI-Powered Sustainability Newsletter for Marketing with Gmail, GPT-4o" workflow. It covers all nodes, configurations, and integration points, enabling advanced users and AI agents to work effectively with the workflow.