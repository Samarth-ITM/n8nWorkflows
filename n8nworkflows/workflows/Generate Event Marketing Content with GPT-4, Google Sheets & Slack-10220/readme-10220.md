Generate Event Marketing Content with GPT-4, Google Sheets & Slack

https://n8nworkflows.xyz/workflows/generate-event-marketing-content-with-gpt-4--google-sheets---slack-10220


# Generate Event Marketing Content with GPT-4, Google Sheets & Slack

---

### 1. Workflow Overview

This workflow automates the generation of comprehensive marketing content for events using GPT-4, Google Sheets, and Slack. It targets marketing teams who want to streamline content creation for email campaigns, social media posts, and paid ads based on event details submitted via an API webhook. The workflow logically divides into the following blocks:

- **1.1 Input Reception:** Receives event data via a webhook and parses it.
- **1.2 AI Content Generation:** Calls GPT-4 APIs to generate email copy, social media posts, and ad copy separately.
- **1.3 Response Parsing:** Extracts and formats AI response JSON content for each content type.
- **1.4 Content Aggregation:** Merges all generated content into a unified data structure.
- **1.5 Data Persistence:** Logs the generated campaign data into Google Sheets for record keeping.
- **1.6 Communication:** Sends a packaged marketing email with the event content and posts a notification to a Slack channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block listens for incoming HTTP POST requests containing event details, then extracts and prepares the data for content generation.

**Nodes Involved:**  
- Webhook Trigger  
- Extract Event Data  

**Node Details:**

- **Webhook Trigger**  
  - Type: Webhook  
  - Role: Entry point accepting POST requests at path `/create-event-content`.  
  - Config: HTTP method POST, no authentication specified (public webhook).  
  - Inputs: External API calls.  
  - Outputs: Event data JSON to next node.  
  - Edge Cases: Missing or malformed payload, request timeout, unauthorized calls if exposed publicly.  

- **Extract Event Data**  
  - Type: Set  
  - Role: Parses and restructures the incoming event payload for consistent downstream usage.  
  - Config: Uses n8n Set node defaults; likely extracts fields like `event_name`, `event_date`, `event_location`, `target_audience`, `key_features`.  
  - Inputs: Webhook Trigger output.  
  - Outputs: Normalized event data JSON for AI calls.  
  - Edge Cases: Missing required fields, incorrect data types.  

---

#### 2.2 AI Content Generation

**Overview:**  
This block invokes GPT-4 APIs to generate tailored marketing content for email, social media posts, and paid advertising based on the event data.

**Nodes Involved:**  
- Generate Email Content  
- Generate Social Posts  
- Generate Ad Copy  

**Node Details:**

- **Generate Email Content**  
  - Type: HTTP Request  
  - Role: Calls OpenAI's chat/completions endpoint with GPT-4 model to generate email marketing content.  
  - Config:  
    - Model: GPT-4  
    - Prompt includes system role "expert email marketer" and user content with event details interpolated from JSON variables.  
    - Requests JSON with keys: subject, body, cta, ps.  
    - Temperature set to 0.7 for creativity balance.  
  - Inputs: Extracted event data.  
  - Outputs: Raw GPT-4 response JSON.  
  - Edge Cases: API authentication failure, rate limit exceeded, invalid prompt interpolation errors, network timeouts.  

- **Generate Social Posts**  
  - Type: HTTP Request  
  - Role: Calls GPT-4 to create social media posts optimized for LinkedIn, Twitter, Instagram, and Facebook.  
  - Config:  
    - Model: GPT-4  
    - System prompt: social media expert.  
    - User prompt includes event data and requests posts formatted as JSON keys: linkedin, twitter, instagram, facebook.  
    - Temperature set to 0.8 for slightly more creative output.  
  - Inputs: Extracted event data.  
  - Outputs: Raw GPT-4 social post JSON.  
  - Edge Cases: Same as above, plus potential formatting issues with emoji-containing Instagram content.  

- **Generate Ad Copy**  
  - Type: HTTP Request  
  - Role: Calls GPT-4 to generate ad copy for Google Search, Facebook, and LinkedIn Sponsored campaigns.  
  - Config:  
    - Model: GPT-4  
    - System prompt: advertising copywriter.  
    - User prompt requests multiple ad formats with strict character limits.  
    - Temperature 0.7.  
  - Inputs: Extracted event data.  
  - Outputs: Raw GPT-4 ad copy JSON.  
  - Edge Cases: Truncation or format errors, API limit issues.  

