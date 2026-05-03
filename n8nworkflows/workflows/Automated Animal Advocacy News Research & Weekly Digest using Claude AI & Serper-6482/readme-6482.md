Automated Animal Advocacy News Research & Weekly Digest using Claude AI & Serper

https://n8nworkflows.xyz/workflows/automated-animal-advocacy-news-research---weekly-digest-using-claude-ai---serper-6482


# Automated Animal Advocacy News Research & Weekly Digest using Claude AI & Serper

### 1. Workflow Overview

This workflow automates the weekly research, summarization, and email delivery of news updates related to animal advocacy topics using AI and real-time search tools. It is designed for advocacy groups or individuals interested in staying informed about recent developments in animal rights, veganism, and related areas.

The workflow is logically divided into these blocks:

- **1.1 Schedule Trigger:** Initiates the workflow automatically once a week.
- **1.2 User Preferences Setup:** Defines topics of interest, custom instructions for research, and the recipient email.
- **1.3 Research Agent Execution:** Invokes a specialized sub-workflow that performs news research using the Serper API and scraping tools, constrained to the past week and specific topics.
- **1.4 HTML Report Generation:** Uses Claude AI (via OpenRouter) to convert the raw research output into a well-formed HTML email report, strictly preserving all URLs.
- **1.5 Email Delivery:** Sends the generated HTML report to the designated recipient using SMTP.

Each block depends sequentially on the prior, forming a clean, automated information pipeline from trigger to email.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger

- **Overview:**  
Triggers the entire workflow once per week at a specified time (default noon).

- **Nodes Involved:**  
`Schedule Trigger`

- **Node Details:**  
  - **Type:** Schedule Trigger (Cron-based)  
  - **Configuration:**  
    - Interval: Weekly  
    - Trigger hour: 12:00 (noon)  
  - **Inputs:** None (trigger node)  
  - **Outputs:** Initiates `Set Preferences` node  
  - **Edge Cases:**  
    - Misconfigured schedule may cause no runs or multiple runs.  
    - Timezone differences may affect trigger time.  
  - **Sticky Note:** Provides recommendations for schedule settings; suggests a daily variant with modified parameters (`"tbs": "qdr:d"`) for daily research.

#### 2.2 User Preferences Setup

- **Overview:**  
Sets research topics, custom instructions on how to summarize the findings, and the recipient email address for the report.

- **Nodes Involved:**  
`Set Preferences`

- **Node Details:**  
  - **Type:** Set node (assigns static values)  
  - **Configuration:**  
    - Topics: List of animal advocacy topics (e.g., legal cases, undercover investigations, plant-based innovations).  
    - Custom Instructions: Guidelines for focusing on credible, high-impact updates in clear language.  
    - Recipient Email: Email address receiving the final report.  
  - **Inputs:** From `Schedule Trigger`  
  - **Outputs:** To `Call Research Agent`  
  - **Edge Cases:**  
    - Incorrect or missing email will cause email delivery failure.  
    - Topics or instructions poorly formulated may affect research quality.  
  - **Sticky Note:** Explains how to customize topics, instructions, and email.

#### 2.3 Research Agent Execution

- **Overview:**  
Calls a sub-workflow (“General Research Agent”) that performs multi-tool research using Serper API restricted to the past week, plus scraping to enrich results. It strictly follows URL handling rules.

- **Nodes Involved:**  
`Call Research Agent`

- **Node Details:**  
  - **Type:** Execute Workflow (sub-workflow call)  
  - **Configuration:**  
    - Workflow ID: Points to “General Research Agent” sub-workflow.  
    - Inputs:  
      - `chatInput`: Detailed prompt instructing research on past-week news, topics, and strict URL handling.  
      - `sessionId`: Random unique string per run for session tracking.  
    - Research constraints:  
      - Time filter `"tbs": "qdr:w"` (past week) for Serper API searches.  
      - Topics and custom instructions passed from `Set Preferences`.  
  - **Inputs:** From `Set Preferences`  
  - **Outputs:** To `Write HTML Report`  
  - **Edge Cases:**  
    - Sub-workflow failure or API rate limits can cause retries or failures.  
    - Scraper blocks or incomplete data are acknowledged but do not halt workflow.  
  - **Sticky Note:** Links to the sub-workflow documentation; highlights research scope and URL rules.

#### 2.4 HTML Report Generation

- **Overview:**  
Uses Claude AI (via OpenRouter) to transform the raw research output into a valid, well-structured HTML report for email, adhering strictly to URL fidelity rules.

- **Nodes Involved:**  
`Write HTML Report`

