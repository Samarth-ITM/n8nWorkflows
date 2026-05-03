Client Proposal Generator with GPT-4o, Tally.so, and Google Slides for Agencies

https://n8nworkflows.xyz/workflows/client-proposal-generator-with-gpt-4o--tally-so--and-google-slides-for-agencies-10777


# Client Proposal Generator with GPT-4o, Tally.so, and Google Slides for Agencies

### 1. Workflow Overview

This workflow automates the creation of customized client proposals for agencies using AI and cloud services. It is designed for scenarios where an agency has just completed a client meeting and wants to quickly generate a professional, detailed proposal and send it via email within minutes. The workflow ingests client input data collected via a form (Tally.so recommended), processes and enriches that data using OpenAI's GPT-4o model, populates a Google Slides presentation template with the generated proposal content, and drafts a follow-up email in Gmail ready for review and sending.

**Target Use Cases:**  
- Agencies needing rapid, high-quality, personalized client proposals after meetings  
- Automating repetitive proposal writing and formatting tasks  
- Integrating AI content generation with Google Slides and Gmail for seamless client communication

**Logical Blocks:**  
- **1.1 Input Reception:** Receiving and structuring client data from a webhook triggered by a form submission.  
- **1.2 AI Proposal Generation:** Using OpenAI GPT-4o to transform raw client data into a fully fleshed proposal JSON.  
- **1.3 Date Formatting:** Formatting current date into a professional string style for proposal use.  
- **1.4 Presentation Preparation:** Copying a Google Slides template and replacing placeholders with AI-generated proposal details.  
- **1.5 Email Drafting:** Creating a draft email in Gmail with a link to the generated proposal for final manual review.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block collects client input via a webhook (from Tally.so form submissions) and assigns the incoming raw data fields to well-defined variables for further processing.

- **Nodes Involved:**  
  - Webhook1  
  - Edit Fields

- **Node Details:**  

  **Webhook1**  
  - *Type & Role:* HTTP Webhook node; entry point receiving POST requests from Tally.so form submissions.  
  - *Configuration:* Listens on path `/unique path` with POST method; receives JSON data containing client info and project details.  
  - *Key Expressions:* N/A (raw JSON from form).  
  - *Input/Output:* No input; outputs webhook payload to next node.  
  - *Potential Failures:* Missing or malformed POST data, webhook path misconfiguration, network errors.  
  - *Note:* This node replaces a disabled Form Trigger node (due to timeout issues).

  **Edit Fields**  
  - *Type & Role:* Set node; maps raw webhook JSON fields to descriptive variables (e.g., firName, lastName, clientBusiness, problem, solution, etc.).  
  - *Configuration:* Assigns each form field value from incoming JSON to named variables for clarity and ease of use downstream.  
  - *Key Expressions:* Uses expressions like `={{ $json.body.data.fields[0].value }}` to extract form data.  
  - *Input/Output:* Receives webhook JSON; outputs structured JSON with named fields.  
  - *Potential Failures:* Incorrect field indexing, missing form fields, malformed JSON.

---

#### 1.2 AI Proposal Generation

- **Overview:**  
  This block uses OpenAI's GPT-4o to convert the structured client input into a detailed, professional proposal formatted as a JSON object with predefined fields.

- **Nodes Involved:**  
  - Presentation Generator

- **Node Details:**  

  **Presentation Generator**  
  - *Type & Role:* OpenAI LLM node via LangChain integration; generates proposal text.  
  - *Configuration:*  
    - Model: GPT-4o  
    - Messages: System prompt positions the AI as a helpful writing agent.  
    - User prompt instructs the AI to generate a spartan, professional proposal JSON with specific fields fully populated.  
    - Includes example input data and expected output format for strong prompt guidance.  
  - *Key Expressions:* Uses expression mode for system and user message content, embedding JSON templates and instructions.  
  - *Input/Output:* Input is the structured client data from "Edit Fields"; output is a JSON object with proposal details.  
  - *Potential Failures:* API rate limits, malformed prompts, incomplete or invalid AI response, network errors, or credential issues.  
  - *Customization:* Prompts can be modified for tone, brand voice, or additional fields.

