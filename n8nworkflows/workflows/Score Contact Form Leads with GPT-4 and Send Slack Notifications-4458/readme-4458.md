Score Contact Form Leads with GPT-4 and Send Slack Notifications

https://n8nworkflows.xyz/workflows/score-contact-form-leads-with-gpt-4-and-send-slack-notifications-4458


# Score Contact Form Leads with GPT-4 and Send Slack Notifications

### 1. Workflow Overview

This n8n workflow is designed to automate the processing of contact form submissions by leveraging OpenAI’s GPT-4 for lead scoring and sending formatted notifications to a Slack channel. It targets businesses or teams who want to quickly prioritize inbound leads based on their expressed interest and immediately notify the sales or social team via Slack.

The workflow is logically grouped into the following blocks:

- **1.1 Input Reception:** Receives contact form submissions via webhook.
- **1.2 Data Extraction:** Cleans and extracts relevant lead details (name, email, message).
- **1.3 AI Processing:** Uses GPT-4 to classify the lead’s interest level (Hot/Warm/Cold).
- **1.4 Slack Notification:** Sends a formatted Slack message alerting the team about the new lead with the scored interest level.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block listens for incoming POST requests at the `/form-submission` webhook endpoint. It triggers whenever a new contact form is submitted and passes the raw payload downstream.

**Nodes Involved:**  
- Receive Form Submission

**Node Details:**

- **Receive Form Submission**  
  - Type: Webhook (HTTP POST)  
  - Configuration: Listens on path `/form-submission` with HTTP method POST; no additional authentication or options configured.  
  - Key Variables: Receives full JSON payload from the contact form submission.  
  - Inputs: External HTTP request (form submission).  
  - Outputs: Raw JSON payload forwarded to the next node.  
  - Failure Modes: Could fail if webhook path is unreachable, if payload is malformed, or if request method is not POST.  
  - Version: v2 webhook node.  
  - Notes: No authentication or validation in place; assumes trusted source or uses external protections.

#### 1.2 Data Extraction

**Overview:**  
Extracts only the needed lead details (name, email, message) from the raw form submission payload, discarding all other data to keep the payload clean for downstream processing.

**Nodes Involved:**  
- Extract Lead Details

**Node Details:**

- **Extract Lead Details**  
  - Type: Set node  
  - Configuration: Assigns new JSON keys `body.name`, `body.email`, and `body.message` by extracting from input JSON’s `body` object with expressions:  
    - `={{ $json["body"]["name"] }}`  
    - `={{ $json["body"]["email"] }}`  
    - `={{ $json["body"]["message"] }}`  
  - Inputs: Raw webhook JSON from "Receive Form Submission".  
  - Outputs: JSON with only the three extracted fields under `body`.  
  - Failure Modes: Fails or returns empty if any of these fields are missing or malformed in the input. No fallback/default values are configured.  
  - Version: Set node v3.4.  
  - Notes: Keeps data minimal for AI consumption.

#### 1.3 AI Processing

**Overview:**  
Uses OpenAI GPT-4 (specifically GPT-4.1) to classify the lead’s message into one of three categories: Hot, Warm, or Cold. The AI’s single-word response is used for lead scoring to prioritize follow-up.

**Nodes Involved:**  
- Rate Lead Interest

**Node Details:**

- **Rate Lead Interest**  
  - Type: OpenAI (LangChain AI node)  
  - Configuration:  
    - Model: `gpt-4.1-2025-04-14` (latest GPT-4.1 model as per node cache)  
    - Prompt: Classifies the message text with explicit instructions to respond with exactly one word: "Hot", "Warm", or "Cold".  
    - Message template:  
      ```
      Classify the following contact form message as "Hot", "Warm", or "Cold" based on how serious the user is about the product:

      "{{ $json["message"] }}"

      Respond only with one word: Hot, Warm, or Cold.
      ```  
  - Inputs: JSON with `body.message` from "Extract Lead Details".  
  - Outputs: JSON containing AI response under `message.content`.  
  - Failure Modes: Possible API authentication errors, timeout, malformed prompt, or model unavailability. AI may respond unexpectedly if prompt is misunderstood.  
  - Version: LangChain OpenAI node v1.8.  
  - Notes: Relies on up-to-date OpenAI credentials configured in n8n.

