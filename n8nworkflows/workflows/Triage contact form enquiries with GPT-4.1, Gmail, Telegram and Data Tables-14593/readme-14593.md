Triage contact form enquiries with GPT-4.1, Gmail, Telegram and Data Tables

https://n8nworkflows.xyz/workflows/triage-contact-form-enquiries-with-gpt-4-1--gmail--telegram-and-data-tables-14593


# Triage contact form enquiries with GPT-4.1, Gmail, Telegram and Data Tables

# 1. Workflow Overview

This workflow triages incoming website contact form submissions into three categories using GPT-4.1 models:

- **Valid enquiries**
- **Seller / outreach messages**
- **Spam / joke / low-quality submissions**

Depending on the classification, the workflow stores the submission in a Data Table, optionally notifies the internal team, and for valid enquiries sends a brand-aligned confirmation email back to the sender.

## 1.1 Input Reception and Normalization

The workflow starts from a **Webhook** that accepts `POST` requests from a website form. The payload is then normalized in a **Set** node so downstream logic can use consistent field names and include company-context text for AI prompting.

## 1.2 AI Classification and Routing

A first LLM chain uses **GPT-4.1-nano** to classify the form submission as `real`, `seller`, or `joke`. A **Switch** node then routes the item into one of three branches.

## 1.3 Valid Enquiry Handling

If the message is classified as a genuine enquiry, the workflow:

- stores it in a Data Table with a `Valid` type,
- sends a Telegram notification to the team,
- generates a tailored confirmation email using **GPT-4.1-mini**,
- parses the AI response into structured fields,
- sends the email through Gmail.

## 1.4 Seller / Outreach Handling

If the message is classified as a sales pitch or unsolicited outreach, the workflow:

- stores it in the same Data Table with a `Seller` type,
- sends a Telegram notification to the team,
- does **not** send any reply to the sender.

## 1.5 Spam / Low-Quality Handling

If the message is classified as `joke` (used here as the spam / low-value bucket), the workflow marks it in a terminal **Set** node and stops further processing. No notification or customer reply is sent.

---

# 2. Block-by-Block Analysis

## Block 1 — Data Intake and Classification

**Overview:**  
This block receives the website form submission, standardizes its fields, and asks GPT-4.1-nano to classify the message. The Switch node uses the returned label to direct the item to the correct processing branch.

**Nodes Involved:**  
- Webhook
- Set data
- OpenAI Chat Model1
- Analyze intend
- Switch

### Webhook

- **Type and technical role:** `n8n-nodes-base.webhook` — entry point for incoming HTTP requests.
- **Configuration choices:**  
  - HTTP method: `POST`
  - Path: `dfc7f035-b32a-436e-804e-361f9cd6a389`
- **Key expressions or variables used:** None in configuration.
- **Input and output connections:**  
  - Input: none, this is a trigger node  
  - Output: `Set data`
- **Version-specific requirements:** Type version `2.1`.
- **Edge cases or potential failure types:**  
  - Wrong HTTP method from website form
  - Payload missing expected fields
  - Production/test webhook confusion during deployment
  - Public endpoint abuse if no upstream validation is added
- **Sub-workflow reference:** None.

### Set data

- **Type and technical role:** `n8n-nodes-base.set` — normalizes and prepares input fields for all later nodes.
- **Configuration choices:**  
  Creates these fields:
  - `Full Name`
  - `Email`
  - `Phone`
  - `Message`
  - `Our Company Information`
  
  In the provided workflow, the contact fields are currently set to empty strings, while company information is hardcoded as:
  - `We are XYZ the best company.....`
- **Key expressions or variables used:** None in the current node configuration.
- **Input and output connections:**  
  - Input: `Webhook`
  - Output: `Analyze intend`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**  
  - As configured, this node overwrites all incoming form fields with empty values. Unless manually changed, classification and notifications will operate on blank inputs.
  - If this is intentional as a template placeholder, it must be replaced with expressions pulling values from the webhook payload.
  - If company information is left generic, auto-response quality will be generic too.
- **Sub-workflow reference:** None.

### OpenAI Chat Model1

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — language model provider for the classification chain.
- **Configuration choices:**  
  - Model: `gpt-4.1-nano`
  - No additional options configured
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - AI language model output goes to `Analyze intend`
- **Version-specific requirements:** Type version `1.2`; requires OpenAI credentials.
- **Edge cases or potential failure types:**  
  - Invalid or missing OpenAI API credentials
  - Model availability changes
  - Rate limits or API quota exhaustion
  - Non-deterministic outputs if the model ignores prompt constraints
- **Sub-workflow reference:** None.

### Analyze intend

