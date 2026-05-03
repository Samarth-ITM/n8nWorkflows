Generate AI-Powered News Summaries with Gnews.io and GPT-4.1 to Slack

https://n8nworkflows.xyz/workflows/generate-ai-powered-news-summaries-with-gnews-io-and-gpt-4-1-to-slack-8660


# Generate AI-Powered News Summaries with Gnews.io and GPT-4.1 to Slack

### 1. Workflow Overview

This workflow automates the process of generating AI-powered news summaries based on user-specified topics. It fetches relevant news articles from the Gnews.io API, uses OpenAI's GPT-4.1 model to intelligently select and summarize the most relevant pieces, and delivers the summarized content to a designated Slack channel. The workflow features a simple form-based input for specifying topics, real-time news aggregation, AI-driven article filtering and summarization, and automated Slack notifications.

Logical blocks include:

- **1.1 Input Reception**: Captures the user input topic via a web form.
- **1.2 News Retrieval**: Queries the Gnews.io API to fetch news articles matching the topic.
- **1.3 Data Preparation**: Extracts and formats article data for AI processing.
- **1.4 AI Processing**: Uses GPT-4.1 via Langchain nodes to select the top 15 relevant articles and summarize them professionally.
- **1.5 Notification Delivery**: Sends the generated summary to a specified Slack channel.
- **1.6 Workflow Documentation**: Provides a sticky note with workflow description and configuration instructions.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Captures user input through a web form for the topic of interest, triggering the workflow execution.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**

  - **On form submission**  
    - Type: `formTrigger` (Webhook Trigger)  
    - Purpose: Starts the workflow when the user submits a form named "News Search" containing a required field "topic".  
    - Configuration:  
      - Form Title: "News Search"  
      - Form Fields: Single required field labeled "topic"  
      - Webhook ID: Unique webhook identifier for receiving form data  
    - Input: HTTP form submission with topic parameter  
    - Output: JSON containing user input, e.g. `{ "topic": "finance" }`  
    - Edge Cases:  
      - Missing or empty topic field will prevent the form submission due to required validation.  
      - Webhook connectivity issues may cause trigger failures.

---

#### 1.2 News Retrieval

- **Overview:**  
  Performs an HTTP request to Gnews.io API to obtain news articles matching the user-provided topic.

- **Nodes Involved:**  
  - Get GNews articles

- **Node Details:**

  - **Get GNews articles**  
    - Type: `httpRequest`  
    - Purpose: Queries Gnews.io API search endpoint to retrieve articles related to the topic.  
    - Configuration:  
      - URL: `https://gnews.io/api/v4/search`  
      - Query Parameters:  
        - `q`: dynamically set to the submitted topic (`={{ $json.topic }}`)  
        - `lang`: fixed to English (`en`)  
        - `apikey`: placeholder `"ADD YOUR API HERE"` to be replaced with a valid API key from Gnews.io  
      - HTTP Method: GET  
    - Input: JSON from form submission containing topic  
    - Output: JSON response from Gnews.io containing an array of articles under `articles` key  
    - Edge Cases:  
      - Missing or invalid API key results in authentication errors.  
      - API rate limiting or downtime may cause request failures or empty responses.  
      - No articles found returns empty or minimal `articles` array.

---

#### 1.3 Data Preparation

- **Overview:**  
  Extracts the `articles` array from the Gnews.io response and maps it into a format suitable for AI summarization.

- **Nodes Involved:**  
  - Map to articles

- **Node Details:**

  - **Map to articles**  
    - Type: `set`  
    - Purpose: Assigns the `articles` array from the previous HTTP response to a new JSON property `articles` for downstream nodes.  
    - Configuration:  
      - Sets field `articles` to the expression `={{ $json.articles }}`  
    - Input: JSON containing full Gnews.io API response  
    - Output: JSON with a top-level `articles` property containing the array of news articles  
    - Edge Cases:  
      - If `articles` key is missing or empty, the AI summarizer will receive no data to process, potentially causing empty summaries or errors.

---

#### 1.4 AI Processing

- **Overview:**  
  Uses Langchain AI nodes with OpenAI GPT-4.1 to select the top 15 most relevant articles related to finance and generate a concise, professional summary including links and the current date.

- **Nodes Involved:**  
  - GPT-4.1 Model  
  - AI News Summarizer

