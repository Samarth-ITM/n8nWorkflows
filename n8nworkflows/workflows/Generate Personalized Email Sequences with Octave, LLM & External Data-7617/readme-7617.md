Generate Personalized Email Sequences with Octave, LLM & External Data

https://n8nworkflows.xyz/workflows/generate-personalized-email-sequences-with-octave--llm---external-data-7617


# Generate Personalized Email Sequences with Octave, LLM & External Data

### 1. Workflow Overview

This workflow automates the generation and delivery of personalized outbound email sequences by dynamically incorporating real-time context about prospect companies. It targets growth teams, SDRs, and outbound marketers who want to move beyond static email sequences by referencing timely, relevant events such as hiring activities or recent funding rounds.

The workflow's logic is organized into five main blocks:

- **1.1 Input Reception:** Captures inbound lead data via webhook.
- **1.2 Context Research:** Queries an external AI agent to derive timely contextual information (e.g., open job roles).
- **1.3 LLM Processing:** Uses a large language model to process and normalize contextual data.
- **1.4 Dynamic Sequence Generation:** Leverages Octave’s sequence agent to create customized email sequences using enriched runtime context.
- **1.5 Email Campaign Integration:** Pushes the personalized sequences to an email platform for campaign execution.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives lead data in real time via an HTTP webhook, serving as the workflow’s entry point.

- **Nodes Involved:**  
  - Lead Data Webhook

- **Node Details:**  
  - **Lead Data Webhook**  
    - *Type:* Webhook  
    - *Role:* Listens for incoming HTTP requests containing lead information (e.g., first name, company domain, email).  
    - *Configuration:*  
      - Webhook path must be replaced with a custom endpoint (`your-webhook-path-here`).  
      - No authentication or advanced options enabled by default; expects JSON payload with lead info.  
    - *Input/Output:*  
      - Input: HTTP POST request from lead source.  
      - Output: JSON object with lead data passed to the next node.  
    - *Edge Cases / Failures:*  
      - Missing or malformed payloads may cause downstream errors.  
      - Webhook path conflict or misconfiguration will block data reception.  
    - *Sticky Note:* "🚀 START HERE\nReplace webhook path.\nConfigure lead source."

#### 2.2 Context Research

- **Overview:**  
  This block extracts timely contextual information about the prospect's company, such as current hiring needs, by querying an external AI agent configured for context research.

- **Nodes Involved:**  
  - Research Company Context

- **Node Details:**  
  - **Research Company Context**  
    - *Type:* Langchain Agent  
    - *Role:* Uses an AI agent to analyze the input company domain and output a relevant open role or a plausible dummy role if none is found.  
    - *Configuration:*  
      - Custom prompt instructs agent to output only the normalized job title without justification or extra text.  
    - *Key Expressions:*  
      - Uses the expression: `{{ $json.body.companyDomain }}` to inject the company domain from the webhook data.  
    - *Input/Output:*  
      - Input: Lead data JSON from webhook.  
      - Output: A string containing a single job title.  
    - *Edge Cases / Failures:*  
      - Agent might fail if company domain is missing or malformed.  
      - AI output could be irrelevant if prompt is misunderstood; no fallback beyond dummy role provided.  
    - *Sticky Note:* "🔍 CONTEXT RESEARCH\nReplace with your data source.\nJob boards, news, enrichment."

#### 2.3 LLM Processing

- **Overview:**  
  This block integrates a large language model (LLM) to further process or normalize the contextual information, ensuring consistent and usable output.

- **Nodes Involved:**  
  - LLM Model

- **Node Details:**  
  - **LLM Model**  
    - *Type:* Langchain LM Chat Anthropic  
    - *Role:* Hosts the LLM responsible for natural language understanding and normalization within the workflow.  
    - *Configuration:*  
      - Model to be selected from a list (e.g., Anthropic Claude or other).  
      - Credentials for LLM API must be configured (`Your LLM API Credentials`).  
    - *Input/Output:*  
      - Input: The prompt or context from previous node(s).  
      - Output: Processed text passed to next step.  
    - *Edge Cases / Failures:*  
      - API authentication errors if credentials are invalid.  
      - Timeouts or rate limits depending on provider.  
      - Unexpected LLM responses requiring prompt tuning.  
    - *Sticky Note:* "🧠 LLM PROCESSING\nAdd your LLM credentials.\nCustomize context extraction."

- **Note:** The connection order shows the LLM Model node feeds into the Research Company Context node, indicating that the AI agent uses this model internally.

#### 2.4 Dynamic Sequence Generation

- **Overview:**  
  This block uses the Octave platform’s sequence agent to generate personalized email sequences that incorporate the runtime context (e.g., the hiring role found).

- **Nodes Involved:**  
  - Generate Sequence with Runtime Context