- **Type and technical role:** `@n8n/n8n-nodes-langchain.chainLlm` — prompt-based classification chain.
- **Configuration choices:**  
  - Prompt includes:
    - `Name: {{ $json['Full Name'] }}`
    - `Email: {{ $json.Email }}`
    - `Message: {{ $json.Message }}`
  - System-style instruction is embedded in the messages configuration.
  - The classifier is instructed to return **exactly one word**: `real`, `seller`, or `joke`.
- **Key expressions or variables used:**  
  - `{{ $json['Full Name'] }}`
  - `{{ $json.Email }}`
  - `{{ $json.Message }}`
- **Input and output connections:**  
  - Main input: `Set data`
  - AI model input: `OpenAI Chat Model1`
  - Main output: `Switch`
- **Version-specific requirements:** Type version `1.9`; requires a compatible LangChain/OpenAI setup in n8n.
- **Edge cases or potential failure types:**  
  - If upstream fields are blank, classification may default unexpectedly
  - If the model returns extra whitespace or wording, the Switch conditions may fail
  - Prompt typo in node name (“intend” instead of “intent”) does not affect execution but may confuse maintainers
- **Sub-workflow reference:** None.

### Switch

- **Type and technical role:** `n8n-nodes-base.switch` — routes items into branches based on classifier output.
- **Configuration choices:**  
  Three named outputs:
  - `Valid` when `{{ $json.text }}` equals `real`
  - `Outreach` when `{{ $json.text }}` equals `seller`
  - `SPAM` when `{{ $json.text }}` equals `joke`
- **Key expressions or variables used:**  
  - `{{ $json.text }}`
- **Input and output connections:**  
  - Input: `Analyze intend`
  - Output 0 / `Valid`: `Insert row`
  - Output 1 / `Outreach`: `Insert row2`
  - Output 2 / `SPAM`: `SPAM Email Detected`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**  
  - If classifier output is not exactly one of the three expected words, no branch may match
  - Case-sensitive matching is enabled
  - Strict type validation is enabled
- **Sub-workflow reference:** None.

---

## Block 2 — Valid Enquiry Processing

**Overview:**  
This block handles genuine customer enquiries. It records the enquiry, alerts the team, generates a personalized confirmation email aligned to company tone, and sends that email back to the sender.

**Nodes Involved:**  
- Insert row
- Send a text message
- OpenAI Chat Model
- Structured Output Parser
- Personalized auto respond
- Send message to Client
- Send a notification (disabled)

### Insert row

- **Type and technical role:** `n8n-nodes-base.dataTable` — inserts the valid enquiry into an n8n Data Table.
- **Configuration choices:**  
  Writes to Data Table `Form` and maps:
  - `Type` = `Valid`
  - `Email` = `{{ $json.Email }}`
  - `Phone` = `{{ $json.Phone }}`
  - `Message` = `{{ $json.Message }}`
  - `Full_Name` = `{{ $json['Full Name'] }}`
- **Key expressions or variables used:**  
  - `{{ $json.Email }}`
  - `{{ $json.Phone }}`
  - `{{ $json.Message }}`
  - `{{ $json['Full Name'] }}`
- **Input and output connections:**  
  - Input: `Switch` valid branch
  - Output: `Send a text message`
- **Version-specific requirements:** Type version `1.1`; requires Data Tables support in the n8n instance.
- **Edge cases or potential failure types:**  
  - Missing Data Table or incorrect table schema
  - Permission issues in project context
  - Null or unexpected field formats
- **Sub-workflow reference:** None.

### Send a text message

- **Type and technical role:** `n8n-nodes-base.telegram` — sends an internal notification via Telegram.
- **Configuration choices:**  
  Sends a text summary containing:
  - name
  - email
  - phone
  - message
- **Key expressions or variables used:**  
  - `{{ $('Set data').item.json['Full Name'] }}`
  - `{{ $('Set data').item.json.Email }}`
  - `{{ $('Set data').item.json.Phone }}`
  - `{{ $('Set data').item.json.Message }}`
- **Input and output connections:**  
  - Input: `Insert row`
  - Output: `Personalized auto respond`
- **Version-specific requirements:** Type version `1.2`; requires Telegram Bot credentials and a valid target chat configuration.
- **Edge cases or potential failure types:**  
  - Missing Telegram chat ID or bot permissions
  - Bot not allowed in destination chat/channel
  - Notification text showing empty values because `Set data` currently sets blanks
- **Sub-workflow reference:** None.

### OpenAI Chat Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — language model provider for response generation.
- **Configuration choices:**  
  - Model: `gpt-4.1-mini`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - AI language model output goes to `Personalized auto respond`
- **Version-specific requirements:** Type version `1.2`; requires OpenAI credentials.
- **Edge cases or potential failure types:**  
  - Authentication failures
  - Quota/rate limit issues
  - Model response not matching expected structure without parser enforcement
