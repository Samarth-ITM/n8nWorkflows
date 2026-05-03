Send Scheduled RSS News Digest Emails with Formatted HTML in Gmail

https://n8nworkflows.xyz/workflows/send-scheduled-rss-news-digest-emails-with-formatted-html-in-gmail-6223


# Send Scheduled RSS News Digest Emails with Formatted HTML in Gmail

### 1. Workflow Overview

This workflow automates the process of sending scheduled news digest emails via Gmail using RSS feed data. The primary use case is to periodically collect news articles from an RSS feed (specifically from "prothomalo.com"), format them into an attractive, responsive HTML email, and send this digest automatically to designated recipients.

The workflow is logically divided into the following blocks:

- **1.1 Scheduling**: Triggers the workflow at set intervals.
- **1.2 RSS Feed Retrieval**: Fetches the RSS XML feed from a news source.
- **1.3 XML Parsing**: Converts the RSS XML into JSON format for easier manipulation.
- **1.4 HTML Email Content Generation**: Builds a styled HTML block representing the news digest.
- **1.5 Email Sending**: Sends the formatted HTML email through Gmail using OAuth2 authentication.

A sticky note node provides a summary and internal comments about workflow steps.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling Block

- **Overview:**  
  Initiates the workflow execution on a scheduled interval, facilitating automatic periodic runs without manual intervention.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger node  
    - Role: Starts workflow execution based on a predefined schedule  
    - Configuration: Runs on a fixed interval (default cron or interval schedule) without specific time configuration visible here  
    - Input: None (trigger node)  
    - Output: Triggers "Get RSS from Prothom Alo" node  
    - Edge Cases: Misconfiguration could cause no triggers or too frequent triggers; time zone considerations may affect scheduling; version 1.2 used (standard)  
    - No sub-workflows invoked

---

#### 2.2 RSS Feed Retrieval Block

- **Overview:**  
  Fetches the raw RSS XML feed from the specified news website URL.

- **Nodes Involved:**  
  - Get RSS from Prothom Alo

- **Node Details:**

  - **Get RSS from Prothom Alo**  
    - Type: HTTP Request node  
    - Role: Performs a GET request to retrieve RSS feed XML data from "https://prothomalo.com/feed"  
    - Configuration: Simple HTTP GET request; no authentication or headers specified; options left default  
    - Input: Trigger from "Schedule Trigger"  
    - Output: Passes XML content to "Convert XML to JSON"  
    - Edge Cases: Network errors, site downtime, invalid URL, HTTP errors (404, 500), unexpected XML structure might cause parsing issues  
    - Version: 4.2  
    - No sub-workflows invoked

---

#### 2.3 XML Parsing Block

- **Overview:**  
  Converts the acquired RSS XML data into JSON format for easier data extraction and processing in subsequent nodes.

- **Nodes Involved:**  
  - Convert XML to JSON

- **Node Details:**

  - **Convert XML to JSON**  
    - Type: XML node  
    - Role: Parses XML feed data into JSON objects  
    - Configuration: Default parsing options, no special namespaces or attributes extraction configured  
    - Input: Raw XML from "Get RSS from Prothom Alo"  
    - Output: Parsed JSON passed to "Generate HTML News Preview"  
    - Edge Cases: Malformed XML, unexpected feed structure, large feed size causing memory issues  
    - Version: 1  
    - No sub-workflows invoked

---

#### 2.4 HTML Email Content Generation Block

- **Overview:**  
  Generates a styled, responsive HTML snippet representing the news digest content, including article titles, summaries, publishing details, and links.

- **Nodes Involved:**  
  - Generate HTML News Preview

