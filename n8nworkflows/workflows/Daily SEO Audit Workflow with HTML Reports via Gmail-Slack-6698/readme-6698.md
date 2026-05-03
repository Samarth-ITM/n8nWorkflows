Daily SEO Audit Workflow with HTML Reports via Gmail/Slack

https://n8nworkflows.xyz/workflows/daily-seo-audit-workflow-with-html-reports-via-gmail-slack-6698


# Daily SEO Audit Workflow with HTML Reports via Gmail/Slack

### 1. Workflow Overview

This workflow is designed to run a **daily, comprehensive SEO audit** of a specified website page, analyzing critical on-page and technical SEO factors such as meta tags, heading structures, Core Web Vitals, accessibility attributes, structured data, and security headers. It automates the process of detecting issues that could affect search engine rankings and generates a detailed **HTML report** with prioritized actionable insights.

The report is then delivered via **Gmail**, but can be adapted for Slack or other delivery methods by modifying the final node.

**Logical Blocks:**

- **1.1 Scheduled Trigger & Configuration:** Starts the workflow on a periodic schedule and sets target URLs and email recipients.
- **1.2 Webpage Content Retrieval:** Fetches the raw HTML content of the target page.
- **1.3 HTML Parsing:** Extracts key on-page elements (title, meta description, headings, images alt text).
- **1.4 SEO Audit Logic:** Applies custom code logic to analyze the extracted data and identify SEO issues.
- **1.5 Report Generation & Delivery:** Formats the audit results into a styled HTML email and sends it via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Configuration

- **Overview:**  
  Initiates the workflow hourly and configures essential variables such as the target site URL and email addresses for sending the report.

- **Nodes Involved:**  
  - Daily Audit Trigger  
  - Configure Target & Recipients

