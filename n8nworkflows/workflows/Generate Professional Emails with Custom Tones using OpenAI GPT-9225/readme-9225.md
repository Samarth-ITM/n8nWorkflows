Generate Professional Emails with Custom Tones using OpenAI GPT

https://n8nworkflows.xyz/workflows/generate-professional-emails-with-custom-tones-using-openai-gpt-9225


# Generate Professional Emails with Custom Tones using OpenAI GPT

---

### 1. Workflow Overview

This workflow, titled **"Generate Professional Emails with Custom Tones using OpenAI GPT"**, automates the creation of professional emails based on user inputs and tone preferences. It is designed for business professionals, customer support, sales teams, or anyone needing quick, tone-customized email drafts.

The logic is divided into the following key blocks:

- **1.1 User Input Collection:** A web form collects recipient name, email subject, email context, and tone preference.
- **1.2 Data Preparation:** Extracts and organizes form data into structured variables.
- **1.3 AI Prompt Construction:** Builds a detailed AI prompt including tone instructions tailored to the user's tone choice.
- **1.4 AI Email Generation:** Uses OpenAI GPT (via LangChain integration) to generate the email content based on the prompt.
- **1.5 Output Formatting & Display:** Formats the AI output and presents the generated email in a user-friendly form for copying.

Supporting these blocks are informative sticky notes that provide descriptions, setup instructions, and step-by-step guidance.

---

### 2. Block-by-Block Analysis

#### 2.1 User Input Collection

- **Overview:**  
  This block gathers all required email information from the user via a form interface, including recipient, subject, context, and preferred tone.

- **Nodes Involved:**  
  - Email Generator Form

- **Node Details:**  

  - **Email Generator Form**  
    - **Type:** Form Trigger node  
    - **Role:** Entry point; presents a user form to collect inputs  
    - **Configuration:**  
      - Form title: "AI Email Generator"  
      - Fields:  
        - Recipient Name (required, text)  
        - Email Subject (required, text)  
        - Email Context (required, textarea)  
        - Tone (required, dropdown with options: Professional, Friendly, Formal, Casual)  
      - Webhook ID: "email-gen-form"  
      - Description: "Generate professional emails with your chosen tone"  
    - **Input/Output:** Incoming webhook request triggers node → outputs form data JSON  
    - **Edge Cases:**  
      - Missing required fields blocks submission  
      - Invalid or unexpected tone values (handled downstream)  
    - **Version Requirements:** n8n v2.2 or higher preferred for form trigger stability  

#### 2.2 Data Preparation

- **Overview:**  
  Extracts individual form fields into clearly named variables to simplify downstream usage.

- **Nodes Involved:**  
  - Extract Form Data

- **Node Details:**  

  - **Extract Form Data**  
    - **Type:** Set node  
    - **Role:** Assigns extracted form fields to named variables for clarity  
    - **Configuration:**  
      - Assigns:  
        - recipient ← "Recipient Name" from form JSON  
        - subject ← "Email Subject"  
        - context ← "Email Context"  
        - tone ← "Tone"  
    - **Input:** Output from Email Generator Form  
    - **Output:** JSON object with four keys: recipient, subject, context, tone  
    - **Edge Cases:**  
      - If any form field is missing, values become undefined (should be prevented by form validation)  
    - **Version Requirements:** Version 3.4 or later recommended for robust variable assignment  

#### 2.3 AI Prompt Construction

- **Overview:**  
  Constructs a detailed prompt text that instructs the AI how to write an email based on inputs and tone.

- **Nodes Involved:**  
  - Build AI Prompt

