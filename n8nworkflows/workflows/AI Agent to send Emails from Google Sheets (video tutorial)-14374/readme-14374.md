AI Agent to send Emails from Google Sheets (video tutorial)

https://n8nworkflows.xyz/workflows/ai-agent-to-send-emails-from-google-sheets--video-tutorial--14374


# AI Agent to send Emails from Google Sheets (video tutorial)

# 1. Workflow Overview

This workflow automates two related email operations around a Google Sheets-based outreach list:

1. **Send personalized emails** for rows marked as ready to send.
2. **Track incoming replies** from Gmail and write the response back into the spreadsheet.

It is designed for lightweight outreach, reminders, or follow-up campaigns where a spreadsheet acts as the control panel. The workflow combines **Google Sheets**, **an AI agent backed by OpenAI**, and **Gmail**.

## 1.1 Outbound Email Processing

This branch starts manually. It reads spreadsheet rows whose `Status` is `To send`, uses an AI agent to generate personalized email text from each row’s `Introduction`, sends the email through Gmail, and then marks the row as `Sent`.

## 1.2 Inbound Reply Tracking

This branch starts automatically via a Gmail trigger that polls every minute. When a new email arrives, it looks up the spreadsheet row where the sender matches the `To` field, then writes the incoming message text into the `Response` column.

## 1.3 Documentation and Operational Notes

The workflow also includes embedded sticky notes describing setup requirements, creator contact details, and a video walkthrough link.

---

# 2. Block-by-Block Analysis

## 2.1 Manual Start and Fetch Pending Rows

### Overview
This block is the entry point for outbound email sending. It starts the workflow manually and retrieves all rows from Google Sheets where `Status = "To send"`.

### Nodes Involved
- Manual Trigger - Start Email Workflow
- Get Pending Emails from Sheet

### Node Details

#### Manual Trigger - Start Email Workflow
- **Type and technical role:** `n8n-nodes-base.manualTrigger`  
  Manual entry node used to launch the outbound branch on demand.
- **Configuration choices:**  
  No custom parameters are configured.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none
  - Output: `Get Pending Emails from Sheet`
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  - No functional failure beyond manual execution availability.
  - This branch will not run automatically unless replaced or complemented with another trigger.
- **Sub-workflow reference:**  
  None.

#### Get Pending Emails from Sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads rows from a Google Sheet using filters.
- **Configuration choices:**  
  - Uses Google Sheets OAuth2 credentials.
  - Targets spreadsheet **Email Manager**
  - Targets sheet/tab **Emails with AI**
  - Filters rows where:
    - `Status` = `To send`
- **Key expressions or variables used:**  
  No dynamic expression in the filter value; it is a static lookup for `To send`.
- **Input and output connections:**  
  - Input: `Manual Trigger - Start Email Workflow`
  - Output: `AI Agent - Generate Personalized Email`
- **Version-specific requirements:**  
  Type version `4.7`.
- **Edge cases or potential failure types:**  
  - Google OAuth permission issues
  - Spreadsheet or sheet not found
  - Header mismatch if the `Status` column is renamed
  - Empty result set: downstream nodes simply receive no items
- **Sub-workflow reference:**  
  None.

---

## 2.2 AI Email Generation

### Overview
This block generates a personalized email body for each selected spreadsheet row. It uses an AI Agent node with an attached OpenAI chat model.

### Nodes Involved
- AI Agent - Generate Personalized Email
- OpenAI Chat Model - GPT

### Node Details

#### AI Agent - Generate Personalized Email
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  LangChain-based AI agent that transforms spreadsheet context into a finished email body.
- **Configuration choices:**  
  - Prompt is defined directly in the node.
  - The prompt instructs the model to:
    - write a short personalized reminder email
    - target a business owner registered for tomorrow’s 9AM automation webinar
    - keep the body to 2–3 sentences
    - use proper email formatting and line breaks
    - end with `Best regards, The Automation Team`
    - return only the email text, with no subject line or commentary
  - The row’s `Introduction` field is appended to the prompt.
- **Key expressions or variables used:**  
  - `{{ $json.Introduction }}`
- **Input and output connections:**  
  - Main input: `Get Pending Emails from Sheet`
  - AI language model input: `OpenAI Chat Model - GPT`
  - Main output: `Send Personalized Email via Gmail`
- **Version-specific requirements:**  
  Type version `3.1`.  
  Requires compatible LangChain/AI node support in the n8n instance.
- **Edge cases or potential failure types:**  
  - Missing or empty `Introduction` may produce generic or low-quality output
  - Prompt formatting issues if the input contains unexpected content
  - OpenAI quota/rate-limit/authentication failures propagate from the model node
  - Output may not perfectly follow formatting instructions in all cases
- **Sub-workflow reference:**  
  None.

