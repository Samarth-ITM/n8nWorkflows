Automated Sales Follow-up with GPT-4, Tavily Research and Gmail

https://n8nworkflows.xyz/workflows/automated-sales-follow-up-with-gpt-4--tavily-research-and-gmail-7221


# Automated Sales Follow-up with GPT-4, Tavily Research and Gmail

### 1. Workflow Overview

This workflow automates personalized sales follow-ups for inbound leads using advanced AI models and live research tools combined with email automation. When a prospect submits a business inquiry form, the workflow:

- Captures the lead’s contact and inquiry details.
- Uses GPT-5 and Tavily AI to research the lead’s business and context online.
- Crafts a tailored, friendly email with a clear call-to-action inviting the lead to book a meeting.
- Sends the email instantly via Gmail.

**Target Use Cases:**  
Ideal for startups, marketing agencies, SaaS companies, B2B sales teams, and consultants who want to automate and personalize lead follow-ups to increase conversion rates and reduce manual workload.

**Logical Blocks:**  
- **1.1 Input Reception:** Capture form submissions from prospects.  
- **1.2 AI Research & Copywriting:** Use GPT-5 and Tavily to research and generate personalized emails.  
- **1.3 Output Parsing:** Structure the AI-generated email subject and body.  
- **1.4 Email Sending:** Deliver the personalized email via Gmail.  
- **1.5 Memory & AI Model Connections:** Manage session memory and link AI model and research tools.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures inbound lead data via an online form titled “Business Inquiry” with required and optional fields.

- **Nodes Involved:**  
  - On form submission  
  - Sticky Note (Form Submission Trigger)

- **Node Details:**  

  **On form submission**  
  - *Type & Role:* Form Trigger node; starts workflow on form submission.  
  - *Configuration:* Form titled “Business Inquiry” with these fields: First Name (required), Last Name (required), Business URL (required), Email (required, email type), Phone Number (optional, number type), and a free text field “How can we help you?”. Description thanks the user for inquiry.  
  - *Inputs:* Triggered externally by form submission.  
  - *Outputs:* Emits lead data as JSON for downstream nodes.  
  - *Edge cases:* Missing required fields prevented by form validation; malformed email or phone number formats; possible form downtime or submission errors.

  **Sticky Note**  
  - Visual annotation labeled "Form Submission Trigger" to clarify node purpose.

---

#### 2.2 AI Research & Copywriting

- **Overview:**  
  Processes lead data through GPT-5 agent enhanced by live Tavily AI research and session memory to compose a personalized sales follow-up email.

- **Nodes Involved:**  
  - GPT-5 Research & Copywriting (LangChain Agent)  
  - Tavily (Research Tool)  
  - Simple Memory (Session Memory)  
  - OpenAI Chat Model (GPT-5 model)  
  - Sticky Note1 (GPT-5 Inbound Lead Research & Email Writing Agent)

- **Node Details:**  

  **GPT-5 Research & Copywriting**  
  - *Type & Role:* LangChain Agent node; orchestrates AI prompt, research, and output.  
  - *Configuration:*  
    - Input text template includes all captured lead fields (name, business URL, email, phone, inquiry).  
    - System message instructs the agent to research the lead’s business and problem statement, using Tavily for online research, then write a friendly, clear email with a call to action to book a meeting at www.calendly.com/book-now.  
    - Output is JSON with separate “Subject Line” and “Body” fields.  
  - *Inputs:* Receives form submission data.  
  - *Outputs:* Sends structured email content JSON.  
  - *Key expressions:* Template expressions populate prompt with lead info.  
  - *Edge cases:* Failure in Tavily API call; timeouts from AI model; incorrect or incomplete output format from AI; session memory key conflicts.  
  - *Sub-workflow:* None explicitly, but uses Tavily as external AI tool and OpenAI Chat Model as language model.

  **Tavily**  
  - *Type & Role:* External AI research tool node.  
  - *Configuration:* Query string dynamically overridden by AI-generated prompt context.  
  - *Inputs/Outputs:* Acts as an AI tool integration invoked by GPT-5 agent.  
  - *Edge cases:* API limit exceeded; network errors; invalid query inputs.

  **Simple Memory**  
  - *Type & Role:* AI memory buffer window node; maintains conversation context using workflow ID as session key.  
  - *Configuration:* SessionIdType set to custom key using workflow ID for session consistency.  
  - *Edge cases:* Memory overflow or truncation if conversation too long.

  **OpenAI Chat Model**  
  - *Type & Role:* Language model node configured for GPT-5.  
  - *Configuration:* Model set to “gpt-5” with caching enabled.  
  - *Edge cases:* API rate limits; authentication errors; model unavailability.

  **Sticky Note1**  
  - Visual annotation describing the block as “GPT-5 Inbound Lead Research & Email Writing Agent.”

