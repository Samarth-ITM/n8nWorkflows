Qualify Real Estate Leads Automatically with OpenAI, Gmail & Airtable CRM

https://n8nworkflows.xyz/workflows/qualify-real-estate-leads-automatically-with-openai--gmail---airtable-crm-5428


# Qualify Real Estate Leads Automatically with OpenAI, Gmail & Airtable CRM

### 1. Workflow Overview

This workflow, titled **"Qualify Real Estate Leads Automatically with OpenAI, Gmail & Airtable CRM"**, automates the lead qualification process for real estate agents or small agencies. It aims to save time by automatically scoring leads submitted via an online property inquiry form, notifying agents by email of qualified (high-value) leads, and storing all lead data in an Airtable CRM.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception:** Captures incoming lead data via a web form submission.
- **1.2 AI Lead Qualification:** Uses an AI model to parse, classify, and score the lead based on predefined criteria.
- **1.3 Data Processing & Decision:** Sets qualification flags and applies conditional logic to differentiate qualified leads.
- **1.4 CRM Logging:** Stores lead data in Airtable for record-keeping and further agent use.
- **1.5 Notification:** Sends an email alert about qualified leads to the agent via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  The entry point of the workflow, this block listens for form submissions from a property inquiry form and outputs the raw lead data for processing.

- **Nodes Involved:**  
  - `On form submission`

- **Node Details:**  
  - **On form submission**  
    - *Type & Role:* `formTrigger` node; triggers workflow on receiving a new submission.  
    - *Configuration:* Listens for submissions to a form titled “Property Form” with required fields: Full Name, Email, Budget Range, Preferred Location, Purchase Timeline, Property Type.  
    - *Key Expressions:* Uses webhook to capture form data; fields mapped directly to JSON properties.  
    - *Input/Output:* No input; outputs JSON with form fields.  
    - *Version Requirements:* v2.2 of formTrigger node.  
    - *Potential Failures:* Webhook connectivity issues, missing required fields if form client-side validation is bypassed.  
    - *Sub-workflow:* None.

#### 2.2 AI Lead Qualification

- **Overview:**  
  This block leverages OpenAI language models to extract structured information from the raw lead data and compute a lead score based on budget, urgency, location, and timeline.

- **Nodes Involved:**  
  - `OpenAI Chat Model`  
  - `Information Extractor`

- **Node Details:**  
  - **OpenAI Chat Model**  
    - *Type & Role:* `lmChatOpenAi` node (Langchain integration); calls the OpenAI GPT-4o-mini model.  
    - *Configuration:* Uses model "gpt-4o-mini" for chat completions; no special options enabled.  
    - *Key Expressions:* Receives prompt content from connected nodes; credentials linked to OpenAI API key.  
    - *Input/Output:* Input from form submission; output passed to Information Extractor.  
    - *Version Requirements:* v1.2; requires valid OpenAI API credentials.  
    - *Potential Failures:* API rate limits, network errors, invalid API key, unexpected model responses.  
    - *Sub-workflow:* None.

  - **Information Extractor**  
    - *Type & Role:* `informationExtractor` node (Langchain); parses AI response into structured JSON.  
    - *Configuration:* Custom prompt instructs parsing of lead details, scoring (0-100), urgency classification, and JSON output schema.  
    - *Key Expressions:*  
      - Template uses input fields (`Full Name`, `Email`, `Budget Range`, etc.) to prompt AI.  
      - Output JSON includes fields: name, email, budget_min, budget_max, location, timeline, urgency, score, qualified.  
    - *Input/Output:* Input from OpenAI Chat Model; outputs structured lead info.  
    - *Version Requirements:* v1; requires Langchain integration.  
    - *Potential Failures:* Improper parsing if AI output format changes, schema mismatch, partial responses.  
    - *Sub-workflow:* None.

#### 2.3 Data Processing & Decision

- **Overview:**  
  Processes the AI-extracted lead data to set a boolean qualification flag based on lead score and routes leads conditionally depending on qualification.

- **Nodes Involved:**  
  - `Edit Fields`  
  - `If`

- **Node Details:**  
  - **Edit Fields**  
    - *Type & Role:* `set` node; modifies data by setting a `qualified` boolean field.  
    - *Configuration:* Adds `qualified` field where `true` if lead score ≥ 70, else `false`.  
    - *Key Expressions:* `={{ $json["output"]["score"] >= 70 }}` sets qualification status.  
    - *Input/Output:* Input from Information Extractor; output to If node.  
    - *Version Requirements:* v3.4.  
    - *Potential Failures:* Expression evaluation errors if score missing or not numeric.

  - **If**  
    - *Type & Role:* Conditional node; checks if lead is qualified (`qualified === true`).  
    - *Configuration:* Condition: `{{$json.qualified}}` must be true (boolean strict equality).  
    - *Input/Output:* Input from Edit Fields; outputs to Airtable and Gmail nodes if true.  
    - *Version Requirements:* v2.2.  
    - *Potential Failures:* Incorrect condition if `qualified` is missing or malformed.