- **Sub-workflow reference:** None.

### Structured Output Parser

- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — enforces structured AI output.
- **Configuration choices:**  
  Expects JSON-like structured output matching:
  - `subject`
  - `body`
  
  Example schema says:
  ```json
  {
    "subject": "Subject in message language",
    "body": "HTML formatted message body"
  }
  ```
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - AI output parser connected to `Personalized auto respond`
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**  
  - Prompt and parser expectation mismatch
  - The prompt asks for:
    - `subject: ...`
    - `body: ...`
    not JSON
  - Parser may fail if the model returns plain text instead of parser-compatible structured content
- **Sub-workflow reference:** None.

### Personalized auto respond

- **Type and technical role:** `@n8n/n8n-nodes-langchain.chainLlm` — generates the customer-facing confirmation email.
- **Configuration choices:**  
  - Main prompt includes sender name and message
  - System instruction defines:
    - professional tone
    - no sales language
    - use company information
    - output with `subject:` and `body:`
  - `hasOutputParser` is enabled
- **Key expressions or variables used:**  
  - `{{ $('Set data').item.json['Full Name'] }}`
  - `{{ $('Set data').item.json.Message }}`
  - `{{ $('Set data').item.json['Our Company Information'] }}`
- **Input and output connections:**  
  - Main input: `Send a text message`
  - AI model input: `OpenAI Chat Model`
  - AI output parser input: `Structured Output Parser`
  - Main output: `Send message to Client`
- **Version-specific requirements:** Type version `1.6`.
- **Edge cases or potential failure types:**  
  - If `Set data` is not populated correctly, email content becomes generic or empty
  - There is a format mismatch risk between parser schema and prompt instructions
  - Typo in prompt text (`respoind eamil`) does not break execution but reduces polish
  - If the model returns non-parseable content, downstream Gmail expressions may fail
- **Sub-workflow reference:** None.

### Send message to Client

- **Type and technical role:** `n8n-nodes-base.gmail` — sends the generated confirmation email to the original sender.
- **Configuration choices:**  
  - To: `{{ $('Set data').item.json.Email }}`
  - Subject: `{{ $('Personalized auto respond').item.json.output.subject }}`
  - Message: `{{ $json.output.body }}`
  - Email type: `text`
  - Attribution disabled
- **Key expressions or variables used:**  
  - `{{ $('Set data').item.json.Email }}`
  - `{{ $('Personalized auto respond').item.json.output.subject }}`
  - `{{ $json.output.body }}`
- **Input and output connections:**  
  - Input: `Personalized auto respond`
  - No downstream nodes
- **Version-specific requirements:** Type version `2.1`; requires Gmail OAuth2 credentials.
- **Edge cases or potential failure types:**  
  - Empty or invalid email address
  - Gmail sender authorization issues
  - If parser output is missing `subject` or `body`, send will fail
  - Body schema says HTML formatted content, but node is configured as `text`; formatting may be degraded
- **Sub-workflow reference:** None.

### Send a notification

- **Type and technical role:** `n8n-nodes-base.gmail` — optional internal email alert for valid enquiries.
- **Configuration choices:**  
  - Sends internal summary email
  - Currently **disabled**
- **Key expressions or variables used:**  
  - References `Set data` fields
- **Input and output connections:**  
  - No active downstream path in current workflow
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**  
  - Disabled node does not execute
  - If enabled, it would require recipient configuration review
- **Sub-workflow reference:** None.

---

## Block 3 — Seller and Outreach Handling

**Overview:**  
This branch handles unsolicited pitches, outreach, or vendor messages. It stores them separately by type and alerts the team, without engaging the sender.

**Nodes Involved:**  
- Insert row2
- Send a text message1
- Send a notification1 (disabled)

### Insert row2

- **Type and technical role:** `n8n-nodes-base.dataTable` — records outreach messages in the same Data Table.
- **Configuration choices:**  
  Writes to Data Table `Form` and maps:
  - `Type` = `Seller`
  - `Email` = `{{ $json.Email }}`
  - `Phone` = `{{ $json.Phone }}`
  - `Message` = `{{ $json.Message }}`
  - `Full_Name` = `{{ $json['Full Name'] }}`
- **Key expressions or variables used:**  
  Same field mappings as the valid branch, but different type label.
- **Input and output connections:**  
  - Input: `Switch` outreach branch
  - Output: `Send a text message1`
- **Version-specific requirements:** Type version `1.1`.
- **Edge cases or potential failure types:**  
  - Same Data Table schema risks as `Insert row`
- **Sub-workflow reference:** None.

### Send a text message1

- **Type and technical role:** `n8n-nodes-base.telegram` — internal Telegram notification for seller/outreach submissions.
- **Configuration choices:**  
  Sends a message prefixed with:
  - `New message from Form, this was selling pitch:`