#### OpenAI Chat Model - GPT
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Provides the language model used by the AI Agent.
- **Configuration choices:**  
  - Model: `gpt-5-mini`
  - No additional options or built-in tools configured
  - Uses OpenAI API credentials
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Output via AI language model connector to `AI Agent - Generate Personalized Email`
- **Version-specific requirements:**  
  Type version `1.3`.
- **Edge cases or potential failure types:**  
  - Invalid API key
  - Model access restrictions
  - API rate limits
  - Temporary upstream OpenAI service errors
- **Sub-workflow reference:**  
  None.

---

## 2.3 Gmail Sending and Spreadsheet Status Update

### Overview
This block sends the AI-generated email via Gmail and updates the corresponding spreadsheet row so it is no longer treated as pending.

### Nodes Involved
- Send Personalized Email via Gmail
- Update Email Status to Sent

### Node Details

#### Send Personalized Email via Gmail
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends outgoing email messages using Gmail OAuth2.
- **Configuration choices:**  
  - Recipient: value from spreadsheet `To`
  - Subject: value from spreadsheet `Subject`
  - Body: AI-generated `output`
  - Email type: plain text
  - Attribution/footer insertion disabled
- **Key expressions or variables used:**  
  - `={{ $json.To }}`
  - `={{ $json.Subject }}`
  - `={{ $json.output }}`
- **Input and output connections:**  
  - Input: `AI Agent - Generate Personalized Email`
  - Output: `Update Email Status to Sent`
- **Version-specific requirements:**  
  Type version `2.2`.
- **Edge cases or potential failure types:**  
  - Gmail OAuth token expiration
  - Invalid or empty recipient email
  - Empty subject/body
  - Gmail sending quotas or anti-abuse restrictions
  - If the AI node does not return `output`, message creation fails
- **Sub-workflow reference:**  
  None.

#### Update Email Status to Sent
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Updates the original spreadsheet row after successful send.
- **Configuration choices:**  
  - Operation: `update`
  - Spreadsheet: **Email Manager**
  - Sheet/tab: **Emails with AI**
  - Matching column: `row_number`
  - Writes:
    - `Status = "Sent"`
    - `row_number` from the original fetched row
  - Type conversion disabled
- **Key expressions or variables used:**  
  - `={{ $('Get Pending Emails from Sheet').item.json.row_number }}`
- **Input and output connections:**  
  - Input: `Send Personalized Email via Gmail`
  - Output: none
- **Version-specific requirements:**  
  Type version `4.7`.
- **Edge cases or potential failure types:**  
  - If `row_number` is missing or mismatched, the update may fail or target no row
  - Sheet schema drift can break column mapping
  - If multiple rows somehow share inconsistent row metadata, updates may be unreliable
- **Sub-workflow reference:**  
  None.

---

## 2.4 Gmail Reply Detection and Sheet Logging

### Overview
This block monitors the Gmail inbox for new messages, matches the sender to a spreadsheet row, and stores the reply text in the `Response` column.

### Nodes Involved
- Gmail Trigger - New Email Responses
- Get Matching Email Row by Sender
- Update Sheet with Email Response

### Node Details

#### Gmail Trigger - New Email Responses
- **Type and technical role:** `n8n-nodes-base.gmailTrigger`  
  Polling trigger that checks Gmail for incoming messages.
- **Configuration choices:**  
  - Polling interval: every minute
  - `simple` mode disabled, so richer Gmail payload data is returned
  - No additional filters configured
- **Key expressions or variables used:**  
  Downstream nodes use values from:
  - `$json.from.value[0].address`
  - `$json.text`
- **Input and output connections:**  
  - Input: none
  - Output: `Get Matching Email Row by Sender`
- **Version-specific requirements:**  
  Type version `1.3`.
- **Edge cases or potential failure types:**  
  - Gmail OAuth permission issues
  - Polling latency or duplicate detection behavior depending on mailbox state
  - Messages without plain-text body may produce incomplete `text`
  - Unexpected sender structure may break downstream expressions
- **Sub-workflow reference:**  
  None.

#### Get Matching Email Row by Sender
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Looks up the spreadsheet row associated with the sender of the incoming email.
- **Configuration choices:**  
  - Spreadsheet: **Email Manager**
  - Sheet/tab: **Emails**
  - Filters:
    - `To` = sender email address from Gmail trigger
    - `Response` present as an additional lookup column entry, though no explicit lookup value is supplied in the JSON
- **Key expressions or variables used:**  
  - `={{ $json.from.value[0].address }}`
- **Input and output connections:**  
  - Input: `Gmail Trigger - New Email Responses`
  - Output: `Update Sheet with Email Response`
- **Version-specific requirements:**  
  Type version `4.7`.
- **Edge cases or potential failure types:**  
  - This branch uses a different tab name (`Emails`) than the outbound branch (`Emails with AI`)
  - If the sender email does not exist in the `To` column, no row is returned
  - If multiple rows use the same recipient address, matching may be ambiguous
  - The extra `Response` filter entry appears incomplete and may cause confusion or filtering issues depending on n8n behavior
