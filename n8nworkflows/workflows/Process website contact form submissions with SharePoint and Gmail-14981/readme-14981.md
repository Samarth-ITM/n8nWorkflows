Process website contact form submissions with SharePoint and Gmail

https://n8nworkflows.xyz/workflows/process-website-contact-form-submissions-with-sharepoint-and-gmail-14981


# Process website contact form submissions with SharePoint and Gmail

The user wants a comprehensive, structured reference document based on the n8n workflow JSON provided. Let me analyze the workflow thoroughly and then produce the document following the required structure.

Let me break down the workflow:

**Workflow Name:** Website Contact Form Processor

**Nodes:**
1. **Overview** - Sticky Note (typeVersion 1) - Contains overall workflow description
2. **Section - SharePoint** - Sticky Note (typeVersion 1) - Section label for SharePoint logging
3. **Section - Email** - Sticky Note (typeVersion 1) - Section label for Email notification
4. **Webhook** - Webhook node (typeVersion 2.1) - Receives POST requests at path "form-processor"
5. **Log Submission to SharePoint** - Microsoft SharePoint node (typeVersion 1) - Creates item in SharePoint list
6. **Build Branded Email HTML** - Code node (typeVersion 2) - Constructs HTML email body
7. **Send Email Notification** - Gmail node (typeVersion 2.2) - Sends the email notification
8. **Sticky Note** - Sticky Note (typeVersion 1) - Contains customization instructions for the Code node

**Connections:**
- Webhook → Log Submission to SharePoint
- Log Submission to SharePoint → Build Branded Email HTML
- Build Branded Email HTML → Send Email Notification

**Logical blocks:**
1. Input Reception (Webhook)
2. Data Persistence / SharePoint Logging (Log Submission to SharePoint)
3. Email Construction (Build Branded Email HTML)
4. Email Delivery (Send Email Notification)

Let me now produce the comprehensive document.### 1. Workflow Overview

This workflow, **Website Contact Form Processor**, automates the handling of website contact-form submissions. Whenever a visitor submits a form on a website, the workflow:

1. **Receives the submission** via a POST Webhook, capturing Name, Email, Phone, Message, Date, and Time.
2. **Logs every field** to a Microsoft SharePoint list for permanent record-keeping.
3. **Constructs a branded HTML email** (with customizable logo, brand name, and colors) that presents the submission data in a professional, styled layout and includes a one-click "Reply" button.
4. **Delivers the notification email** via Gmail to a designated recipient, with the Reply-To header automatically set to the submitter's email address for instant one-click responses.

The flow is strictly sequential: Webhook → SharePoint → Code → Gmail. There are no branching paths, loops, or sub-workflow calls.

---

### 2. Block-by-Block Analysis

---

#### Block 1 — Input Reception

**Overview:** The Webhook node acts as the sole entry point. It exposes a POST endpoint that your website form must target. The entire form payload becomes available to downstream nodes.

| Node | Details |
|---|---|
| **Webhook** | **Type:** `n8n-nodes-base.webhook` (v2.1). **Role:** Receives an HTTP POST request at the path `form-processor`. **Configuration:** HTTP method set to POST; no authentication is required on the webhook by default. **Key expressions/variables:** The request body (`$json.body`) carries the fields: `Name`, `Email`, `Phone`, `Message`, `Date`, `Time`. **Connections:** Output → `Log Submission to SharePoint`. **Edge cases:** If the client sends a GET instead of POST, n8n will return a 405 error. If the body is empty or missing expected fields, downstream Code node defaults those to "(not provided)". If n8n is restarted or the workflow deactivated, the webhook URL becomes unavailable. No request-body size validation is enforced by n8n beyond server limits. |

---

#### Block 2 — Data Persistence (SharePoint Logging)

**Overview:** After the webhook receives data, the submission is written as a new item in a Microsoft SharePoint list. This provides an auditable, persistent record independent of email.

| Node | Details |
|---|---|
| **Log Submission to SharePoint** | **Type:** `n8n-nodes-base.microsoftSharePoint` (v1). **Role:** Creates a new list item on a SharePoint site with all form fields. **Configuration:** Resource = `item`, Operation = `create`. The Site dropdown is left unconfigured ("-- Select your SharePoint site --") and must be set by the user. The List and column mappings must be configured inside the node's `columns` parameter to map: `Name`, `Email`, `Phone`, `Message`, `Date`, `Time` to the corresponding SharePoint list columns. **Credentials:** Requires a `microsoftSharePointOAuth2Api` credential (Microsoft SharePoint OAuth2). **Connections:** Input ← `Webhook`; Output → `Build Branded Email HTML`. **Edge cases:** If the credential is missing, expired, or lacks write permissions on the target list, the node will fail with an authentication or permissions error. If the list schema does not match the mapped columns (e.g., column name mismatch or data type mismatch such as date format), item creation will fail. If the site/list selection is left at the placeholder, execution will produce a configuration error. Timeout issues may arise if the SharePoint API is slow to respond. |

