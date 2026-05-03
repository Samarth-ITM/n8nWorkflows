Generate and Send AI News Newsletters Automatically with GPT & Gmail

https://n8nworkflows.xyz/workflows/generate-and-send-ai-news-newsletters-automatically-with-gpt---gmail-4847


# Generate and Send AI News Newsletters Automatically with GPT & Gmail

### 1. Workflow Overview

This n8n workflow automates the generation and sending of a weekly AI news newsletter using AI-generated content and Gmail for distribution. It is designed to run on a scheduled basis (daily or weekly) to:

- Fetch and summarize the latest developments in artificial intelligence
- Format the content into a visually appealing HTML email newsletter
- Send the newsletter automatically to a predefined recipient list via Gmail

The workflow is logically divided into the following blocks:

**1.1 Scheduling and Date Setup**  
Trigger the workflow on a schedule and prepare date variables used for content generation or filtering.

**1.2 AI Content Generation**  
Use a structured prompt to instruct an AI agent (via Azure OpenAI GPT-4 model) to generate a detailed, multi-section AI newsletter based on the latest news and trends.

**1.3 Email Formatting**  
Transform the raw AI-generated text into a polished HTML email with proper styling and a professional layout, including subject line generation.

**1.4 Email Sending**  
Send the formatted newsletter via Gmail using OAuth2 credentials to a specified recipient.

Supporting the workflow are sticky notes that provide documentation and setup instructions.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Scheduling and Date Setup

**Overview:**  
This block initiates the workflow on a scheduled interval and prepares relevant date variables for the AI prompt or metadata.

**Nodes Involved:**  
- Schedule Trigger  
- Set Dates1

**Node Details:**  

- **Schedule Trigger**  
  - *Type & Role:* Schedule Trigger node; initiates the workflow automatically based on time.  
  - *Configuration:* Runs at a daily interval by default (configured via the empty interval array, which defaults to daily).  
  - *Inputs/Outputs:* No input; output triggers "Set Dates1".  
  - *Edge Cases:* Misconfiguration can cause the workflow not to run; time zone considerations may affect execution time.  
  - *Sticky Note:* Setup Instructions mention modifying this node to change schedule time.

- **Set Dates1**  
  - *Type & Role:* Set node; assigns current date and a date 7 days prior to variables for use downstream.  
  - *Configuration:*  
    - `currentDate` set to current date in "yyyy-MM-dd" format.  
    - `lastweekDate` set to 7 days before current date.  
  - *Expressions Used:* `$now.format('yyyy-MM-dd')` and `$now.minus({days:7}).format('yyyy-MM-dd')`.  
  - *Inputs/Outputs:* Triggered by Schedule Trigger; outputs to AI Agent.  
  - *Edge Cases:* Date formatting errors unlikely, but incorrect time zone on server could lead to off-by-one day issues.

---

#### 2.2 AI Content Generation

**Overview:**  
This block uses Azure OpenAI GPT-4 via LangChain agent node to generate a structured AI newsletter based on a detailed prompt specifying newsletter sections and style.

**Nodes Involved:**  
- Azure OpenAI Chat Model  
- AI Agent

**Node Details:**  

- **Azure OpenAI Chat Model**  
  - *Type & Role:* Language model node (LangChain integration) connecting to Azure OpenAI GPT-4.  
  - *Configuration:* Model set to "gpt-4.1".  
  - *Credentials:* Azure OpenAI API credentials configured.  
  - *Inputs/Outputs:* Provides language model interface to AI Agent node.  
  - *Edge Cases:* API key expiration, rate limits, network timeouts, or model unavailability could cause failures.

- **AI Agent**  
  - *Type & Role:* LangChain Agent node; uses a structured system prompt and user prompt to generate newsletter content following a multi-section format.  
  - *Configuration:*  
    - Detailed prompt defines newsletter structure: title, editor’s note, AI main story, headlines, tool of the week, learning corner, prompt of the week, AI stat, community shoutout, and next week preview.  
    - Uses markdown formatting and instructs tone and style.  
  - *Inputs/Outputs:* Receives date variables as input context; sends output JSON to "Format Email Content".  
  - *Edge Cases:* Improper prompt could yield irrelevant or incomplete content. API errors or malformed responses could cause node failures.