- **Key expressions or variables used:**  
  - `{{ $('Set data').item.json['Full Name'] }}`
  - `{{ $('Set data').item.json.Email }}`
  - `{{ $('Set data').item.json.Phone }}`
  - `{{ $('Set data').item.json.Message }}`
- **Input and output connections:**  
  - Input: `Insert row2`
  - No downstream nodes
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**  
  - Telegram credential/chat issues
  - Empty fields if upstream normalization is not fixed
- **Sub-workflow reference:** None.

### Send a notification1

- **Type and technical role:** `n8n-nodes-base.gmail` — optional internal email alert for seller/outreach submissions.
- **Configuration choices:**  
  - Subject: `New message from Seller`
  - Currently **disabled**
- **Key expressions or variables used:**  
  - References `Set data` fields
- **Input and output connections:**  
  - No active downstream path in current workflow
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**  
  - Disabled node does not execute
- **Sub-workflow reference:** None.

---

## Block 4 — Spam and Low-Quality Submissions

**Overview:**  
This branch catches spam, joke, and other low-value submissions. The workflow marks them and ends processing without notifying the team or replying to the sender.

**Nodes Involved:**  
- SPAM Email Detected

### SPAM Email Detected

- **Type and technical role:** `n8n-nodes-base.set` — terminal marker node for spam/joke submissions.
- **Configuration choices:**  
  No assignments are defined; it acts as a stop/end marker.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: `Switch` spam branch
  - No downstream nodes
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**  
  - Since it does not explicitly write a spam label, no persistent record is created in the current design
  - If later reporting is needed, this branch would need Data Table insertion or logging
- **Sub-workflow reference:** None.

---

## Block 5 — Documentation and In-Canvas Notes

**Overview:**  
These nodes are non-executable documentation elements embedded in the workflow canvas. They describe behavior, setup, and warnings for maintainers.

**Nodes Involved:**  
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6
- Sticky Note7

### Sticky Note / Sticky Note1 / Sticky Note2 / Sticky Note3 / Sticky Note4 / Sticky Note5 / Sticky Note6 / Sticky Note7

- **Type and technical role:** `n8n-nodes-base.stickyNote` — visual documentation only.
- **Configuration choices:**  
  Notes document the workflow purpose, branch behavior, setup checklist, customization points, and some generic warnings.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None at runtime.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | Webhook | Receives POST submissions from the website form |  | Set data | ## Data intake and classification<br>Receives the form submission, normalizes the fields, and uses GPT-4.1-nano to classify the message as real, seller, or spam. Switch routes it into the correct branch.<br><br>## Form submission triage and auto-response<br>### How it works<br>1. Webhook receives incoming POST requests from the website contact form.<br>2. Set data normalizes the submission fields and injects company information for use by the AI.<br>3. Analyze intend uses GPT-4.1-nano to classify the message as real, seller, or joke.<br>4. Switch routes the submission into one of three branches based on the classification result.<br>5. Valid enquiries are saved to the DataTable, the team is notified via Telegram, and a personalized confirmation email is generated by GPT-4.1-mini and sent back to the sender.<br>6. Seller submissions are saved separately and the team is notified via Telegram. No reply is sent to the sender.<br>7. Spam and joke submissions are marked and discarded. No notification or reply is triggered.<br><br>### Setup<br>- [ ] Connect the Webhook node to your website contact form and set the correct POST endpoint.<br>- [ ] In Set data, fill in your company name, tone, and description in the Our Company Information field.<br>- [ ] Connect your OpenAI API credential to both OpenAI Chat Model nodes.<br>- [ ] Connect your Telegram Bot credential and set the correct Chat ID in both Telegram nodes.<br>- [ ] Connect your Gmail OAuth2 credential to Send message to Client and verify the sender address.<br>- [ ] Optionally enable Send a notification nodes for email-based internal alerts.<br><br>### Customization<br>Edit the classification prompt in Analyze intend to adjust what counts as real, seller, or spam. Edit the system prompt in Personalized auto respond to change the tone or format of the confirmation email. |