---

#### 2.3 Response Parsing

**Overview:**  
Parses the raw JSON string from each GPT-4 response, cleaning and structuring the content for merging.

**Nodes Involved:**  
- Parse Email Response  
- Parse Social Response  
- Parse Ad Response  

**Node Details:**

- **Parse Email Response**  
  - Type: Code (JavaScript)  
  - Role: Parses the JSON string from the `Generate Email Content` node's response.  
  - Script: `const email = JSON.parse($input.first().json.choices[0].message.content); return { email_content: email };`  
  - Inputs: GPT-4 email content response.  
  - Outputs: Parsed email content object.  
  - Edge Cases: JSON parse errors if GPT-4 returns invalid JSON.  

- **Parse Social Response**  
  - Type: Code (JavaScript)  
  - Role: Parses social media posts JSON string.  
  - Script: `const social = JSON.parse($input.first().json.choices[0].message.content); return { social_media: social };`  
  - Inputs: GPT-4 social content response.  
  - Outputs: Parsed social media content object.  
  - Edge Cases: Same JSON parse risks.  

- **Parse Ad Response**  
  - Type: Code (JavaScript)  
  - Role: Parses ad copy JSON string from GPT-4 response.  
  - Script: `const ads = JSON.parse($input.first().json.choices[0].message.content); return { ad_content: ads };`  
  - Inputs: GPT-4 ad content response.  
  - Outputs: Parsed ad content object.  
  - Edge Cases: Same as above.  

---

#### 2.4 Content Aggregation

**Overview:**  
Combines parsed email, social media, and ad content into one cohesive data structure for logging and delivery.

**Nodes Involved:**  
- Merge All Content  

**Node Details:**

- **Merge All Content**  
  - Type: Merge  
  - Role: Combines the three parsed content streams by position (index) into a single JSON object.  
  - Config: Mode set to "combine" with "mergeByPosition" to join corresponding items.  
  - Inputs: Parse Email Response, Parse Social Response, Parse Ad Response.  
  - Outputs: Unified content object containing all generated marketing materials.  
  - Edge Cases: Mismatched item counts causing incomplete merges.  

---

#### 2.5 Data Persistence

**Overview:**  
Logs key event and generated content data into a Google Sheets spreadsheet for tracking marketing campaigns.

**Nodes Involved:**  
- Log to Google Sheets  

**Node Details:**

- **Log to Google Sheets**  
  - Type: Google Sheets  
  - Role: Appends a new row to the "Marketing_Campaigns" sheet in a specified Google Sheets document.  
  - Config:  
    - Uses service account authentication.  
    - Columns mapped include Timestamp, Event_Name, Event_Date, Location, Email_Subject, Status (set to "Generated").  
    - Document ID and sheet name fixed in parameters.  
  - Inputs: Merged content output.  
  - Outputs: Confirmation of row append.  
  - Edge Cases: Sheet access permission errors, API quota limits, invalid data mappings.  

---

#### 2.6 Communication

**Overview:**  
Delivers the generated marketing content via email to the events team and sends a notification message to a Slack channel.

**Nodes Involved:**  
- Email Content Package  
- Notify Team on Slack  

**Node Details:**

- **Email Content Package**  
  - Type: Email Send  
  - Role: Sends an email containing the full event marketing content package.  
  - Config:  
    - From: marketing@company.com  
    - To: events@company.com  
    - CC: marketing-team@company.com  
    - Subject includes event name dynamically.  
    - SMTP credentials required.  
  - Inputs: Output from Google Sheets node (which receives merged content).  
  - Outputs: Email delivery status.  
  - Edge Cases: SMTP authentication issues, email delivery failures, invalid recipient addresses.  