---

#### 2.3 Output Parsing

- **Overview:**  
  Parses the AI agent’s JSON output to extract the email subject line and body for sending.

- **Nodes Involved:**  
  - Structured Output Parser

- **Node Details:**  

  **Structured Output Parser**  
  - *Type & Role:* LangChain output parser node; enforces JSON schema validation and extraction.  
  - *Configuration:* Uses example JSON schema with “Subject Line” and “Body” fields as output format.  
  - *Inputs:* Receives AI agent raw output.  
  - *Outputs:* Produces validated, structured JSON for email node.  
  - *Edge cases:* Parsing errors if AI output is malformed or deviates from schema.

---

#### 2.4 Email Sending

- **Overview:**  
  Sends the personalized email to the lead’s email address using Gmail.

- **Nodes Involved:**  
  - Gmail  
  - Sticky Note2 (Send Email Node)

- **Node Details:**  

  **Gmail**  
  - *Type & Role:* Email sending node using Gmail API.  
  - *Configuration:*  
    - Recipient (“sendTo”) dynamically set from form submission email field.  
    - Subject and message set from parsed AI output fields.  
    - Email type set to plain text.  
    - No additional options configured.  
  - *Inputs:* Receives structured email content JSON and form data.  
  - *Outputs:* Confirmation of email sent or error.  
  - *Edge cases:* Authentication failures (OAuth token expired or missing), invalid recipient email format, Gmail API rate limits, network failures.

  **Sticky Note2**  
  - Visual annotation labeled “Send Email Node.”

---

#### 2.5 Memory & AI Model Connections

- **Overview:**  
  Support nodes that connect the AI agent to memory, AI model, and research tool nodes.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - Tavily  
  - Simple Memory  

- **Node Details:**  
  These nodes serve as interfaces and are connected via AI tool, AI language model, and AI memory ports to the GPT-5 Research & Copywriting agent node, enabling multi-modal AI integration.

---

#### 2.6 Documentation & Workflow Context

- **Nodes Involved:**  
  - Sticky Note3 (Workflow overview and marketing description)

- **Node Details:**  

  **Sticky Note3**  
  - Contains a detailed description of the workflow purpose, benefits, use cases, and links to tutorial videos:  
    - Highlights AI-powered research, natural copywriting, instant follow-ups, and no-code automation.  
    - Link: https://www.youtube.com/@Automatewithmarc  
  - Useful for onboarding and understanding the workflow context.

---

### 3. Summary Table

| Node Name                   | Node Type                                  | Functional Role                      | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                                     |
|-----------------------------|--------------------------------------------|------------------------------------|------------------------|--------------------------|----------------------------------------------------------------------------------------------------------------|
| On form submission          | Form Trigger                               | Capture inbound lead form data     | —                      | GPT-5 Research & Copywriting | Form Submission Trigger                                                                                         |
| GPT-5 Research & Copywriting | LangChain Agent                            | AI research and email drafting     | On form submission      | Gmail                    | GPT-5 Inbound Lead Research & Email Writing Agent                                                              |
| Tavily                      | Tavily AI Research Tool                    | Online business research for leads | GPT-5 Research & Copywriting (ai_tool) | GPT-5 Research & Copywriting |                                                                                                                |
| Simple Memory               | LangChain Memory Buffer                    | Store AI session context           | GPT-5 Research & Copywriting (ai_memory) | GPT-5 Research & Copywriting |                                                                                                                |
| OpenAI Chat Model           | Language Model (GPT-5)                     | Provide GPT-5 AI model             | GPT-5 Research & Copywriting (ai_languageModel) | GPT-5 Research & Copywriting |                                                                                                                |
| Structured Output Parser    | LangChain Output Parser                     | Parse AI output JSON               | GPT-5 Research & Copywriting (ai_outputParser) | GPT-5 Research & Copywriting |                                                                                                                |
| Gmail                      | Gmail Node                                 | Send personalized email            | GPT-5 Research & Copywriting | —                        | Send Email Node                                                                                                |
| Sticky Note                 | Sticky Note                                | Annotation                        | —                      | —                        | Form Submission Trigger                                                                                        |
| Sticky Note1                | Sticky Note                                | Annotation                        | —                      | —                        | GPT-5 Inbound Lead Research & Email Writing Agent                                                             |
| Sticky Note2                | Sticky Note                                | Annotation                        | —                      | —                        | Send Email Node                                                                                                |
| Sticky Note3                | Sticky Note                                | Documentation & marketing overview | —                      | —                        | 🚀 GPT-5 AI Lead Research & Auto-Email Agent – Instant Personalized Follow-Ups for Inbound Leads; tutorial link: https://www.youtube.com/@Automatewithmarc |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**:  
   - Type: Form Trigger  
   - Configure form title as “Business Inquiry”.  
   - Add fields:  
     - First Name (required)  
     - Last Name (required)  
     - Business URL (required)  
     - Email (required, type email)  
     - Phone Number (optional, type number)  
     - How can we help you? (optional text)  
   - Add form description: “Thank you for your inquiry, we'll get back to you soon!”  
   - Position node at workflow start.