- **Sub-workflow reference:**  
  None.

#### Update Sheet with Email Response
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Writes the reply text back into the matching spreadsheet row.
- **Configuration choices:**  
  - Operation: `update`
  - Spreadsheet: **Email Manager**
  - Sheet/tab: **Emails**
  - Matching column: `row_number`
  - Writes:
    - `Response` = incoming email plain text
    - `row_number` = row selected in previous step
- **Key expressions or variables used:**  
  - `={{ $('Gmail Trigger - New Email Responses').item.json.text }}`
  - `={{ $json.row_number }}`
- **Input and output connections:**  
  - Input: `Get Matching Email Row by Sender`
  - Output: none
- **Version-specific requirements:**  
  Type version `4.7`.
- **Edge cases or potential failure types:**  
  - Empty or null `text` body may overwrite response with blank content
  - `row_number` mismatch causes failed or skipped update
  - If no row was matched upstream, this node receives no items
  - HTML-only replies may not be captured well in the `text` field
- **Sub-workflow reference:**  
  None.

---

## 2.5 Embedded Notes and Reference Material

### Overview
These nodes are not executable business logic. They provide operational guidance, setup instructions, contact details, and a video link.

### Nodes Involved
- Workflow Description
- Creator Contact Info
- Video Walkthrough

### Node Details

#### Workflow Description
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual documentation note on the canvas.
- **Configuration choices:**  
  Large note containing:
  - required credentials
  - expected Google Sheet structure
  - explanation of sending branch
  - explanation of response branch
  - configuration reminders
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  None operational.
- **Sub-workflow reference:**  
  None.

#### Creator Contact Info
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual branding/contact note.
- **Configuration choices:**  
  Contains creator branding, email, booking link, YouTube link, and LinkedIn profile.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  None operational.
- **Sub-workflow reference:**  
  None.