#### 2.4 CRM Logging

- **Overview:**  
  Logs lead data into an Airtable base, allowing agents to track and manage qualified leads.

- **Nodes Involved:**  
  - `Airtable`

- **Node Details:**  
  - **Airtable**  
    - *Type & Role:* Airtable node; creates new record in CRM table.  
    - *Configuration:*  
      - Operation: `create` record.  
      - Columns mapped from AI output fields: Name, Email, Score, budget (budget_min), Location, Timeline.  
      - Base and Table selected from credential-linked lists (user-defined).  
    - *Key Expressions:* Uses expressions like `={{ $('Information Extractor').item.json.output.name }}` to map fields.  
    - *Input/Output:* Input from If node (filtered qualified leads); output to Gmail node.  
    - *Version Requirements:* v2.1; requires Airtable API token credential.  
    - *Potential Failures:* API errors (authentication, rate limits), missing base/table selection, field mapping errors.

#### 2.5 Notification

- **Overview:**  
  Sends an email notification to the agent for every qualified lead, summarizing key information for quick action.

- **Nodes Involved:**  
  - `Gmail`

- **Node Details:**  
  - **Gmail**  
    - *Type & Role:* Gmail node; sends email alert.  
    - *Configuration:*  
      - Sends to lead's email (`sendTo: "email"` - likely a misconfiguration, should be agent’s email).  
      - Subject: "🔥 NEW QUALIFIED LEAD".  
      - Message body includes lead details formatted from AI output fields.  
    - *Key Expressions:* Uses expressions to fetch lead data from `Information Extractor` node.  
    - *Input/Output:* Input from Airtable node; no outputs.  
    - *Version Requirements:* v2.1; requires Gmail OAuth2 credential.  
    - *Potential Failures:* OAuth token expiration, SMTP sending errors, incorrect recipient address (note on misconfiguration).  
    - *Sub-workflow:* None.

---

### 3. Summary Table

| Node Name            | Node Type                       | Functional Role             | Input Node(s)          | Output Node(s)        | Sticky Note                                                                                                             |
|----------------------|--------------------------------|----------------------------|------------------------|-----------------------|-------------------------------------------------------------------------------------------------------------------------|
| On form submission    | formTrigger                    | Input Reception             | -                      | Information Extractor  |                                                                                                                         |
| OpenAI Chat Model     | lmChatOpenAi                   | AI Lead Qualification       | -                      | Information Extractor  |                                                                                                                         |
| Information Extractor | informationExtractor           | AI Lead Qualification       | On form submission, OpenAI Chat Model | Edit Fields           |                                                                                                                         |
| Edit Fields          | set                           | Data Processing & Decision  | Information Extractor   | If                    |                                                                                                                         |
| If                   | if                            | Data Processing & Decision  | Edit Fields             | Airtable, Gmail        |                                                                                                                         |
| Airtable             | airtable                      | CRM Logging                 | If                     | Gmail                  |                                                                                                                         |
| Gmail                | gmail                         | Notification                | Airtable                | -                      |                                                                                                                         |
| Sticky Note          | stickyNote                    | Notes / Documentation       | -                      | -                      | ## Flow                                                                                                                 |
| Sticky Note1         | stickyNote                    | Notes / Documentation       | -                      | -                      | ## Engine                                                                                                               |
| Sticky Note2         | stickyNote                    | Notes / Documentation       | -                      | -                      | ## CRM                                                                                                                  |
| Sticky Note3         | stickyNote                    | Notes / Documentation       | -                      | -                      | ## Follow Up                                                                                                            |
| Sticky Note4         | stickyNote                    | Notes / Documentation       | -                      | -                      | ## AI Agent Leads Processor<br>- **Problem**<br>Real estate agents waste time on low-quality leads and slow follow-ups.<br>- **Solution**<br>An AI-powered agent that automatically scores and filters leads, sends alerts for hot leads, and stores them in a CRM.<br>- **This AI Agent is for...**<br>Solo real estate agents or small agencies handling 10–50 buyer inquiries per week — especially those using online forms.<br>- **Scope**<br>✓ Auto-lead intake from form<br>✓ AI lead scoring & qualification<br>✓ Email notification for qualified leads<br>✓ CRM logging (Airtable)<br>✗ No built-in WhatsApp reply (optional add-on)<br>✗ No auto-scheduling (optional add-on)<br>- **How to Set Up**<br>1. Replace the form with your own (Typeform/Google Form/etc.)<br>2. Link your OpenAI key and Gmail account<br>3. Connect your Airtable base<br>4. Activate the workflow<br>5. Test by submitting a dummy lead |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node:**
   - Add a `formTrigger` node named **"On form submission"**.
   - Configure it to listen for submissions to a form titled **"Property Form"**.
   - Define required fields: Full Name, Email, Budget Range, Preferred Location, Purchase Timeline, Property Type.
   - Set webhook ID is autogenerated.
   - Position this node at top-left for clarity.