- **Node Details:**  
  - **Type:** Chain LLM node (LangChain with OpenRouter Claude model)  
  - **Configuration:**  
    - Model: `anthropic/claude-sonnet-4` via OpenRouter API  
    - Prompt instructions:  
      - Generate a full HTML document starting with `<!DOCTYPE html>`.  
      - Use only URLs explicitly provided, wrapped in `<a>` tags, no alterations or fabrications.  
      - Display “Link not available.” if any link is missing.  
      - Include user topics and custom instructions context.  
  - **Inputs:** From `Call Research Agent` output  
  - **Outputs:** To `Send email`  
  - **Edge Cases:**  
    - AI model may generate invalid HTML if prompt is misunderstood.  
    - API failures or rate limits may cause retries.  
  - **Sticky Note:** Explains strict URL and HTML generation rules.

#### 2.5 Email Delivery

- **Overview:**  
Sends the generated HTML report to the recipient email via SMTP.

- **Nodes Involved:**  
`Send email`

- **Node Details:**  
  - **Type:** Email Send node  
  - **Configuration:**  
    - Subject: “Weekly Update from Open Paws” (timestamped dynamically)  
    - To: Recipient email from `Set Preferences`  
    - From: Fixed sender email (`email@example.com`)  
    - Email body: HTML content from `Write HTML Report`  
    - SMTP Credentials: ProtonMail SMTP configured  
  - **Inputs:** From `Write HTML Report`  
  - **Outputs:** None (end node)  
  - **Edge Cases:**  
    - SMTP authentication errors or network issues can cause send failure.  
    - Invalid recipient email causes bounce or delivery failure.  
  - **Sticky Note:** Describes subject and content usage.

---

### 3. Summary Table

| Node Name           | Node Type                            | Functional Role                               | Input Node(s)          | Output Node(s)        | Sticky Note                                                                                           |
|---------------------|------------------------------------|-----------------------------------------------|------------------------|-----------------------|-----------------------------------------------------------------------------------------------------|
| Schedule Trigger     | Schedule Trigger                   | Triggers workflow weekly                       | None                   | Set Preferences        | # ⏰ Schedule Node (Cron)  Triggers the workflow weekly with recommended settings and tips for daily variant. |
| Set Preferences     | Set                                | Defines topics, instructions, recipient email | Schedule Trigger       | Call Research Agent    | # 🧩 Set Preferences  Explains how to customize topics, instructions, and email.                     |
| Call Research Agent  | Execute Workflow                   | Runs research sub-workflow using Serper API   | Set Preferences        | Write HTML Report      | # 🔎 Research Agent  Calls research sub-workflow; uses strict URL rules and past-week search.       |
| Write HTML Report    | Chain LLM (LangChain)              | Generates valid HTML report from research data| Call Research Agent    | Send email             | # 🧱 Write HTML  Converts research output to valid HTML email; preserves URLs exactly.               |
| Send email           | Email Send                        | Sends the HTML report via SMTP                 | Write HTML Report      | None                  | # ✉️ Send Email  Sends final report with subject and HTML body to recipient email.                   |
| Sticky Note1         | Sticky Note                       | Documentation                                  | None                   | None                  | See Schedule Trigger row                                                                             |
| Sticky Note2         | Sticky Note                       | Documentation                                  | None                   | None                  | See Set Preferences row                                                                              |
| Sticky Note3         | Sticky Note                       | Documentation                                  | None                   | None                  | See Call Research Agent row                                                                          |
| Sticky Note4         | Sticky Note                       | Documentation                                  | None                   | None                  | See Write HTML Report row                                                                            |
| Sticky Note5         | Sticky Note                       | Documentation                                  | None                   | None                  | See Send email row                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n and name it** “Weekly Animal Advocacy Brief”.

2. **Add a Schedule Trigger node:**  
   - Set mode to **Cron**  
   - Configure to trigger **weekly** (e.g., every Monday) at 12:00 (noon) or preferred time.

3. **Add a Set node named “Set Preferences”:**  
   - Create three fields:  
     - `Topics` (string):  
       *Example:* “Animal rights legal cases, undercover investigations in factory farming, corporate pledges to improve animal welfare, plant-based and cultivated meat innovations, veganism trends and campaigns, government policies affecting farmed animals.”  
     - `Custom Instructions` (string):  
       *Example:* “Focus on high-impact, verifiable updates that matter for animal advocacy campaigns. Prioritize credible sources (NGOs, major news outlets, scientific publications). Summarize key events in clear, non-technical language, and highlight how they affect animal rights or vegan advocacy efforts.”  
     - `Recipient Email` (string):  
       *Example:* “email@example.com” (replace with actual recipient)  
   - Connect `Schedule Trigger` output to this node input.

