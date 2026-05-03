Daily Financial News Summary with Ollama LLM - Automated Email Report

https://n8nworkflows.xyz/workflows/daily-financial-news-summary-with-ollama-llm---automated-email-report-5405


# Daily Financial News Summary with Ollama LLM - Automated Email Report

### 1. Workflow Overview

This workflow automates the daily collection, summarization, and email distribution of financial news headlines using n8n and an AI language model (Ollama LLM). It targets financial analysts, market teams, or newsletter publishers who require a concise, investor-friendly digest of key market news every morning.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger:** Initiates the process daily at 7 AM.
- **1.2 News Data Acquisition:** Fetches the Financial Times homepage, waits briefly to ensure full content load, then extracts relevant headlines and news sections via HTML parsing.
- **1.3 Data Cleaning & Preparation:** Consolidates and formats the extracted news content into a structured text block.
- **1.4 AI Summarization:** Uses a Langchain Ollama LLM agent to generate a concise summary with investor insights.
- **1.5 Email Dispatch:** Sends the generated summary as a plain-text email to a predefined recipient list.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block schedules the workflow to run automatically every day at 7:00 AM.

- **Nodes Involved:**  
  - Schedule Daily Trigger

- **Node Details:**  
  - **Schedule Daily Trigger**  
    - Type: Schedule Trigger (n8n-nodes-base.scheduleTrigger)  
    - Configuration: Trigger set to fire daily at hour 7 (7 AM).  
    - Inputs: None (trigger node).  
    - Outputs: Connects to "Fetch Financial News Webpage".  
    - Edge Cases: Workflow will not run if n8n server is offline or the scheduler is disabled.  
    - Notes: Ensures timely daily execution.

#### 1.2 News Data Acquisition

- **Overview:**  
  This block fetches the latest financial news webpage, waits for content to fully load, then extracts selected headlines and news sections using CSS selectors.

- **Nodes Involved:**  
  - Fetch Financial News Webpage  
  - Delay to Ensure Page Load  
  - Extract News Headlines & Text

- **Node Details:**

  - **Fetch Financial News Webpage**  
    - Type: HTTP Request (n8n-nodes-base.httpRequest)  
    - Configuration: GET request to https://www.ft.com/ with 10-second timeout.  
    - Inputs: From "Schedule Daily Trigger".  
    - Outputs: HTML content to "Delay to Ensure Page Load".  
    - Edge Cases: Timeout if page is slow; HTTP errors if site unreachable; content structure changes may affect extraction.  

  - **Delay to Ensure Page Load**  
    - Type: Wait (n8n-nodes-base.wait)  
    - Configuration: Default wait (no delay time configured, likely minimal wait to allow previous node completion).  
    - Inputs: From "Fetch Financial News Webpage".  
    - Outputs: To "Extract News Headlines & Text".  
    - Edge Cases: None significant; node ensures proper sequencing.

  - **Extract News Headlines & Text**  
    - Type: HTML Extract (n8n-nodes-base.html)  
    - Configuration: Extracts multiple news sections using detailed CSS selectors:  
      - Headline #1, Headline #2  
      - Editor's Picks  
      - Top Stories  
      - Spotlight  
      - Various News  
      - Europe News  
    - Text cleanup enabled to remove unwanted formatting.  
    - Inputs: From "Delay to Ensure Page Load".  
    - Outputs: Extracted, structured JSON to "Clean Extracted News Data".  
    - Edge Cases: Changes in FT page layout or CSS selectors could break extraction; some sections skip H2 tags to avoid headlines duplication.

#### 1.3 Data Cleaning & Preparation

- **Overview:**  
  This block consolidates all extracted news snippets into a single formatted string, preparing it for AI summarization.

- **Nodes Involved:**  
  - Clean Extracted News Data

- **Node Details:**

  - **Clean Extracted News Data**  
    - Type: Set Node (n8n-nodes-base.set)  
    - Configuration: Creates a single string field "All data" combining all extracted news parts with labels and line breaks for readability. Uses expressions to pull data from prior HTML extraction node.  
    - Inputs: From "Extract News Headlines & Text".  
    - Outputs: Passes formatted text to "AI Financial News Summarizer".  
    - Edge Cases: Missing or empty extracted fields may result in incomplete summaries; expression failures if upstream data missing.

