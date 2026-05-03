Generate PDF pricing proposals from Excel with Gotenberg and Outlook

https://n8nworkflows.xyz/workflows/generate-pdf-pricing-proposals-from-excel-with-gotenberg-and-outlook-14982


# Generate PDF pricing proposals from Excel with Gotenberg and Outlook

We need to analyze an n8n workflow JSON for "Generate PDF pricing proposals from Excel with Gotenberg and Outlook". The workflow is described with a title and description. Let's parse.

The JSON includes nodes:

- Sticky Notes:
  - "Overview" (big overview)
  - "Section – Intake"
  - "Section – Data"
  - "Section – Processing"
  - "Section – Build"
  - "Section – PDF"
  - "Section – Email"

- Active nodes:
  - "Pricing Request Webhook" (webhook)
  - "Fetch Pricing Sheet" (Microsoft Excel node)
  - "Process & Price" (Code node)
  - "Build HTML Proposal" (Code node)
  - "Gotenberg → PDF" (HTTP Request node)
  - "Send a message" (Microsoft Outlook node)

Connections:
- Pricing Request Webhook → Fetch Pricing Sheet
- Fetch Pricing Sheet → Process & Price
- Process & Price → Build HTML Proposal
- Build HTML Proposal → Gotenberg → PDF
- Gotenberg → PDF → Send a message

The output is a PDF email.

We need to produce sections: 1. Workflow Overview, 2. Block-by-Block Analysis, 3. Summary Table, 4. Reproducing the Workflow from Scratch, 5. General Notes & Resources.

The sticky notes have a lot of content. Must incorporate into summary table sticky note column.

Need to consider any sub-workflow references: none.

Also note the node "Send a message" is actually Outlook, not Gmail, but description says Gmail OAuth2, but the node is Microsoft Outlook (type n8n-nodes-base.microsoftOutlook). So note discrepancy.

Now produce a comprehensive document.

First, workflow overview: talk about purpose: generate PDF pricing proposals from Excel data via webhook intake, fetch pricing sheet, process, build HTML, convert via Gotenberg, email via Outlook. Provide logical blocks: Intake, Data Fetch, Processing, HTML Build, PDF Conversion, Email Delivery.

Then block-by-block analysis: for each block, overview, nodes involved, node details.

Block 1: Intake. Node: Pricing Request Webhook. Provide details: HTTP method POST, path pricing-request, receives JSON body containing fields: prospectName, date, services (array or string), discountLevel (Discount/Retail/Premium), requestorName (format "Full Name (user@example.com)"). Output: pass to next node.

Edge cases: webhook authentication, missing fields, service array not provided, discount level invalid.

Block 2: Data. Node: Fetch Pricing Sheet (Microsoft Excel node). Provide details: resource=table, operation=getRows, returnAll=true, select workbook pricing_request.xlsx, worksheet Sheet1, table Table1. Credential: Microsoft Excel OAuth2. Output: array of rows with columns Title, Description, Cost, Discount, Retail, Premium, Unit.

Edge cases: credentials, workbook not found, table not found, column mismatch.

Block 3: Processing. Node: Process & Price (Code node). Code extracts form data, processes services selection, applies discount level to pricing items, calculates total. Output JSON includes prospectName, date, requestorName, requestorEmail, discountLevel, selectedServices, pricingItems array, total.

Edge cases: services not in Excel (filter will drop them), discountLevel unknown (fallback to Retail), parsing requestorName incorrectly.

Block 4: Build HTML. Node: Build HTML Proposal (Code node). Takes input JSON, builds HTML string with branding variables (logo URL, company name, etc.), calculates display date and valid until date (30 days). Produces binary data for HTML (index.html) and JSON with htmlContent, fileName, requestorEmail, requestorName, prospectName.

Edge cases: logo placeholder not replaced, date formatting issues, missing fields cause undefined.

Block 5: PDF Conversion. Node: Gotenberg → PDF (HTTP Request). POST multipart form-data with binary field "index.html" referencing data field (binary). Sends to endpoint https://YOUR_GOTENBERG_INSTANCE/forms/chromium/convert/html. Sends header Gotenberg-Output-Filename from fileName. Auth: generic Basic Auth. Response format set to file. Returns binary PDF.

Edge cases: Gotenberg instance unreachable, auth failure, incorrect binary field name, wrong content type.

Block 6: Email Delivery. Node: Send a message (Microsoft Outlook). Uses Outlook OAuth2 credential. Subject line dynamic referencing prospectName. Body referencing prospectName and requestorName. Recipients: YOUR_INTERNAL_EMAIL + requestorEmail (semi-colon separated). Attachment uses binary field "data" (the PDF). Edge cases: missing credentials, Outlook OAuth2 token expiration, missing recipient emails, attachment missing.

Note that there's discrepancy with description about Gmail vs Outlook.

Now summary table with columns: Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note.

We have sticky notes per node? There are section sticky notes covering multiple nodes. For each node, we must duplicate relevant sticky note content. Node mapping:

- Pricing Request Webhook is in Section – Intake.
- Fetch Pricing Sheet is in Section – Data.
- Process & Price is in Section – Processing.
- Build HTML Proposal is in Section – Build.
- Gotenberg → PDF is in Section – PDF.
- Send a message is in Section – Email.

Sticky note content per section:

Section – Intake: "## 1 · Intake\nReceives the form POST from your web form."

Section – Data: "## 2 · Data\nPulls all rows from the pricing Excel table in OneDrive."

Section – Processing: "## 3 · Processing\nFilters services, applies price tier (Discount / Retail / Premium), and calculates total."

Section – Build: "## 4 · Proposal Build\nRenders the branded HTML document for PDF conversion.\n\nUpdate `YOUR_LOGO_URL` and the brand color variables to match your identity."

Section – PDF: "## 5 · PDF Conversion\nSelf-hosted [Gotenberg](https://gotenberg.dev) converts HTML → PDF.\n\nUpdate the URL to your Gotenberg instance and attach a **Basic Auth** credential."

Section – Email: "## 6 · Email Delivery\nSends the PDF to the requester via Gmail OAuth2.\n\nReplace `YOUR_INTERNAL_EMAIL` in the **Email Proposal** node with your own address."