- **Node Details:**

  - **Daily Audit Trigger**  
    - Type: Schedule Trigger  
    - Role: Starts the workflow every hour (interval set to 1 hour).  
    - Configuration: Interval field set to hours.  
    - Inputs: None (trigger node).  
    - Outputs: Triggers "Configure Target & Recipients".  
    - Edge cases: Workflow will not run if n8n instance is down; no retry logic on trigger.  

  - **Configure Target & Recipients**  
    - Type: Set  
    - Role: Defines variables for the target URL and email addresses used later.  
    - Configuration:  
      - `siteUrl`: The URL to audit (default: https://example.com/).  
      - `emailFrom`: Sender's email address (exampleFrom@example.com).  
      - `emailTo`: Recipient's email address (exampleTo@example.com).  
    - Inputs: Receives trigger from "Daily Audit Trigger".  
    - Outputs: Passes variables downstream to "HTTP Fetch Page Content".  
    - Edge cases: Missing or invalid URLs may cause fetch failures downstream.

---

#### 2.2 Webpage Content Retrieval

- **Overview:**  
  Fetches the HTML content of the target URL to enable further analysis.

- **Nodes Involved:**  
  - HTTP Fetch Page Content

- **Node Details:**

  - **HTTP Fetch Page Content**  
    - Type: HTTP Request  
    - Role: Downloads the raw HTML content of the target page URL.  
    - Configuration:  
      - URL set dynamically using expression from `siteUrl` variable in "Configure Target & Recipients".  
      - Default HTTP GET method.  
    - Inputs: Receives URL from "Configure Target & Recipients".  
    - Outputs: Raw HTML content passed to "Parse On‑Page Elements".  
    - Edge cases:  
      - HTTP errors (404, 500, timeouts) may cause incomplete or no data.  
      - Site blocking automated requests (e.g., via User-Agent restrictions) is possible.  
      - No retries configured; could be added for robustness.

---

#### 2.3 HTML Parsing

- **Overview:**  
  Extracts key SEO-relevant elements from the fetched HTML content for detailed analysis.

- **Nodes Involved:**  
  - Parse On‑Page Elements

- **Node Details:**

  - **Parse On‑Page Elements**  
    - Type: HTML Extract  
    - Role: Parses HTML to extract elements: title, meta description, h1/h2 tags, image alt attributes.  
    - Configuration:  
      - Extraction targets configured using CSS selectors and attributes:  
        - `title` from `<title>` tag  
        - `metaDesc` from `<meta name="description">` content attribute  
        - `h1Tags` array from all `<h1>` tags  
        - `h2Tags` array from all `<h2>` tags  
        - `imgAlts` array from `alt` attributes of all `<img>` tags  
    - Inputs: Receives raw HTML from "HTTP Fetch Page Content".  
    - Outputs: Parsed JSON with extracted elements to "Run SEO Audit Logic".  
    - Edge cases:  
      - Missing elements return null or empty arrays (handled downstream).  
      - Complex or malformed HTML may yield incomplete extraction.

---

#### 2.4 SEO Audit Logic

- **Overview:**  
  Applies custom JavaScript logic to evaluate extracted page elements against SEO best practices and generates a structured audit report including issue codes, messages, and suggestions.

- **Nodes Involved:**  
  - Run SEO Audit Logic

- **Node Details:**

  - **Run SEO Audit Logic**  
    - Type: Code (JavaScript)  
    - Role: Implements detailed SEO checks on extracted data.  
    - Configuration:  
      - Checks include:  
        - Title presence and length (30–60 characters)  
        - Meta description presence and length (50–160 characters)  
        - Presence of meta robots, viewport, canonical link, and HTML lang attribute  
        - Word count threshold (≥300 words)  
        - Heading tags structure: exactly one h1, at least one h2, h3 presence  
        - Image alt text completeness  
        - Broken links count (status ≥ 400)  
        - Presence of structured data (JSON-LD schema)  
        - Core Web Vitals thresholds: LCP ≤ 2500ms, INP ≤ 200ms, CLS ≤ 0.1  
      - Builds a markdown summary with issues and suggestions.  
    - Inputs: Receives extracted page elements from "Parse On‑Page Elements".  
    - Outputs: JSON with audit results, issues array, suggestions map, markdown report to "Email Audit Report".  
    - Edge cases:  
      - Missing or partial data handled gracefully with appropriate issue flags.  
      - Errors in code execution (e.g., undefined properties) should be minimal given controlled inputs.  
      - If metrics or structured data are missing, corresponding issues are added.  
    - Version: Requires n8n supporting JavaScript code nodes (typeVersion 2).

---

#### 2.5 Report Generation & Delivery

- **Overview:**  
  Formats the SEO audit findings into a polished HTML email and sends the report to the configured recipients via Gmail.

- **Nodes Involved:**  
  - Email Audit Report

- **Node Details:**

  - **Email Audit Report**  
    - Type: Gmail (OAuth2)  
    - Role: Sends the completed SEO audit report via email.  
    - Configuration:  
      - `sendTo` dynamically set from `emailTo` variable.  
      - `subject` set to the audited site's URL.  
      - `message` is a fully styled HTML document embedding:  
        - Audit date  
        - URL and overall pass/fail status  
        - Top 3 priority issues  
        - Categorized issues lists (On-Page vs Technical) with suggestions  
        - Next steps and instructions  
    - Credentials: Gmail OAuth2 account configured externally.  
    - Inputs: Receives audit results and markdown from "Run SEO Audit Logic".  
    - Outputs: None (email sent).  
    - Edge cases:  
      - Gmail API quota limits or auth errors must be handled externally.  
      - If `emailTo` is invalid, email will fail.  
      - Email rendering depends on client support for HTML/CSS used.

---

### 3. Summary Table

| Node Name               | Node Type              | Functional Role                         | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                                                                                                                                                                                                                                                                                          |
|-------------------------|------------------------|---------------------------------------|-----------------------------|----------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note             | Sticky Note            | Documentation and overview note       | None                        | None                       | ## Complete SEO Audit Workflow with Automated HTML Reports via Gmail or Slack  Purpose: Run daily, in-depth audits of pages checking meta tags, headings, Core Web Vitals, accessibility, structured data, security headers. Benefits: Proactive monitoring, time savings, actionable insights. Output: HTML report via Gmail/Slack. Requirements: Gmail/Slack creds, customizable URLs and frequency. |
| Daily Audit Trigger     | Schedule Trigger       | Triggers audit run every hour          | None                        | Configure Target & Recipients |                                                                                                                                                                                                                                                                                                                                                                    |
| Configure Target & Recipients | Set                    | Sets URL and email addresses           | Daily Audit Trigger          | HTTP Fetch Page Content     |                                                                                                                                                                                                                                                                                                                                                                    |
| HTTP Fetch Page Content | HTTP Request           | Fetches target page HTML               | Configure Target & Recipients | Parse On‑Page Elements      |                                                                                                                                                                                                                                                                                                                                                                    |
| Parse On‑Page Elements  | HTML Extract           | Extracts SEO-relevant elements (title, meta, headings, images) | HTTP Fetch Page Content      | Run SEO Audit Logic         |                                                                                                                                                                                                                                                                                                                                                                    |
| Run SEO Audit Logic     | Code                   | Analyzes extracted data, detects SEO issues, generates report | Parse On‑Page Elements       | Email Audit Report          |                                                                                                                                                                                                                                                                                                                                                                    |
| Email Audit Report      | Gmail                  | Sends formatted HTML SEO audit report | Run SEO Audit Logic          | None                       |                                                                                                                                                                                                                                                                                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**
   - Name: `Daily Audit Trigger`
   - Set interval to 1 hour (every hour) under the "Rule" section.

3. **Add a Set node:**
   - Name: `Configure Target & Recipients`
   - Create three string fields:  
     - `siteUrl` (e.g., "https://example.com/")  
     - `emailFrom` (e.g., "exampleFrom@example.com")  
     - `emailTo` (e.g., "exampleTo@example.com")  
   - Connect `Daily Audit Trigger` output to this Set node input.

4. **Add an HTTP Request node:**
   - Name: `HTTP Fetch Page Content`
   - Set HTTP Method to GET (default).
   - Set URL to expression: `{{$node["Configure Target & Recipients"].json.siteUrl}}`
   - Connect `Configure Target & Recipients` output to this node.

5. **Add an HTML Extract node:**
   - Name: `Parse On‑Page Elements`
   - Operation: Extract HTML Content.
   - Data Property Name: `data` (default, matches HTTP Response Body)
   - Extraction Values:  
     - Key: `title`, CSS Selector: `title`  
     - Key: `metaDesc`, CSS Selector: `meta[name="description"]`, Return Value: attribute `content`  
     - Key: `h1Tags`, CSS Selector: `h1`, Return Array: true  
     - Key: `h2Tags`, CSS Selector: `h2`, Return Array: true  
     - Key: `imgAlts`, CSS Selector: `img`, Return Array: true, Return Value: attribute `alt`  
   - Connect `HTTP Fetch Page Content` output here.

6. **Add a Code node:**
   - Name: `Run SEO Audit Logic`
   - Language: JavaScript (Node.js)
   - Copy the custom audit logic code provided in the workflow. This code processes the extracted HTML elements, checks SEO best practices, and compiles a markdown report and issue list.
   - Connect `Parse On‑Page Elements` output here.

7. **Add a Gmail node:**
   - Name: `Email Audit Report`
   - Operation: Send Email
   - Credentials: Configure Gmail OAuth2 credentials in n8n and select here.
   - To: Expression `{{$node["Configure Target & Recipients"].json.emailTo}}`
   - Subject: Expression `{{$node["Configure Target & Recipients"].json.siteUrl}}`
   - Message: Use the full HTML email template from the workflow, which dynamically includes audit results, issue lists, and next steps using mustache-like expressions.
   - Connect `Run SEO Audit Logic` output here.

8. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                     |
|-----------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------|
| The workflow performs a comprehensive SEO audit covering meta data, headings, Core Web Vitals, accessibility, structured data, and technical SEO elements. | Sticky Note content at workflow start.                            |
| Email report uses advanced HTML and inline CSS styling for readability and clear categorization of issues.      | Final node "Email Audit Report" HTML content.                     |
| To customize frequency, adjust the `Schedule Trigger` node interval.                                           | "Daily Audit Trigger" node configuration.                         |
| Gmail OAuth2 credentials must be pre-configured in n8n credentials manager for the Email node to function.      | "Email Audit Report" node credential setup.                       |
| The audit logic code can be extended to include additional checks or integrate third-party SEO tools.          | "Run SEO Audit Logic" node - code comments and suggestions.       |

---

**Disclaimer:** The text provided comes exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.