#### 1.4 AI Summarization

- **Overview:**  
  This block uses a Langchain agent with Ollama LLM to generate a concise investor-oriented summary of the daily financial news.

- **Nodes Involved:**  
  - AI Financial News Summarizer  
  - LLM Chat Model

- **Node Details:**

  - **AI Financial News Summarizer**  
    - Type: Langchain Agent (AI language model integration)  
    - Configuration:  
      - Takes combined news text from "Clean Extracted News Data".  
      - System message instructs the AI to act as a financial analyst summarizing the news with investor outlook and market sentiment indications (bullish/bearish/neutral).  
      - Prompt type: define (custom prompt).  
    - Inputs: From "Clean Extracted News Data".  
    - Outputs: AI-generated summary text to "Email Daily Financial Summary".  
    - Edge Cases: Model latency, API errors, or malformed inputs can cause failures; system message must align with desired output style.

  - **LLM Chat Model**  
    - Type: Ollama Language Model Chat Node (@n8n/n8n-nodes-langchain.lmChatOllama)  
    - Configuration: Uses "llama3.2-16000:latest" Ollama model with default options.  
    - Credentials: Requires Ollama API credentials.  
    - Inputs: Connected as AI language model provider to "AI Financial News Summarizer".  
    - Outputs: AI text output streamed back to agent node.  
    - Edge Cases: API authentication issues, model unavailability, rate limits.

#### 1.5 Email Dispatch

- **Overview:**  
  Sends the AI-generated daily financial summary as a plain-text email to a predefined recipient.

- **Nodes Involved:**  
  - Email Daily Financial Summary

- **Node Details:**

  - **Email Daily Financial Summary**  
    - Type: Email Send (n8n-nodes-base.emailSend)  
    - Configuration:  
      - Email subject: "Today's News"  
      - Recipient: abc@gmail.com  
      - Sender: xyz@gmail.com  
      - Email body: AI summary output from "AI Financial News Summarizer" (plain text)  
      - Format: text only  
    - Credentials: SMTP account configured and selected for sending email.  
    - Inputs: From "AI Financial News Summarizer".  
    - Outputs: None (end node).  
    - Edge Cases: SMTP authentication failures, email delivery issues, invalid email addresses.

---

### 3. Summary Table

| Node Name                     | Node Type                       | Functional Role                 | Input Node(s)               | Output Node(s)                   | Sticky Note                                                                                              |
|-------------------------------|--------------------------------|--------------------------------|-----------------------------|---------------------------------|---------------------------------------------------------------------------------------------------------|
| Schedule Daily Trigger         | Schedule Trigger                | Initiate workflow daily         | None                        | Fetch Financial News Webpage     |                                                                                                         |
| Fetch Financial News Webpage   | HTTP Request                   | Fetch FT homepage HTML          | Schedule Daily Trigger      | Delay to Ensure Page Load        | Url : https://www.ft.com/                                                                                |
| Delay to Ensure Page Load      | Wait                           | Allow content to load fully     | Fetch Financial News Webpage| Extract News Headlines & Text    |                                                                                                         |
| Extract News Headlines & Text  | HTML Extract                   | Extract headlines & news texts  | Delay to Ensure Page Load   | Clean Extracted News Data        | Extract selected headlines, editor's picks, spotlight etc.                                              |
| Clean Extracted News Data      | Set                            | Format extracted news for AI    | Extract News Headlines & Text| AI Financial News Summarizer     |                                                                                                         |
| AI Financial News Summarizer   | Langchain Agent (AI LLM)       | Generate news summary           | Clean Extracted News Data   | Email Daily Financial Summary    |                                                                                                         |
| LLM Chat Model                | Ollama LLM Chat Node           | Provide AI language model       | AI Financial News Summarizer (as provider) | AI Financial News Summarizer (via ai_languageModel input) |                                                                                                         |
| Email Daily Financial Summary  | Email Send                    | Send summary via email          | AI Financial News Summarizer| None                            |                                                                                                         |
| Sticky Note                   | Sticky Note                    | Informational note              | None                        | None                            | 📌 **Try It Out!** This workflow auto-fetches top financial headlines, cleans the content, and uses AI to summarize it into a short investor-friendly email. 🧠 Ideal for: Market analysts, finance teams, or daily newsletters. **Powered by:** n8n + LLM (e.g., GPT-4 or Gemini) + Email Node. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Parameters: Set to trigger daily at 7:00 AM  
   - No inputs  
   - Connect output to next node.