We also have overview sticky note with bigger content. Should we include that in each node? The overview note is separate, maybe we can put it for the first node or for workflow overview only. The table likely includes each node's sticky note. The overview note is a generic one; it is not directly covering any specific node but overall. Probably we include overview content as sticky note for all nodes? The instructions: "If a sticky note visually covers multiple nodes, duplicate its content in each affected row." The Overview sticky note visually covers the entire workflow (all nodes). So we could duplicate its content for each node row. That would be a large content. But maybe we can note as "See Overview note". Actually instruction: "duplicate its content in each affected row". So for each node, we should include the Overview content. But that might be lengthy. Could summarise. Let's include the Overview content for each node, but maybe only key sections. Provide full content for each node.

We also need to ensure we do not include raw JSON. Provide interpreted configuration.

Now for Reproducing the Workflow from Scratch: Step-by-step.

1. Create a new workflow named "Pricing Proposal Generator".
2. Add a Webhook node.
   - Name: "Pricing Request Webhook"
   - HTTP Method: POST
   - Path: "pricing-request"
   - Respond: default
3. Add Microsoft Excel node.
   - Name: "Fetch Pricing Sheet"
   - Resource: Table
   - Operation: Get Rows
   - Return All: true
   - Workbook: pricing_request.xlsx
   - Worksheet: Sheet1
   - Table: Table1
   - Credential: Microsoft Excel OAuth2
4. Add Code node "Process & Price". Insert the provided JavaScript code. It references $('Pricing Request Webhook') and $('Fetch Pricing Sheet').
   - In the code, handle extraction of form fields.
5. Add Code node "Build HTML Proposal". Insert provided HTML builder code. Replace placeholders.
6. Add HTTP Request node "Gotenberg → PDF".
   - Method: POST
   - URL: https://YOUR_GOTENBERG_INSTANCE/forms/chromium/convert/html (replace with your actual Gotenberg instance)
   - Authentication: Generic Credential Type -> Basic Auth, create a credential for Gotenberg.
   - Send Body: true
   - Content Type: multipart/form-data
   - Body Parameters: one field:
     - Name: "index.html"
     - Parameter Type: formBinaryData
     - Input Data Field Name: =data (binary data name from Build HTML Proposal)
   - Header Parameters: add "Gotenberg-Output-Filename" with expression: ={{ $('Build HTML Proposal').item.json.fileName }}
   - Options > Response > Response Format: File
7. Add Microsoft Outlook node "Send a message".
   - Operation: Send
   - Subject: =Pricing Proposal prepared for {{ $('Build HTML Proposal').item.json.prospectName }}
   - Body Content: =Please find the attached pricing proposal for {{ $('Build HTML Proposal').item.json.prospectName }}, requested by {{ $('Build HTML Proposal').item.json.requestorName }}. The attached pricing is valid for 30 days.
   - To Recipients: =YOUR_INTERNAL_EMAIL;{{ $('Build HTML Proposal').item.json.requestorEmail }}
   - Attachments: add binary property name: "data" (the PDF from Gotenberg)
   - Credential: Microsoft Outlook OAuth2 (e.g., Training Outlook)
8. Connect nodes in order:
   - Pricing Request Webhook → Fetch Pricing Sheet
   - Fetch Pricing Sheet → Process & Price
   - Process & Price → Build HTML Proposal
   - Build HTML Proposal → Gotenberg → PDF
   - Gotenberg → PDF → Send a message

9. Replace placeholders:
   - In Build HTML Proposal: replace YOUR_LOGO_URL, companyName, tagline, brand color variables.
   - In Gotenberg → PDF: update URL and set Basic Auth credential.
   - In Send a message: replace YOUR_INTERNAL_EMAIL with your internal email address.

10. Create and upload the Excel file "pricing_request.xlsx" to OneDrive with table "Table1" on "Sheet1" containing columns: Title, Description, Cost, Discount, Retail, Premium, Unit.

11. Activate the workflow.

Potential edge cases: webhook path must be unique, Outlook OAuth2 may need admin consent, Gotenberg instance must be accessible.

Now General Notes & Resources: Include note about Gotenberg self-hosted, Excel template download link: https://iportgpt.com/n8n_assets/pricing_request.xlsx . Provide link to Gotenberg website: https://gotenberg.dev . Provide note about Outlook vs Gmail description mismatch. Provide branding update note.

Also note: The "Overview" sticky note includes a download template link: https://iportgpt.com/n8n_assets/pricing_request.xlsx . Should be included.

Also mention that the default validity period is 30 days, can be changed in Build HTML Proposal (the 30 multiplier and email body).

Potential issues: Excel credentials may require read permission, Gotenberg Basic Auth must be set up.

Now let's think about each node's details in block-by-block analysis.

Will produce each block with overview, nodes involved, and node details.

Will also mention version numbers (typeVersion) as relevant.

Block 1: Intake.

- Pricing Request Webhook:
  - Node type: n8n-nodes-base.webhook
  - HTTP method: POST
  - Path: "pricing-request"
  - Options: default (none)
  - Version: 2
  - Input: none (trigger)
  - Output: 1 item with json containing body (incoming request), includes fields.
  - Potential errors: no payload, invalid JSON, missing fields, authentication if required.
  - No sticky note specifically for this node but Section – Intake sticky note covers it.

Block 2: Data Fetch.

- Fetch Pricing Sheet:
  - Type: n8n-nodes-base.microsoftExcel
  - Version: 2.2
  - Resource: table
  - Operation: getRows
  - Return all rows: true
  - Workbook: pricing_request.xlsx (list mode)
  - Worksheet: Sheet1
  - Table: Table1
  - Credential: Microsoft Excel OAuth2 (credential-id)
  - Output: array of items with columns as per Excel.

Edge Cases: Could be empty table, missing columns, credentials.

Block 3: Processing.

- Process & Price:
  - Type: n8n-nodes-base.code (JS)
  - Version: 2
  - Code extracts body from webhook, gets all rows from Fetch Pricing Sheet.
  - Handles services as array.
  - Extracts requestor email via regex.
  - Filters rows based on selected services matching Title.
  - Applies discount level mapping: Discount column, Premium column, else Retail.
  - Calculates total sum.
  - Output: JSON object with fields: prospectName, date, requestorName, requestorEmail, discountLevel, selectedServices, pricingItems (array), total.

Edge Cases: No matching services yields empty pricingItems and total 0, missing columns.

Block 4: Build HTML Proposal.

- Build HTML Proposal:
  - Type: n8n-nodes-base.code
  - Version: 2
  - Code uses branding variables.
  - Calculates display date and valid until date (30 days later).
  - Builds HTML string with CSS.
  - Converts HTML to Buffer and sets binary property "data" with base64.
  - Also returns JSON with htmlContent, fileName (generated from prospect name + date), requestorEmail, requestorName, prospectName.