- **Node Details:**  
  - **Generate Sequence with Runtime Context**  
    - *Type:* Octave Node  
    - *Role:* Runs the Octave sequence agent to dynamically create email sequences tailored to the lead and context.  
    - *Configuration:*  
      - Uses lead details such as job title, first name, company name, domain, LinkedIn profile URL from webhook data.  
      - Runtime context is constructed as a string like `"they are hiring a {{ job title }}"`.  
      - Runtime instructions direct the agent to mention the role when reaching out.  
      - Requires Octave API credentials (`Octave API Credentials`).  
    - *Input/Output:*  
      - Input: Lead data JSON plus context string from previous nodes.  
      - Output: Generated email sequences (structured).  
    - *Edge Cases / Failures:*  
      - API authentication or connectivity issues.  
      - Invalid or incomplete runtime context might produce suboptimal sequences.  
      - Missing required fields like company domain or first name.  
    - *Sticky Note:* "⚡ RUNTIME CONTEXT\nOctave + external data.\nConfigure context & instructions."

#### 2.5 Email Campaign Integration

- **Overview:**  
  This block submits the generated email sequences and lead data to an email platform (Instantly) to add the lead to a campaign with customized emails.

- **Nodes Involved:**  
  - Add Lead to Email Campaign

- **Node Details:**  
  - **Add Lead to Email Campaign**  
    - *Type:* HTTP Request  
    - *Role:* Sends a POST request to the email platform API to add the lead with personalized sequence variables.  
    - *Configuration:*  
      - URL: `https://api.instantly.ai/api/v2/leads`  
      - Method: POST  
      - Authentication: Bearer token via `Email Platform API Key` credential.  
      - Body Parameters include campaign ID, email, names, company name, and custom variables for up to 3 emails with subjects extracted from the generated sequences.  
      - Header `Content-Type` set to `application/json`.  
    - *Key Expressions:*  
      - Uses expressions like `{{ $('Lead Data Webhook').item.json.body.email }}` to map lead data.  
      - Extracts email content and subjects from Octave output JSON arrays.  
    - *Input/Output:*  
      - Input: Lead data and generated sequences.  
      - Output: HTTP response from email platform.  
    - *Edge Cases / Failures:*  
      - API errors due to invalid token, campaign ID, or malformed payload.  
      - Network or timeout issues.  
      - Missing sequence data may cause incomplete payloads.  
    - *Sticky Note:* "📧 EMAIL PLATFORM\nDynamic sequences to campaign.\nUpdate platform & variables."

---

### 3. Summary Table

| Node Name                         | Node Type                      | Functional Role                           | Input Node(s)            | Output Node(s)                      | Sticky Note                                                                                  |
|----------------------------------|--------------------------------|------------------------------------------|--------------------------|-----------------------------------|----------------------------------------------------------------------------------------------|
| Lead Data Webhook                | Webhook                       | Receives incoming lead data               | —                        | Research Company Context           | 🚀 START HERE<br>Replace webhook path.<br>Configure lead source.                            |
| Research Company Context         | Langchain Agent               | Derives hiring context from company domain | Lead Data Webhook         | Generate Sequence with Runtime Context | 🔍 CONTEXT RESEARCH<br>Replace with your data source.<br>Job boards, news, enrichment.      |
| LLM Model                       | Langchain LLM Chat Anthropic | Processes and normalizes context data    | (Feeds into Research Company Context) | Research Company Context (indirect) | 🧠 LLM PROCESSING<br>Add your LLM credentials.<br>Customize context extraction.             |
| Generate Sequence with Runtime Context | Octave Node                  | Generates personalized email sequences using runtime context | Research Company Context  | Add Lead to Email Campaign         | ⚡ RUNTIME CONTEXT<br>Octave + external data.<br>Configure context & instructions.           |
| Add Lead to Email Campaign       | HTTP Request                  | Adds lead with sequences to email campaign | Generate Sequence with Runtime Context | —                                 | 📧 EMAIL PLATFORM<br>Dynamic sequences to campaign.<br>Update platform & variables.         |
| Sticky Note - Main Overview      | Sticky Note                   | Provides high-level project explanation  | —                        | —                                 | 🎯 DYNAMIC EMAIL SEQUENCES WITH RUNTIME CONTEXT<br>FOR: Growth teams, SDRs, outbound marketers who want to reference real-time prospect information.<br>... |
| Sticky Note - Webhook Setup      | Sticky Note                   | Highlights webhook config start point    | —                        | —                                 | 🚀 START HERE<br>Replace webhook path.<br>Configure lead source.                            |
| Sticky Note - Context Research   | Sticky Note                   | Advises on context data source replacement | —                        | —                                 | 🔍 CONTEXT RESEARCH<br>Replace with your data source.<br>Job boards, news, enrichment.      |
| Sticky Note - LLM Processing     | Sticky Note                   | Reminds user to add LLM credentials      | —                        | —                                 | 🧠 LLM PROCESSING<br>Add your LLM credentials.<br>Customize context extraction.             |
| Sticky Note - Octave Runtime     | Sticky Note                   | Notes Octave runtime context configuration | —                        | —                                 | ⚡ RUNTIME CONTEXT<br>Octave + external data.<br>Configure context & instructions.           |
| Sticky Note - Email Platform     | Sticky Note                   | Notes email platform integration details | —                        | —                                 | 📧 EMAIL PLATFORM<br>Dynamic sequences to campaign.<br>Update platform & variables.         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Type: Webhook  
   - Name: "Lead Data Webhook"  
   - Parameters:  
     - Set custom webhook path (e.g., `your-webhook-path-here`)  
     - Method: POST  
     - No authentication unless needed  
   - Position at left side of canvas.