- **Node Details:**

  - **GPT-4.1 Model**  
    - Type: `lmChatOpenAi` (Langchain OpenAI Chat Model)  
    - Purpose: Provides GPT-4.1 language model capabilities for natural language processing.  
    - Configuration:  
      - Model selection: `gpt-4.1`  
      - No additional options specified  
      - Credentials: OpenAI API key configured under "OpenAi account"  
    - Input: Receives prompt from AI News Summarizer node.  
    - Output: Model-generated text to AI News Summarizer node for final processing.  
    - Edge Cases:  
      - API authentication failures if credentials are invalid or quota exceeded.  
      - Timeout or rate limiting by OpenAI API.  
      - Unexpected model response formats.

  - **AI News Summarizer**  
    - Type: `agent` (Langchain agent node)  
    - Purpose: Orchestrates the summarization task by defining prompt instructions and handling the AI interaction.  
    - Configuration:  
      - Text prompt:  
        ```
        ** Select Top Articles **
        From {{$json.articles}}, choose the 15 most relevant articles related to finances advancements, tools, research, and applications.

        ** Summarize Clearly **
        Write in clear, professional, and readable language, summarizing the main point of each article in 1-2 short sentences.

        ** Include Links **
        After each summary, add the article's URL for readers to explore more.

        ** Add the Date **
        Begin the summary with today’s date:
        {{ $now.format('yyyy/MM/dd') }}
        ```
      - System message:  
        ```
        ** Output Format **
        Only return the summary in plain text – no explanations or additional formatting. Keep the response up to 2000 characters
        ```
      - Prompt type: `define` (custom prompt definition)  
    - Input: Receives articles array from "Map to articles" node  
    - Output: Summarized news text forwarded to Slack notification node  
    - Edge Cases:  
      - Input articles array empty or malformed may yield empty or irrelevant summaries.  
      - AI response exceeding 2000 characters might be truncated.  
      - API errors from OpenAI propagate here.  
      - Date formatting expression failure if `$now` context is missing.

---

#### 1.5 Notification Delivery

- **Overview:**  
  Sends the generated news summary text as a message to a specified Slack channel.

- **Nodes Involved:**  
  - Completed Notification

- **Node Details:**

  - **Completed Notification**  
    - Type: `slack` (Slack node)  
    - Purpose: Posts the AI-generated news summary to a Slack channel for team or user consumption.  
    - Configuration:  
      - Text: dynamically set to the summary output (`={{ $json.output }}`) from AI News Summarizer  
      - Channel Selection: Fixed Slack channel ID `C099YS0V3M2` selected by ID mode  
      - Webhook ID: preconfigured webhook for Slack integration  
      - Execute Once: Enabled to prevent duplicate sends on retries  
      - Credentials: Slack API OAuth2 credentials under "Slack account"  
    - Input: Summarized text JSON from AI News Summarizer  
    - Output: Slack post response (usually ignored downstream)  
    - Edge Cases:  
      - Invalid or expired Slack credentials causing authorization errors.  
      - Slack API rate limits or downtime.  
      - Message length limits on Slack (usually large enough, but possible truncation).  
      - Channel ID incorrect or user lacks posting permissions.

---

#### 1.6 Workflow Documentation

- **Overview:**  
  Contains a sticky note node with detailed workflow purpose, instructions, and configuration notes for users or maintainers.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**

  - **Sticky Note**  
    - Type: `stickyNote`  
    - Purpose: Provide human-readable documentation embedded within the workflow canvas.  
    - Content includes:  
      - Workflow description and unique features  
      - Instructions for obtaining and configuring Gnews.io API Key with a link to https://gnews.io  
    - Position: Left side of the canvas for easy visibility  
    - No input or output connections (informational only)  
    - Edge Cases: None (non-executable node)

---

### 3. Summary Table

| Node Name           | Node Type                        | Functional Role                 | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                                             |
|---------------------|---------------------------------|--------------------------------|-------------------------|-------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| On form submission   | formTrigger                     | Capture user topic input       | —                       | Get GNews articles      |                                                                                                                                        |
| Get GNews articles   | httpRequest                    | Fetch news articles API        | On form submission      | Map to articles         |                                                                                                                                        |
| Map to articles      | set                            | Extract articles array         | Get GNews articles      | AI News Summarizer      |                                                                                                                                        |
| GPT-4.1 Model        | lmChatOpenAi (Langchain)        | Provide GPT-4.1 model          | AI News Summarizer      | AI News Summarizer      |                                                                                                                                        |
| AI News Summarizer   | agent (Langchain)               | Select & summarize articles    | Map to articles, GPT-4.1 Model | Completed Notification |                                                                                                                                        |
| Completed Notification | slack                         | Post summary to Slack channel  | AI News Summarizer      | —                       |                                                                                                                                        |
| Sticky Note          | stickyNote                     | Workflow documentation         | —                       | —                       | ## Daily News AI Agent using Gnews.io; Instructions to get and configure API key at https://gnews.io                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node**  
   - Add a node of type `Form Trigger`.  
   - Set the form title to `"News Search"`.  
   - Add a single form field: label `"topic"`, set as required.  
   - Save the node; note the generated webhook URL for triggering.