- **Node Details:**  

  - **Build AI Prompt**  
    - **Type:** Code node (JavaScript)  
    - **Role:** Creates a prompt string for the AI with custom tone instructions  
    - **Configuration:**  
      - Extracts recipient, subject, context, and tone from input JSON  
      - Uses a switch statement to select tone-specific instructions:  
        - Professional: clear, concise, respectful, business-appropriate  
        - Friendly: warm, conversational, approachable but professional  
        - Formal: highly formal, diplomatic, strict formality  
        - Casual: relaxed, conversational, natural  
        - Default: generic tone description  
      - Builds a multi-line prompt specifying:  
        - Recipient, subject, context  
        - Tone instructions  
        - Formatting instructions (greeting, body, closing, no signature line)  
      - Returns JSON containing prompt, recipient, subject, tone  
    - **Key Expressions:** Uses `$input.item.json` to reference variables; `.toLowerCase()` for tone normalization  
    - **Input:** JSON from Extract Form Data  
    - **Output:** JSON with the constructed prompt text and relevant metadata  
    - **Edge Cases:**  
      - Unexpected tone strings default to generic instructions  
      - Code errors or syntax issues could break prompt generation  
    - **Version Requirements:** Version 2 or higher for JavaScript code node  

#### 2.4 AI Email Generation

- **Overview:**  
  The AI model receives the prompt and generates the email content accordingly.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - Generate Email

- **Node Details:**  

  - **OpenAI Chat Model**  
    - **Type:** LangChain OpenAI Chat Model node  
    - **Role:** Provides the AI language model interface for generation  
    - **Configuration:**  
      - Model: gpt-3.5-turbo  
      - Max tokens: 500 (controls response length)  
      - Temperature: 0.7 (moderate creativity)  
    - **Credentials:** OpenAI API key configured via n8n credential "OpenAi account"  
    - **Input:** Receives prompt text from Generate Email node (linked via LangChain)  
    - **Output:** AI-generated response to prompt (email content)  
    - **Edge Cases:**  
      - API key invalid or rate limits reached cause errors  
      - Timeout or network issues with OpenAI endpoint  
    - **Version Requirements:** LangChain integration enabled, n8n version supporting LangChain nodes  

  - **Generate Email**  
    - **Type:** LangChain Agent node  
    - **Role:** Defines and sends the prompt to the OpenAI Chat Model node and retrieves the response  
    - **Configuration:**  
      - Prompt text assigned via expression `={{ $json.prompt }}`  
      - Prompt type: "define" (indicates custom prompt)  
    - **Input:** JSON containing prompt from Build AI Prompt  
    - **Output:** JSON containing AI-generated email content in `output` field  
    - **Edge Cases:**  
      - If no prompt text, generation fails  
      - AI output may be incomplete or off-tone if prompt is ambiguous  
    - **Version Requirements:** Version 2.2 or higher for LangChain agent node  

#### 2.5 Output Formatting & Display

- **Overview:**  
  Formats the generated email content with metadata and displays it in a completion form for the user to copy.

- **Nodes Involved:**  
  - Format Output  
  - Display Generated Email

- **Node Details:**  

  - **Format Output**  
    - **Type:** Set node  
    - **Role:** Organizes AI output and original form data for final display  
    - **Configuration:**  
      - Assigns:  
        - email_body ← AI output text (`$json.output`)  
        - subject ← original subject from Extract Form Data  
        - tone ← original tone from Extract Form Data  
        - recipient ← original recipient  
    - **Input:** Output from Generate Email  
    - **Output:** JSON with all display-ready fields  
    - **Edge Cases:**  
      - AI output missing or malformed leads to empty email body  
    - **Version Requirements:** 3.4 or higher recommended  

  - **Display Generated Email**  
    - **Type:** Form node (completion operation)  
    - **Role:** Shows the generated email in a formatted message after generation  
    - **Configuration:**  
      - Completion title: "✅ Email Generated Successfully!"  
      - Completion message uses Markdown formatting to display:  
        - Subject, Tone, Recipient  
        - The generated email body with separators  
        - Instruction to copy and add signature manually  
      - Webhook ID: "email-completion"  
    - **Input:** JSON from Format Output  
    - **Output:** Displays read-only completion form to user via webhook response  
    - **Edge Cases:**  
      - Display errors if input JSON malformed  
      - User experience depends on frontend rendering  
    - **Version Requirements:** v1 or higher of form node completion operation  

---

### 3. Summary Table

