Automated News Summarizer with GPT-4o + Email Delivery

https://n8nworkflows.xyz/workflows/automated-news-summarizer-with-gpt-4o---email-delivery-4864


# Automated News Summarizer with GPT-4o + Email Delivery

### 1. Workflow Overview

This workflow automates the daily summarization of top news headlines from the US using GPT-4o and sends the summarized news via email to a list of recipients stored in a Google Sheet. It is designed for use cases such as daily newsletter distribution, personalized news briefing, or automated content curation.

The workflow is logically grouped into the following blocks:

- **1.1 Scheduled Trigger and News Retrieval:** Periodically triggers the workflow and pulls the latest news headlines from a news API.
- **1.2 AI Processing and Summarization:** Uses an AI agent powered by GPT-4o to summarize the collected news headlines into five bullet points.
- **1.3 Recipient List Retrieval:** Fetches the email recipients from a Google Sheets document.
- **1.4 Email Delivery:** Sends personalized emails containing the summarized news to each recipient.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger and News Retrieval

- **Overview:**  
  This block initiates the workflow every 10 minutes and retrieves the latest top news headlines from the US using an HTTP request node.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Pull News

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates workflow execution on a fixed time interval.  
    - Configuration: Triggers every 10 minutes.  
    - Inputs: None (start node).  
    - Outputs: Connects to Pull News node.  
    - Edge Cases: Potential failure if the n8n instance is offline or schedule misconfigured.

  - **Pull News**  
    - Type: HTTP Request  
    - Role: Calls the NewsAPI to retrieve top headlines for the US.  
    - Configuration:  
      - URL: `https://newsapi.org/v2/top-headlines`  
      - Query Parameters: `country=us`, `apiKey=NEWS_API_KEY` (placeholder, must be replaced with a valid API key).  
    - Inputs: Triggered by Schedule Trigger.  
    - Outputs: Passes response JSON to AI Agent.  
    - Edge Cases:  
      - API rate limiting or invalid API key can cause HTTP errors.  
      - Network timeouts or malformed responses may cause failures.

#### 2.2 AI Processing and Summarization

- **Overview:**  
  This block leverages GPT-4o via an AI agent to process the fetched news headlines and summarize them concisely in five bullet points.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model

- **Node Details:**

  - **OpenAI Chat Model**  
    - Type: Language Model Chat (OpenAI)  
    - Role: Provides GPT-4o model capabilities for AI Agent.  
    - Configuration:  
      - Model: `gpt-4o-mini` (a GPT-4 variant optimized for cost/performance).  
      - Credentials: OpenAI API key configured via `OpenAi account`.  
    - Inputs: None (used as AI language model for AI Agent).  
    - Outputs: Connected internally to AI Agent.  
    - Edge Cases:  
      - API quota exhaustion or invalid credentials cause authentication errors.  
      - Model unavailability or network issues cause timeouts.

  - **AI Agent**  
    - Type: Langchain Agent (AI processing node)  
    - Role: Orchestrates AI prompt to summarize news headlines.  
    - Configuration:  
      - Text prompt: "You are act as news expert and collect all news headlines in below and summarise in 5 bullets. {{ $json.articles[1].title }}"  
      - Uses output from Pull News node (`articles` array) dynamically in prompt.  
      - Prompt Type: Define (custom prompt).  
      - Uses OpenAI Chat Model node as the backend language model.  
    - Inputs: Receives news data from Pull News.  
    - Outputs: Summarized news text, passed to Email list node.  
    - Edge Cases:  
      - Expression failures if `articles` or `title` fields are missing or empty.  
      - AI response may be incomplete or off-topic if prompt is malformed.

#### 2.3 Recipient List Retrieval

- **Overview:**  
  Retrieves the list of email recipients from a specified Google Sheets document.

- **Nodes Involved:**  
  - Email list

- **Node Details:**

  - **Email list**  
    - Type: Google Sheets  
    - Role: Reads recipient data (names and emails) from sheet "Sheet1" (gid=0).  
    - Configuration:  
      - Document ID: `1L2dLObdw_aicD6fTd-ElHSBBJIj1aBmKT7FecMtbyyE` (placeholder, must be replaced by actual Google Sheet ID).  
      - Sheet Name: `gid=0` (Sheet1).  
      - Credentials: Google Sheets OAuth2 account.  
    - Inputs: Receives summarized news output from AI Agent.  
    - Outputs: Passes recipient records to Send Mail node.  
    - Edge Cases:  
      - Permission issues if OAuth token is invalid or access revoked.  
      - Empty or malformed sheet data causing no recipients to be found.

#### 2.4 Email Delivery

- **Overview:**  
  Sends personalized emails to each recipient with the summarized news headlines.

- **Nodes Involved:**  
  - Send Mail