#### Video Walkthrough
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Visual note linking to a YouTube walkthrough.
- **Configuration choices:**  
  Includes clickable thumbnail linked to the video.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  None operational.
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Manual Trigger - Start Email Workflow | n8n-nodes-base.manualTrigger | Manual entry point for outbound email sending |  | Get Pending Emails from Sheet | ## Workflow Overview<br><br>This workflow automates sending and tracking emails directly from Google Sheets, saving hours of manual work each week. It reads email data from your spreadsheet, generates personalized content using AI, sends the messages via Gmail, and automatically tracks responses—all without manual intervention.<br><br>### First Setup<br><br>**Required Credentials:**<br>- Google Sheets OAuth2 connection for reading and updating spreadsheet data<br>- Gmail OAuth2 connection for sending emails and monitoring replies<br>- OpenAI API credentials for AI-powered email personalization<br><br>**Google Sheet Structure:**<br>Create a spreadsheet with these columns:<br>- **To**: Recipient email address<br>- **Subject**: Email subject line<br>- **Introduction**: Brief one-sentence recipient context for AI personalization<br>- **Status**: Track email state ("To send" or "Sent")<br>- **Response**: Captures reply content automatically<br><br>### How It Works<br><br>**Email Sending Branch:**<br>Manually triggered, this flow fetches all rows with Status = "To send", passes the Introduction field to an AI agent that generates personalized email content following your prompt template, sends the email via Gmail, and updates the Status to "Sent".<br><br>**Response Tracking Branch:**<br>Runs automatically every minute, monitoring Gmail for new messages. When a reply arrives, it matches the sender to your spreadsheet and logs the response content in the Response column.<br><br>### Configuration<br><br>Reconfigure the Google Sheets document ID and sheet names to point to your own spreadsheet. Adjust the AI prompt template in the AI Agent node to match your email style and use case. Update Gmail credentials to your sending account. |
| Get Pending Emails from Sheet | n8n-nodes-base.googleSheets | Reads rows marked `To send` from Google Sheets | Manual Trigger - Start Email Workflow | AI Agent - Generate Personalized Email | ## Workflow Overview<br><br>This workflow automates sending and tracking emails directly from Google Sheets, saving hours of manual work each week. It reads email data from your spreadsheet, generates personalized content using AI, sends the messages via Gmail, and automatically tracks responses—all without manual intervention.<br><br>### First Setup<br><br>**Required Credentials:**<br>- Google Sheets OAuth2 connection for reading and updating spreadsheet data<br>- Gmail OAuth2 connection for sending emails and monitoring replies<br>- OpenAI API credentials for AI-powered email personalization<br><br>**Google Sheet Structure:**<br>Create a spreadsheet with these columns:<br>- **To**: Recipient email address<br>- **Subject**: Email subject line<br>- **Introduction**: Brief one-sentence recipient context for AI personalization<br>- **Status**: Track email state ("To send" or "Sent")<br>- **Response**: Captures reply content automatically<br><br>### How It Works<br><br>**Email Sending Branch:**<br>Manually triggered, this flow fetches all rows with Status = "To send", passes the Introduction field to an AI agent that generates personalized email content following your prompt template, sends the email via Gmail, and updates the Status to "Sent".<br><br>**Response Tracking Branch:**<br>Runs automatically every minute, monitoring Gmail for new messages. When a reply arrives, it matches the sender to your spreadsheet and logs the response content in the Response column.<br><br>### Configuration<br><br>Reconfigure the Google Sheets document ID and sheet names to point to your own spreadsheet. Adjust the AI prompt template in the AI Agent node to match your email style and use case. Update Gmail credentials to your sending account. |
| AI Agent - Generate Personalized Email | @n8n/n8n-nodes-langchain.agent | Generates personalized email body from sheet data | Get Pending Emails from Sheet; OpenAI Chat Model - GPT | Send Personalized Email via Gmail | ## Workflow Overview<br><br>This workflow automates sending and tracking emails directly from Google Sheets, saving hours of manual work each week. It reads email data from your spreadsheet, generates personalized content using AI, sends the messages via Gmail, and automatically tracks responses—all without manual intervention.<br><br>### First Setup<br><br>**Required Credentials:**<br>- Google Sheets OAuth2 connection for reading and updating spreadsheet data<br>- Gmail OAuth2 connection for sending emails and monitoring replies<br>- OpenAI API credentials for AI-powered email personalization<br><br>**Google Sheet Structure:**<br>Create a spreadsheet with these columns:<br>- **To**: Recipient email address<br>- **Subject**: Email subject line<br>- **Introduction**: Brief one-sentence recipient context for AI personalization<br>- **Status**: Track email state ("To send" or "Sent")<br>- **Response**: Captures reply content automatically<br><br>### How It Works<br><br>**Email Sending Branch:**<br>Manually triggered, this flow fetches all rows with Status = "To send", passes the Introduction field to an AI agent that generates personalized email content following your prompt template, sends the email via Gmail, and updates the Status to "Sent".<br><br>**Response Tracking Branch:**<br>Runs automatically every minute, monitoring Gmail for new messages. When a reply arrives, it matches the sender to your spreadsheet and logs the response content in the Response column.<br><br>### Configuration<br><br>Reconfigure the Google Sheets document ID and sheet names to point to your own spreadsheet. Adjust the AI prompt template in the AI Agent node to match your email style and use case. Update Gmail credentials to your sending account. |
| OpenAI Chat Model - GPT | @n8n/n8n-nodes-langchain.lmChatOpenAi | Supplies the LLM used by the AI agent |  | AI Agent - Generate Personalized Email | ## Workflow Overview<br><br>This workflow automates sending and tracking emails directly from Google Sheets, saving hours of manual work each week. It reads email data from your spreadsheet, generates personalized content using AI, sends the messages via Gmail, and automatically tracks responses—all without manual intervention.<br><br>### First Setup<br><br>**Required Credentials:**<br>- Google Sheets OAuth2 connection for reading and updating spreadsheet data<br>- Gmail OAuth2 connection for sending emails and monitoring replies<br>- OpenAI API credentials for AI-powered email personalization<br><br>**Google Sheet Structure:**<br>Create a spreadsheet with these columns:<br>- **To**: Recipient email address<br>- **Subject**: Email subject line<br>- **Introduction**: Brief one-sentence recipient context for AI personalization<br>- **Status**: Track email state ("To send" or "Sent")<br>- **Response**: Captures reply content automatically<br><br>### How It Works<br><br>**Email Sending Branch:**<br>Manually triggered, this flow fetches all rows with Status = "To send", passes the Introduction field to an AI agent that generates personalized email content following your prompt template, sends the email via Gmail, and updates the Status to "Sent".<br><br>**Response Tracking Branch:**<br>Runs automatically every minute, monitoring Gmail for new messages. When a reply arrives, it matches the sender to your spreadsheet and logs the response content in the Response column.<br><br>### Configuration<br><br>Reconfigure the Google Sheets document ID and sheet names to point to your own spreadsheet. Adjust the AI prompt template in the AI Agent node to match your email style and use case. Update Gmail credentials to your sending account. |
| Send Personalized Email via Gmail | n8n-nodes-base.gmail | Sends AI-generated email through Gmail | AI Agent - Generate Personalized Email | Update Email Status to Sent | ## Workflow Overview<br><br>This workflow automates sending and tracking emails directly from Google Sheets, saving hours of manual work each week. It reads email data from your spreadsheet, generates personalized content using AI, sends the messages via Gmail, and automatically tracks responses—all without manual intervention.<br><br>### First Setup<br><br>**Required Credentials:**<br>- Google Sheets OAuth2 connection for reading and updating spreadsheet data<br>- Gmail OAuth2 connection for sending emails and monitoring replies<br>- OpenAI API credentials for AI-powered email personalization<br><br>**Google Sheet Structure:**<br>Create a spreadsheet with these columns:<br>- **To**: Recipient email address<br>- **Subject**: Email subject line<br>- **Introduction**: Brief one-sentence recipient context for AI personalization<br>- **Status**: Track email state ("To send" or "Sent")<br>- **Response**: Captures reply content automatically<br><br>### How It Works<br><br>**Email Sending Branch:**<br>Manually triggered, this flow fetches all rows with Status = "To send", passes the Introduction field to an AI agent that generates personalized email content following your prompt template, sends the email via Gmail, and updates the Status to "Sent".<br><br>**Response Tracking Branch:**<br>Runs automatically every minute, monitoring Gmail for new messages. When a reply arrives, it matches the sender to your spreadsheet and logs the response content in the Response column.<br><br>### Configuration<br><br>Reconfigure the Google Sheets document ID and sheet names to point to your own spreadsheet. Adjust the AI prompt template in the AI Agent node to match your email style and use case. Update Gmail credentials to your sending account. |
| Update Email Status to Sent | n8n-nodes-base.googleSheets | Marks sent rows as `Sent` in Google Sheets | Send Personalized Email via Gmail |  | ## Workflow Overview<br><br>This workflow automates sending and tracking emails directly from Google Sheets, saving hours of manual work each week. It reads email data from your spreadsheet, generates personalized content using AI, sends the messages via Gmail, and automatically tracks responses—all without manual intervention.<br><br>### First Setup<br><br>**Required Credentials:**<br>- Google Sheets OAuth2 connection for reading and updating spreadsheet data<br>- Gmail OAuth2 connection for sending emails and monitoring replies<br>- OpenAI API credentials for AI-powered email personalization<br><br>**Google Sheet Structure:**<br>Create a spreadsheet with these columns:<br>- **To**: Recipient email address<br>- **Subject**: Email subject line<br>- **Introduction**: Brief one-sentence recipient context for AI personalization<br>- **Status**: Track email state ("To send" or "Sent")<br>- **Response**: Captures reply content automatically<br><br>### How It Works<br><br>**Email Sending Branch:**<br>Manually triggered, this flow fetches all rows with Status = "To send", passes the Introduction field to an AI agent that generates personalized email content following your prompt template, sends the email via Gmail, and updates the Status to "Sent".<br><br>**Response Tracking Branch:**<br>Runs automatically every minute, monitoring Gmail for new messages. When a reply arrives, it matches the sender to your spreadsheet and logs the response content in the Response column.<br><br>### Configuration<br><br>Reconfigure the Google Sheets document ID and sheet names to point to your own spreadsheet. Adjust the AI prompt template in the AI Agent node to match your email style and use case. Update Gmail credentials to your sending account. |
| Gmail Trigger - New Email Responses | n8n-nodes-base.gmailTrigger | Polls Gmail for new incoming replies |  | Get Matching Email Row by Sender | ## Workflow Overview<br><br>This workflow automates sending and tracking emails directly from Google Sheets, saving hours of manual work each week. It reads email data from your spreadsheet, generates personalized content using AI, sends the messages via Gmail, and automatically tracks responses—all without manual intervention.<br><br>### First Setup<br><br>**Required Credentials:**<br>- Google Sheets OAuth2 connection for reading and updating spreadsheet data<br>- Gmail OAuth2 connection for sending emails and monitoring replies<br>- OpenAI API credentials for AI-powered email personalization<br><br>**Google Sheet Structure:**<br>Create a spreadsheet with these columns:<br>- **To**: Recipient email address<br>- **Subject**: Email subject line<br>- **Introduction**: Brief one-sentence recipient context for AI personalization<br>- **Status**: Track email state ("To send" or "Sent")<br>- **Response**: Captures reply content automatically<br><br>### How It Works<br><br>**Email Sending Branch:**<br>Manually triggered, this flow fetches all rows with Status = "To send", passes the Introduction field to an AI agent that generates personalized email content following your prompt template, sends the email via Gmail, and updates the Status to "Sent".<br><br>**Response Tracking Branch:**<br>Runs automatically every minute, monitoring Gmail for new messages. When a reply arrives, it matches the sender to your spreadsheet and logs the response content in the Response column.<br><br>### Configuration<br><br>Reconfigure the Google Sheets document ID and sheet names to point to your own spreadsheet. Adjust the AI prompt template in the AI Agent node to match your email style and use case. Update Gmail credentials to your sending account. |
| Get Matching Email Row by Sender | n8n-nodes-base.googleSheets | Finds the sheet row associated with an incoming sender email | Gmail Trigger - New Email Responses | Update Sheet with Email Response | ## Workflow Overview<br><br>This workflow automates sending and tracking emails directly from Google Sheets, saving hours of manual work each week. It reads email data from your spreadsheet, generates personalized content using AI, sends the messages via Gmail, and automatically tracks responses—all without manual intervention.<br><br>### First Setup<br><br>**Required Credentials:**<br>- Google Sheets OAuth2 connection for reading and updating spreadsheet data<br>- Gmail OAuth2 connection for sending emails and monitoring replies<br>- OpenAI API credentials for AI-powered email personalization<br><br>**Google Sheet Structure:**<br>Create a spreadsheet with these columns:<br>- **To**: Recipient email address<br>- **Subject**: Email subject line<br>- **Introduction**: Brief one-sentence recipient context for AI personalization<br>- **Status**: Track email state ("To send" or "Sent")<br>- **Response**: Captures reply content automatically<br><br>### How It Works<br><br>**Email Sending Branch:**<br>Manually triggered, this flow fetches all rows with Status = "To send", passes the Introduction field to an AI agent that generates personalized email content following your prompt template, sends the email via Gmail, and updates the Status to "Sent".<br><br>**Response Tracking Branch:**<br>Runs automatically every minute, monitoring Gmail for new messages. When a reply arrives, it matches the sender to your spreadsheet and logs the response content in the Response column.<br><br>### Configuration<br><br>Reconfigure the Google Sheets document ID and sheet names to point to your own spreadsheet. Adjust the AI prompt template in the AI Agent node to match your email style and use case. Update Gmail credentials to your sending account. |
| Update Sheet with Email Response | n8n-nodes-base.googleSheets | Writes incoming reply text into the spreadsheet | Get Matching Email Row by Sender |  | ## Workflow Overview<br><br>This workflow automates sending and tracking emails directly from Google Sheets, saving hours of manual work each week. It reads email data from your spreadsheet, generates personalized content using AI, sends the messages via Gmail, and automatically tracks responses—all without manual intervention.<br><br>### First Setup<br><br>**Required Credentials:**<br>- Google Sheets OAuth2 connection for reading and updating spreadsheet data<br>- Gmail OAuth2 connection for sending emails and monitoring replies<br>- OpenAI API credentials for AI-powered email personalization<br><br>**Google Sheet Structure:**<br>Create a spreadsheet with these columns:<br>- **To**: Recipient email address<br>- **Subject**: Email subject line<br>- **Introduction**: Brief one-sentence recipient context for AI personalization<br>- **Status**: Track email state ("To send" or "Sent")<br>- **Response**: Captures reply content automatically<br><br>### How It Works<br><br>**Email Sending Branch:**<br>Manually triggered, this flow fetches all rows with Status = "To send", passes the Introduction field to an AI agent that generates personalized email content following your prompt template, sends the email via Gmail, and updates the Status to "Sent".<br><br>**Response Tracking Branch:**<br>Runs automatically every minute, monitoring Gmail for new messages. When a reply arrives, it matches the sender to your spreadsheet and logs the response content in the Response column.<br><br>### Configuration<br><br>Reconfigure the Google Sheets document ID and sheet names to point to your own spreadsheet. Adjust the AI prompt template in the AI Agent node to match your email style and use case. Update Gmail credentials to your sending account. |
| Workflow Description | n8n-nodes-base.stickyNote | Embedded workflow documentation |  |  | ## Workflow Overview<br><br>This workflow automates sending and tracking emails directly from Google Sheets, saving hours of manual work each week. It reads email data from your spreadsheet, generates personalized content using AI, sends the messages via Gmail, and automatically tracks responses—all without manual intervention.<br><br>### First Setup<br><br>**Required Credentials:**<br>- Google Sheets OAuth2 connection for reading and updating spreadsheet data<br>- Gmail OAuth2 connection for sending emails and monitoring replies<br>- OpenAI API credentials for AI-powered email personalization<br><br>**Google Sheet Structure:**<br>Create a spreadsheet with these columns:<br>- **To**: Recipient email address<br>- **Subject**: Email subject line<br>- **Introduction**: Brief one-sentence recipient context for AI personalization<br>- **Status**: Track email state ("To send" or "Sent")<br>- **Response**: Captures reply content automatically<br><br>### How It Works<br><br>**Email Sending Branch:**<br>Manually triggered, this flow fetches all rows with Status = "To send", passes the Introduction field to an AI agent that generates personalized email content following your prompt template, sends the email via Gmail, and updates the Status to "Sent".<br><br>**Response Tracking Branch:**<br>Runs automatically every minute, monitoring Gmail for new messages. When a reply arrives, it matches the sender to your spreadsheet and logs the response content in the Response column.<br><br>### Configuration<br><br>Reconfigure the Google Sheets document ID and sheet names to point to your own spreadsheet. Adjust the AI prompt template in the AI Agent node to match your email style and use case. Update Gmail credentials to your sending account. |
| Creator Contact Info | n8n-nodes-base.stickyNote | Creator branding and contact information |  |  | # Contact Us:<br>## Milan @ SmoothWork - [Book a Free Consulting Call](https://smoothwork.ai/book-a-call/)<br>![Milan](https://gravatar.com/avatar/95700d17ba300a9f14c1b8cacf933df7720027b3adda9cbe6183d89142925422?r=pg&d=retro&size=100)<br><br>### We help businesses eliminate busywork by building compact business tools tailored to your process.<br>### Contact us for customizing this, or building similar automations.<br><br>📧 hello@smoothwork.ai<br>▶️ [Check us on YouTube](https://www.youtube.com/@vasarmilan)<br>📞 [Book a Free Consulting Call](https://smoothwork.ai/book-a-call/)<br>💼 [Add me on Linkedin](https://www.linkedin.com/in/mil%C3%A1n-v%C3%A1s%C3%A1rhelyi-3a9985123/) |
| Video Walkthrough | n8n-nodes-base.stickyNote | Video reference link |  |  | # Video Walkthrough<br>[![image.png](https://vasarmilan-public.s3.us-east-1.amazonaws.com/blog_thumbnails/thumbnail_recrji7w9W5rhpyhr.jpg)](https://www.youtube.com/watch?v=jxT6XO4eUwI) |