Edge Cases: If prospectName includes special characters, regex replace replaces non-alphanumeric with hyphens.

Block 5: PDF conversion.

- Gotenberg → PDF:
  - Type: n8n-nodes-base.httpRequest
  - Version: 4.4
  - Method: POST
  - URL: https://YOUR_GOTENBERG_INSTANCE/forms/chromium/convert/html (replace placeholder)
  - Authentication: genericCredentialType (httpBasicAuth)
  - Headers: Gotenberg-Output-Filename: {{ $('Build HTML Proposal').item.json.fileName }}
  - Body: multipart/form-data with one field: "index.html" referencing binary field name =data (from Build HTML Proposal)
  - Options > Response > responseFormat: file (expects binary PDF)
  - Output: binary data (PDF) presumably stored under default "data" property.

Edge Cases: If Gotenberg URL unreachable or auth fails, HTTP error. If no binary returned, downstream email may fail.

Block 6: Email Delivery.

- Send a message:
  - Type: n8n-nodes-base.microsoftOutlook
  - Version: 2
  - Operation: send message (presumably)
  - Subject: dynamic expression referencing prospectName
  - Body: dynamic expression referencing prospectName and requestorName
  - To: YOUR_INTERNAL_EMAIL; {{ $('Build HTML Proposal').item.json.requestorEmail }}
  - Additional fields: attachments using binary property name "data" (the PDF)
  - Credential: MicrosoftOutlookOAuth2Api (Training Outlook)
  - Output: confirmation or error.

Edge Cases: Recipient format may need commas or semicolons; Outlook node expects semicolons; missing requestorEmail leads to missing recipient; attachment may be missing if no PDF.

Now the summary table.

Will list nodes:

1. Overview (sticky note) - not a processing node. Not needed? It is sticky note, not a node that processes data. Should we include sticky notes in table? Probably not; the table is for workflow nodes (not sticky notes). The instruction says "List all workflow nodes in a single table". Sticky note nodes are also nodes. But they are non-processing, so we might include them. However, we need to include all nodes, including sticky notes. Include each sticky note as node name "Overview", type stickyNote, functional role: Documentation comment. Input node(s) blank, output blank. Sticky Note content = its content.

Better to include all nodes for completeness, but the table may be huge. Probably we include only processing nodes (the functional ones). However the instruction says "all workflow nodes". So include all. We'll include sticky notes with their content in Sticky Note column (maybe same content). For each sticky note, the Sticky Note column should show the content of that sticky note. Since sticky note already is content, we can duplicate.

But to avoid duplication, we can note that the sticky note "Overview" covers entire workflow and include its content. But for the row for "Overview", we include its content. Then for other rows, if sticky note covers them, we duplicate. However we also have individual section sticky notes for each block; they will be associated to the processing nodes within that block.

Thus:

- Node Name: Pricing Request Webhook
  - Node Type: n8n-nodes-base.webhook
  - Functional Role: Receives inbound POST request with form data
  - Input Node(s): None (trigger)
  - Output Node(s): Fetch Pricing Sheet
  - Sticky Note: "## 1 · Intake\nReceives the form POST from your web form."

- Node Name: Fetch Pricing Sheet
  - Node Type: n8n-nodes-base.microsoftExcel
  - Functional Role: Retrieves all rows from Excel pricing table
  - Input Node(s): Pricing Request Webhook
  - Output Node(s): Process & Price
  - Sticky Note: "## 2 · Data\nPulls all rows from the pricing Excel table in OneDrive."

- Node Name: Process & Price
  - Node Type: n8n-nodes-base.code
  - Functional Role: Filters rows by selected services, selects price tier, computes total
  - Input Node(s): Fetch Pricing Sheet
  - Output Node(s): Build HTML Proposal
  - Sticky Note: "## 3 · Processing\nFilters services, applies price tier (Discount / Retail / Premium), and calculates total."

- Node Name: Build HTML Proposal
  - Node Type: n8n-nodes-base.code
  - Functional Role: Generates branded HTML document for proposal
  - Input Node(s): Process & Price
  - Output Node(s): Gotenberg → PDF
  - Sticky Note: "## 4 · Proposal Build\nRenders the branded HTML document for PDF conversion.\n\nUpdate `YOUR_LOGO_URL` and the brand color variables to match your identity."

- Node Name: Gotenberg → PDF
  - Node Type: n8n-nodes-base.httpRequest
  - Functional Role: Converts HTML to PDF via Gotenberg
  - Input Node(s): Build HTML Proposal
  - Output Node(s): Send a message
  - Sticky Note: "## 5 · PDF Conversion\nSelf-hosted [Gotenberg](https://gotenberg.dev) converts HTML → PDF.\n\nUpdate the URL to your Gotenberg instance and attach a **Basic Auth** credential."

- Node Name: Send a message
  - Node Type: n8n-nodes-base.microsoftOutlook
  - Functional Role: Emails the PDF proposal
  - Input Node(s): Gotenberg → PDF
  - Output Node(s): None
  - Sticky Note: "## 6 · Email Delivery\nSends the PDF to the requester via Gmail OAuth2.\n\nReplace `YOUR_INTERNAL_EMAIL` in the **Email Proposal** node with your own address."

- Node Name: Overview (sticky)
  - Node Type: n8n-nodes-base.stickyNote
  - Functional Role: Documentation
  - Input Node(s): None
  - Output Node(s): None
  - Sticky Note: Content of Overview note (the full description, setup, customization etc). Should include the full note. Since it's lengthy, we will copy it. Use markdown.

- Node Name: Section – Intake
- Node Name: Section – Data
- Node Name: Section – Processing
- Node Name: Section – Build
- Node Name: Section – PDF
- Node Name: Section – Email

We should include each sticky note node row. The Sticky Note column for each sticky note can be the content it displays.

Potentially large table. Use concise content or include as is.

Better to include each sticky note node row. We'll include them.

Now Reproducing the Workflow from Scratch: Provide step-by-step instructions.

Will detail each node creation, configuration, credentials, connections.

Now General Notes & Resources: Provide note about Excel template link, Gotenberg documentation, branding variable placeholders, Outlook vs Gmail discrepancy, 30-day validity.

Potentially also include note about the default validity period and how to change.

Let's produce final document.