---

#### Block 3 — Email Construction (Branded HTML)

**Overview:** The Code node takes the raw form data and constructs a fully branded HTML email body. Branding constants (logo URL, brand name, brand website URL, three hex colors) are defined at the top of the script for easy customization. The node outputs the HTML string plus the individual fields for downstream use.

| Node | Details |
|---|---|
| **Build Branded Email HTML** | **Type:** `n8n-nodes-base.code` (v2). **Role:** Generates a styled HTML email body from form data. **Configuration:** JavaScript code runs in the n8n sandbox. The script reads the Webhook's first item: `$('Webhook').first().json.body` and extracts `Name`, `Email`, `Phone`, `Message`, `Date`, `Time`. Missing fields default to `"(not provided)"`. Date and Time are concatenated as `"Date at Time"`. **Customizable constants:** `LOGO_URL` (default `'https://yourwebsite.com/path/to/your-logo.png'`), `BRAND_NAME` (default `'Your Company Name'`), `BRAND_WEBSITE` (default `'yourwebsite.com'`), `COLOR_HEADER` (default `'#004080'`), `COLOR_ACCENT` (default `'#ffc107'`), `COLOR_BUTTON` (default `'#008080'`). The generated HTML includes: a header banner with logo, an accent stripe, a data table with each form field, a "Reply to {name}" CTA button (mailto link), and a branded footer. **Output:** Returns `{ html, name, email, phone, date, message }`. **Connections:** Input ← `Log Submission to SharePoint`; Output → `Send Email Notification`. **Edge cases:** If the Webhook node's body is null or undefined (e.g., malformed POST), accessing `body.Name` etc. will result in "(not provided)" fallbacks. If `LOGO_URL` is unreachable, email clients will display a broken image. Some email clients strip `<style>` blocks; the HTML uses inline styles to mitigate this. XSS risk: user-supplied values (`name`, `email`, `phone`, `message`) are inserted unescaped; if the message contains raw HTML, it will be rendered. Consider sanitizing input if the form is public. |

---

#### Block 4 — Email Delivery (Gmail)

**Overview:** The Gmail node sends the branded HTML email to a fixed recipient, using the HTML body generated in the previous block and setting the Reply-To header to the submitter's email.