| Set data | Set | Normalizes form fields and adds company context | Webhook | Analyze intend | ## Data intake and classification<br>Receives the form submission, normalizes the fields, and uses GPT-4.1-nano to classify the message as real, seller, or spam. Switch routes it into the correct branch.<br><br>warning<br><br>## Form submission triage and auto-response<br>### How it works<br>1. Webhook receives incoming POST requests from the website contact form.<br>2. Set data normalizes the submission fields and injects company information for use by the AI.<br>3. Analyze intend uses GPT-4.1-nano to classify the message as real, seller, or joke.<br>4. Switch routes the submission into one of three branches based on the classification result.<br>5. Valid enquiries are saved to the DataTable, the team is notified via Telegram, and a personalized confirmation email is generated by GPT-4.1-mini and sent back to the sender.<br>6. Seller submissions are saved separately and the team is notified via Telegram. No reply is sent to the sender.<br>7. Spam and joke submissions are marked and discarded. No notification or reply is triggered.<br><br>### Setup<br>- [ ] Connect the Webhook node to your website contact form and set the correct POST endpoint.<br>- [ ] In Set data, fill in your company name, tone, and description in the Our Company Information field.<br>- [ ] Connect your OpenAI API credential to both OpenAI Chat Model nodes.<br>- [ ] Connect your Telegram Bot credential and set the correct Chat ID in both Telegram nodes.<br>- [ ] Connect your Gmail OAuth2 credential to Send message to Client and verify the sender address.<br>- [ ] Optionally enable Send a notification nodes for email-based internal alerts.<br><br>### Customization<br>Edit the classification prompt in Analyze intend to adjust what counts as real, seller, or spam. Edit the system prompt in Personalized auto respond to change the tone or format of the confirmation email. |
| OpenAI Chat Model1 | OpenAI Chat Model | Supplies GPT-4.1-nano for classification |  | Analyze intend | ## Data intake and classification<br>Receives the form submission, normalizes the fields, and uses GPT-4.1-nano to classify the message as real, seller, or spam. Switch routes it into the correct branch.<br><br>## Form submission triage and auto-response<br>### How it works<br>1. Webhook receives incoming POST requests from the website contact form.<br>2. Set data normalizes the submission fields and injects company information for use by the AI.<br>3. Analyze intend uses GPT-4.1-nano to classify the message as real, seller, or joke.<br>4. Switch routes the submission into one of three branches based on the classification result.<br>5. Valid enquiries are saved to the DataTable, the team is notified via Telegram, and a personalized confirmation email is generated by GPT-4.1-mini and sent back to the sender.<br>6. Seller submissions are saved separately and the team is notified via Telegram. No reply is sent to the sender.<br>7. Spam and joke submissions are marked and discarded. No notification or reply is triggered.<br><br>### Setup<br>- [ ] Connect the Webhook node to your website contact form and set the correct POST endpoint.<br>- [ ] In Set data, fill in your company name, tone, and description in the Our Company Information field.<br>- [ ] Connect your OpenAI API credential to both OpenAI Chat Model nodes.<br>- [ ] Connect your Telegram Bot credential and set the correct Chat ID in both Telegram nodes.<br>- [ ] Connect your Gmail OAuth2 credential to Send message to Client and verify the sender address.<br>- [ ] Optionally enable Send a notification nodes for email-based internal alerts.<br><br>### Customization<br>Edit the classification prompt in Analyze intend to adjust what counts as real, seller, or spam. Edit the system prompt in Personalized auto respond to change the tone or format of the confirmation email. |
| Analyze intend | LLM Chain | Classifies the submission as real, seller, or joke | Set data, OpenAI Chat Model1 | Switch | ## Data intake and classification<br>Receives the form submission, normalizes the fields, and uses GPT-4.1-nano to classify the message as real, seller, or spam. Switch routes it into the correct branch.<br><br>## Form submission triage and auto-response<br>### How it works<br>1. Webhook receives incoming POST requests from the website contact form.<br>2. Set data normalizes the submission fields and injects company information for use by the AI.<br>3. Analyze intend uses GPT-4.1-nano to classify the message as real, seller, or joke.<br>4. Switch routes the submission into one of three branches based on the classification result.<br>5. Valid enquiries are saved to the DataTable, the team is notified via Telegram, and a personalized confirmation email is generated by GPT-4.1-mini and sent back to the sender.<br>6. Seller submissions are saved separately and the team is notified via Telegram. No reply is sent to the sender.<br>7. Spam and joke submissions are marked and discarded. No notification or reply is triggered.<br><br>### Setup<br>- [ ] Connect the Webhook node to your website contact form and set the correct POST endpoint.<br>- [ ] In Set data, fill in your company name, tone, and description in the Our Company Information field.<br>- [ ] Connect your OpenAI API credential to both OpenAI Chat Model nodes.<br>- [ ] Connect your Telegram Bot credential and set the correct Chat ID in both Telegram nodes.<br>- [ ] Connect your Gmail OAuth2 credential to Send message to Client and verify the sender address.<br>- [ ] Optionally enable Send a notification nodes for email-based internal alerts.<br><br>### Customization<br>Edit the classification prompt in Analyze intend to adjust what counts as real, seller, or spam. Edit the system prompt in Personalized auto respond to change the tone or format of the confirmation email. |
| Switch | Switch | Routes classified submissions to the correct branch | Analyze intend | Insert row, Insert row2, SPAM Email Detected | ## Data intake and classification<br>Receives the form submission, normalizes the fields, and uses GPT-4.1-nano to classify the message as real, seller, or spam. Switch routes it into the correct branch.<br><br>## Form submission triage and auto-response<br>### How it works<br>1. Webhook receives incoming POST requests from the website contact form.<br>2. Set data normalizes the submission fields and injects company information for use by the AI.<br>3. Analyze intend uses GPT-4.1-nano to classify the message as real, seller, or joke.<br>4. Switch routes the submission into one of three branches based on the classification result.<br>5. Valid enquiries are saved to the DataTable, the team is notified via Telegram, and a personalized confirmation email is generated by GPT-4.1-mini and sent back to the sender.<br>6. Seller submissions are saved separately and the team is notified via Telegram. No reply is sent to the sender.<br>7. Spam and joke submissions are marked and discarded. No notification or reply is triggered.<br><br>### Setup<br>- [ ] Connect the Webhook node to your website contact form and set the correct POST endpoint.<br>- [ ] In Set data, fill in your company name, tone, and description in the Our Company Information field.<br>- [ ] Connect your OpenAI API credential to both OpenAI Chat Model nodes.<br>- [ ] Connect your Telegram Bot credential and set the correct Chat ID in both Telegram nodes.<br>- [ ] Connect your Gmail OAuth2 credential to Send message to Client and verify the sender address.<br>- [ ] Optionally enable Send a notification nodes for email-based internal alerts.<br><br>### Customization<br>Edit the classification prompt in Analyze intend to adjust what counts as real, seller, or spam. Edit the system prompt in Personalized auto respond to change the tone or format of the confirmation email. |
| Insert row | Data Table | Stores valid enquiries in the Form table | Switch | Send a text message | ## Valid enquiry processing<br>Saves the lead to the DataTable, notifies the team via Telegram, generates a brand-aligned confirmation email with GPT-4.1-mini, and sends it to the sender. |
| Send a text message | Telegram | Sends internal Telegram alert for valid enquiries | Insert row | Personalized auto respond | ## Valid enquiry processing<br>Saves the lead to the DataTable, notifies the team via Telegram, generates a brand-aligned confirmation email with GPT-4.1-mini, and sends it to the sender.<br><br>warning |
| OpenAI Chat Model | OpenAI Chat Model | Supplies GPT-4.1-mini for customer reply generation |  | Personalized auto respond | ## Valid enquiry processing<br>Saves the lead to the DataTable, notifies the team via Telegram, generates a brand-aligned confirmation email with GPT-4.1-mini, and sends it to the sender. |
| Structured Output Parser | Structured Output Parser | Parses generated email into subject/body fields |  | Personalized auto respond | ## Valid enquiry processing<br>Saves the lead to the DataTable, notifies the team via Telegram, generates a brand-aligned confirmation email with GPT-4.1-mini, and sends it to the sender. |
| Personalized auto respond | LLM Chain | Generates confirmation email content | Send a text message, OpenAI Chat Model, Structured Output Parser | Send message to Client | ## Valid enquiry processing<br>Saves the lead to the DataTable, notifies the team via Telegram, generates a brand-aligned confirmation email with GPT-4.1-mini, and sends it to the sender. |
| Send message to Client | Gmail | Sends the AI-generated confirmation email to the sender | Personalized auto respond |  | ## Valid enquiry processing<br>Saves the lead to the DataTable, notifies the team via Telegram, generates a brand-aligned confirmation email with GPT-4.1-mini, and sends it to the sender. |
| Send a notification | Gmail | Optional internal email alert for valid enquiries |  |  | ## Valid enquiry processing<br>Saves the lead to the DataTable, notifies the team via Telegram, generates a brand-aligned confirmation email with GPT-4.1-mini, and sends it to the sender. |
| Insert row2 | Data Table | Stores seller/outreach submissions in the Form table | Switch | Send a text message1 | ## Seller and outreach handling<br>Saves the submission to the DataTable with a Seller label and notifies the team via Telegram. No reply is sent to the sender. |
| Send a text message1 | Telegram | Sends internal Telegram alert for seller/outreach submissions | Insert row2 |  | ## Seller and outreach handling<br>Saves the submission to the DataTable with a Seller label and notifies the team via Telegram. No reply is sent to the sender.<br><br>warning |
| Send a notification1 | Gmail | Optional internal email alert for seller/outreach submissions |  |  | ## Seller and outreach handling<br>Saves the submission to the DataTable with a Seller label and notifies the team via Telegram. No reply is sent to the sender. |
| SPAM Email Detected | Set | Terminal marker for spam/joke submissions | Switch |  | ## Spam and low-quality submissions<br>Marks the submission as spam and stops processing. No notification or reply is triggered. |
| Sticky Note | Sticky Note | Canvas documentation for valid branch |  |  |  |
| Sticky Note1 | Sticky Note | Canvas documentation for seller branch |  |  |  |
| Sticky Note2 | Sticky Note | Canvas documentation for spam branch |  |  |  |
| Sticky Note4 | Sticky Note | Canvas documentation for intake/classification block |  |  |  |
| Sticky Note6 | Sticky Note | Global workflow documentation and setup checklist |  |  |  |
| Sticky Note3 | Sticky Note | Warning note |  |  |  |
| Sticky Note5 | Sticky Note | Warning note |  |  |  |
| Sticky Note7 | Sticky Note | Warning note |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a Webhook trigger**
   - Add a **Webhook** node.
   - Set:
     - Method: `POST`
     - Path: choose a custom path, e.g. `contact-form-triage`
   - Expect your website to submit fields such as:
     - `Full Name`
     - `Email`
     - `Phone`
     - `Message`

