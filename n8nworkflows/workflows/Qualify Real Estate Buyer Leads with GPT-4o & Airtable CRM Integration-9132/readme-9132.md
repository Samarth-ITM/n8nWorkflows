Qualify Real Estate Buyer Leads with GPT-4o & Airtable CRM Integration

https://n8nworkflows.xyz/workflows/qualify-real-estate-buyer-leads-with-gpt-4o---airtable-crm-integration-9132


# Qualify Real Estate Buyer Leads with GPT-4o & Airtable CRM Integration

### 1. Workflow Overview

This workflow automates the qualification of real estate buyer leads collected via an online form, leveraging AI processing and CRM integration. It is designed for solo agents or small agencies to efficiently score and filter inbound leads based on budget, urgency, and location criteria, then notify agents via email and log qualified leads in Airtable.

**Logical Blocks:**

- **1.1 Input Reception:** Captures lead data submitted through a property inquiry form.
- **1.2 AI Processing:** Uses OpenAI GPT-4o model to extract structured details and score the lead quality.
- **1.3 Qualification Logic:** Applies a threshold to classify leads as qualified or not based on AI scoring.
- **1.4 CRM Integration:** Creates a record for qualified leads in Airtable.
- **1.5 Follow-Up Notification:** Sends an email alert to the agent for new qualified leads.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Receives incoming property buyer lead submissions through a web form trigger node.

**Nodes Involved:**  
- On form submission

**Node Details:**

- **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point to capture user-submitted data from the "Property Form" with required fields: Full Name, Email, Budget Range, Preferred Location, Purchase Timeline, and Property Type.  
  - Configuration: Form fields explicitly defined and required; form description set as "Fill the form to buy!"  
  - Input: HTTP webhook triggered by form submission  
  - Output: JSON object with all form fields passed downstream  
  - Edge Cases: Missing required fields prevented by form; webhook connectivity or latency issues possible  
  - Version: 2.2  

#### 1.2 AI Processing

**Overview:**  
Extracts structured information from the raw form data and scores the lead using GPT-4o-powered language models.

**Nodes Involved:**  
- OpenAI Chat Model  
- Information Extractor  
- Edit Fields

**Node Details:**

- **OpenAI Chat Model**  
  - Type: Langchain OpenAI Chat  
  - Role: Provides AI language model interface using GPT-4o-mini for contextual understanding and processing  
  - Configuration: Model set to "gpt-4o-mini"; no additional options enabled  
  - Credentials: Requires valid OpenAI API key  
  - Input: None directly; invoked as AI engine for downstream nodes  
  - Output: AI response passed to Information Extractor  
  - Edge Cases: API key invalid or quota exceeded; API timeout or rate limits; model version availability

- **Information Extractor**  
  - Type: Langchain Information Extractor  
  - Role: Parses and classifies lead data, extracts normalized fields, estimates urgency, and computes lead score (0-100)  
  - Configuration: Custom prompt instructs parsing of budget, location, urgency estimation, scoring logic (high budget, urgency, location in Sydney), returning structured JSON with fields including urgency category and qualification boolean  
  - Key Expressions: Uses interpolation to insert form fields dynamically (e.g., `{{ $json['Full Name'] }}`, `{{ $json.Email }}`)  
  - Input: Raw form fields from "On form submission" node  
  - Output: JSON with structured lead data and score  
  - Edge Cases: AI parsing errors, unexpected input formats, incomplete data from form, malformed JSON output  
  - Version: 1

- **Edit Fields**  
  - Type: Set Node  
  - Role: Post-processes AI output to set a boolean `qualified` field where score ≥ 70 indicates qualification  
  - Configuration: Assigns `"qualified" = {{ $json["output"]["score"] >= 70 }}`  
  - Input: Output from Information Extractor  
  - Output: Enriched JSON with explicit `qualified` boolean  
  - Edge Cases: Missing or invalid score field could cause assignment failure  
  - Version: 3.4  

#### 1.3 Qualification Logic

**Overview:**  
Determines if the lead is qualified and routes the workflow accordingly.

**Nodes Involved:**  
- If

**Node Details:**