2. **Add an HTTP Request node**  
   - Type: HTTP Request  
   - Parameters:  
     - HTTP Method: GET  
     - URL: https://www.ft.com/  
     - Timeout: 10000 ms (10 seconds)  
   - Connect input from Schedule Trigger.

3. **Add a Wait node**  
   - Type: Wait  
   - Parameters: Default (no delay value needed, just sequencing)  
   - Connect input from HTTP Request node.

4. **Add an HTML Extract node**  
   - Type: HTML Extract  
   - Parameters:  
     - Operation: Extract HTML Content  
     - Clean Up Text: Enabled  
     - Extraction Values: Add multiple keys with CSS selectors:  
       - Headline #1  
       - Headline #2  
       - Editor's Picks  
       - Top Stories  
       - Spotlight  
       - Various News  
       - Europe News  
     - Use provided CSS selectors as per FT website layout (adjust if FT changes their DOM).  
     - Skip H2 tags in some selectors to avoid duplicate titles.  
   - Connect input from Wait node.

5. **Add a Set node**  
   - Type: Set  
   - Parameters:  
     - Add a new string field named "All data"  
     - Use expressions to concatenate all extracted fields into one formatted string with labels and line breaks.  
       For example:  
       ```
       News :  
       {{$json["Headline #1"]}}


       Financial news :

       {{$json["Headline #1"]}};
       {{$json["Headline #2"]}};
       {{$json["Editor's Picks"]}};
       {{$json["Top Stories"]}};
       {{$json["Spotlight"]}};
       {{$json["Various News"]}};
       {{$json["Europe News"]}};
       ```  
   - Connect input from HTML Extract node.

6. **Add a Langchain Agent node**  
   - Type: Langchain Agent (AI language model node)  
   - Parameters:  
     - Text prompt: `"Summarise this news :\n\n{{$json['All data']}}`  
     - System message: Set to instruct AI as a financial analyst providing an investor outlook, including sentiment and upcoming events.  
     - Prompt type: define  
   - Connect input from Set node.

7. **Add an Ollama LLM Chat node**  
   - Type: Ollama LLM Chat node  
   - Parameters:  
     - Model: "llama3.2-16000:latest"  
     - Options: Default  
   - Credentials: Configure Ollama API credentials for authentication.  
   - Connect output of this node as the AI language model provider to the Langchain Agent node.

8. **Add an Email Send node**  
   - Type: Email Send  
   - Parameters:  
     - Subject: "Today's News"  
     - To Email: abc@gmail.com (replace with actual recipient)  
     - From Email: xyz@gmail.com (replace with authenticated sender email)  
     - Email Format: Text  
     - Text: Use expression `{{$json.output}}` from Langchain node output.  
   - Credentials: Configure SMTP credentials for sending email.  
   - Connect input from Langchain Agent node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                     | Context or Link                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| 📌 **Try It Out!** This workflow auto-fetches top financial headlines, cleans the content, and uses AI to summarize it into a short investor-friendly email. 🧠 Ideal for: Market analysts, finance teams, or daily newsletters. **Powered by:** n8n + LLM (e.g., GPT-4 or Gemini) + Email Node. | Sticky Note node content in the workflow.           |
| Workflow depends heavily on the FT website's DOM structure; CSS selectors may require maintenance if site layout changes.                                         | FT website https://www.ft.com/                        |
| Ollama LLM node requires valid API credentials configured in n8n’s credentials manager.                                                                           | Ollama official docs for API setup                   |
| SMTP email sending requires a properly configured SMTP server or email service with credentials stored in n8n.                                                    | Configure SMTP credentials in n8n                    |
| The Langchain Agent node abstracts AI prompt management but requires a compatible LLM node (here Ollama LLM Chat) configured as provider.                       | n8n Langchain integration documentation              |


---

**Disclaimer:**  
The provided content is exclusively derived from an automated workflow created with n8n, an integration and automation platform. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.