2. **Add a Set node to normalize incoming fields**
   - Add a **Set** node after the Webhook and name it `Set data`.
   - Create fields:
     - `Full Name`
     - `Email`
     - `Phone`
     - `Message`
     - `Our Company Information`
   - Recommended configuration:
     - `Full Name` = expression from webhook payload, for example `{{ $json["Full Name"] }}`
     - `Email` = `{{ $json.Email }}`
     - `Phone` = `{{ $json.Phone }}`
     - `Message` = `{{ $json.Message }}`
     - `Our Company Information` = plain text describing your company tone, services, positioning, and preferred writing style
   - Important: do **not** leave the first four fields as empty static strings unless you are intentionally mocking input.

3. **Create the classification model node**
   - Add an **OpenAI Chat Model** node and name it `OpenAI Chat Model1`.
   - Select model `gpt-4.1-nano`.
   - Attach your **OpenAI API credential**.

4. **Create the classification chain**
   - Add a **Basic LLM Chain / LangChain Chain** node and name it `Analyze intend`.
   - Set its main prompt text to include:
     - sender name
     - email
     - message
   - Add a strong classification instruction that allows only:
     - `real`
     - `seller`
     - `joke`
   - Connect:
     - `Set data` → `Analyze intend` (main)
     - `OpenAI Chat Model1` → `Analyze intend` (AI language model)