- **Node Details:**

  - **Generate HTML News Preview**  
    - Type: Code (JavaScript) node  
    - Role: Processes JSON feed entries to build a formatted HTML block for embedding in the email body  
    - Configuration:  
      - Uses JavaScript to iterate over feed entries (assumed to be under `feed.entry`)  
      - For each article, extracts title, link, published date, author, summary, category  
      - Formats dates in 'en-US' and 'bn-BD' locales  
      - Constructs inline-styled HTML blocks for each article with consistent branding colors (#cc0000 red) and responsive design  
      - Returns a single JSON output with an `html` property containing the full HTML snippet  
    - Inputs: JSON from "Convert XML to JSON"  
    - Outputs: JSON with `html` key passed to "Send a message" node  
    - Expressions/Variables: Uses `$input.first().json.feed.entry` to access feed entries; uses template literals for HTML construction  
    - Edge Cases: Missing or unexpected fields in feed entries (e.g., no author, no category), empty feed, date parsing errors, locale formatting issues  
    - Version: 2  
    - No sub-workflows invoked

---

#### 2.5 Email Sending Block

- **Overview:**  
  Sends the constructed HTML email digest to recipients using Gmail with OAuth2 authentication.

- **Nodes Involved:**  
  - Send a message

- **Node Details:**

  - **Send a message**  
    - Type: Gmail node  
    - Role: Sends an email using Gmail API with OAuth2 authentication  
    - Configuration:  
      - Email subject: "News Digest From RSS"  
      - Message body: Full HTML email template with embedded CSS styling and dynamic content insertion via `{{ $json.html }}` placeholder (populated from previous node output)  
      - The HTML includes a custom styled email container with brand colors (#cc0000), responsive design, and accessibility tags  
      - No additional options or attachments configured  
    - Credentials: Uses Gmail OAuth2 credential named "Gmail account"  
    - Inputs: JSON with `html` content from "Generate HTML News Preview"  
    - Outputs: None (final node)  
    - Edge Cases: Authentication failure, API limits, email sending errors, invalid HTML causing rendering issues in email clients  
    - Version: 2.1  
    - No sub-workflows invoked

---

#### 2.6 Documentation Block

- **Overview:**  
  Provides internal documentation, workflow summary, and commentary for maintainers or collaborators.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**

  - **Sticky Note**  
    - Type: Sticky Note node  
    - Role: Describes the workflow steps, flow, and design purpose  
    - Configuration: Contains a detailed textual overview of the workflow steps and flowchart  
    - Inputs/Outputs: None  
    - Edge Cases: None  
    - Version: 1

---

### 3. Summary Table

| Node Name               | Node Type               | Functional Role                         | Input Node(s)            | Output Node(s)              | Sticky Note                                                                                   |
|-------------------------|-------------------------|---------------------------------------|--------------------------|----------------------------|-----------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger        | Initiates workflow on schedule        | None                     | Get RSS from Prothom Alo    | Scheduled RSS News Digest Automation Workflow... (shared note covering entire workflow)       |
| Get RSS from Prothom Alo | HTTP Request            | Fetches RSS XML feed                   | Schedule Trigger          | Convert XML to JSON         | Scheduled RSS News Digest Automation Workflow...                                            |
| Convert XML to JSON      | XML                     | Parses RSS XML to JSON                 | Get RSS from Prothom Alo  | Generate HTML News Preview  | Scheduled RSS News Digest Automation Workflow...                                            |
| Generate HTML News Preview | Code (JavaScript)       | Builds formatted HTML news digest     | Convert XML to JSON       | Send a message              | Scheduled RSS News Digest Automation Workflow...                                            |
| Send a message           | Gmail                   | Sends formatted HTML email             | Generate HTML News Preview| None                       | Scheduled RSS News Digest Automation Workflow...                                            |
| Sticky Note              | Sticky Note             | Internal documentation and comments    | None                     | None                       | Scheduled RSS News Digest Automation Workflow...                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Configuration: Set to run on your preferred interval (e.g., daily, hourly). Default interval can be used.  
   - Position: Start node

2. **Add HTTP Request node ("Get RSS from Prothom Alo")**  
   - Type: HTTP Request  
   - Parameters: Method = GET; URL = `https://prothomalo.com/feed`  
   - Connect output of Schedule Trigger to this node

3. **Add XML node ("Convert XML to JSON")**  
   - Type: XML  
   - Configuration: Default options for parsing XML to JSON  
   - Connect output of HTTP Request node to this node

4. **Add Code node ("Generate HTML News Preview")**  
   - Type: Code (JavaScript)  
   - Parameters: Use the following JS code (adapted to your feed structure):

   ```js
   const entries = $input.first().json.feed.entry || [];

   let html = `
     <div style="max-width:600px;margin:0 auto;background:#fff;padding:20px;font-family:Arial,sans-serif;color:#333;">
       <h2 style="text-align:center;color:#cc0000;">${new Date().toLocaleDateString('en-US', { year: 'numeric', month: 'long', day: 'numeric' })}</h2>
   `;

   for (const entry of entries) {
     const title = entry.title || 'No Title';
     const link = entry.link?.href || '#';
     const published = entry.published ? new Date(entry.published).toLocaleDateString('bn-BD', { year: 'numeric', month: 'long', day: 'numeric' }) : '';
     const author = entry.author?.name || '';
     const summary = entry.summary || '';
     const category = entry.category?.term || '';

     html += `
       <div style="border-bottom:1px solid #eee;padding:15px 0;">
         <a href="${link}" target="_blank" style="color:#cc0000;font-size:18px;font-weight:bold;text-decoration:none;">${title}</a>
         <p style="color:#555;margin:8px 0;">${summary}</p>
         <p style="font-size:12px;color:#999;margin:0;">Published: ${published} | Author: ${author} | Category: ${category}</p>
         <a href="${link}" target="_blank" style="display:inline-block;margin-top:10px;padding:8px 12px;background:#cc0000;color:#fff;text-decoration:none;border-radius:4px;">Read More</a>
       </div>
     `;
   }

   html += '</div>';

   return [{ json: { html } }];
   ```

   - Connect output of XML node to this code node

5. **Add Gmail node ("Send a message")**  
   - Type: Gmail  
   - Parameters:  
     - Subject: "News Digest From RSS"  
     - Message: Paste the full HTML email template (including inline CSS) with the placeholder `{{ $json.html }}` where the dynamic news content is inserted.  
     - Example snippet for message body:

     ```html
     <!DOCTYPE html>
     <html lang="en" >
     <head>
       <meta charset="UTF-8" />
       <meta name="viewport" content="width=device-width, initial-scale=1" />
       <title>News Digest</title>
       <style>
         /* Include all the CSS styles from the original workflow for consistent styling */
       </style>
     </head>
     <body>
       <div class="email-container" role="article" aria-roledescription="email" aria-label="News Digest">
         <div class="header">
           News RSS Feed
         </div>
         <div class="content">
           {{ $json.html }}
         </div>
       </div>
     </body>
     </html>
     ```

   - Credentials: Configure with Gmail OAuth2 credentials (create or reuse existing with appropriate scopes for sending email)  
   - Connect output of Code node to this Gmail node

6. **Add Sticky Note node (optional for documentation)**  
   - Type: Sticky Note  
   - Add internal commentary or instructions as needed  
   - Position for easy reference

7. **Connect nodes in this order:**  
   Schedule Trigger → Get RSS from Prothom Alo → Convert XML to JSON → Generate HTML News Preview → Send a message

8. **Activate the workflow** after testing.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                            |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| The workflow is designed for automatic periodic news digest emails with responsive styling in Gmail. It uses inline CSS for consistent appearance across email clients.                                                         | Internal design consideration                              |
| Styling uses a bold red brand color (#cc0000) with white backgrounds and clear typography for readability.                                                                                                                      | Branding and UI                                             |
| The RSS feed URL is currently set to "https://prothomalo.com/feed" but can be replaced with any other valid RSS feed URL matching similar structure.                                                                            | RSS Feed customization                                      |
| Gmail OAuth2 credentials must have the correct scopes enabled to send emails via the Gmail API.                                                                                                                                 | Gmail developer console                                     |
| For troubleshooting email formatting, test with multiple email clients to ensure responsive design renders correctly.                                                                                                           | Email client compatibility                                 |
| Sticky Note node contains a concise summary of the workflow steps and is useful for onboarding or collaborative editing.                                                                                                        | Internal documentation                                      |

---

**Disclaimer:** The provided text is exclusively derived from an n8n automation workflow. The processing strictly complies with applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly available.