2. **Add GPT-5 Research & Copywriting Agent Node**:  
   - Type: LangChain Agent node  
   - Connect input from “On form submission”.  
   - Set prompt text template to include all lead fields (First Name, Last Name, Business URL, Email, Phone Number, Inquiry).  
   - Add system message instructing:  
     - You are an inbound lead email agent.  
     - Research lead’s business and problem using Tavily.  
     - Write a friendly email with subject and body, inviting to book a meeting at www.calendly.com/book-now.  
     - Output JSON with “Subject Line” and “Body”.  
   - Enable output parser.  
   - Configure AI tool, memory, and language model connections as below.

3. **Add Tavily AI Research Tool Node**:  
   - Type: Tavily  
   - Connect as AI tool to the GPT-5 agent node.  
   - Configure query to be dynamically overridden by AI prompt context.

4. **Add Simple Memory Node**:  
   - Type: LangChain Memory Buffer Window  
   - Set session key to workflow ID or unique identifier.  
   - Connect as AI memory to the GPT-5 agent node.

5. **Add OpenAI Chat Model Node**:  
   - Type: OpenAI Chat Model (LangChain)  
   - Set model to “gpt-5”.  
   - Connect as AI language model to GPT-5 agent node.  
   - Provide OpenAI API credentials with GPT-5 access.

6. **Add Structured Output Parser Node**:  
   - Type: LangChain Output Parser Structured  
   - Use example JSON schema with fields:  
     ```json
       {
         "Subject Line": "string",
         "Body": "string"
       }
     ```  
   - Connect AI output parser port from GPT-5 agent.  
   - This ensures output is valid JSON for email content.

7. **Add Gmail Node**:  
   - Type: Gmail  
   - Connect main input from GPT-5 agent’s main output (after parsing).  
   - Set “Send To” to the lead’s email from form submission data.  
   - Set “Subject” to parsed “Subject Line” field.  
   - Set “Message” to parsed “Body” field.  
   - Set email type to plain text.  
   - Configure Gmail OAuth2 credentials with sending permissions.

8. **Add Sticky Notes for Clarity (Optional)**:  
   - Add sticky notes near nodes with descriptions:  
     - “Form Submission Trigger” near form trigger node.  
     - “GPT-5 Inbound Lead Research & Email Writing Agent” near GPT-5 agent node.  
     - “Send Email Node” near Gmail node.  
     - A large note summarizing the workflow purpose and providing tutorial video link: https://www.youtube.com/@Automatewithmarc

9. **Connect Nodes Sequentially**:  
   - “On form submission” → “GPT-5 Research & Copywriting”  
   - “GPT-5 Research & Copywriting” AI ports: connect to Tavily (ai_tool), Simple Memory (ai_memory), OpenAI Chat Model (ai_languageModel)  
   - GPT-5 agent output parser port → Structured Output Parser  
   - GPT-5 agent main output port → Gmail node

10. **Test Workflow**:  
    - Submit test form with valid lead data.  
    - Verify AI generates personalized email subject and body.  
    - Confirm email is sent to the test recipient.  
    - Monitor for errors in AI calls, parsing, and Gmail sending.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| 🚀 GPT-5 AI Lead Research & Auto-Email Agent – Instant Personalized Follow-Ups for Inbound Leads. Use this workflow to turn every inbound lead into a booked meeting automatically by combining lead capture, AI research, copywriting, and email sending. Free tutorial videos for similar workflows are available at https://www.youtube.com/@Automatewithmarc. Key features include AI-powered research for up-to-date insights, natural, persuasive copywriting, and instant follow-ups. Perfect for startups, agencies, SaaS, and B2B sales. | https://www.youtube.com/@Automatewithmarc                       |
| The workflow’s call-to-action directs leads to book meetings at www.calendly.com/book-now. Adjust this URL as needed for your scheduling system.                                                                                                                                                                                                                                                                                                                                                                          | www.calendly.com/book-now                                        |
| Ensure that API credentials for OpenAI GPT-5 and Gmail OAuth2 are correctly configured and have appropriate permissions before activating the workflow. Monitor API usage quotas and errors to maintain smooth operation.                                                                                                                                                                                                                                                                                                     | n8n Credentials Setup                                            |

---

**Disclaimer:**  
The text above is exclusively derived from an automated workflow created with n8n, respecting all content policies. It contains no illegal, offensive, or protected material. All processed data is legal and public.