5. **Add a Switch node for branch routing**
   - Add a **Switch** node after `Analyze intend`.
   - Create 3 outputs with renamed outputs:
     - `Valid`
     - `Outreach`
     - `SPAM`
   - Configure conditions against the classifier text output:
     - Output `Valid`: `{{ $json.text }}` equals `real`
     - Output `Outreach`: `{{ $json.text }}` equals `seller`
     - Output `SPAM`: `{{ $json.text }}` equals `joke`
   - Keep in mind this is exact matching.

6. **Create the Data Table for storage**
   - In n8n Data Tables, create a table named something like `Form`.
   - Add columns:
     - `Full_Name` (string)
     - `Phone` (string)
     - `Email` (string)
     - `Message` (string)
     - `Type` (string)

7. **Build the valid enquiry storage node**
   - Add a **Data Table** node named `Insert row`.
   - Operation: insert row.
   - Select the `Form` table.
   - Map:
     - `Full_Name` = `{{ $json["Full Name"] }}`
     - `Phone` = `{{ $json.Phone }}`
     - `Email` = `{{ $json.Email }}`
     - `Message` = `{{ $json.Message }}`
     - `Type` = `Valid`
   - Connect `Switch` valid output to `Insert row`.

8. **Build the valid enquiry Telegram notification**
   - Add a **Telegram** node named `Send a text message`.
   - Connect Telegram Bot credentials.
   - Set the destination chat ID/channel according to your environment.
   - Set message text similar to:
     - Name, email, phone, message
   - You can reference `Set data` values directly:
     - `{{ $('Set data').item.json['Full Name'] }}`
     - etc.
   - Connect `Insert row` → `Send a text message`.

9. **Create the reply-generation model**
   - Add another **OpenAI Chat Model** node named `OpenAI Chat Model`.
   - Select model `gpt-4.1-mini`.
   - Use the same or another OpenAI credential.

10. **Create the structured output parser**
    - Add a **Structured Output Parser** node named `Structured Output Parser`.
    - Define a schema with:
      - `subject`
      - `body`
    - Recommended actual expectation: plain string values.
    - If you want robust parsing, align the generation prompt with the parser format exactly.