2. **Create Langchain LLM Chat Node:**  
   - Type: Langchain LLM Chat Anthropic (or your LLM provider)  
   - Name: "LLM Model"  
   - Parameters:  
     - Select your LLM model from the list (e.g., Claude, GPT)  
     - Configure API credentials under "Anthropic API" or equivalent  
   - Position below webhook node.

3. **Create Langchain Agent Node:**  
   - Type: Langchain Agent  
   - Name: "Research Company Context"  
   - Parameters:  
     - Prompt:  
       ```
       Output the name of an open role that {{ $json.body.companyDomain }} is hiring for. If you can't find a role, make up a plausible one. Normalize titles to common internal terms. Output only the job title.
       ```  
     - Set prompt type to "define".  
   - Connect input from "Lead Data Webhook".  
   - Connect "LLM Model" node as the language model for this agent.

4. **Create Octave Node:**  
   - Type: Octave Sequence Agent  
   - Name: "Generate Sequence with Runtime Context"  
   - Parameters:  
     - Agent ID: your Octave sequence agent ID  
     - Map lead fields from webhook data: jobTitle, firstName, companyName, companyDomain, LinkedInProfile  
     - Runtime context: `"they are hiring a {{ $json.output }}"` (from Research Company Context output)  
     - Runtime instructions: `"mention the role they're hiring for and that's why you reached out"`  
   - Credentials: Set Octave API credentials.  
   - Connect input from "Research Company Context".

5. **Create HTTP Request Node:**  
   - Type: HTTP Request  
   - Name: "Add Lead to Email Campaign"  
   - Parameters:  
     - URL: `https://api.instantly.ai/api/v2/leads`  
     - Method: POST  
     - Authentication: Bearer Token (configure with your email platform API key)  
     - Headers: `Content-Type: application/json`  
     - Body Parameters (JSON):  
       ```json
       {
         "campaign": "your-campaign-id-here",
         "email": "={{ $('Lead Data Webhook').item.json.body.email }}",
         "first_name": "={{ $('Lead Data Webhook').item.json.body.firstName }}",
         "last_name": "={{ $('Lead Data Webhook').item.json.body.lastName }}",
         "company_name": "={{ $('Lead Data Webhook').item.json.body.companyName }}",
         "custom_variables": {
           "email1": "={{ $json.emails[0].email || '' }}",
           "subject1": "={{ $json.emails[0].subject || '' }}",
           "email2": "={{ $json.emails[1].email || '' }}",
           "subject2": "={{ $json.emails[1].subject || '' }}",
           "email3": "={{ $json.emails[2].email || '' }}",
           "subject3": "={{ $json.emails[2].subject || '' }}"
         }
       }
       ```  
   - Connect input from "Generate Sequence with Runtime Context".

6. **Add Sticky Notes for Documentation:**  
   - Add sticky notes at appropriate canvas locations with the provided content for setup guidance, context research, LLM instructions, Octave configuration, and email platform notes.

7. **Check All Credentials:**  
   - Configure and verify:  
     - Webhook accessibility  
     - LLM API credentials  
     - Octave API credentials  
     - Email platform API key (Bearer token)  

8. **Test the Workflow:**  
   - Trigger the webhook with sample lead data including company domain and contact info.  
   - Confirm that context is researched, sequences generated, and lead added to campaign successfully.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                  |
|------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| 🎯 Dynamic email sequences with runtime context enable referencing real-time prospect info for outbound marketing. | Workflow main purpose summary                    |
| Replace the webhook path with your own to integrate your lead source.                                       | Webhook Setup sticky note                        |
| Customize the external data source for context research; can be job boards, news, or data enrichment APIs. | Context Research sticky note                     |
| Add and configure your LLM API credentials to tailor context extraction and normalization.                  | LLM Processing sticky note                       |
| Use Octave to combine external data and runtime instructions to generate personalized sequences.            | Octave Runtime sticky note                        |
| Configure your email platform credentials and update variables to push sequences to campaigns dynamically.   | Email Platform sticky note                        |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.