- **If**  
  - Type: Conditional Node  
  - Role: Checks if `qualified` is true to decide on further processing  
  - Configuration: Condition set to boolean true on `$json.qualified` with strict type validation and case sensitivity  
  - Input: Output from Edit Fields node  
  - Output: True branch for qualified leads (routes to Airtable and Gmail), false branch omitted (no further action)  
  - Edge Cases: Missing `qualified` field, type mismatch, unexpected boolean values  
  - Version: 2.2  
  - Always outputs data to maintain workflow stability  

#### 1.4 CRM Integration

**Overview:**  
Stores qualified lead data into an Airtable base for CRM management.

**Nodes Involved:**  
- Airtable

**Node Details:**

- **Airtable**  
  - Type: Airtable Node  
  - Role: Creates a new record in a defined Airtable base and table with lead details  
  - Configuration: Uses dynamic expressions to map AI-extracted fields to Airtable columns, including Name, Email, Score, Budget, Location, and Timeline  
  - Credentials: Requires Airtable API token credentials  
  - Input: True branch output from If node (qualified lead)  
  - Output: Confirmation of record creation  
  - Edge Cases: API authentication failures, base/table misconfiguration, data type mismatch, rate limiting  
  - Version: 2.1  

#### 1.5 Follow-Up Notification

**Overview:**  
Sends an email notification alerting the agent of a new qualified lead.

**Nodes Involved:**  
- Gmail

**Node Details:**

- **Gmail**  
  - Type: Gmail Node  
  - Role: Sends an email to the lead's email address with summary details of the qualified lead  
  - Configuration:  
    - Recipient: dynamic to lead’s email  
    - Subject: "🔥 NEW QUALIFIED LEAD"  
    - Body: Includes Name, Email, Location, Budget, Timeline, and Score extracted by AI  
  - Credentials: Requires OAuth2 Gmail credentials  
  - Input: True branch output from If node (qualified lead)  
  - Output: Email sent status  
  - Edge Cases: SMTP authentication errors, invalid email addresses, Gmail API limits, connectivity issues  
  - Version: 2.1  

---

### 3. Summary Table

| Node Name           | Node Type                           | Functional Role                 | Input Node(s)          | Output Node(s)      | Sticky Note                                                                                              |
|---------------------|-----------------------------------|--------------------------------|-----------------------|---------------------|--------------------------------------------------------------------------------------------------------|
| On form submission   | Form Trigger                      | Capture lead input from form    | -                     | Information Extractor |                                                                                                        |
| OpenAI Chat Model    | Langchain OpenAI Chat             | AI language model engine        | -                     | Information Extractor |                                                                                                        |
| Information Extractor| Langchain Information Extractor   | Parse and score lead data       | On form submission, OpenAI Chat Model | Edit Fields         |                                                                                                        |
| Edit Fields         | Set Node                         | Assign qualified boolean based on score | Information Extractor | If                  |                                                                                                        |
| If                  | Conditional Node                 | Route based on qualification    | Edit Fields            | Airtable, Gmail      |                                                                                                        |
| Airtable            | Airtable Node                   | Store qualified lead in CRM     | If                     | Gmail                |                                                                                                        |
| Gmail               | Gmail Node                      | Email notification for qualified leads | If                     | -                   |                                                                                                        |
| Sticky Note         | Sticky Note                     | Visual annotation - Flow        | -                      | -                   | ## Flow                                                                                                |
| Sticky Note1        | Sticky Note                     | Visual annotation - Engine      | -                      | -                   | ## Engine                                                                                              |
| Sticky Note2        | Sticky Note                     | Visual annotation - CRM         | -                      | -                   | ## CRM                                                                                                 |
| Sticky Note3        | Sticky Note                     | Visual annotation - Follow Up   | -                      | -                   | ## Follow Up                                                                                           |
| Sticky Note4        | Sticky Note                     | Workflow summary & setup notes  | -                      | -                   | ## AI Agent Leads Processor  \n- Problem: Low-quality leads waste time, manual qualification burnout \n- Solution: AI scoring, CRM logging, email alerts \n- For solo/small agencies handling 10-50 inquiries/week \n- Scope: Auto intake, AI scoring, email alerts, Airtable logging \n- Setup: Replace form, link API keys, activate, test |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Add a **Form Trigger** node named "On form submission".  
   - Configure webhook with form titled "Property Form".  
   - Add fields (all required): Full Name, Email, Budget Range, Preferred Location, Purchase Timeline, Property Type.  
   - Set form description to "Fill the form to buy!".  