---

# 4. Reproducing the Workflow from Scratch

## Prerequisites
1. Create or prepare a Google Sheet with at least these columns:
   - `To`
   - `Subject`
   - `Introduction`
   - `Status`
   - `Response`

2. Prepare credentials in n8n:
   - **Google Sheets OAuth2**
   - **Gmail OAuth2**
   - **OpenAI API**

3. Ensure your Gmail account has permission to:
   - send emails
   - allow Gmail trigger polling

4. Ensure your OpenAI account has access to the selected chat model, or choose another supported model if needed.

---

## Build the outbound branch

1. **Create a Manual Trigger node**
   - Node type: `Manual Trigger`
   - Name it: `Manual Trigger - Start Email Workflow`

2. **Create a Google Sheets node to read pending emails**
   - Node type: `Google Sheets`
   - Name it: `Get Pending Emails from Sheet`
   - Credential: your Google Sheets OAuth2 credential
   - Select your spreadsheet document
   - Select the tab intended for outbound sending, matching the source workflow’s `Emails with AI`
   - Configure filtering so only rows with:
     - `Status` = `To send`
   - Connect:
     - `Manual Trigger - Start Email Workflow` → `Get Pending Emails from Sheet`

3. **Create an OpenAI Chat Model node**
   - Node type: `OpenAI Chat Model`
   - Name it: `OpenAI Chat Model - GPT`
   - Credential: your OpenAI API credential
   - Model: `gpt-5-mini`
   - Leave options default unless you need stricter control