- **Node Details:**

  - **Send Mail**  
    - Type: Gmail (email sending)  
    - Role: Sends emails to each recipient with the news summary.  
    - Configuration:  
      - Send To: Dynamic expression `{{ $json.Email }}` from Google Sheets data.  
      - Subject: "News Headlines"  
      - Message Body:  
        ```
        Hi {{ $json.Name }},
        Please find the top news headlines as below:

        {{ $('AI Agent').item.json.output }}
        ```  
      - Email Type: Plain text.  
      - Credentials: Gmail OAuth2 account.  
    - Inputs: Recipient data from Email list node and AI Agent summary via expression.  
    - Outputs: None (end node).  
    - Edge Cases:  
      - OAuth token expiration or insufficient Gmail API scope causes authentication failure.  
      - Invalid email addresses cause delivery errors.  
      - Sending limits on Gmail account may cause throttling.

---

### 3. Summary Table

| Node Name      | Node Type                           | Functional Role                    | Input Node(s)       | Output Node(s)  | Sticky Note                                                |
|----------------|-----------------------------------|----------------------------------|---------------------|-----------------|------------------------------------------------------------|
| Schedule Trigger | Schedule Trigger                  | Initiates workflow every 10 mins | None                | Pull News       |                                                            |
| Pull News      | HTTP Request                      | Fetches top US news headlines    | Schedule Trigger    | AI Agent        |                                                            |
| AI Agent       | Langchain AI Agent                | Summarizes news headlines with GPT-4o | Pull News          | Email list      |                                                            |
| OpenAI Chat Model | OpenAI Chat Model (GPT-4o)       | Provides GPT-4o language model   | None (AI Agent backend) | AI Agent       | Requires OpenAI API credentials                            |
| Email list     | Google Sheets                    | Retrieves recipient emails        | AI Agent            | Send Mail       |                                                            |
| Send Mail      | Gmail                            | Sends summarized news emails      | Email list           | None            | Ensure Gmail OAuth2 credentials with send email permission |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set interval to trigger every 10 minutes.

2. **Create HTTP Request node "Pull News"**  
   - Type: HTTP Request  
   - URL: `https://newsapi.org/v2/top-headlines`  
   - Query Parameters:  
     - country = us  
     - apiKey = (Your News API key)  
   - Connect Schedule Trigger’s output to Pull News input.

3. **Create OpenAI Chat Model node**  
   - Type: Langchain OpenAI Chat Model  
   - Model: `gpt-4o-mini`  
   - Set OpenAI API credentials (API key under "OpenAi account").  
   - No direct input connections (used as backend).

4. **Create AI Agent node**  
   - Type: Langchain Agent  
   - Prompt:  
     ```
     You are act as news expert and collect all news headlines in below and summarise in 5 bullets.

     {{ $json.articles[1].title }}
     ```  
   - Set Prompt Type to "define".  
   - Configure AI Agent to use the OpenAI Chat Model node as the language model.  
   - Connect Pull News output to AI Agent input.

5. **Create Google Sheets node "Email list"**  
   - Type: Google Sheets  
   - Document ID: (Your Google Sheet ID containing recipients)  
   - Sheet Name: `Sheet1` (gid=0)  
   - Authenticate with Google Sheets OAuth2 credentials.  
   - Connect AI Agent output to Email list input.

6. **Create Gmail node "Send Mail"**  
   - Type: Gmail  
   - Set credentials with Gmail OAuth2 account with sending permissions.  
   - Parameters:  
     - Send To: `={{ $json.Email }}` (from Google Sheet)  
     - Subject: "News Headlines"  
     - Message:  
       ```
       Hi {{ $json.Name }},

       Please find the top news headlines as below:

       {{ $('AI Agent').item.json.output }}
       ```  
     - Email Type: Plain text.  
   - Connect Email list output to Send Mail input.

7. **Validate connections**  
   - Schedule Trigger → Pull News → AI Agent → Email list → Send Mail

8. **Test the workflow**  
   - Ensure API keys and OAuth credentials are valid.  
   - Confirm the Google Sheet contains columns "Email" and "Name".  
   - Trigger manually or wait for schedule to execute.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                                  |
|----------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| The NewsAPI key placeholder `NEWS_API_KEY` must be replaced with a valid API key from https://newsapi.org. | News API documentation                                           |
| Google Sheets document ID and sheet name must match your actual sheet containing recipient emails and names. | Google Sheets setup                                             |
| Gmail OAuth2 credentials must be configured with proper scopes for sending emails (https://developers.google.com/gmail/api/auth/scopes). | Gmail API authentication                                         |
| GPT-4o-mini is an OpenAI model variant optimized for cost and performance; ensure your OpenAI plan supports it. | OpenAI model info                                                |
| Expression in AI Agent uses `{{ $json.articles[1].title }}` which assumes the presence of at least two news articles in the response. Adjust if needed. | Potential expression failure if API response structure changes   |

---

**Disclaimer:**  
The provided content exclusively originates from an automated workflow created with n8n, an integration and automation tool. The processing fully complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.