Make sure to not use prohibited words (tutorial, cheatsheet, README). Use professional English.

Will format sections with headings and bullet points.

Will ensure sticky note content includes links, and format them as links.

Will include all relevant details.

Now begin writing. 1. Workflow Overview  

The **Pricing Proposal Generator** workflow turns a simple web‑form submission into a professional, branded PDF pricing proposal and delivers it by email. It follows a linear pipeline:

| # | Logical Block | Purpose |
|---|----------------|---------|
| 1 | **Intake** | A Webhook receives a POST request containing prospect details, selected services, discount level, and requestor information. |
| 2 | **Data Fetch** | A Microsoft Excel node reads every row from a OneDrive/SharePoint pricing table (`pricing_request.xlsx`). |
| 3 | **Processing** | A Code node filters the Excel rows to the selected services, selects the appropriate price tier (Discount / Retail / Premium) per the requestor’s discount level, and computes the total. |
| 4 | **Proposal Build** | A second Code node renders a full‑page HTML document containing cover information, line‑item pricing, legal terms and a signature block. Branding placeholders (logo URL, colours, company name) are defined at the top of the script. |
| 5 | **PDF Conversion** | An HTTP Request node sends the HTML to a self‑hosted Gotenberg instance, which returns a print‑ready PDF. |
| 6 | **Email Delivery** | An Outlook (Microsoft 365) node emails the PDF to the requestor and an internal address, with a dynamic subject and body. |

The workflow is fully self‑contained; the only external dependencies are:  

* A Microsoft Excel OAuth2 credential for OneDrive/SharePoint.  
* A running Gotenberg service with Basic Auth.  
* An Outlook OAuth2 credential (the description mentions Gmail, but the node uses Outlook).  

---

## 2. Block‑by‑Block Analysis  

### Block 1 – Intake  

**Overview**  
A webhook endpoint listens for an HTTP POST containing the form data needed to generate a proposal. This is the sole entry point of the workflow.

| Node | Type & Technical Role | Key Configuration | Expressions / Variables | Input → Output | Edge Cases / Failure Types |
|------|-----------------------|-------------------|--------------------------|----------------|---------------------------|
| Pricing Request Webhook | `n8n‑nodes‑base.webhook` (v2) – Trigger | • HTTP Method: **POST** <br>• Path: **pricing‑request** <br>• Options: default | Access inbound data via `$('Pricing Request Webhook').item.json.body` | No input (trigger) → Item with `body` containing `prospectName`, `date`, `services` (array or string), `discountLevel`, `requestorName` | • No payload or malformed JSON → node fails. <br>• Missing required fields → downstream Code node receives `undefined`. <br>• Path conflict with another webhook. |

---

### Block 2 – Data Fetch  

**Overview**  
Retrieves the complete pricing catalogue from a designated Excel table stored in OneDrive/SharePoint. The full dataset is later filtered by the Code node.

| Node | Type & Technical Role | Key Configuration | Expressions / Variables | Input → Output | Edge Cases / Failure Types |
|------|-----------------------|-------------------|--------------------------|----------------|---------------------------|
| Fetch Pricing Sheet | `n8n‑nodes‑base.microsoftExcel` (v2.2) – Data source | • Resource: **Table** <br>• Operation: **Get Rows** <br>• Return All: **true** <br>• Workbook: **pricing_request.xlsx** (list mode) <br>• Worksheet: **Sheet1** <br>• Table: **Table1** <br>• Credential: **Microsoft Excel OAuth2** (named “Microsoft Excel account”) | Output columns expected: `Title`, `Description`, `Cost`, `Discount`, `Retail`, `Premium`, `Unit` | Receives item from Webhook (ignored) → Array of rows, each row as an item with `json` containing the columns above | • Missing or mis‑named columns → downstream Code node will read `undefined`. <br>• Credential not authorized or workbook not found → node error. <br>• Empty table → later filtering yields no items. |

---

### Block 3 – Processing & Pricing  

**Overview**  
A JavaScript Code node extracts the form data, matches selected services against the Excel rows, selects the price tier, and calculates the total amount.

| Node | Type & Technical Role | Key Configuration | Expressions / Variables | Input → Output | Edge Cases / Failure Types |
|------|-----------------------|-------------------|--------------------------|----------------|---------------------------|
| Process & Price | `n8n‑nodes‑base.code` (v2) – Business logic | **JS Code** (full script below) | • Form data: `$('Pricing Request Webhook').item.json.body` <br>• Pricing rows: `$('Fetch Pricing Sheet').all()` <br>• Extracts `prospectName`, `date`, `services`, `discountLevel`, `requestorName` <br>• Parses requestor email via regex `\(([^)]+)\)` <br>• Filters rows where `item.json.Title` is in `selectedServices` <br>• `switch` on `discountLevel`: **Discount** → `data.Discount`; **Premium** → `data.Premium`; else → `data.Retail` <br>• Calculates `total` as sum of selected prices | Input: Items from “Fetch Pricing Sheet” (and the webhook item) → Output: Single item with JSON `{ prospectName, date, requestorName, requestorEmail, discountLevel, selectedServices, pricingItems, total }` | • No matching service titles → `pricingItems` empty, `total` = 0. <br>• `discountLevel` value not recognised → falls back to Retail (default). <br>• Regex fails to extract email → `requestorEmail` becomes empty string. <br>• Excel rows lack expected columns → runtime error. |

**JS Code (Process & Price)**  
```javascript
// Extract form data
const formData = $('Pricing Request Webhook').item.json.body;
const prospectName   = formData.prospectName;
const date           = formData.date;
const selectedServices = Array.isArray(formData.services)
                       ? formData.services
                       : [formData.services];
const discountLevel  = formData.discountLevel; // 'Discount' | 'Retail' | 'Premium'
const requestorInfo  = formData.requestorName; // "Jane Smith (user@example.com)"

// Extract email from requestor info
const emailMatch    = requestorInfo.match(/\(([^)]+)\)/);
const requestorEmail = emailMatch ? emailMatch[1] : '';
const requestorName  = requestorInfo.replace(/\s*\([^)]*\)/, '').trim();

// Get pricing data from Excel node
const pricingData = $('Fetch Pricing Sheet').all();

// Filter rows for selected services (match on Title)
const filteredPricing = pricingData.filter(item =>
  selectedServices.includes(item.json.Title)
);

// Map pricing based on discount level
const processedPricing = filteredPricing.map(item => {
  const data = item.json;
  let price;
  switch (discountLevel) {
    case 'Discount': price = data.Discount; break;
    case 'Premium':  price = data.Premium;  break;
    default:        price = data.Retail;
  }
  return {
    service:     data.Title,
    description: data.Description,
    cost:        data.Cost,
    price:       price,
    unit:        data.Unit
  };
});

// Calculate total
const total = processedPricing.reduce(
  (sum, item) => sum + (parseFloat(item.price) || 0),
  0
);

return [{
  json: {
    prospectName,
    date,
    requestorName,
    requestorEmail,
    discountLevel,
    selectedServices,
    pricingItems: processedPricing,
    total: total.toFixed(2)
  }
}];
```