2. **Add OpenAI Chat Model Node**  
   - Add a **Langchain OpenAI Chat** node named "OpenAI Chat Model".  
   - Choose model "gpt-4o-mini".  
   - Link OpenAI API credentials.  
   - Leave options default.  

3. **Add Information Extractor Node**  
   - Add a **Langchain Information Extractor** node named "Information Extractor".  
   - Configure prompt with instructions to parse input fields (use expressions to bind form data):  
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
   - Use "fromJson" schema type with example JSON reflecting expected output.  
   - Connect "On form submission" main output to this node's input.  
   - Connect "OpenAI Chat Model" AI language model input to this node's AI input.  

4. **Add Set Node to Edit Fields**  
   - Add a **Set** node named "Edit Fields".  
   - Add assignment: `qualified` (boolean) = evaluate if `output.score >= 70`.  
   - Connect "Information Extractor" output to "Edit Fields" input.  

5. **Add If Node**  
   - Add an **If** node named "If".  
   - Set condition: `$json.qualified` is boolean true (strict validation).  
   - Connect "Edit Fields" output to "If" input.  

6. **Add Airtable Node for CRM**  
   - Add an **Airtable** node named "Airtable".  
   - Configure operation: Create record.  
   - Select your Airtable base and table (set these lists according to your account).  
   - Map fields using expressions from `Information Extractor` output: Name, Email, Score, Budget (budget_min), Location, Timeline.  
   - Connect "If" node true output to this node.  
   - Link Airtable API token credentials.  

7. **Add Gmail Node for Notification**  
   - Add a **Gmail** node named "Gmail".  
   - Configure to send email to lead's email address dynamically (`={{ $('Information Extractor').item.json.output.email }}`).  
   - Set subject: "🔥 NEW QUALIFIED LEAD".  
   - Compose message body with lead details from `Information Extractor` output (Name, Email, Location, Budget, Timeline, Score).  
   - Connect "If" node true output to this node.  
   - Link Gmail OAuth2 credentials.  

8. **Connect Nodes**  
   - Connect "On form submission" → "Information Extractor" → "Edit Fields" → "If"  
   - Connect "If" true output → "Airtable" and "Gmail" nodes  

9. **Add Sticky Notes (Optional for clarity)**  
   - Add sticky notes to visually group "Flow", "Engine", "CRM", and "Follow Up" sections.  
   - Add a detailed sticky note summarizing problem, solution, scope, and setup instructions as per workflow description.  

10. **Activate Workflow**  
    - Ensure all credentials are set and valid.  
    - Activate the workflow and test by submitting dummy leads through the configured form.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                         | Context or Link                         |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------|
| The AI Agent Leads Processor addresses inefficiencies in manual lead qualification by automating scoring and CRM logging, enabling agents to focus on high-quality prospects only.                                                | Workflow Sticky Note4 (summary note)  |
| Setup requires replacing the sample form with your own (Typeform, Google Form, etc.), linking OpenAI and Gmail credentials, and configuring your Airtable base and table appropriately.                                          | Workflow Sticky Note4 (setup notes)   |
| No built-in WhatsApp replies or auto-scheduling features included; these can be added as optional extensions.                                                                                                                     | Workflow Sticky Note4 (scope limitations) |
| For detailed OpenAI API usage, refer to: https://platform.openai.com/docs/api-reference/chat/create                                                                                                                             | External Resource                     |
| For Airtable API documentation and base/table setup: https://airtable.com/api                                                                                                                                                       | External Resource                     |
| Gmail OAuth2 setup instructions and limitations: https://developers.google.com/gmail/api/guides                                                                                                                                    | External Resource                     |

---

**Disclaimer:**  
The provided content is exclusively derived from an n8n automated workflow. It complies fully with all applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and publicly available.