4. **Create an AI Agent node**
   - Node type: `AI Agent`
   - Name it: `AI Agent - Generate Personalized Email`
   - Set prompt mode to define the prompt directly
   - Use a prompt equivalent to this:

     ```text
     You are writing a short, personalized reminder email for a business owner who has registered for tomorrow's 9AM automation webinar. The message should feel warm and relevant — referencing something specific from their background to make it feel tailored, not generic. Keep the body to 2-3 sentences. Format it as an email with proper newlines and close with "Best regards, The Automation Team". Return only the email text directly — no subject line, no labels, no extra commentary.

     Here is an example of the expected output:
     Hi Sarah,
     Just a reminder that your seat is reserved for tomorrow's automation webinar at 9AM — we think you'll find it especially useful given your work scaling operations at GreenLeaf Market. We'll be covering practical tools that can save you hours every week.
     Best regards,
     The Automation Team

     {{ $json.Introduction }}
     ```

   - Connect:
     - `Get Pending Emails from Sheet` → `AI Agent - Generate Personalized Email`
     - `OpenAI Chat Model - GPT` → AI language model port on `AI Agent - Generate Personalized Email`

5. **Create a Gmail node to send the generated email**
   - Node type: `Gmail`
   - Name it: `Send Personalized Email via Gmail`
   - Credential: your Gmail OAuth2 credential
   - Configure for sending email
   - Set:
     - **To** = `{{ $json.To }}`
     - **Subject** = `{{ $json.Subject }}`
     - **Message** = `{{ $json.output }}`
     - **Email Type** = `Text`
     - **Append Attribution** = disabled
   - Connect:
     - `AI Agent - Generate Personalized Email` → `Send Personalized Email via Gmail`

