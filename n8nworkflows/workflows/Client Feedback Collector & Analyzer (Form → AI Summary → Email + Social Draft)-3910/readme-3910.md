Client Feedback Collector & Analyzer (Form → AI Summary → Email + Social Draft)

https://n8nworkflows.xyz/workflows/client-feedback-collector---analyzer--form---ai-summary---email---social-draft--3910


# Client Feedback Collector & Analyzer (Form → AI Summary → Email + Social Draft)

### 1. Workflow Overview

This workflow, named **Client Feedback Collector & Analyzer (Form → AI Summary → Email + Social Draft)**, automates the collection, analysis, and dissemination of client feedback. It is designed to intake raw feedback data submitted via a form (such as Tally, Typeform, or Google Forms), utilize AI to generate a structured summary and social media draft, and then distribute these outputs via email and Telegram (or other messaging integrations).

The workflow is organized into the following logical blocks:

- **1.1 Input Reception:** Captures client feedback data submitted through an HTTP webhook.
- **1.2 AI Processing:** Prepares and sends the feedback to an AI service for analysis and content generation.
- **1.3 Output Formatting:** Parses the AI response into distinct outputs for reporting and social media.
- **1.4 Distribution:** Sends the summarized feedback report via email and the social media draft via Telegram.

This modular design supports easy adaptation, such as swapping messaging platforms or customizing AI prompts.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives the client feedback data via an HTTP POST webhook endpoint. It expects a field named `feedback` carrying the client's textual input.

- **Nodes Involved:**  
  - Receive Feedback

- **Node Details:**

  - **Receive Feedback**  
    - Type: Webhook node  
    - Role: Entry point capturing POST requests from feedback forms.  
    - Configuration:  
      - HTTP Method: POST  
      - Path: `/client-feedback`  
      - No additional options enabled.  
    - Inputs: None (start node)  
    - Outputs: Passes JSON containing the client’s feedback field to the next node.  
    - Edge cases:  
      - Missing or empty `feedback` field results in default handling downstream.  
      - Potential HTTP errors if the webhook URL is misconfigured or the form does not POST correctly.

#### 2.2 AI Processing

- **Overview:**  
  This block constructs a prompt for the AI model based on the incoming feedback and sends it to the AI API to generate an analysis and social media draft.

- **Nodes Involved:**  
  - Prepare AI Prompt  
  - Analyze with AI

- **Node Details:**

  - **Prepare AI Prompt**  
    - Type: Function node  
    - Role: Constructs a text prompt embedding the client's feedback for AI analysis.  
    - Configuration:  
      - JavaScript code extracts `feedback` from the input JSON; defaults to `"No feedback provided."` if absent.  
      - Builds a prompt requesting:  
        1. Summary of positive points  
        2. Suggestions for improvement  
        3. Short social media post based on positives  
    - Key Expression:  
      ```js
      const feedback = $json.feedback || "No feedback provided.";
      return [{
        json: {
          prompt: `Analyze this client feedback: "${feedback}"\n\n1. Summarize the positive points.\n2. Suggest improvements.\n3. Generate a short social media post based on the positive elements.`
        }
      }];
      ```  
    - Inputs: JSON with `feedback` field from the webhook  
    - Outputs: JSON with `prompt` string for the AI API  
    - Edge cases: Empty or malformed feedback handled by default text.

  - **Analyze with AI**  
    - Type: HTTP Request node  
    - Role: Sends the prepared prompt to the DeepSeek AI API (or compatible GPT service) and receives the generated text.  
    - Configuration:  
      - URL: `https://api.deepseek.com/generate`  
      - HTTP Method: POST (implied)  
      - Authentication: Predefined credential (DeepSeek API key or compatible)  
      - Request body: JSON containing the prompt from the previous node  
      - JSON parameters enabled for request formatting  
    - Inputs: Prompt JSON from "Prepare AI Prompt"  
    - Outputs: AI-generated response JSON  
    - Version-specific: Uses n8n HTTP Request v2 node for better JSON handling  
    - Edge cases:  
      - API authentication failure  
      - Network timeout  
      - Unexpected API response structure