---

#### 2.3 Email Formatting

**Overview:**  
This block formats the AI-generated newsletter text into a polished, styled HTML email and generates the email subject line and plain text alternative.

**Nodes Involved:**  
- Format Email Content

**Node Details:**  

- **Format Email Content**  
  - *Type & Role:* Code node running JavaScript to transform AI output into email-ready content.  
  - *Configuration:*  
    - Extracts AI output text, cleans whitespace, converts markdown-like syntax to HTML tags with inline styles.  
    - Generates a dynamic email subject line with the current date: "🤖 AI Weekly Pulse - [Date]: ChatGPT Voice Update & Latest AI News".  
    - Builds both plain text and HTML versions of the email body with professional styling and branding.  
  - *Expressions Used:* Uses `$input.all()[0].json.output` or fallback properties to extract AI text.  
  - *Inputs/Outputs:* Input from AI Agent; output to Gmail1 node.  
  - *Edge Cases:* If AI output is malformed or empty, the email content may be broken or incomplete. Errors in regex replacements could break HTML formatting.

---

#### 2.4 Email Sending

**Overview:**  
This block sends the formatted newsletter email to recipients using Gmail with OAuth2 authentication.

**Nodes Involved:**  
- Gmail1

**Node Details:**  

- **Gmail1**  
  - *Type & Role:* Gmail node; sends email messages.  
  - *Configuration:*  
    - Sends email to a fixed recipient address: `jyothi.swarup@techdome.net.in`.  
    - Uses the subject and HTML content generated by the previous node.  
    - Email body is in HTML format with an optional plain text fallback.  
  - *Credentials:* Uses Gmail OAuth2 credentials for authentication.  
  - *Inputs/Outputs:* Input from "Format Email Content". No further output nodes.  
  - *Edge Cases:* Authentication failures, quota limits, or network issues could prevent email sending. Recipient email hardcoded; lack of dynamic recipient handling limits scalability.

---

#### 2.5 Documentation and Setup Notes (Sticky Notes)

**Overview:**  
Provides user guidance on workflow purpose, configuration, and customization.

**Nodes Involved:**  
- Workflow Documentation (sticky note)  
- Setup Instructions (sticky note)

**Node Details:**  

- **Workflow Documentation**  
  - *Type & Role:* Sticky Note; describes workflow purpose, schedule, news categories, and setup requirements.  
  - *Content Highlights:* Runs daily at 9 AM, fetches AI news via Perplexity (though Perplexity node is not explicitly present in this workflow), categorizes content, formats, and sends email.  
  - *Usage:* Reference for users to understand workflow scope.

- **Setup Instructions**  
  - *Type & Role:* Sticky Note; outlines configuration steps including API keys, Gmail credentials, timing adjustments, and prompt customization.

---

### 3. Summary Table

| Node Name           | Node Type                            | Functional Role                    | Input Node(s)           | Output Node(s)        | Sticky Note                                                                                  |
|---------------------|------------------------------------|----------------------------------|------------------------|-----------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger     | Schedule Trigger                   | Initiates workflow on schedule   |                        | Set Dates1            | Modify schedule time in "Daily Trigger" node (Setup Instructions sticky note)                |
| Set Dates1          | Set                                | Sets current and last week dates | Schedule Trigger       | AI Agent              |                                                                                              |
| Azure OpenAI Chat Model | LangChain Azure OpenAI Chat Model | Provides GPT-4 model access      |                        | AI Agent (ai_languageModel) |                                                                                              |
| AI Agent            | LangChain Agent                    | Generates newsletter content     | Set Dates1, Azure OpenAI Chat Model | Format Email Content | Change categories in AI Agent prompt (Setup Instructions sticky note)                        |
| Format Email Content | Code                              | Formats AI output into email     | AI Agent               | Gmail1                | Email should be in HTML format (Setup Instructions sticky note)                              |
| Gmail1              | Gmail                             | Sends newsletter email           | Format Email Content   |                       | Add your Gmail Credentials (Setup Instructions sticky note)                                  |
| Workflow Documentation | Sticky Note                      | Workflow purpose and overview    |                        |                       | Detailed workflow purpose, logic, and setup notes                                           |
| Setup Instructions  | Sticky Note                       | Configuration and customization  |                        |                       | Steps for API key setup, email, timing, and content customization                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**
   - Set interval to run daily at 9 AM or desired time.
   - No additional parameters needed.