6. **Create a Google Sheets node to update sent status**
   - Node type: `Google Sheets`
   - Name it: `Update Email Status to Sent`
   - Credential: same Google Sheets OAuth2 credential
   - Operation: `Update`
   - Select the same spreadsheet and same outbound tab used in step 2
   - Configure column mapping manually
   - Match rows using:
     - `row_number`
   - Write:
     - `Status` = `Sent`
     - `row_number` = `{{ $('Get Pending Emails from Sheet').item.json.row_number }}`
   - Disable type coercion if you want behavior close to the source workflow
   - Connect:
     - `Send Personalized Email via Gmail` → `Update Email Status to Sent`

---

## Build the inbound reply-tracking branch

7. **Create a Gmail Trigger node**
   - Node type: `Gmail Trigger`
   - Name it: `Gmail Trigger - New Email Responses`
   - Credential: the same Gmail OAuth2 credential
   - Polling interval: every minute
   - Disable simple mode so the node returns rich sender/body metadata
   - Leave filters empty unless you want to restrict which inbox messages count

8. **Create a Google Sheets lookup node for matching sender**
   - Node type: `Google Sheets`
   - Name it: `Get Matching Email Row by Sender`
   - Credential: your Google Sheets OAuth2 credential
   - Select the spreadsheet document
   - Select the reply-tracking sheet/tab matching the source workflow’s `Emails`
   - Add a filter where:
     - `To` = `{{ $json.from.value[0].address }}`
   - Note: the source JSON also includes an extra filter entry for `Response` without a visible lookup value. Reproducing the workflow exactly would mean preserving that unusual filter entry if your n8n UI allows it, but in most rebuilds it is safer to omit it unless you specifically need additional filtering.
   - Connect:
     - `Gmail Trigger - New Email Responses` → `Get Matching Email Row by Sender`