| Node Name               | Node Type                           | Functional Role           | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                          |
|-------------------------|-----------------------------------|---------------------------|------------------------|-------------------------|----------------------------------------------------------------------------------------------------|
| Email Generator Form     | Form Trigger                      | Collect user email inputs | (webhook trigger)      | Extract Form Data        | **Step 1:** User Input<br>Form collects email details and tone preference                           |
| Extract Form Data        | Set                              | Extract and organize form data | Email Generator Form    | Build AI Prompt          | **Step 2:** Extract & Organize<br>Prepare data for AI processing                                   |
| Build AI Prompt          | Code                             | Build AI prompt text with tone instructions | Extract Form Data        | Generate Email           | **Step 3:** Build Prompt<br>Create AI instructions based on selected tone                           |
| OpenAI Chat Model        | LangChain LM Chat OpenAI          | AI language model interface | Generate Email (ai_languageModel) | Generate Email           | ⚙️ **Configure Your API Key Here**<br>Add your OpenAI credentials                                  |
| Generate Email           | LangChain Agent                   | Send prompt & receive AI output | Build AI Prompt          | Format Output            | **Step 4:** AI Generation<br>OpenAI creates the email content                                      |
| Format Output            | Set                              | Format AI output & metadata | Generate Email           | Display Generated Email  | **Step 5:** Format & Display<br>Show the generated email to the user                               |
| Display Generated Email  | Form (completion operation)      | Display generated email    | Format Output            | (end)                   |                                                                                                    |
| Main Description         | Sticky Note                      | Project overview & instructions | (none)                 | (none)                  | ![5min Logo](https://assets.zyrosite.com/cdn-cgi/image/format=auto,w=175,q=95/d9573nnb9LioR05g/logo_tr-Yleq6B4J5WtDk64q.png)<br> Detailed overview and usage instructions |
| Setup Instructions       | Sticky Note                      | Setup steps for API and customization | (none)                 | (none)                  | Detailed API key setup and customization steps                                                    |
| Step 1                  | Sticky Note                      | Describes step 1 user input | (none)                 | (none)                  | **Step 1:** User Input<br>Form collects email details and tone preference                           |
| Step 2                  | Sticky Note                      | Describes step 2 data extraction | (none)                 | (none)                  | **Step 2:** Extract & Organize<br>Prepare data for AI processing                                   |
| Step 3                  | Sticky Note                      | Describes step 3 prompt build | (none)                 | (none)                  | **Step 3:** Build Prompt<br>Create AI instructions based on selected tone                           |
| Step 4                  | Sticky Note                      | Describes step 4 AI generation | (none)                 | (none)                  | **Step 4:** AI Generation<br>OpenAI creates the email content                                      |
| Step 5                  | Sticky Note                      | Describes step 5 formatting and display | (none)                 | (none)                  | **Step 5:** Format & Display<br>Show the generated email to the user                               |
| API Key Note            | Sticky Note                      | Reminder to configure OpenAI API key | (none)                 | (none)                  | ⚙️ **Configure Your API Key Here**<br>Add your OpenAI credentials                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node: "Email Generator Form"**  
   - Node Type: Form Trigger  
   - Configure form title: "AI Email Generator"  
   - Add fields (all required):  
     - Recipient Name (text)  
     - Email Subject (text)  
     - Email Context (textarea)  
     - Tone (dropdown): Options = Professional, Friendly, Formal, Casual  
   - Set webhook ID: "email-gen-form"  
   - Save node  

2. **Create a Set Node: "Extract Form Data"**  
   - Node Type: Set  
   - Assign variables:  
     - recipient = `{{$json["Recipient Name"]}}`  
     - subject = `{{$json["Email Subject"]}}`  
     - context = `{{$json["Email Context"]}}`  
     - tone = `{{$json["Tone"]}}`  
   - Connect "Email Generator Form" → "Extract Form Data"  

3. **Create a Code Node: "Build AI Prompt"**  
   - Node Type: Code (JavaScript)  
   - Paste the following JavaScript logic:  

```javascript
const recipient = $input.item.json.recipient;
const subject = $input.item.json.subject;
const context = $input.item.json.context;
const tone = $input.item.json.tone.toLowerCase();

let toneInstructions = '';

switch(tone) {
  case 'professional':
    toneInstructions = 'Use a professional, business-appropriate tone. Be clear, concise, and respectful. Maintain a balance between friendliness and formality.';
    break;
  case 'friendly':
    toneInstructions = 'Use a warm, friendly tone while remaining appropriate. Be conversational and approachable, but maintain professionalism.';
    break;
  case 'formal':
    toneInstructions = 'Use a highly formal, diplomatic tone. Use proper salutations and maintain strict formality throughout. Be respectful and courteous.';
    break;
  case 'casual':
    toneInstructions = 'Use a casual, relaxed tone. Be conversational and natural, as if writing to a colleague or friend.';
    break;
  default:
    toneInstructions = `Use a ${tone} tone.`;
}

const prompt = `Write a complete email with the following details:

Recipient: ${recipient}
Subject: ${subject}
Context: ${context}

Tone Instructions: ${toneInstructions}

Format the email properly with:
- An appropriate greeting
- A clear body that addresses the context
- A professional closing
- Do not include a signature line (the user will add their own)

Write only the email content, nothing else.`;

return {
  json: {
    prompt: prompt,
    recipient: recipient,
    subject: subject,
    tone: tone
  }
};
```
   - Connect "Extract Form Data" → "Build AI Prompt"  

4. **Create a LangChain OpenAI Chat Model Node: "OpenAI Chat Model"**  
   - Node Type: LangChain LM Chat OpenAI  
   - Model: gpt-3.5-turbo  
   - Options: maxTokens = 500, temperature = 0.7  
   - Assign OpenAI credentials (must create credential with API key via n8n Credentials panel)  
   - Keep node ready for downstream connections  

5. **Create a LangChain Agent Node: "Generate Email"**  
   - Node Type: LangChain Agent  
   - Parameters:  
     - Text: `={{ $json.prompt }}`  
     - Prompt type: "define"  
   - Connect "Build AI Prompt" → "Generate Email" (main connection)  
   - Connect "OpenAI Chat Model" → "Generate Email" (ai_languageModel connection)  

6. **Create a Set Node: "Format Output"**  
   - Node Type: Set  
   - Assign:  
     - email_body = `={{ $json.output }}` (AI generated email content)  
     - subject = `={{ $('Extract Form Data').item.json.subject }}`  
     - tone = `={{ $('Extract Form Data').item.json.tone }}`  
     - recipient = `={{ $('Extract Form Data').item.json.recipient }}`  
   - Connect "Generate Email" → "Format Output"  

7. **Create a Form Node (Completion): "Display Generated Email"**  
   - Node Type: Form (operation: completion)  
   - Configure:  
     - Completion Title: "✅ Email Generated Successfully!"  
     - Completion Message (Markdown):  
```
**Subject:** {{ $json.subject }}

**Tone:** {{ $json.tone }}

**Recipient:** {{ $json.recipient }}

---

{{ $json.email_body }}

---

*Copy the email above and add your signature before sending.*
```
     - Webhook ID: "email-completion"  
   - Connect "Format Output" → "Display Generated Email"  

8. **Activate the Workflow**  
   - Test by accessing the form webhook URL (provided by n8n after activation)  
   - Fill in details and submit to see generated email  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                         | Context or Link                                             |
|------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| Workflow created by Biznova with logo and branding: ![5min Logo](https://assets.zyrosite.com/cdn-cgi/image/format=auto,w=175,q=95/d9573nnb9LioR05g/logo_tr-Yleq6B4J5WtDk64q.png) | Project branding and creator info                           |
| Setup instructions emphasize obtaining OpenAI API key at https://platform.openai.com/api-keys and configuring it in the workflow credentials panel | OpenAI API key setup                                        |
| Workflow supports customization of tones by editing the "Build AI Prompt" node and form dropdown options                                            | Tone customization guide                                    |
| Suggested AI model upgrade to GPT-4 and tuning parameters (temperature, max tokens) for improved email quality                                     | AI model tuning recommendations                             |
| Users are instructed to copy generated email content and add their own signature manually                                                          | User email finalization instructions                        |

---

This document fully dissects the workflow, enabling replication, modification, and error anticipation for advanced users and AI agents alike.

---