---

### Block 4 – Proposal Build (HTML Generation)  

**Overview**  
A second Code node receives the processed data and constructs a complete, brand‑styled HTML document. It also produces a binary representation of the HTML (named `data`) for PDF conversion, and a JSON payload containing the generated file name and email metadata.

| Node | Type & Technical Role | Key Configuration | Expressions / Variables | Input → Output | Edge Cases / Failure Types |
|------|-----------------------|-------------------|--------------------------|----------------|---------------------------|
| Build HTML Proposal | `n8n‑nodes‑base.code` (v2) – Document builder | **JS Code** (full script below) | • Branding variables at top of script (`logoUrl`, `companyName`, `colorNavy`, etc.) – placeholders that **must be replaced** before activation. <br>• `displayDate` = current locale date <br>• `validUntil` = today + 30 days <br>• Builds table rows from `data.pricingItems` <br>• Assembles full HTML with `<style>` and `<body>` <br>• Creates `Buffer.from(htmlContent, 'utf-8')` → base64 binary under property `data` <br>• Returns JSON with `htmlContent`, `fileName`, `prospectName`, `requestorEmail`, `requestorName` | Input: JSON from “Process & Price” → Output: item with `json` (metadata) and `binary.data` (HTML file) | • `prospectName` containing illegal filename characters → replaced with hyphens via regex `/[^a-zA-Z0-9]/g`. <br>• Branding placeholders left unchanged → PDF will show generic values. <br>• Locale date formatting may vary based on n8n server timezone. <br>• Large number of pricing items → long HTML; ensure Gotenberg can handle it. |

**JS Code (Build HTML Proposal)**  
```javascript
/**
 * Pricing Proposal – HTML Builder
 * Replace the variables below with your own branding before activating.
 */
const data = $json;

// ── BRANDING – update these ──────────────────────────────────────────────────
const logoUrl    = 'YOUR_LOGO_URL';          // e.g. https://example.com/logo.png
const companyName = 'Your Company Name';
const tagline    = 'A division of Your Parent Company';
const colorNavy  = '#002657';
const colorBlue  = '#0021a5';
const colorOrange = '#fa4616';
const colorGold  = '#f2a900';
const colorDark  = '#343741';
// ─────────────────────────────────────────────────────────────────────────────

const displayDate = new Date().toLocaleDateString('en-US', { year: 'numeric', month: 'long', day: 'numeric' });
const validUntil  = new Date(Date.now() + 30 * 24 * 60 * 60 * 1000).toLocaleDateString('en-US', { year: 'numeric', month: 'long', day: 'numeric' });

const pricingRows = data.pricingItems.map((item, index) =>
  '<tr style="' + (index % 2 === 0 ? '' : 'background-color:#f9f9f9;') + '">'
  + '<td style="padding:12px;border-bottom:1px solid #eee;font-weight:bold;color:' + colorNavy + ';">' + item.service + '</td>'
  + '<td style="padding:12px;border-bottom:1px solid #eee;font-size:11px;">' + item.description + '</td>'
  + '<td style="padding:12px;border-bottom:1px solid #eee;text-align:center;">' + item.unit + '</td>'
  + '<td style="padding:12px;border-bottom:1px solid #eee;text-align:right;font-weight:bold;color:' + colorBlue + ';">$' + parseFloat(item.price).toFixed(2) + '</td>'
  + '</tr>'
).join('');

const htmlContent = '<!DOCTYPE html>'
+ '<html lang="en"><head><meta charset="UTF-8">'
+ '<title>Pricing Proposal - ' + data.prospectName + '</title>'
+ '<style>'
+ '@page{size:letter;margin:15mm;}'
+ 'body{font-family:Helvetica Neue,Helvetica,Arial,sans-serif;line-height:1.5;color:' + colorDark + ';margin:0;padding:0;}'
+ '.header{display:flex;justify-content:space-between;align-items:center;border-bottom:4px solid ' + colorOrange + ';padding-bottom:20px;margin-bottom:30px;}'
+ '.logo{height:70px;width:auto;}'
+ '.proposal-title{font-size:32px;font-weight:900;color:' + colorNavy + ';margin:10px 0;text-transform:uppercase;letter-spacing:-1px;}'
+ '.cover-info{background-color:#fcfcfc;padding:25px;border-radius:4px;border:1px solid #eee;margin:20px 0;}'
+ '.info-row{display:flex;margin:8px 0;font-size:14px;}'
+ '.label{font-weight:bold;color:' + colorBlue + ';width:150px;text-transform:uppercase;font-size:11px;}'
+ '.pricing-table{width:100%;border-collapse:collapse;margin:25px 0;}'
+ '.pricing-table th{background-color:' + colorNavy + ';color:white;padding:15px 12px;text-align:left;text-transform:uppercase;font-size:11px;}'
+ '.legal-section{margin-top:40px;padding:20px;background-color:#fffcf5;border-left:5px solid ' + colorGold + ';font-size:12px;line-height:1.7;}'
+ '.signature-section{margin-top:40px;padding:25px;background-color:#f9f9f9;border-radius:4px;}'
+ '.sig-line{border-bottom:2px solid ' + colorDark + ';display:inline-block;width:100%;margin-top:30px;}'
+ '</style></head><body>'
+ '<div class="header">'
+ '<img src="' + logoUrl + '" class="logo" alt="' + companyName + '">'
+ '<div style="text-align:right;color:' + colorNavy + ';font-size:10px;font-weight:bold;">' + tagline + '</div>'
+ '</div>'
+ '<div class="proposal-title">Pricing Proposal</div>'
+ '<div class="cover-info">'
+ '<div class="info-row"><span class="label">Prepared For:</span><span><strong>' + data.prospectName + '</strong></span></div>'
+ '<div class="info-row"><span class="label">Date:</span><span>' + displayDate + '</span></div>'
+ '<div class="info-row"><span class="label">Prepared By:</span><span>' + data.requestorName + '</span></div>'
+ '<div class="info-row"><span class="label">Valid Until:</span><span>' + validUntil + '</span></div>'
+ '</div>'
+ '<div>'
+ '<h2 style="color:' + colorNavy + ';border-bottom:2px solid ' + colorOrange + ';padding-bottom:10px;font-size:18px;">Proposed Services &amp; Pricing</h2>'
+ '<table class="pricing-table"><thead><tr>'
+ '<th>Service</th><th>Description</th><th style="text-align:center;">Unit</th><th style="text-align:right;">Price</th>'
+ '</tr></thead><tbody>' + pricingRows + '</tbody></table>'
+ '</div>'
+ '<div class="legal-section">'
+ '<h3 style="color:' + colorNavy + ';margin-top:0;font-size:14px;">Terms &amp; Conditions</h3>'
+ '<p>This pricing proposal is valid for <strong>thirty (30) days</strong>. After this period, all prices are subject to change. A binding contract is only created upon execution of a separate Services Agreement.</p>'
+ '</div>'
+ '<div class="signature-section">'
+ '<h3 style="color:' + colorNavy + ';margin-top:0;font-size:14px;">Authorization &amp; Acceptance</h3>'
+ '<table style="width:100%;margin-top:20px;"><tr>'
+ '<td style="width:60%;padding-right:20px;"><div class="sig-line"></div><div style="font-size:10px;color:' + colorDark + ';margin-top:5px;">Client Authorized Signature</div></td>'
+ '<td style="width:30%;"><div class="sig-line"></div><div style="font-size:10px;color:' + colorDark + ';margin-top:5px;">Date</div></td>'
+ '</tr><tr><td colspan="2" style="padding-top:20px;">'
+ '<div class="sig-line" style="width:60%;"></div>'
+ '<div style="font-size:10px;color:' + colorDark + ';margin-top:5px;">Printed Name &amp; Title</div>'
+ '</td></tr></table></div>'
+ '</body></html>';

const htmlBuffer = Buffer.from(htmlContent, 'utf-8');

return [{
  binary: {
    data: {
      data: htmlBuffer.toString('base64'),
      mimeType: 'text/html',
      fileName: 'index.html',
      fileExtension: 'html'
    }
  },
  json: {
    htmlContent:    htmlContent,
    fileName:       data.prospectName.replace(/[^a-zA-Z0-9]/g, '-') + '-Pricing-Proposal-' + new Date().toISOString().split('T')[0] + '.pdf',
    prospectName:   data.prospectName,
    requestorEmail: data.requestorEmail,
    requestorName:  data.requestorName
  }
}];
```