9. **Create a Google Sheets update node for saving reply text**
   - Node type: `Google Sheets`
   - Name it: `Update Sheet with Email Response`
   - Credential: your Google Sheets OAuth2 credential
   - Operation: `Update`
   - Select the same sheet/tab as step 8
   - Match rows using:
     - `row_number`
   - Write:
     - `Response` = `{{ $('Gmail Trigger - New Email Responses').item.json.text }}`
     - `row_number` = `{{ $json.row_number }}`
   - Connect:
     - `Get Matching Email Row by Sender` → `Update Sheet with Email Response`

---

## Add documentation notes

10. **Create a Sticky Note for workflow instructions**
    - Node type: `Sticky Note`
    - Name it: `Workflow Description`
    - Include:
      - credential requirements
      - expected sheet columns
      - outbound branch explanation
      - inbound branch explanation
      - configuration reminders

11. **Create a Sticky Note for contact details**
    - Node type: `Sticky Note`
    - Name it: `Creator Contact Info`
    - Include links if desired:
      - https://smoothwork.ai/book-a-call/
      - https://www.youtube.com/@vasarmilan
      - https://www.linkedin.com/in/mil%C3%A1n-v%C3%A1s%C3%A1rhelyi-3a9985123/

12. **Create a Sticky Note for video reference**
    - Node type: `Sticky Note`
    - Name it: `Video Walkthrough`
    - Add the YouTube video link:
      - https://www.youtube.com/watch?v=jxT6XO4eUwI

---

## Important implementation notes

13. **Decide whether both branches should use the same sheet tab**
   - In the provided workflow:
     - outbound branch uses `Emails with AI`
     - inbound branch uses `Emails`
   - If this is intentional, keep both tabs.
   - If not, unify them to avoid reply logging into a different sheet than the send process.

14. **Verify row_number support**
   - The workflow relies on Google Sheets row metadata.
   - Make sure your Google Sheets node version exposes `row_number` so updates can target the original row.

15. **Test with sample data**
   - Add at least one row with:
     - a valid email in `To`
     - a meaningful `Subject`
     - a short `Introduction`
     - `Status = To send`
   - Run the manual branch.
   - Confirm:
     - AI generated text appears in Gmail send action
     - email is sent
     - row status changes to `Sent`

16. **Test the reply branch**
   - Send a reply from the target inbox or an external mailbox.
   - Wait for the Gmail trigger polling cycle.
   - Confirm the `Response` column is updated with the email text.

17. **Optionally harden the workflow**
   - Add an IF node before send to validate `To` and `Subject`
   - Add deduplication logic for repeated replies
   - Add error handling branch for failed sends
   - Add a timestamp column for send time and response time
   - Restrict Gmail trigger to specific labels or threads if inbox noise is a concern

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Creator contact and consulting offer: Milan @ SmoothWork | https://smoothwork.ai/book-a-call/ |
| SmoothWork general outreach email | hello@smoothwork.ai |
| Creator YouTube channel | https://www.youtube.com/@vasarmilan |
| Creator LinkedIn | https://www.linkedin.com/in/mil%C3%A1n-v%C3%A1s%C3%A1rhelyi-3a9985123/ |
| Video walkthrough for this workflow | https://www.youtube.com/watch?v=jxT6XO4eUwI |
| Embedded thumbnail image used in the video note | https://vasarmilan-public.s3.us-east-1.amazonaws.com/blog_thumbnails/thumbnail_recrji7w9W5rhpyhr.jpg |