11. **Create the customer reply generation chain**
    - Add another **Basic LLM Chain / LangChain Chain** node named `Personalized auto respond`.
    - Main user prompt should include:
      - sender name
      - sender message
    - System instruction should tell the model to:
      - confirm receipt only
      - avoid marketing/sales language
      - use company information
      - write a concise professional email
      - return `subject` and `body`
    - Enable output parsing.
    - Connect:
      - `Send a text message` → `Personalized auto respond` (main)
      - `OpenAI Chat Model` → `Personalized auto respond` (AI language model)
      - `Structured Output Parser` → `Personalized auto respond` (AI output parser)

12. **Create the Gmail node for customer reply**
    - Add a **Gmail** node named `Send message to Client`.
    - Attach **Gmail OAuth2 credentials**.
    - Configure:
      - To = `{{ $('Set data').item.json.Email }}`
      - Subject = `{{ $('Personalized auto respond').item.json.output.subject }}`
      - Body = `{{ $json.output.body }}`
      - Email type = `text` or `html`
   - Recommendation: if your AI body is HTML, set email type to `html`.
   - Connect `Personalized auto respond` → `Send message to Client`.

13. **Build the seller/outreach storage node**
   - Add a second **Data Table** node named `Insert row2`.
   - Use the same table and same mappings as `Insert row`, except:
     - `Type` = `Seller`
   - Connect `Switch` outreach output → `Insert row2`.

14. **Build the seller/outreach Telegram notification**
   - Add another **Telegram** node named `Send a text message1`.
   - Configure bot/chat ID.
   - Message should identify the submission as a selling pitch.
   - Connect `Insert row2` → `Send a text message1`.

15. **Build the spam terminal node**
   - Add a **Set** node named `SPAM Email Detected`.
   - You may leave it empty as a terminal stop node, or optionally assign:
     - `Type = Spam`
     - `Reason = Classified as joke`
   - Connect `Switch` spam output → `SPAM Email Detected`.

16. **Optionally add internal Gmail alert nodes**
   - Add one Gmail node for valid enquiries (`Send a notification`)
   - Add one Gmail node for seller messages (`Send a notification1`)
   - Keep them disabled if Telegram is sufficient
   - If enabling them, configure:
     - recipient inbox
     - subject
     - message body with submission details

17. **Test classification thoroughly**
   - Send sample POST requests:
     - real enquiry: “Need a quote for an automation project”
     - seller: “We provide SEO services for your business”
     - joke/spam: “test”
   - Verify branch selection and stored `Type` values.

18. **Fix the parser/prompt consistency**
   - In the provided design, the reply prompt requests:
     - `subject: ...`
     - `body: ...`
     while the parser expects structured output.
   - For a more stable rebuild, choose one of these:
     - **Option A:** output strict JSON and parse JSON
     - **Option B:** remove parser and parse plain text manually
   - Option A is preferable for automation reliability.

19. **Deploy the webhook**
   - Switch from test URL to production URL.
   - Update your website form action or backend POST target.
   - Ensure your form sends the expected field names.

20. **Add operational safeguards**
   - Recommended improvements when rebuilding:
     - validate required fields before LLM usage
     - add fallback branch if Switch gets unexpected text
     - log spam records if analytics matter
     - set Gmail body type to HTML if returning formatted content
     - add rate-limit or CAPTCHA upstream on the website form

### Credential configuration summary

- **OpenAI**
  - Required for:
    - `OpenAI Chat Model1`
    - `OpenAI Chat Model`
- **Telegram Bot**
  - Required for:
    - `Send a text message`
    - `Send a text message1`
  - Also configure the target chat/channel in node parameters.
- **Gmail OAuth2**
  - Required for:
    - `Send message to Client`
    - optional notification nodes

### Input/output expectations

- **Webhook input**
  - Expected fields:
    - `Full Name`
    - `Email`
    - `Phone`
    - `Message`
- **Classification output**
  - Must be exactly one of:
    - `real`
    - `seller`
    - `joke`
- **Reply generation output**
  - Must provide:
    - `subject`
    - `body`

### Sub-workflow setup

This workflow does **not** invoke any sub-workflow and does not contain an Execute Workflow node.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow title indicates a triage design using GPT-4.1, Gmail, Telegram, and Data Tables. | Workflow metadata/title |
| The canvas includes an embedded setup checklist reminding the user to connect the webhook endpoint, OpenAI credentials, Telegram Bot, Gmail OAuth2, and company information text. | In-workflow documentation |
| The workflow uses the word `joke` as the low-quality/spam class, even though the branch is labeled `SPAM`. | Classification design detail |
| The current `Set data` node appears to be a template placeholder and will overwrite incoming values with empty strings unless corrected. | Important implementation note |
| The current reply-generation prompt and structured parser are not perfectly aligned; this should be corrected for reliable execution. | Reliability note |
| Several nodes are marked with simple `warning` sticky notes near the intake and notification areas, but they do not include detailed explanatory text. | Canvas warning notes |