#### 2.3 Output Formatting

- **Overview:**  
  This block processes the AI response to separate the analysis summary and the social media post draft.

- **Nodes Involved:**  
  - Format AI Output

- **Node Details:**

  - **Format AI Output**  
    - Type: Function node  
    - Role: Parses the AI response text to extract the report summary and social media post separately.  
    - Configuration:  
      - Extracts the AI output text from either `response` or `choices[0].text` fields.  
      - Searches for the start of item "3." to split summary and post.  
      - Returns two JSON fields: `report` (summary + improvements) and `post` (social media draft).  
    - Key Expression:  
      ```js
      const output = $json.response || $json.choices?.[0]?.text || "No AI output.";
      const splitIndex = output.indexOf("3.");
      let summary = output;
      let post = "No post generated.";

      if (splitIndex !== -1) {
        summary = output.substring(0, splitIndex).trim();
        post = output.substring(splitIndex).replace(/^3\./, "").trim();
      }

      return [{
        json: {
          report: summary,
          post: post
        }
      }];
      ```  
    - Inputs: AI response JSON  
    - Outputs: JSON with `report` and `post` fields for downstream nodes  
    - Edge cases:  
      - AI output missing or malformed results in default messages  
      - Unexpected AI text formatting could break splitting logic

#### 2.4 Distribution

- **Overview:**  
  This block sends the formatted feedback summary report via email and the social media draft via Telegram.

- **Nodes Involved:**  
  - Send Feedback Report  
  - Send Social Draft

- **Node Details:**

  - **Send Feedback Report**  
    - Type: Email Send node  
    - Role: Emails the AI-generated feedback report to the specified team address.  
    - Configuration:  
      - Subject: "Client Feedback Summary"  
      - To: `team@email.com` (placeholder; must be customized)  
      - From: `your@email.com` (placeholder; must be customized)  
      - Text body: Uses the `report` field from input JSON  
      - SMTP credentials required (configured in n8n credentials)  
    - Inputs: JSON with `report` text from "Format AI Output"  
    - Outputs: None (terminal node)  
    - Edge cases:  
      - SMTP failures (auth, connection)  
      - Invalid email addresses

  - **Send Social Draft**  
    - Type: Telegram node  
    - Role: Sends the AI-generated social media post draft to a Telegram chat.  
    - Configuration:  
      - Text: Uses `post` field from input JSON  
      - Chat ID: Placeholder `YOUR_TELEGRAM_CHAT_ID` (must be updated)  
      - No additional fields set  
      - Telegram credentials configured in n8n  
    - Inputs: JSON with `post` text from "Format AI Output"  
    - Outputs: None (terminal node)  
    - Edge cases:  
      - Incorrect chat ID or invalid Telegram credentials  
      - Message length or formatting issues  
      - Optional replacement with Slack, Buffer, or other integrations as per setup instructions

---

### 3. Summary Table

| Node Name            | Node Type          | Functional Role                     | Input Node(s)       | Output Node(s)               | Sticky Note                                                                                          |
|----------------------|--------------------|-----------------------------------|---------------------|-----------------------------|----------------------------------------------------------------------------------------------------|
| Receive Feedback     | Webhook            | Receives client feedback via POST | None                | Prepare AI Prompt            | Connect your form tool to this webhook using POST method; ensure it sends a `feedback` field.       |
| Prepare AI Prompt    | Function           | Builds AI prompt from feedback    | Receive Feedback    | Analyze with AI              | Customize the prompt here to change tone or language if desired.                                   |
| Analyze with AI      | HTTP Request       | Sends prompt to AI API and gets response | Prepare AI Prompt | Format AI Output             | Add your DeepSeek or compatible GPT API key in credentials for this node.                           |
| Format AI Output     | Function           | Parses AI response into summary and post | Analyze with AI     | Send Feedback Report, Send Social Draft |                                                                                                    |
| Send Feedback Report | Email Send         | Emails the feedback summary       | Format AI Output    | None                        | Configure SMTP credentials and recipient email address here.                                       |
| Send Social Draft    | Telegram           | Sends social media draft via Telegram | Format AI Output    | None                        | Replace Telegram with Slack, Buffer, or other service as needed; update chat ID and credentials.   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "Receive Feedback"**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `client-feedback`  
   - No additional options  
   - This node will serve as the entry point for form submissions.