4. **Add an Execute Workflow node named “Call Research Agent”:**  
   - Set `Workflow ID` to the ID of the “General Research Agent” sub-workflow.  
   - Configure inputs mapping:  
     - `chatInput` (string) with this prompt:  
       ```
       Your job is to research news and updates strictly from the past week, only using the Serper API. You must set the time parameter as follows to ensure only past-week articles are retrieved:

       "tbs": "qdr:w" (past week)

       You are searching for news and articles on the following topics, but **only as they relate to animal rights, animal welfare, vegetarianism, and veganism**:

       {{ $json.Topics }}

       Also consider the additional context of the user's instructions for the type of information they want you to find:

       {{ $json['Custom Instructions'] }}

       The current date and time is {{ $now }}.

       **Your final answer should:**
       - Be as long and detailed as possible.
       - Include every direct URL you find, using exactly the same URL string returned by the tools.
       - Never modify, shorten, omit, or "clean up" URLs in any way.
       - Clearly show all URLs, even if some are repeated across different findings.

       **Research requirements:**
       - Make multiple API calls to Serper if needed to get complete coverage.
       - Use scraping tools (text scraping, URL scraping) on the sites you find to extract additional details, but never modify or fabricate URLs.
       - If a website blocks scraping, acknowledge it and continue with the available data (this does not mean the tool is broken).

       **Prohibited behavior:**
       - Never say "I don't have access to real-time news data or current internet content" or any similar disclaimer, as you do have tools for real-time news retrieval.
       - Never guess or invent URLs.

       **IMPORTANT:** Always include every URL exactly as found by the tools, without any alterations.
       ```  
     - `sessionId` (string): Set to unique string expression:  
       `={{ (Math.random().toString(36).substring(2) + Date.now().toString(36)) }}`  
   - Connect `Set Preferences` output to this node input.

5. **Add a Chain LLM node named “Write HTML Report”:**  
   - Choose model `anthropic/claude-sonnet-4` via OpenRouter API credentials.  
   - Set prompt with the following instructions (use expression to dynamically insert data):  
     ```
     Write an HTML formatted report on the latest updates relevant to the user based on the context provided and the user's request. 

     Rules for links:
     - You must ONLY use URLs explicitly provided in the context. 
     - Do NOT invent, guess, or modify any URLs. If a link is missing or incomplete, state: "Link not available."
     - Use the exact URL as it appears in the context without shortening, changing parameters, or cleaning them in any way.
     - When displaying a link, wrap it in an <a href="URL"> tag for presentation, but the URL must remain untouched.

     User topics interested in:
     {{ $('Set Preferences').item.json.Topics }}

     User instructions:
     {{ $('Set Preferences').item.json['Custom Instructions'] }}

     Context:
     {{ $json.output }}

     EXTREMELY IMPORTANT: Your response must be entirely valid HTML code and start directly with <!DOCTYPE html>. Do not include any backticks or code block formatting. 

     Check your response carefully to ensure:
     - Every hyperlink exactly matches a URL from the context.
     - There are no fabricated or missing URLs.
     - The HTML structure is correct and well-formed.
     ```  
   - Connect `Call Research Agent` output to this node input.

6. **Add an Email Send node named “Send email”:**  
   - Set SMTP credentials with your valid SMTP (example uses ProtonMail SMTP).  
   - Configure parameters:  
     - To Email: Expression referencing `Recipient Email` from `Set Preferences`:  
       `={{ $('Set Preferences').item.json['Recipient Email'] }}`  
     - From Email: Fixed email address (e.g., `email@example.com`)  
     - Subject: “Weekly Update from Open Paws” or dynamically include date if desired.  
     - HTML content: Use the output of `Write HTML Report`:  
       `={{ $json.text }}`  
   - Connect `Write HTML Report` output to this node input.

7. **(Optional) Add Sticky Note nodes for documentation:**  
   - Add notes as per blocks for clarity and maintenance.

8. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                 | Context or Link                                                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| The workflow uses the “General Research Agent” sub-workflow: This sub-workflow performs multi-tool research using Serper API and scraping tools. Ensure you have imported and configured it properly before using this main workflow.          | [General Research Agent Workflow](https://n8n.io/workflows/5588-multi-tool-research-agent-for-animal-advocacy-with-openrouter-serper-and-open-paws-db/) |
| Schedule node is recommended to run weekly on Monday at 09:00 or 12:00. For daily updates, duplicate the workflow and adjust the time parameter (`"tbs"`) to `"qdr:d"`.                                                                       | Sticky Note1 content                                                                                                                |
| Email sending requires valid SMTP credentials; ProtonMail is used in this example. Update credentials and sender email accordingly.                                                                                                        | Sticky Note5 content                                                                                                                |
| Strict URL handling throughout the workflow ensures no link is modified, shortened, or fabricated, preserving data integrity and transparency in reporting.                                                                                  | Sticky Notes 3 and 4 content                                                                                                       |
| Use clear, non-technical language in custom instructions to ensure accessible and impactful summaries for advocacy stakeholders.                                                                                                           | Sticky Note2 content                                                                                                                |

---

**Disclaimer:** The text provided is derived exclusively from an n8n workflow automation. It complies fully with all content policies and contains no illegal, offensive, or protected elements. All data handled are legal and publicly available.