2. **Create the HTTP Request Node to Gnews.io**  
   - Add a node of type `HTTP Request`.  
   - Set HTTP Method to `GET`.  
   - Set URL to `https://gnews.io/api/v4/search`.  
   - Add query parameters:  
     - `q` = Expression referencing input topic: `={{ $json.topic }}`  
     - `lang` = `"en"` (fixed)  
     - `apikey` = `"YOUR_GNEWS_API_KEY"` (replace with valid key)  
   - Connect output of Form Trigger to this node's input.

3. **Create Set Node to Map Articles**  
   - Add a node of type `Set`.  
   - Add a field named `articles` with value expression: `={{ $json.articles }}`.  
   - Connect output of HTTP Request node to this Set node.

4. **Create Langchain GPT-4.1 Model Node**  
   - Add a node of type `lmChatOpenAi` (Langchain OpenAI Chat Model).  
   - Select model `"gpt-4.1"`.  
   - Configure OpenAI credentials (API key) under "OpenAi account".  
   - No special options needed.  

5. **Create Langchain Agent Node for AI News Summarization**  
   - Add a node of type `agent` (Langchain agent node).  
   - In the text prompt field, enter:  
     ```
     ** Select Top Articles **
     From {{$json.articles}}, choose the 15 most relevant articles related to finances advancements, tools, research, and applications.

     ** Summarize Clearly **
     Write in clear, professional, and readable language, summarizing the main point of each article in 1-2 short sentences.

     ** Include Links **
     After each summary, add the article's URL for readers to explore more.

     ** Add the Date **
     Begin the summary with today’s date:
     {{ $now.format('yyyy/MM/dd') }}
     ```  
   - Set system message with:  
     ```
     ** Output Format **
     Only return the summary in plain text – no explanations or additional formatting. Keep the response up to 2000 characters
     ```  
   - Set prompt type to `define`.  
   - Connect output of Set node to this agent node's input.  
   - Connect GPT-4.1 Model node as the language model for this agent node.

6. **Create Slack Notification Node**  
   - Add a Slack node.  
   - Set the message text to: `={{ $json.output }}` (summary from AI agent).  
   - Select the Slack channel by ID (e.g., `"C099YS0V3M2"`).  
   - Configure Slack OAuth2 credentials under "Slack account".  
   - Enable "Execute Once" to send the message only once per workflow run.  
   - Connect output of AI News Summarizer node to this Slack node.

7. **Create Sticky Note Node (Optional for Documentation)**  
   - Add a sticky note to the canvas.  
   - Paste the workflow overview, unique features, and API configuration instructions.  
   - Position it for easy access and reference.

8. **Connect the Nodes**  
   - Ensure the node connection order is:  
     `On form submission` → `Get GNews articles` → `Map to articles` → `AI News Summarizer` (with `GPT-4.1 Model` as AI backend) → `Completed Notification`.  

9. **Credential Setup**  
   - Add and configure OpenAI API credentials with valid key.  
   - Add and configure Slack OAuth2 credentials with required scopes for sending messages.  
   - Obtain a valid Gnews.io API key and insert it into the HTTP Request node’s query parameters.

10. **Test the Workflow**  
    - Trigger the workflow by submitting the form with a topic.  
    - Verify news articles are fetched, summarized, and posted to Slack.  
    - Monitor for errors such as API failures, empty responses, or authentication issues.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                |
|-------------------------------------------------------------------------------------------------|------------------------------------------------|
| Workflow automates news aggregation and summarization with AI, delivering to Slack in real-time. | Workflow purpose overview                       |
| Obtain Gnews.io API key by signing up at: https://gnews.io                                      | Gnews.io API documentation and signup          |
| Uses OpenAI GPT-4.1 model via Langchain nodes for intelligent summarization                      | OpenAI GPT-4.1 integration                      |
| Slack OAuth2 credentials needed for posting messages                                            | Slack API documentation                         |
| Form trigger node provides simple web interface for topic input                                | n8n formTrigger node documentation              |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated n8n workflow, fully compliant with current content policies, containing no illegal, offensive, or protected material. All data processed is legal and publicly accessible.