2. **Add OpenAI Chat Model Node:**
   - Add an `lmChatOpenAi` node named **"OpenAI Chat Model"**.
   - Set model to **"gpt-4o-mini"**.
   - Connect this node to the output of **On form submission** (or as per design where AI is called).
   - Add OpenAI API credentials (OpenAI API key).
   - No special options needed.
   - Position near the form trigger for logical flow.

3. **Add Information Extractor Node:**
   - Add an `informationExtractor` node named **"Information Extractor"**.
   - Configure prompt text as follows (use expressions to inject form data):
     ```
     You are a real estate assistant. Based on the input, classify the lead quality, extract structured info, and give a lead score (0-100).

     Input:
     Name: {{ $json['Full Name'] }}
     Email: {{ $json.Email }}
     Budget: {{ $json['Budget Range'] }}
     Location: {{ $json['Preferred Location'] }}
     Timeline: {{ $json['Purchase Timeline'] }}
     Property Type: {{ $json['Property Type'] }}

     Instructions:
     - Parse the budget into numeric range
     - Estimate urgency from timeline
     - Score lead (0-100) based on high budget, urgency, and location being Sydney
     - Return JSON like:
     {
       "name": "...",
       "email": "...",
       "budget_min": ...,
       "budget_max": ...,
       "location": "...",
       "timeline": "...",
       "urgency": "high | medium | low",
       "score": ...,
       "qualified": 
     }
     ```
   - Set JSON schema type to parse the AI output.
   - Connect input from OpenAI Chat Model.

4. **Add Edit Fields (Set) Node:**
   - Add a `set` node named **"Edit Fields"**.
   - Add a boolean field `qualified` with expression:
     ```
     {{$json["output"]["score"] >= 70}}
     ```
   - Connect input from **Information Extractor** node.

5. **Add If Node:**
   - Add an `if` node named **"If"**.
   - Set condition: check if `qualified` field is boolean true.
   - Connect input from **Edit Fields** node.
   - Connect the "true" output to both **Airtable** and **Gmail** nodes (parallel outputs).

6. **Add Airtable Node:**
   - Add an `airtable` node named **"Airtable"**.
   - Set operation to **create**.
   - Configure base and table using existing Airtable credentials.
   - Map columns from AI output:
     - Name → `output.name`
     - Email → `output.email`
     - Score → `output.score`
     - budget → `output.budget_min`
     - Location → `output.location`
     - Timeline → `output.timeline`
   - Connect input from the "true" output of the **If** node.

7. **Add Gmail Node:**
   - Add a `gmail` node named **"Gmail"**.
   - Configure credentials with Gmail OAuth2.
   - Set recipient email (should be the agent’s email address; note the original uses `"email"` which might be incorrect).
   - Set subject to `"🔥 NEW QUALIFIED LEAD"`.
   - Compose message body using expressions from **Information Extractor** output:
     ```
     Name: {{ $('Information Extractor').item.json.output.name }}
     Email: {{ $('Information Extractor').item.json.output.email }}
     Location: {{ $('Information Extractor').item.json.output.location }}
     Budget: {{ $('Information Extractor').item.json.output.budget_min }}
     Timeline: {{ $('Information Extractor').item.json.output.timeline }}
     Score: {{ $('Information Extractor').item.json.output.score }}
     ```
   - Connect input from Airtable node.

8. **Activate Workflow and Test:**
   - Ensure all credentials are correctly set: OpenAI, Gmail OAuth2, Airtable token.
   - Activate the workflow.
   - Submit a test form lead to verify processing, scoring, CRM entry, and email notification.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                              |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow addresses the common pain points of real estate agents who struggle with manually qualifying and following up on leads, offering an automated AI-powered solution.                                                                                                                                                                                               | Sticky Note4 content within workflow.                                                                        |
| Setup requires replacing the form trigger with your own form provider (e.g., Typeform or Google Forms) if desired, along with configuring API keys and credentials for OpenAI, Gmail, and Airtable.                                                                                                                                                                               | Sticky Note4 content within workflow.                                                                        |
| No built-in WhatsApp reply or auto-scheduling features are included but can be added as optional extensions.                                                                                                                                                                                                                                                                 | Sticky Note4 content within workflow.                                                                        |
| For further customization or troubleshooting, verify that the AI output JSON schema matches expected fields exactly, as any deviation can cause parsing or expression errors downstream.                                                                                                                                                                                     | General best practice for Langchain and n8n workflows.                                                      |
| Gmail node’s `sendTo` parameter currently references the lead’s email; typically, notifications should go to the agent’s email instead. Adjust accordingly to avoid sending lead notifications to the lead themselves.                                                                                                                                                         | Potential misconfiguration noted during node analysis.                                                      |
| For more information on n8n Langchain nodes and OpenAI integration, refer to: https://docs.n8n.io/integrations/builtin/n8n-nodes-langchain/                                                                                                                                                                                                                                  | Official n8n documentation.                                                                                   |

---

**Disclaimer:**  
The text provided comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly available.