---

### Block 5 – PDF Conversion  

**Overview**  
The HTTP Request node posts the HTML binary to a Gotenberg endpoint, which renders the HTML with Chromium and returns a PDF binary. The node is configured to treat the response as a file.

| Node | Type & Technical Role | Key Configuration | Expressions / Variables | Input → Output | Edge Cases / Failure Types |
|------|-----------------------|-------------------|--------------------------|----------------|---------------------------|
| Gotenberg → PDF | `n8n‑nodes‑base.httpRequest` (v4.4) – External service call | • Method: **POST** <br>• URL: `https://YOUR_GOTENBERG_INSTANCE/forms/chromium/convert/html` (placeholder) <br>• Authentication: **Generic → HTTP Basic Auth** (credential must be created) <br>• Send Body: **true** <br>• Content Type: **multipart/form-data** <br>• Body Parameters: **1** item <br> – Name: `index.html` <br> – Parameter Type: **formBinaryData** <br> – Input Data Field Name: `=data` (binary property from previous node) <br>• Header Parameters: **1** item <br> – Name: `Gotenberg-Output-Filename` <br> – Value: `={{ $('Build HTML Proposal').item.json.fileName }}` <br>• Options → Response → Response Format: **File** | `$('Build HTML Proposal').item.json.fileName` for header; binary field `data` from previous node | Input: Binary HTML (`data`) from “Build HTML Proposal” + JSON metadata → Output: Binary PDF stored under default binary name `data` | • Gotenberg URL unreachable → HTTP timeout/connection error. <br>• Incorrect Basic Auth → 401 Unauthorized. <br>• Missing or misnamed binary field → multipart body empty. <br>• Gotenberg processing error → non‑PDF response, downstream node may fail. |

---

### Block 6 – Email Delivery  

**Overview**  
An Outlook node composes an email, attaches the generated PDF, and sends it to the requestor plus a configurable internal address.

| Node | Type & Technical Role | Key Configuration | Expressions / Variables | Input → Output | Edge Cases / Failure Types |
|------|-----------------------|-------------------|--------------------------|----------------|---------------------------|
| Send a message | `n8n‑nodes‑base.microsoftOutlook` (v2) – Email sender | • Operation: **Send** <br>• Subject: `=Pricing Proposal prepared for {{ $('Build HTML Proposal').item.json.prospectName }}` <br>• Body Content: `=Please find the attached pricing proposal for {{ $('Build HTML Proposal').item.json.prospectName }}, requested by {{ $('Build HTML Proposal').item.json.requestorName }}. The attached pricing is valid for 30 days.` <br>• To Recipients: `=YOUR_INTERNAL_EMAIL;{{ $('Build HTML Proposal').item.json.requestorEmail }}` (semicolon‑separated) <br>• Attachments: binary property name **data** (the PDF from Gotenberg) <br>• Credential: **Microsoft Outlook OAuth2** (named “Training Outlook”) | Dynamic fields: `prospectName`, `requestorName`, `requestorEmail` from “Build HTML Proposal” item | Input: PDF binary (`data`) → Output: confirmation (or error) | • Outlook OAuth2 token expired → 401 error. <br>• `YOUR_INTERNAL_EMAIL` not replaced → email may bounce. <br>• `requestorEmail` empty or malformed → recipient invalid. <br>• Attachment name missing → email may lack file. |

---