2. **Create Function Node: "Prepare AI Prompt"**  
   - Connect output of "Receive Feedback" to this node.  
   - Set code to:  
     ```js
     const feedback = $json.feedback || "No feedback provided.";
     return [{
       json: {
         prompt: `Analyze this client feedback: "${feedback}"\n\n1. Summarize the positive points.\n2. Suggest improvements.\n3. Generate a short social media post based on the positive elements.`
       }
     }];
     ```  
   - This node formats the prompt text for AI processing.

3. **Create HTTP Request Node: "Analyze with AI"**  
   - Connect output of "Prepare AI Prompt" to this node.  
   - URL: `https://api.deepseek.com/generate`  
   - HTTP Method: POST (default)  
   - Enable "JSON/RAW Parameters" for request body  
   - Authentication: Select or create credential with your DeepSeek or compatible GPT API key  
   - Payload: Use JSON with the `prompt` property passed from previous node  
   - This node sends the prompt to the AI API and retrieves the result.

4. **Create Function Node: "Format AI Output"**  
   - Connect output of "Analyze with AI" to this node.  
   - Set code to:  
     ```js
     const output = $json.response || $json.choices?.[0]?.text || "No AI output.";
     const splitIndex = output.indexOf("3.");
     let summary = output;
     let post = "No post generated.";

     if (splitIndex !== -1) {
       summary = output.substring(0, splitIndex).trim();
       post = output.substring(splitIndex).replace(/^3\./, "").trim();
     }

     return [{
       json: {
         report: summary,
         post: post
       }
     }];
     ```  
   - This node splits the AI output into summary and social media draft.

5. **Create Email Send Node: "Send Feedback Report"**  
   - Connect output of "Format AI Output" to this node.  
   - Set "To Email" to your team's email address (e.g., `team@email.com`)  
   - Set "From Email" to your sender address (e.g., `your@email.com`)  
   - Subject: `Client Feedback Summary`  
   - Text: Expression referencing `{{ $json["report"] }}`  
   - Configure SMTP credentials in n8n for sending emails.

6. **Create Telegram Node: "Send Social Draft"**  
   - Connect output of "Format AI Output" to this node.  
   - Set "Chat ID" to your Telegram chat ID (`YOUR_TELEGRAM_CHAT_ID`)  
   - Text: Expression referencing `{{ $json["post"] }}`  
   - Configure Telegram credentials in n8n.  
   - Optionally replace this node with Slack, Buffer, or other messaging nodes as preferred.

7. **Activate and Test**  
   - Deploy the workflow inactive by default.  
   - Send test POST requests with `feedback` field to the webhook URL.  
   - Confirm email receipt and Telegram message delivery.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                             |
|------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Estimated setup time is approximately 15 minutes for credential and webhook configuration.                       | Setup instructions in the workflow description.                                                            |
| Sticky notes in the workflow provide visual guidance on key configuration steps such as API keys and webhook use.| Present within the n8n editor on respective nodes.                                                         |
| Replace the Telegram node with Slack, Buffer, or other messaging services by changing node type and credentials.  | Allows adaptation to different team communication platforms.                                               |
| DeepSeek API is used as an example GPT-compatible AI service; other providers can be configured similarly.        | Requires corresponding API credentials and endpoint adjustments.                                           |
| Useful for creating actionable insights from customer feedback instantly, benefiting freelancers, educators, etc.| Enables marketing to react quickly via social drafts, and support teams to address improvements.           |

---

This comprehensive analysis and reconstruction guide provides a clear, stepwise understanding of the workflow’s structure, rationale, node configurations, and integration points to facilitate seamless reproduction and customization.