- **Notify Team on Slack**  
  - Type: Slack  
  - Role: Posts a formatted notification to the #marketing-updates Slack channel summarizing campaign readiness.  
  - Config:  
    - Uses Slack API credentials.  
    - Message dynamically inserts event name, date, location, and confirms content generation.  
  - Inputs: Output from Google Sheets node.  
  - Outputs: Slack message post confirmation.  
  - Edge Cases: Slack API rate limits, invalid token, channel permissions.  

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                        | Input Node(s)               | Output Node(s)                  | Sticky Note                                                                                         |
|-----------------------|---------------------|-------------------------------------|-----------------------------|--------------------------------|---------------------------------------------------------------------------------------------------|
| Webhook Trigger       | Webhook             | Receives event data via HTTP POST  | External API Requests        | Extract Event Data              | 🚀 **WEBHOOK TRIGGER** Receives event details: Event name, Date & time, Location, Target audience  |
| Extract Event Data    | Set                 | Parses and prepares event data     | Webhook Trigger             | Generate Email Content, Generate Social Posts, Generate Ad Copy | 📋 **EXTRACT DATA** Parses incoming event info, Prepares for content generation                   |
| Generate Email Content | HTTP Request        | Requests GPT-4 for email content   | Extract Event Data          | Parse Email Response            | 📧 **EMAIL GENERATOR** Creates personalized email: Subject line, Body copy, CTA button, PS line   |
| Generate Social Posts | HTTP Request        | Requests GPT-4 for social posts    | Extract Event Data          | Parse Social Response           | 📱 **SOCIAL MEDIA POSTS** Generates for 4 platforms: LinkedIn, Twitter/X, Instagram, Facebook    |
| Generate Ad Copy      | HTTP Request        | Requests GPT-4 for paid ad copy    | Extract Event Data          | Parse Ad Response               | 💰 **AD COPY GENERATOR** Creates paid ad content: Google Search, Facebook Ads, LinkedIn Sponsored |
| Parse Email Response  | Code (JavaScript)   | Parses GPT-4 email JSON response   | Generate Email Content      | Merge All Content               | 🔄 **PARSE EMAIL** Extracts JSON from API, Cleans up response                                     |
| Parse Social Response | Code (JavaScript)   | Parses GPT-4 social JSON response  | Generate Social Posts       | Merge All Content               | 🔄 **PARSE SOCIAL** Extracts JSON from API, Cleans up response                                   |
| Parse Ad Response     | Code (JavaScript)   | Parses GPT-4 ads JSON response     | Generate Ad Copy            | Merge All Content               | 🔄 **PARSE ADS** Extracts JSON from API, Cleans up response                                     |
| Merge All Content     | Merge               | Aggregates all marketing content   | Parse Email Response, Parse Social Response, Parse Ad Response | Log to Google Sheets           | 🔀 **MERGE CONTENT** Combines all generated: Email content, Social posts, Ad copy                |
| Log to Google Sheets  | Google Sheets       | Logs campaign data for tracking    | Merge All Content           | Email Content Package, Notify Team on Slack | 💾 **SAVE TO SHEETS** Sheet: Marketing_Campaigns, Tracks all campaigns, Doc: 5x4w3v2u1t0s9r8q   |
| Email Content Package | Email Send          | Sends marketing email to events team | Log to Google Sheets        | —                              | 📧 **EMAIL DELIVERY** To: events@company.com, CC: marketing-team@company.com, All content in email body |
| Notify Team on Slack  | Slack               | Posts notification to Slack channel | Log to Google Sheets        | —                              | 💬 **SLACK NOTIFICATION** Channel: #marketing-updates, Channel ID: C11223MARKETING, Team alert with summary |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `create-event-content`  
   - No authentication (or secure as preferred).  
   - Position: Left side (starting node).  

2. **Create Extract Event Data Node**  
   - Type: Set  
   - Connect from Webhook Trigger output.  
   - Configure to extract and set variables from incoming JSON such as:  
     - `event_name`  
     - `event_date`  
     - `event_location`  
     - `target_audience`  
     - `key_features`  
   - Position: Right of Webhook Trigger.  

3. **Create Generate Email Content Node**  
   - Type: HTTP Request  
   - Connect from Extract Event Data.  
   - Method: POST  
   - URL: `https://api.openai.com/v1/chat/completions`  
   - Authentication: Add OpenAI API credentials (Bearer Token).  
   - Body (JSON):  
     ```json
     {
       "model": "gpt-4",
       "messages": [
         {"role": "system", "content": "You are an expert email marketer. Create compelling email content."},
         {"role": "user", "content": "Create email campaign for Event: {{ $json.event_name }}, Date: {{ $json.event_date }}, Location: {{ $json.event_location }}, Audience: {{ $json.target_audience }}, Features: {{ $json.key_features }}. Generate: 1) Subject line (under 60 chars) 2) Email body (150-200 words) 3) CTA button text 4) PS line. Format as JSON with keys: subject, body, cta, ps"}
       ],
       "temperature": 0.7
     }
     ```  
   - Position: Below Extract Event Data and to the right.  