## 3. Summary Table  

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|-----------|-----------|----------------|---------------|----------------|-------------|
| Overview | `n8n‑nodes‑base.stickyNote` | Documentation & setup guide | – | – | **Full Overview Note** (see below) |
| Section – Intake | `n8n‑nodes‑base.stickyNote` | Visual label for Intake block | – | – | **1 · Intake** – Receives the form POST from your web form. |
| Section – Data | `n8n‑nodes‑base.stickyNote` | Visual label for Data block | – | – | **2 · Data** – Pulls all rows from the pricing Excel table in OneDrive. |
| Section – Processing | `n8n‑nodes‑base.stickyNote` | Visual label for Processing block | – | – | **3 · Processing** – Filters services, applies price tier (Discount / Retail / Premium), and calculates total. |
| Section – Build | `n8n‑nodes‑base.stickyNote` | Visual label for Build block | – | – | **4 · Proposal Build** – Renders the branded HTML document for PDF conversion. Update `YOUR_LOGO_URL` and the brand color variables to match your identity. |
| Section – PDF | `n8n‑nodes‑base.stickyNote` | Visual label for PDF block | – | – | **5 · PDF Conversion** – Self‑hosted [Gotenberg](https://gotenberg.dev) converts HTML → PDF. Update the URL to your Gotenberg instance and attach a **Basic Auth** credential. |
| Section – Email | `n8n‑nodes‑base.stickyNote` | Visual label for Email block | – | – | **6 · Email Delivery** – Sends the PDF to the requester via Gmail OAuth2. Replace `YOUR_INTERNAL_EMAIL` in the **Email Proposal** node with your own address. |
| Pricing Request Webhook | `n8n‑nodes‑base.webhook` (v2) | Receives inbound POST with form data | – | Fetch Pricing Sheet | **1 · Intake** – Receives the form POST from your web form. |
| Fetch Pricing Sheet | `n8n‑nodes‑base.microsoftExcel` (v2.2) | Retrieves all rows from `pricing_request.xlsx` table in OneDrive | Pricing Request Webhook | Process & Price | **2 · Data** – Pulls all rows from the pricing Excel table in OneDrive. |
| Process & Price | `n8n‑nodes‑base.code` (v2) | Filters rows, applies discount tier, computes total | Fetch Pricing Sheet | Build HTML Proposal | **3 · Processing** – Filters services, applies price tier (Discount / Retail / Premium), and calculates total. |
| Build HTML Proposal | `n8n‑nodes‑base.code` (v2) | Generates branded HTML and binary representation of it | Process & Price | Gotenberg → PDF | **4 · Proposal Build** – Renders the branded HTML document for PDF conversion. Update `YOUR_LOGO_URL` and the brand color variables to match your identity. |
| Gotenberg → PDF | `n8n‑nodes‑base.httpRequest` (v4.4) | Sends HTML to Gotenberg, receives PDF | Build HTML Proposal | Send a message | **5 · PDF Conversion** – Self‑hosted [Gotenberg](https://gotenberg.dev) converts HTML → PDF. Update the URL to your Gotenberg instance and attach a **Basic Auth** credential. |
| Send a message | `n8n‑nodes‑base.microsoftOutlook` (v2) | Emails the PDF proposal to requestor & internal address | Gotenberg → PDF | – | **6 · Email Delivery** – Sends the PDF to the requester via Gmail OAuth2. Replace `YOUR_INTERNAL_EMAIL` in the **Email Proposal** node with your own address. |

**Full Overview Sticky Note (copied verbatim for reference):**  

> ## 📄 Pricing Proposal Generator
> 
> Generates a branded PDF pricing proposal on demand and delivers it by email.
> 
> ### How it works
> 1. **Webhook Trigger** — A web form POSTs prospect details, selected services, discount level, and requestor info to the webhook endpoint.
> 2. **Fetch Pricing Sheet** — Retrieves all rows from your `pricing_request.xlsx` Excel table in OneDrive/SharePoint.
>    Download template: https://iportgpt.com/n8n_assets/pricing_request.xlsx
> 3. **Process & Price** — Filters the pricing rows to the selected services, applies the correct Discount / Retail / Premium price tier, and calculates the total.
> 4. **Build HTML** — Renders a fully branded HTML proposal document with cover info, line‑item pricing table, terms, and a signature block.
> 5. **Convert to PDF** — Sends the HTML to a self‑hosted [Gotenberg](https://gotenberg.dev) instance, which returns a print‑ready PDF.
> 6. **Email Proposal** — Delivers the PDF as an attachment via Gmail OAuth2 to the requester and any internal recipients you configure.
> 
> ### Setup
> 1. **Webhook** — Set the `path` field to a value that matches your form POST URL (e.g. `pricing-request`).
> 2. **Fetch Pricing Sheet** — Connect a Microsoft Excel credential, then select your OneDrive/SharePoint workbook, worksheet, and table.
> 3. **Process & Price** — Confirm the column names in the Code node match your Excel table (`Title`, `Description`, `Cost`, `Discount`, `Retail`, `Premium`, `Unit`).
> 4. **Build HTML Proposal** — Replace `YOUR_LOGO_URL` and brand color variables with your own values.
> 5. **Gotenberg → PDF** — Update the URL to point to your Gotenberg instance and attach a Basic Auth credential.
> 6. **Email Proposal** — Connect a Gmail OAuth2 credential. Update `YOUR_INTERNAL_EMAIL` in the `sendTo` field.
> 
> ### Customization
> - Add new price tiers by extending the `switch` block in **Process & Price**.
> - Adjust proposal branding in **Build HTML Proposal** by updating the CSS color variables and logo URL.
> - Validity period defaults to 30 days — change the `30` multiplier in **Build HTML Proposal** and the email body text.

---

## 4. Reproducing the Workflow from Scratch  

Follow these steps to rebuild the workflow in a fresh n8n instance.

1. **Create a new workflow**  
   - Name: **Pricing Proposal Generator** (or any name you prefer).

2. **Add the Intake trigger**  
   - Node type: **Webhook** (`n8n-nodes-base.webhook`)  
   - Settings:  
     - HTTP Method: **POST**  
     - Path: `pricing-request` (or your chosen path)  
   - Leave default response options (immediate “200 OK”).

3. **Add the data source node**  
   - Node type: **Microsoft Excel** (`n8n-nodes-base.microsoftExcel`)  
   - Settings:  
     - Resource: **Table**  
     - Operation: **Get Rows**  
     - Return All: **true**  
     - Workbook: **pricing_request.xlsx** (select from list after authenticating)  
     - Worksheet: **Sheet1**  
     - Table: **Table1**  
   - Credential: Create / select a **Microsoft Excel OAuth2** credential with read access to the OneDrive/SharePoint file.

4. **Add the processing Code node**  
   - Node type: **Code** (`n8n-nodes-base.code`)  
   - Mode: **JavaScript** (default)  
   - Paste the **Process & Price** script (see section 2).  
   - No special configuration; the script uses `$()` to reference the previous nodes.

5. **Add the HTML builder Code node**  
   - Node type: **Code** (`n8n-nodes-base.code`)  
   - Mode: **JavaScript**  
   - Paste the **Build HTML Proposal** script (section 2).  
   - **Important**: Replace placeholder values:  
     - `YOUR_LOGO_URL` → your logo’s public URL.  
     - `companyName`, `tagline`, colour variables (`colorNavy`, `colorBlue`, `colorOrange`, `colorGold`, `colorDark`) → your brand colours.  
   - The script writes the HTML to a binary property named `data` (base‑64 encoded) and returns JSON metadata (`fileName`, `prospectName`, `requestorEmail`, `requestorName`).

6. **Add the PDF conversion HTTP Request node**  
   - Node type: **HTTP Request** (`n8n-nodes-base.httpRequest`)  
   - Settings:  
     - Method: **POST**  
     - URL: `https://YOUR_GOTENBERG_INSTANCE/forms/chromium/convert/html` (replace with your instance’s address)  
     - Authentication: **Generic Credential Type → HTTP Basic Auth** (create a credential with the Gotenberg user/password)  
     - Send Body: **true**  
     - Content Type: **multipart/form-data**  
     - Body Parameters → Add one item:  
       - Name: `index.html`  
       - Parameter Type: **formBinaryData**  
       - Input Data Field Name: `=data` (binary field from the previous node)  
     - Header Parameters → Add one item:  
       - Name: `Gotenberg-Output-Filename`  
       - Value: `={{ $('Build HTML Proposal').item.json.fileName }}`  
     - Options → Response → Response Format: **File**  
   - The node will output a binary PDF in the default `data` property.

7. **Add the email delivery node**  
   - Node type: **Microsoft Outlook** (`n8n-nodes-base.microsoftOutlook`)  
   - Settings:  
     - Operation: **Send** (or “Send a message”)  
     - Subject: `=Pricing Proposal prepared for {{ $('Build HTML Proposal').item.json.prospectName }}`  
     - Body Content: `=Please find the attached pricing proposal for {{ $('Build HTML Proposal').item.json.prospectName }}, requested by {{ $('Build HTML Proposal').item.json.requestorName }}. The attached pricing is valid for 30 days.`  
     - To Recipients: `=YOUR_INTERNAL_EMAIL;{{ $('Build HTML Proposal').item.json.requestorEmail }}` (replace `YOUR_INTERNAL_EMAIL` with your internal address)  
     - Additional Fields → Attachments → Add attachment:  
       - Binary Property Name: `data` (the PDF from Gotenberg)  
   - Credential: Create / select a **Microsoft Outlook OAuth2** credential (e.g., “Training Outlook”) with permission to send mail.

8. **Wire the nodes** (in this order):  
   - **Pricing Request Webhook** → **Fetch Pricing Sheet** → **Process & Price** → **Build HTML Proposal** → **Gotenberg → PDF** → **Send a message**  

9. **Create the Excel data source**  
   - Upload the template `pricing_request.xlsx` to your OneDrive/SharePoint location.  
   - Ensure the workbook contains a table named **Table1** on **Sheet1** with columns: `Title`, `Description`, `Cost`, `Discount`, `Retail`, `Premium`, `Unit`.  
   - Template download link: https://iportgpt.com/n8n_assets/pricing_request.xlsx  

10. **Deploy Gotenberg** (if not already running)  
    - Deploy a Gotenberg Docker container or other self‑hosted instance.  
    - Secure it with Basic Auth and record the credentials used in the HTTP Request node.  
    - Verify the endpoint `/forms/chromium/convert/html` is reachable from your n8n instance.

11. **Activate the workflow**  
    - Click **Activate** in n8n.  
    - Use the webhook URL (e.g., `https://<your-n8n-instance>/webhook/pricing-request`) as the POST target from your web form.

12. **Test end‑to‑end**  
    - Send a sample POST request with fields: `prospectName`, `date`, `services` (array), `discountLevel` (`Discount`, `Retail`, or `Premium`), `requestorName` (e.g., `"Jane Smith (jane@example.com)"`).  
    - Verify the email arrives with the PDF attachment and that the PDF reflects the correct pricing, branding, and 30‑day validity.

---

## 5. General Notes & Resources  

| Note Content | Context or Link |
|--------------|-----------------|
| **Excel template** – Use the provided template to ensure column names match the Code nodes. | https://iportgpt.com/n8n_assets/pricing_request.xlsx |
| **Gotenberg documentation** – Installation, configuration, and API reference for the Chromium‑based PDF conversion service. | https://gotenberg.dev |
| **Outlook vs Gmail** – The description mentions Gmail OAuth2, but the workflow actually uses a Microsoft Outlook node. Use the Outlook node with an Outlook OAuth2 credential. | – |
| **Branding placeholders** – Replace `YOUR_LOGO_URL` and the colour variables (`colorNavy`, `colorBlue`, `colorOrange`, `colorGold`, `colorDark`) in the **Build HTML Proposal** script before activating. | – |
| **Gotenberg URL placeholder** – Replace `YOUR_GOTENBERG_INSTANCE` in the HTTP Request node with the actual host (e.g., `https://gotenberg.mycompany.com`). | – |
| **Internal email address** – Replace `YOUR_INTERNAL_EMAIL` in the **Send a message** node’s `toRecipients` field with the address that should always receive a copy of the proposal. | – |
| **Validity period** – The proposal defaults to a 30‑day validity period. To change it, edit the `30` multiplier in the **Build HTML Proposal** script (line `const validUntil = …`) and update the email body text accordingly. | – |
| **Price tier extension** – Adding a new price tier requires extending the `switch` block in the **Process & Price** script and adding a corresponding column in the Excel table. | – |
| **Error handling** – Consider adding an **Error Trigger** node or **IF** checks after each major node to capture failures (e.g., missing Excel rows, Gotenberg errors, Outlook auth issues) and notify administrators. | – |
| **Security** – The webhook is unauthenticated by default. If required, enable authentication (e.g., Basic Auth, Header Auth) in the Webhook node’s options. | – |
| **Testing tip** – Use the **Test** button on the Webhook node to capture a sample payload, then inspect the data flow through each node to validate field mappings. | – |