#### 1.4 Slack Notification

**Overview:**  
Sends a Slack message to a designated channel (#social) with a formatted alert including the lead’s name, email, original message, and the AI-assigned interest level, complete with emoji indicators.

**Nodes Involved:**  
- Send Lead Alert to Slack

**Node Details:**

- **Send Lead Alert to Slack**  
  - Type: Slack node  
  - Configuration:  
    - Channel: `social`  
    - Text: Constructed dynamically with expressions combining:  
      - Emoji based on interest level: 🔥 for Hot, 🌤 for Warm, ❄️ for Cold  
      - Lead name and email from "Extract Lead Details"  
      - Original message text  
      - Interest level label with emoji  
    - Expression used for text:  
      ```js
      ={
        ($json["message"]["content"] === "Hot" ? "🔥" :
         $json["message"]["content"] === "Warm" ? "🌤" : "❄️") +
        " New Lead: " + $node["Extract Lead Details"].json["body"]["name"] +
        " (" + $node["Extract Lead Details"].json["body"]["email"] + ")\n\n" +
        "Message: " + $node["Extract Lead Details"].json["body"]["message"] + "\n\n" +
        "Triage: " +
        ($json["message"]["content"] === "Hot" ? "🔥 Hot" :
         $json["message"]["content"] === "Warm" ? "🌤 Warm" : "❄️ Cold")
      }
      ```  
  - Inputs: AI classification output from "Rate Lead Interest" and lead details from "Extract Lead Details".  
  - Outputs: Sends message to Slack; no further output.  
  - Failure Modes: Slack API authentication failure, invalid channel name, message formatting errors, rate limits.  
  - Version: Slack node v2.3.  
  - Notes: Requires OAuth2 Slack credentials with chat:write permissions.

---

### 3. Summary Table

| Node Name             | Node Type               | Functional Role          | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                         |
|-----------------------|-------------------------|-------------------------|------------------------|-------------------------|---------------------------------------------------------------------------------------------------------------------|
| Receive Form Submission| Webhook                 | Input Reception         | -                      | Extract Lead Details     | Receive Form Submission: • Listens for incoming POSTs at /form-submission  • Triggers whenever someone hits your contact form  • Passes the raw payload downstream |
| Extract Lead Details   | Set                     | Data Extraction         | Receive Form Submission | Rate Lead Interest       | Extract Lead Details: • Extracts only what we need: name, email, message; drops all other fields for a clean payload |
| Rate Lead Interest     | OpenAI LangChain        | AI Processing           | Extract Lead Details    | Send Lead Alert to Slack | Rate Lead Interest: • Uses GPT-4.1 to read the message text  • Replies with one word: “Hot”, “Warm”, or “Cold”  • Auto-rates eagerness |
| Send Lead Alert to Slack| Slack                   | Slack Notification      | Rate Lead Interest      | -                       | Send Lead Alert to Slack: • Sends formatted alert into #social with interest level, name, email, and message       |
| Sticky Note            | Sticky Note             | Documentation           | -                      | -                       | Form-to-Slack AI Triager: workflow description                                                                       |
| Sticky Note1           | Sticky Note             | Documentation           | -                      | -                       | Receive Form Submission explanation                                                                                  |
| Sticky Note2           | Sticky Note             | Documentation           | -                      | -                       | Extract Lead Details explanation                                                                                      |
| Sticky Note3           | Sticky Note             | Documentation           | -                      | -                       | Rate Lead Interest explanation                                                                                        |
| Sticky Note4           | Sticky Note             | Documentation           | -                      | -                       | Send Lead Alert to Slack explanation                                                                                  |
| Sticky Note5           | Sticky Note             | Documentation           | -                      | -                       | Connection Overview: Webhook → Set → OpenAI → Slack (Triggers → Clean data → Classify → Notify team)                 |
| Sticky Note8           | Sticky Note             | Documentation           | -                      | -                       | Image Generator: sends prompt to OpenAI’s image endpoint (unrelated to this workflow’s core logic)                   |
| Sticky Note12          | Sticky Note             | Support Info            | -                      | -                       | Workflow Assistance contact and resources (YouTube, LinkedIn links)                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Name: `Receive Form Submission`  
   - Type: Webhook (HTTP)  
   - HTTP Method: POST  
   - Path: `/form-submission`  
   - Authentication: None (or as per your security needs)  
   - Save and activate webhook.

2. **Create Set Node**  
   - Name: `Extract Lead Details`  
   - Type: Set  
   - Add three assignments:  
     - `body.name` = `={{ $json["body"]["name"] }}`  
     - `body.email` = `={{ $json["body"]["email"] }}`  
     - `body.message` = `={{ $json["body"]["message"] }}`  
   - Connect output from `Receive Form Submission` to this node.

3. **Create OpenAI LangChain Node**  
   - Name: `Rate Lead Interest`  
   - Type: `@n8n/n8n-nodes-langchain.openAi`  
   - Model: Select or enter GPT-4.1 model (e.g., `gpt-4.1-2025-04-14`)  
   - Messages: Add a single system/user message with content:  
     ```
     Classify the following contact form message as "Hot", "Warm", or "Cold" based on how serious the user is about the product:

     "{{ $json["message"] }}"

     Respond only with one word: Hot, Warm, or Cold.
     ```  
   - Connect output from `Extract Lead Details` to this node.  
   - Ensure OpenAI credentials are configured in n8n with sufficient quota and permissions.

4. **Create Slack Node**  
   - Name: `Send Lead Alert to Slack`  
   - Type: Slack  
   - Channel: `social` (or other target Slack channel name)  
   - Text: Use expression mode and enter:  
     ```js
     ={
       ($json["message"]["content"] === "Hot" ? "🔥" :
        $json["message"]["content"] === "Warm" ? "🌤" : "❄️") +
       " New Lead: " + $node["Extract Lead Details"].json["body"]["name"] +
       " (" + $node["Extract Lead Details"].json["body"]["email"] + ")\n\n" +
       "Message: " + $node["Extract Lead Details"].json["body"]["message"] + "\n\n" +
       "Triage: " +
       ($json["message"]["content"] === "Hot" ? "🔥 Hot" :
        $json["message"]["content"] === "Warm" ? "🌤 Warm" : "❄️ Cold")
     }
     ```  
   - Connect output from `Rate Lead Interest` to this node.  
   - Configure Slack OAuth2 credentials with chat:write permission to the selected channel.

5. **Activate the workflow**  
   - Ensure all nodes are connected as: `Receive Form Submission` → `Extract Lead Details` → `Rate Lead Interest` → `Send Lead Alert to Slack`.  
   - Test by submitting a POST request with JSON body including `name`, `email`, and `message` fields to the webhook URL.  
   - Verify Slack receives the formatted message with correct lead scoring emoji and details.

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                                                                  |
|--------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| For any questions or support, contact: Yaron@nofluff.online                                                  | Sticky Note12 in workflow                                                                       |
| Explore video tutorials and tips here: https://www.youtube.com/@YaronBeen/videos                              | Sticky Note12                                                                                   |
| Connect professionally with the author: https://www.linkedin.com/in/yaronbeen/                               | Sticky Note12                                                                                   |
| The workflow is branded “Form-to-Slack AI Triager” and automates lead scoring for contact forms using GPT-4 | Sticky Note (general workflow description)                                                     |
| Slack channel used is `social`; update as needed to fit your workspace                                       | Slack node configuration                                                                        |
| OpenAI GPT-4.1 model used for classification; ensure API quota and permissions are correctly set             | OpenAI node configuration                                                                      |

---

**Disclaimer:**  
The provided text is generated exclusively from an automated n8n workflow. It complies fully with applicable content policies and contains no illegal, offensive, or protected content. All data processed is legal and publicly accessible.