---

#### 1.3 Date Formatting

- **Overview:**  
  This block formats the current date into a human-friendly, polished string to be used in the proposal and presentation.

- **Nodes Involved:**  
  - Set Date Format

- **Node Details:**  

  **Set Date Format**  
  - *Type & Role:* Code node executing JavaScript to format date.  
  - *Configuration:* Formats `new Date()` into string like "November 8 2025".  
  - *Key Expressions:* Uses JS's `toLocaleString` for month name, extracts day and year, concatenates.  
  - *Input/Output:* Receives AI output JSON; outputs JSON with `currentDate` field.  
  - *Potential Failures:* JS code errors (unlikely), timezone discrepancies.

---

#### 1.4 Presentation Preparation

- **Overview:**  
  Copies a Google Slides template presentation, then replaces placeholder text fields with the AI-generated proposal content for client-ready presentation.

- **Nodes Involved:**  
  - Copy Template  
  - Replace text

- **Node Details:**  

  **Copy Template**  
  - *Type & Role:* Google Drive node; duplicates a pre-existing Google Slides template file.  
  - *Configuration:*  
    - Uses a fixed Google Slides file ID as the template.  
    - Names copied file dynamically as "[ClientBusiness] Proposal".  
  - *Key Expressions:* `={{ $('Edit Fields').item.json.clientBusiness }} Proposal` for naming.  
  - *Input/Output:* Takes date formatting output; outputs new presentation ID.  
  - *Potential Failures:* Google Drive permission errors, invalid template ID.

  **Replace text**  
  - *Type & Role:* Google Slides node; replaces multiple placeholder texts in the copied presentation with AI-generated content.  
  - *Configuration:*  
    - Replaces placeholders like `{{proposalTitle}}`, `{{clientName}}`, `{{oneParagraphProblemSummary}}`, etc., with corresponding fields from the "Presentation Generator" output JSON.  
    - Uses dynamic expressions referencing "Presentation Generator" node results to insert AI content.  
    - Also inserts formatted date from webhook data and current date from "Set Date Format".  
  - *Input/Output:* Input is copied presentation file ID; outputs updated presentation.  
  - *Potential Failures:* Placeholder mismatch, API limit errors, invalid presentation ID.

---

#### 1.5 Email Drafting

- **Overview:**  
  Drafts an email in Gmail with a message linking to the generated Google Slides proposal, ready for manual review and sending.

- **Nodes Involved:**  
  - Draft Email (Text)

- **Node Details:**  

  **Draft Email (Text)**  
  - *Type & Role:* Gmail node; creates an email draft with a personalized message.  
  - *Configuration:*  
    - Uses dynamic expressions to personalize email with client first name and business name.  
    - Inserts link to the copied Google Slides presentation using its file ID.  
    - Sends draft to client email collected from form submission.  
    - Subject fixed as "Great Meeting".  
  - *Input/Output:* Input is the updated presentation node output and client email; outputs Gmail draft.  
  - *Potential Failures:* Gmail OAuth2 authentication issues, invalid email addresses, API quota limits.

---

### 3. Summary Table