4. **Create Generate Social Posts Node**  
   - Type: HTTP Request  
   - Connect from Extract Event Data.  
   - Same OpenAI API settings, with prompt adjusted for social media:  
     - System: "You are a social media expert. Create engaging platform-optimized content."  
     - User: Requests posts for LinkedIn, Twitter, Instagram (with emojis), Facebook with format JSON keys.  
     - Temperature: 0.8  
   - Position: Right of Extract Event Data.  

5. **Create Generate Ad Copy Node**  
   - Type: HTTP Request  
   - Connect from Extract Event Data.  
   - OpenAI API with prompt for ad copywriter and specific ad formats with character limits.  
   - Temperature: 0.7  
   - Position: Below Generate Social Posts.  

6. **Create Parse Email Response Node**  
   - Type: Code (JavaScript)  
   - Connect from Generate Email Content.  
   - Code:  
     ```js
     const email = JSON.parse($input.first().json.choices[0].message.content);
     return { email_content: email };
     ```  
   - Position: Right of Generate Email Content.  

7. **Create Parse Social Response Node**  
   - Type: Code (JavaScript)  
   - Connect from Generate Social Posts.  
   - Code:  
     ```js
     const social = JSON.parse($input.first().json.choices[0].message.content);
     return { social_media: social };
     ```  
   - Position: Right of Generate Social Posts.  

8. **Create Parse Ad Response Node**  
   - Type: Code (JavaScript)  
   - Connect from Generate Ad Copy.  
   - Code:  
     ```js
     const ads = JSON.parse($input.first().json.choices[0].message.content);
     return { ad_content: ads };
     ```  
   - Position: Right of Generate Ad Copy.  

9. **Create Merge All Content Node**  
   - Type: Merge  
   - Connect inputs from Parse Email Response, Parse Social Response, and Parse Ad Response.  
   - Mode: Combine  
   - Combination Mode: Merge by position  
   - Position: Right of all Parse nodes.  

10. **Create Log to Google Sheets Node**  
    - Type: Google Sheets  
    - Connect from Merge All Content.  
    - Operation: Append  
    - Document ID: Use your Google Sheets document ID  
    - Sheet Name: `Marketing_Campaigns`  
    - Authentication: Service Account Google API credentials  
    - Map columns: Timestamp (current time), Event_Name, Event_Date, Location, Email_Subject (from merged content), Status (set to "Generated")  
    - Position: Right of Merge All Content.  

11. **Create Email Content Package Node**  
    - Type: Email Send  
    - Connect from Log to Google Sheets.  
    - From Email: marketing@company.com  
    - To Email: events@company.com  
    - CC Email: marketing-team@company.com  
    - Subject: Use expression to include event name (e.g., `✨ Event Marketing Content Ready - {{ $json.event_name }}`)  
    - SMTP Credentials: Configure with your SMTP server details.  
    - Position: Right and above Log to Google Sheets.  

12. **Create Notify Team on Slack Node**  
    - Type: Slack  
    - Connect from Log to Google Sheets.  
    - Use Slack API credentials with appropriate scopes to post messages.  
    - Channel: #marketing-updates (or specify channel ID)  
    - Message: Include event name, date, location, and confirmations of content readiness using expressions.  
    - Position: Right and below Email Content Package.  

**Optional:** Add Sticky Notes for clarity at each major block.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                  |
|-------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Workflow uses GPT-4 model calls via OpenAI API; ensure API key has GPT-4 access enabled.        | OpenAI API Documentation: https://platform.openai.com/docs    |
| Google Sheets node requires a Service Account with write permissions on the target spreadsheet. | Google Sheets API setup: https://developers.google.com/sheets/api |
| Slack notifications require a bot token with `chat:write` scope in the target workspace.        | Slack API Docs: https://api.slack.com/messaging/sending          |
| Prompts include strict JSON formatting instructions to simplify parsing downstream.             | Consistent prompt engineering reduces parse errors.             |
| Error handling is recommended for API nodes to catch and retry on rate limits or network errors. | n8n documentation on error workflows: https://docs.n8n.io/nodes/error-handling/ |

---

**Disclaimer:**  
The text provided is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with content policies and includes no illegal, offensive, or protected elements. All data handled is legal and public.

---