3. **Add a Set node named "Set Dates1":**
   - Add two variables:  
     - `currentDate` with value `={{ $now.format('yyyy-MM-dd') }}`  
     - `lastweekDate` with value `={{ $now.minus({days: 7}).format('yyyy-MM-dd') }}`
   - Connect Schedule Trigger’s output to this node.

4. **Add an Azure OpenAI Chat Model node:**
   - Set model to `gpt-4.1`.  
   - Configure credentials with your Azure OpenAI API key.
   - No direct input connections here (it's linked via AI Agent).

5. **Add a LangChain Agent node named "AI Agent":**
   - Set `text` parameter with the detailed newsletter prompt specifying newsletter structure and style (as described in section 2.2).  
   - Set the system message to instruct the AI as a newsletter writing assistant.  
   - In the "AI Language Model" credential configuration, select the Azure OpenAI Chat Model node created in step 4.  
   - Connect "Set Dates1" output to this node input (to provide date context).  
   - Set output to send generated JSON content downstream.

6. **Add a Code node named "Format Email Content":**
   - Paste the JavaScript code that:  
     - Extracts AI output, cleans and formats text.  
     - Converts markdown-like syntax to styled HTML with inline CSS.  
     - Generates a dynamic subject line with the current date.  
     - Builds both plain text and HTML email bodies.  
   - Connect "AI Agent" output to this node.

7. **Add a Gmail node named "Gmail1":**
   - Set "Send To" to your recipient’s email address (e.g., `jyothi.swarup@techdome.net.in`).  
   - Set "Subject" to `={{ $json.subject }}`.  
   - Set message body to `={{ $json.html }}` for rich HTML content.  
   - Configure Gmail OAuth2 credentials for authentication.  
   - Connect "Format Email Content" output to this node.

8. **Add Sticky Notes for documentation and setup instructions (optional but recommended):**
   - Workflow Documentation: Describe overall workflow purpose and logic.  
   - Setup Instructions: Outline API key setup, Gmail credentials, scheduling, and prompt customization.

9. **Activate the workflow and test manually or wait for scheduled trigger.**

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow runs daily at 9 AM by default; adjust schedule in Schedule Trigger node.                           | See Setup Instructions sticky note                                                              |
| Requires Azure OpenAI API key and Gmail OAuth2 credentials.                                               | Credentials must be set up in n8n credentials manager                                            |
| The AI Agent prompt defines newsletter sections: title, editor note, main story, headlines, tools, etc.   | Prompt is customizable for tone, style, and content focus                                        |
| The email formatting code converts markdown-like AI output into styled HTML with inline CSS for readability.| Located in "Format Email Content" node                                                          |
| The recipient email address is currently hardcoded in the Gmail node; modify for multiple recipients as needed.| Gmail1 node configuration                                                                        |
| The workflow mentions Perplexity API in the documentation, but no Perplexity node is included in this version.| Possibly from an earlier version or intended extension                                           |
| For unsubscribe or replies, recipients can reply directly to the sent email address.                       | Included in email footer text                                                                    |

---

**Disclaimer:** The provided text and workflow are generated exclusively from an automated n8n workflow. All data and content comply with applicable content policies and contain no illegal, offensive, or protected elements. Data processed is legal and publicly available.