| Node Name           | Node Type                  | Functional Role                  | Input Node(s)       | Output Node(s)      | Sticky Note                                                                                                                  |
|---------------------|----------------------------|--------------------------------|---------------------|---------------------|------------------------------------------------------------------------------------------------------------------------------|
| Webhook1            | Webhook                    | Receive client form submissions | None                | Edit Fields         | Use webhook instead of form node to avoid timeouts and data loss.                                                           |
| Edit Fields         | Set                        | Assign and rename input fields  | Webhook1            | Presentation Generator | Clear naming of variables from form data for easy referencing.                                                               |
| Presentation Generator | OpenAI (LangChain)         | Generate detailed proposal JSON | Edit Fields         | Set Date Format     | AI expands notes into professional proposal JSON with strict prompt rules.                                                   |
| Set Date Format     | Code                       | Format date string for proposal | Presentation Generator | Copy Template      | Formats current date to a polished, client-ready string.                                                                      |
| Copy Template       | Google Drive               | Copy Google Slides template file | Set Date Format      | Replace text        | Copy template presentation and rename with client business name.                                                             |
| Replace text        | Google Slides              | Replace placeholders in slides  | Copy Template       | Draft Email (Text)  | Replace placeholders in presentation with AI-generated proposal content and dates.                                            |
| Draft Email (Text)  | Gmail                      | Draft client follow-up email    | Replace text        | None                | Draft email with proposal link personalized for client, ready for review before sending.                                       |
| Sticky Note         | Sticky Note                | Documentation and instructions  | None                | None                | Overview of workflow purpose and quick proposal generation process.                                                           |
| Sticky Note1        | Sticky Note                | Documentation and instructions  | None                | None                | How the workflow integrates Tally.so, OpenAI, Google Slides, Gmail; setup notes.                                              |
| Sticky Note2        | Sticky Note                | Documentation on webhook usage  | None                | None                | Recommends webhook over form node due to timeout issues.                                                                      |
| Sticky Note3        | Sticky Note                | Documentation on variable setup | None                | None                | Explains naming variables for clarity.                                                                                        |
| Sticky Note4        | Sticky Note                | Documentation on AI node        | None                | None                | Explains AI proposal generation node and prompt importance.                                                                   |
| Sticky Note5        | Sticky Note                | Documentation on date formatting| None                | None                | Explains date formatting node purpose.                                                                                        |
| Sticky Note6        | Sticky Note                | Documentation on slide text replacement | None          | None                | Explains placeholder replacement in Google Slides.                                                                            |
| Sticky Note7        | Sticky Note                | Documentation on copying template | None               | None                | Explains Google Slides template setup and copying.                                                                            |
| Sticky Note9        | Sticky Note                | Documentation on email drafting | None                | None                | Explains Gmail email draft creation for review.                                                                               |
| Sticky Note10       | Sticky Note                | Closing note                   | None                | None                | Friendly reminder and encouragement message from workflow author.                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: unique path (or any unique endpoint)  
   - Purpose: receive JSON data from Tally.so form submissions.  
   - No credentials required.

2. **Create Set Node (Edit Fields):**  
   - Type: Set  
   - Assign variables by extracting data from the webhook JSON. For example:  
     - firName = `{{$json.body.data.fields[0].value}}`  
     - lastName = `{{$json.body.data.fields[1].value}}`  
     - date = `{{$json.body.data.fields[2].value}}`  
     - clientBusiness = `{{$json.body.data.fields[3].value}}`  
     - email = `{{$json.body.data.fields[4].value}}`  
     - problem = `{{$json.body.data.fields[5].value}}`  
     - solution = `{{$json.body.data.fields[6].value}}`  
     - cost = `{{$json.body.data.fields[7].value}}`  
     - scope = `{{$json.body.data.fields[8].value}}`  
     - timeline = `{{$json.body.data.fields[9].value}}`  
     - extraDetails = `{{$json.body.data.fields[10].value}}`  
   - Connect Webhook output to this node.

3. **Create OpenAI Node (Presentation Generator):**  
   - Type: OpenAI (LangChain)  
   - Credentials: OpenAI API key with GPT-4o access  
   - Model: GPT-4o  
   - Messages:  
     - System message: "You are a helpful, intelligent writing agent."  
     - User message: Provide detailed prompt instructing to generate a professional proposal JSON object with all fields fully populated, spartan tone, professional style. Include example input and output formats for guidance.  
   - Input: Output of "Edit Fields" node.  
   - Enable JSON output parsing.

4. **Create Code Node (Set Date Format):**  
   - Type: Code  
   - Language: JavaScript  
   - Code:  
     ```js
     const now = new Date();
     const month = now.toLocaleString('en-US', { month: 'long' });
     const year = now.getFullYear();
     const date = now.getDate();
     const formattedDate = `${month} ${date} ${year}`;
     return [{ json: { currentDate: formattedDate } }];
     ```  
   - Connect OpenAI node output to this node.