| Node | Details |
|---|---|
| **Send Email Notification** | **Type:** `n8n-nodes-base.gmail` (v2.2). **Role:** Dispatches a notification email for each form submission. **Configuration:** **Send To:** `user@example.com` (must be changed to the real recipient). **Subject:** Dynamic expression `New Website Inquiry from {{ $json.name }}`. **Message:** Dynamic expression `{{ $json.html }}` (the HTML body from the Code node). **Options:** `Reply-To` is set to `{{ $json.email }}` (the submitter's email). `Append Attribution` is disabled (set to false), so no n8n signature is appended. **Credentials:** Requires a Gmail OAuth2 credential (`gmailOAuth2`). **Connections:** Input ← `Build Branded Email HTML`; no downstream connections (terminal node). **Edge cases:** If the Gmail credential is missing, revoked, or lacks sending permissions, the node will fail. If the `sendTo` address is invalid or the inbox is full, delivery will bounce. Gmail imposes sending limits (typically 500/day for standard accounts); high-volume sites may hit rate limits. The Reply-To header only works if the recipient's email client respects it. The `appendAttribution: false` prevents any n8n branding from being injected. |

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Overview | n8n-nodes-base.stickyNote | Workflow description and setup instructions | — | — | ## Website Contact Form Processor\n\n### How it works\nThis workflow fires whenever a visitor submits a contact form on your website.\n\n1. **Webhook** receives the POST payload (Name, Email, Phone, Message, Date, Time) from your website form.\n2. **Log Submission to SharePoint** writes each field into a SharePoint list for permanent record-keeping.\n3. **Build Branded Email HTML** — Code node constructs a fully branded HTML notification email (customize colors and logo to match your brand).\n4. **Send Email Notification** delivers the notification via Gmail to your designated recipient address, with Reply-To automatically set to the submitter's email so you can respond in one click.\n\n### Setup\n1. Connect your **Microsoft SharePoint** credential and select your target site and list.\n2. Connect your **Gmail OAuth2** credential.\n3. In the **Send Email Notification** node, enter the recipient email address in the `Send To` field.\n4. In the **Build Branded Email HTML** node, replace the placeholder logo URL and brand name with your own.\n5. Point your website form's action URL to the webhook path: `form-processor`.\n\n### Customization\n- To CC or BCC additional staff, add those addresses in the Gmail node Options.\n- Adjust brand colors in the Code node (search for the hex color values).\n- To add or remove SharePoint columns, update both the `columns` mapping and the list schema in the SharePoint node. |
| Webhook | n8n-nodes-base.webhook (v2.1) | Receives POST form submission | — (entry point) | Log Submission to SharePoint | (covered by Overview sticky note — see above) |
| Log Submission to SharePoint | n8n-nodes-base.microsoftSharePoint (v1) | Creates a new SharePoint list item with form data | Webhook | Build Branded Email HTML | 📋 Log to SharePoint — Writes all form fields to the **SharePoint** list for record-keeping. |
| Build Branded Email HTML | n8n-nodes-base.code (v2) | Generates styled HTML email body from form data | Log Submission to SharePoint | Send Email Notification | ✉️ Notify Recipient — Builds branded HTML email and delivers via Gmail. Reply-To is set to the submitter's address. |
| Send Email Notification | n8n-nodes-base.gmail (v2.2) | Sends branded notification email via Gmail | Build Branded Email HTML | — (terminal) | ✉️ Notify Recipient — Builds branded HTML email and delivers via Gmail. Reply-To is set to the submitter's address. |
| Section - SharePoint | n8n-nodes-base.stickyNote | Visual section label | — | — | 📋 Log to SharePoint — Writes all form fields to the **SharePoint** list for record-keeping. |
| Section - Email | n8n-nodes-base.stickyNote | Visual section label | — | — | ✉️ Notify Recipient — Builds branded HTML email and delivers via Gmail. Reply-To is set to the submitter's address. |
| Sticky Note | n8n-nodes-base.stickyNote | Customization reminder for the Code node | — | — | ## Update this part of the Build Branded Email HTML node with your information\n\nconst LOGO_URL      = 'https://yourwebsite.com/path/to/your-logo.png';\nconst BRAND_NAME    = 'Your Company Name';\nconst BRAND_WEBSITE = 'yourwebsite.com';\nconst COLOR_HEADER  = '#004080';   // Primary / header background\nconst COLOR_ACCENT  = '#ffc107';   // Accent stripe\nconst COLOR_BUTTON  = '#008080';   // CTA button |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to rebuild the workflow in a new n8n instance:

1. **Create a new workflow** and name it `Website Contact Form Processor`.

2. **Add the Overview sticky note:**
   - Type: `Sticky Note`
   - Position: top-left of the canvas (approximately x=0, y=0)
   - Size: Width 440, Height 768
   - Content: Paste the full overview text (How it works, Setup, Customization) as shown in the workflow JSON.

3. **Add the Webhook node:**
   - Type: `Webhook`
   - HTTP Method: `POST`
   - Path: `form-processor`
   - Options: leave default (no authentication)
   - After saving, copy the **Webhook URL** — this is the endpoint your website form will POST to.

4. **Add the SharePoint section sticky note:**
   - Type: `Sticky Note`
   - Position: x=496, y=32
   - Size: Width 332, Height 300
   - Color: 7 (blue)
   - Content: `## 📋 Log to SharePoint\nWrites all form fields to the **SharePoint** list for record-keeping.`

5. **Add the "Log Submission to SharePoint" node:**
   - Type: `Microsoft SharePoint`
   - Resource: `Item`
   - Operation: `Create`
   - Site: Select your SharePoint site from the dropdown (you must connect a credential first).
   - List: Select the target list.
   - Columns: Map each form field to the corresponding SharePoint column:
     - `Name` → SharePoint column (e.g., `Title` or `Name`)
     - `Email` → SharePoint column `Email`
     - `Phone` → SharePoint column `Phone`
     - `Message` → SharePoint column `Message`
     - `Date` → SharePoint column `Date`
     - `Time` → SharePoint column `Time`
   - Credential: Create or select a `Microsoft SharePoint OAuth2 API` credential and authorize it.
   - Connect: **Webhook → Log Submission to SharePoint** (Webhook output → SharePoint input).

6. **Add the Email section sticky note:**
   - Type: `Sticky Note`
   - Position: x=832, y=32
   - Size: Width 340, Height 300
   - Color: 7 (blue)
   - Content: `## ✉️ Notify Recipient\nBuilds branded HTML email and delivers via Gmail.\nReply-To is set to the submitter's address.`

7. **Add the "Build Branded Email HTML" node:**
   - Type: `Code`
   - Language: JavaScript
   - Paste the full JavaScript code from the workflow (see the `jsCode` parameter).
   - **Customize the constants at the top of the script:**
     - `LOGO_URL`: Replace `'https://yourwebsite.com/path/to/your-logo.png'` with your actual hosted logo URL.
     - `BRAND_NAME`: Replace `'Your Company Name'` with your company name.
     - `BRAND_WEBSITE`: Replace `'yourwebsite.com'` with your website domain.
     - `COLOR_HEADER`: Change `'#004080'` to your primary brand color if desired.
     - `COLOR_ACCENT`: Change `'#ffc107'` to your accent color if desired.
     - `COLOR_BUTTON`: Change `'#008080'` to your CTA button color if desired.
   - Connect: **Log Submission to SharePoint → Build Branded Email HTML**.

8. **Add the "Send Email Notification" node:**
   - Type: `Gmail`
   - Send To: Enter the recipient email address (replace `user@example.com` with your actual notification recipient).
   - Subject: Use expression `=New Website Inquiry from {{ $json.name }}`
   - Message (HTML): Use expression `={{ $json.html }}`
   - Options → Reply-To: `={{ $json.email }}`
   - Options → Append Attribution: `false` (uncheck / disabled)
   - If you need CC or BCC recipients, add them under Options → CC / BCC fields.
   - Credential: Create or select a `Gmail OAuth2` credential and authorize it.
   - Connect: **Build Branded Email HTML → Send Email Notification**.

9. **Add the customization reminder sticky note:**
   - Type: `Sticky Note`
   - Position: x=720, y=368
   - Size: Width 512, Height 304
   - Color: 5 (yellow)
   - Content: `## Update this part of the Build Branded Email HTML node with your information\n\nconst LOGO_URL      = 'https://yourwebsite.com/path/to/your-logo.png';\nconst BRAND_NAME    = 'Your Company Name';\nconst BRAND_WEBSITE = 'yourwebsite.com';\nconst COLOR_HEADER  = '#004080';   // Primary / header background\nconst COLOR_ACCENT  = '#ffc107';   // Accent stripe\nconst COLOR_BUTTON  = '#008080';   // CTA button`

10. **Configure your SharePoint list:**
    - In SharePoint, create a list (or use an existing one) with columns matching the mapped fields: Name (Text), Email (Text), Phone (Text), Message (Multiple lines of text), Date (Date), Time (Text or Date/Time).
    - Ensure the SharePoint OAuth2 credential has Write access to this list.

11. **Configure your website form:**
    - Set the form's `action` attribute to the Webhook URL (e.g., `https://your-n8n-instance.com/webhook/form-processor`).
    - Ensure the form sends a POST request with fields: `Name`, `Email`, `Phone`, `Message`, `Date`, `Time`.

12. **Activate the workflow:**
    - Click the toggle to activate the workflow. The Webhook node will now listen for POST requests.

13. **Test the workflow:**
    - Submit a test entry from your website form or use a tool like `curl` or Postman to POST sample data:
      ```
      curl -X POST https://your-n8n-instance.com/webhook/form-processor \
        -H "Content-Type: application/json" \
        -d '{"Name":"Test User","Email":"test@example.com","Phone":"555-0100","Message":"Hello","Date":"2024-01-15","Time":"14:30"}'
      ```
    - Verify the SharePoint list item is created.
    - Verify the email arrives at the designated recipient with correct branding and Reply-To header.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The SharePoint node's Site and List selectors are intentionally left blank in the template. You must select your own site and list before the workflow will run. | Setup step — SharePoint configuration |
| The Gmail node's `Send To` address defaults to `user@example.com`. Replace it with your actual notification inbox. | Setup step — Gmail configuration |
| The Code node inserts user-supplied values directly into HTML without sanitization. For public-facing forms, consider adding HTML-escaping logic (e.g., replace `<`, `>`, `&` with entities) to prevent XSS in email clients that render HTML. | Security consideration |
| Gmail standard accounts have a sending limit of approximately 500 emails per day. If you expect higher volume, consider a Google Workspace account or an alternative email service. | Rate limit awareness |
| The Webhook endpoint has no built-in authentication. If your form is public, anyone who discovers the URL can submit data. Consider adding n8n's Basic Auth or Header Auth option on the Webhook node for protection. | Security consideration |
| To add CC or BCC recipients, open the Gmail node's Options section and populate the CC / BCC fields with comma-separated email addresses. | Customization tip |
| The HTML email uses inline CSS to maximize compatibility across email clients. Avoid adding a `<style>` block, as many clients strip it. | Email rendering best practice |
| The `LOGO_URL` must point to a publicly accessible image. If the URL is behind authentication or returns a redirect, many email clients will fail to render it. | Branding tip |
| If you need to capture additional form fields, add them to the Webhook payload, update the SharePoint column mapping, and extend the Code node's HTML template with new rows. | Extensibility note |