5. **Create Google Drive Node (Copy Template):**  
   - Type: Google Drive  
   - Credentials: Authorized Google Drive account  
   - Operation: Copy  
   - File ID: Google Slides template ID (must be prepared in advance)  
   - Name: `={{ $json.clientBusiness }} Proposal` (dynamic naming)  
   - Connect "Set Date Format" output to this node.

6. **Create Google Slides Node (Replace text):**  
   - Type: Google Slides  
   - Credentials: Authorized Google Slides account  
   - Operation: Replace Text  
   - Presentation ID: `={{ $json.id }}` (copied file ID from previous node)  
   - Text replacements: Map placeholders like `{{proposalTitle}}`, `{{clientName}}`, etc. to corresponding fields from "Presentation Generator" output JSON using expressions such as:  
     - `={{ $('Presentation Generator').item.json.message.content.clientBusiness }}`  
     - Include date from webhook and formatted current date.  
   - Connect "Copy Template" node to this node.

7. **Create Gmail Node (Draft Email):**  
   - Type: Gmail  
   - Credentials: Gmail OAuth2 account  
   - Operation: Create draft email  
   - Subject: "Great Meeting"  
   - Message:  
     ```
     Hey {{ $('Edit Fields').item.json.firName }},

     Thanks for the great call earlier. I had a chance to mock up a detailed proposal for {{ $('Edit Fields').item.json.clientBusiness }} for your convenience.

     https://docs.google.com/presentation/d/{{ $('Copy Template').item.json.id }}/edit?slide=id.g3366476a56f_0_651#slide=id.g3366476a56f_0_651

     Best, JS
     ```  
   - Send To: `={{ $('On form submission').item.json.Email }}` or from "Edit Fields" email variable  
   - Connect "Replace text" node to this node.

8. **Final Workflow Connections:**  
   - Webhook1 → Edit Fields → Presentation Generator → Set Date Format → Copy Template → Replace text → Draft Email (Text)

9. **Additional Setup:**  
   - Prepare Google Slides template with placeholders matching those in the Replace text node.  
   - Ensure Google APIs (Drive, Slides) are enabled in Google Cloud Console.  
   - Configure OpenAI API key with GPT-4o access in n8n credentials.  
   - Configure Gmail OAuth2 credentials for email drafting.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                    | Context or Link                                                                                                 |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| This workflow is designed to generate a client proposal within minutes after a meeting, creating a strong first impression of professionalism and efficiency.                                                                                   | Overview sticky note in workflow.                                                                              |
| Use Tally.so for client intake form creation; integrate it via webhook to send client data to n8n. This avoids timeout issues from n8n's Form Trigger node.                                                                                      | Sticky Note1 and Sticky Note2.                                                                                  |
| Prepare your Google Slides template with placeholders exactly matching the variable names used in the workflow to ensure accurate replacement.                                                                                                | Sticky Note6 and Sticky Note7.                                                                                  |
| The AI prompt is highly customizable — adjust to align with your agency’s tone and service style. The prompt includes strict instructions to fill all fields and maintain professionalism.                                                     | Sticky Note4.                                                                                                   |
| The email draft node allows you to review and edit the email before sending, ensuring no unpolished or incomplete proposals are sent.                                                                                                         | Sticky Note9.                                                                                                   |
| The workflow replaces the default timestamp with a clean, polished date string (e.g., “November 8, 2025”) for better client-facing presentation appearance.                                                                                   | Sticky Note5.                                                                                                   |
| For setup help, review Google Cloud Console for enabling Drive and Slides APIs, OpenAI dashboard for API keys, and Gmail OAuth2 credential configuration in n8n.                                                                              | Sticky Note1.                                                                                                   |
| Thank you and encouragement to stay automated from the workflow author.                                                                                                                                                                       | Sticky Note10.                                                                                                  |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created using n8n, adhering strictly to content policies and containing no illegal or offensive material. All data processed is legal and public.