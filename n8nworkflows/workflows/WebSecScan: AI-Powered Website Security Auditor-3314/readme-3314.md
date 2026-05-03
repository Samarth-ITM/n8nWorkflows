WebSecScan: AI-Powered Website Security Auditor

https://n8nworkflows.xyz/workflows/websecscan--ai-powered-website-security-auditor-3314


# WebSecScan: AI-Powered Website Security Auditor

### 1. Workflow Overview

The **WebSecScan: AI-Powered Website Security Auditor** workflow is designed to perform a comprehensive, non-invasive security analysis of any given website URL. It leverages OpenAI's GPT-4o-mini language models to conduct dual-layered security audits: one focusing on HTTP headers and configuration, and the other on client-side vulnerabilities in the website’s HTML and JavaScript content. The results are aggregated, processed to assign a security grade, and formatted into a professional, mobile-friendly HTML report that is emailed directly via Gmail.

**Target Use Cases:**  
- Website owners and developers seeking automated, quick security assessments without intrusive scanning.  
- Security analysts who want a preliminary client-side security overview before deeper penetration testing.  
- Teams requiring detailed, actionable security reports delivered via email.

**Logical Blocks:**  
- **1.1 Input Reception:** Captures the target website URL via a web form trigger.  
- **1.2 Website Content Retrieval:** Fetches the website HTML and HTTP headers.  
- **1.3 Parallel AI Security Analysis:**  
  - Header Configuration Audit (HTTP headers, cookies, CSP, etc.)  
  - Vulnerability Assessment (client-side HTML/JS vulnerabilities)  
- **1.4 Results Aggregation and Processing:** Merges AI outputs, extracts header details, computes security grade, and prepares data.  
- **1.5 Report Generation:** Converts processed data into a detailed HTML report with visual grading and recommendations.  
- **1.6 Email Delivery:** Sends the generated report via Gmail to a configured recipient.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures the website URL input from the user through a web-accessible form.

- **Nodes Involved:**  
  - Landing Page Url

- **Node Details:**  
  - **Landing Page Url**  
    - Type: Form Trigger  
    - Configuration:  
      - Form title: "Website Security Scanner"  
      - Single required field labeled "Landing Page Url" with placeholder "https://example.com"  
      - Form description instructs users to check their website for vulnerabilities  
    - Inputs: None (trigger node)  
    - Outputs: Passes form data (URL) downstream  
    - Edge Cases: Invalid or missing URL input will prevent triggering; no URL validation beyond required field.  
    - Version: 2.2

#### 2.2 Website Content Retrieval

- **Overview:**  
  Fetches the full HTTP response (headers and HTML content) from the provided URL, following redirects up to 5 times.

- **Nodes Involved:**  
  - Scrape Website

- **Node Details:**  
  - **Scrape Website**  
    - Type: HTTP Request  
    - Configuration:  
      - URL dynamically set from form input (`{{$json['Landing Page Url']}}`)  
      - Follows up to 5 redirects  
      - Returns full HTTP response including headers and body as text  
    - Inputs: From Landing Page Url  
    - Outputs: JSON containing raw headers and HTML content  
    - Edge Cases:  
      - Network errors, invalid URLs, or unreachable sites may cause request failure  
      - Large pages may cause timeout or memory issues  
    - Version: 4.2

#### 2.3 Parallel AI Security Analysis

- **Overview:**  
  Runs two parallel OpenAI-powered agents analyzing different security aspects: headers/configuration and client-side vulnerabilities.

- **Nodes Involved:**  
  - Extract Headers for Debug  
  - OpenAI Headers Analysis  
  - Security Configuration Audit  
  - OpenAI Content Analysis  
  - Security Vulnerabilities Audit

- **Node Details:**  

  - **Extract Headers for Debug**  
    - Type: Code  
    - Role: Formats raw HTTP headers into a readable string and preserves original headers for AI input  
    - Inputs: From Scrape Website  
    - Outputs: JSON with `formattedHeaders` and `originalHeaders`  
    - Edge Cases: Missing headers field in HTTP response  
    - Version: 2

  - **OpenAI Headers Analysis**  
    - Type: Langchain Chat OpenAI  
    - Role: Provides the language model interface for header analysis agent  
    - Configuration: Uses GPT-4o-mini model  
    - Credentials: OpenAI API key required  
    - Inputs: Connected as AI language model for Security Configuration Audit  
    - Outputs: AI-generated header audit text  
    - Edge Cases: API rate limits, auth errors, model timeouts  
    - Version: 1.2

  - **Security Configuration Audit**  
    - Type: Langchain Agent  
    - Role: Analyzes HTTP headers, cookies, and configuration for security misconfigurations  
    - Configuration: Custom prompt instructing detailed header presence, values, and issues detection  
    - Inputs: Receives formatted headers from Extract Headers for Debug  
    - Outputs: Detailed markdown-style audit report on headers and config  
    - Edge Cases: Incomplete headers, ambiguous AI output formatting  
    - Version: 1.7

  - **OpenAI Content Analysis**  
    - Type: Langchain Chat OpenAI  
    - Role: Provides the language model interface for client-side vulnerability analysis agent  
    - Configuration: Uses GPT-4o-mini model  
    - Credentials: OpenAI API key required  
    - Inputs: Connected as AI language model for Security Vulnerabilities Audit  
    - Outputs: AI-generated vulnerability audit text  
    - Edge Cases: API limits, auth errors, large HTML content causing truncation  
    - Version: 1.2

  - **Security Vulnerabilities Audit**  
    - Type: Langchain Agent  
    - Role: Analyzes HTML and visible content for client-side security vulnerabilities (XSS, info leakage, JS issues)  
    - Configuration: Custom prompt detailing audit structure and output format  
    - Inputs: Receives website HTML content from Scrape Website  
    - Outputs: Detailed markdown-style vulnerability report  
    - Edge Cases: Complex or obfuscated JS may reduce detection accuracy  
    - Version: 1.7

#### 2.4 Results Aggregation and Processing

- **Overview:**  
  Merges the two AI audit outputs, extracts and analyzes security headers, computes a security grade, counts issues, and prepares data for report generation.

- **Nodes Involved:**  
  - Merge Security Results  
  - Aggregate Audit Results  
  - Process Audit Results

- **Node Details:**  

  - **Merge Security Results**  
    - Type: Merge  
    - Role: Combines outputs from Security Configuration Audit and Security Vulnerabilities Audit in parallel branches  
    - Inputs: From both audit nodes  
    - Outputs: Merged array of audit results  
    - Edge Cases: Mismatched or missing outputs from either branch  
    - Version: 3

  - **Aggregate Audit Results**  
    - Type: Aggregate  
    - Role: Aggregates merged results into a single item for processing  
    - Inputs: From Merge Security Results  
    - Outputs: Single aggregated item containing both audit outputs  
    - Version: 1

  - **Process Audit Results**  
    - Type: Code  
    - Role:  
      - Extracts presence and values of key security headers from raw headers and AI output  
      - Detects unsafe inline CSP usage  
      - Calculates a security grade (A+ to F) based on header presence and quality  
      - Counts critical and warning issues  
      - Prepares structured audit data including timestamps and URLs  
    - Inputs: Aggregated audit results  
    - Outputs: JSON enriched with auditData object containing all processed info  
    - Edge Cases: Parsing errors, missing headers, unexpected AI output format  
    - Version: 2

#### 2.5 Report Generation

- **Overview:**  
  Converts the processed audit data into a detailed, styled HTML report including visual grade, header badges, vulnerability details, configuration issues, warnings, and implementation guidance.

- **Nodes Involved:**  
  - convert to HTML

- **Node Details:**  

  - **convert to HTML**  
    - Type: Code  
    - Role:  
      - Generates a responsive HTML email report with:  
        - Security grade visualization (color-coded badge)  
        - Summary table with site URL, timestamp, critical issues, warnings  
        - Header badges indicating presence and warnings  
        - Warnings section with explanations for CSP unsafe-inline, HSTS max-age, missing X-XSS-Protection  
        - Raw headers table with color-coded status  
        - Detailed vulnerability and configuration issues sections with formatted recommendations and code blocks  
        - Additional info about common headers  
        - Implementation guide for remediation steps  
        - Footer with disclaimers and copyright  
      - Uses inline CSS for styling and mobile-friendly layout  
    - Inputs: From Process Audit Results (auditData)  
    - Outputs: JSON with `emailHtml` field containing full HTML content  
    - Edge Cases: Large or malformed audit data causing formatting issues  
    - Version: 2

#### 2.6 Email Delivery

- **Overview:**  
  Sends the generated HTML security report via Gmail to a configured recipient email address.

- **Nodes Involved:**  
  - Send Security Report

- **Node Details:**  

  - **Send Security Report**  
    - Type: Gmail  
    - Role: Sends email with subject including scanned URL and body containing the HTML report  
    - Configuration:  
      - Recipient email hardcoded as `example@here.com` (should be customized)  
      - Subject: "Website Security Audit - [scanned URL]"  
      - Message: Uses `emailHtml` from previous node  
    - Credentials: Gmail OAuth2 required  
    - Inputs: From convert to HTML  
    - Outputs: Email sending status  
    - Edge Cases: OAuth token expiration, Gmail API limits, invalid recipient address  
    - Version: 2.1

---

### 3. Summary Table

| Node Name                  | Node Type                          | Functional Role                          | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                              |
|----------------------------|----------------------------------|----------------------------------------|-------------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------|
| Landing Page Url            | Form Trigger                     | User input of website URL               | None                          | Scrape Website                | How To Use This Workflow: Steps to deploy, access form, submit URL, and receive report via email       |
| Scrape Website             | HTTP Request                    | Fetch website HTML and headers          | Landing Page Url               | Security Vulnerabilities Audit, Extract Headers for Debug | Security Audit Process: Explains dual parallel analysis paths and non-invasive approach                 |
| Extract Headers for Debug   | Code                            | Format and extract HTTP headers         | Scrape Website                | Security Configuration Audit  | Security Audit Process (shared)                                                                         |
| OpenAI Headers Analysis     | Langchain Chat OpenAI           | OpenAI interface for header analysis    | Security Configuration Audit (AI model input) | Security Configuration Audit  | OpenAI Security Analysis: Notes on credentials, model choice, and analysis focus                        |
| Security Configuration Audit| Langchain Agent                 | Analyze headers and config for security | Extract Headers for Debug      | Merge Security Results        | Security Audit Process (shared)                                                                         |
| OpenAI Content Analysis     | Langchain Chat OpenAI           | OpenAI interface for content analysis   | Security Vulnerabilities Audit (AI model input) | Security Vulnerabilities Audit | OpenAI Security Analysis (shared)                                                                       |
| Security Vulnerabilities Audit | Langchain Agent              | Analyze HTML/JS for client-side vulnerabilities | Scrape Website                | Merge Security Results        | Security Audit Process (shared)                                                                         |
| Merge Security Results      | Merge                           | Combine header and vulnerability results| Security Configuration Audit, Security Vulnerabilities Audit | Aggregate Audit Results       | Results Processing: Explains analysis of AI output and grading                                          |
| Aggregate Audit Results     | Aggregate                      | Aggregate merged results into one item  | Merge Security Results         | Process Audit Results         | Results Processing (shared)                                                                             |
| Process Audit Results       | Code                            | Extract headers, compute grade, prepare data | Aggregate Audit Results        | convert to HTML               | Results Processing (shared)                                                                             |
| convert to HTML             | Code                            | Generate detailed HTML email report     | Process Audit Results          | Send Security Report          | Report Formatting: Details on HTML report structure, styling, and content                              |
| Send Security Report        | Gmail                           | Send HTML report via Gmail               | convert to HTML                | None                         | Email Configuration: Notes on Gmail OAuth2 setup and email formatting                                 |
| Sticky Note - Setup Instructions | Sticky Note               | Setup guide for credentials and activation | None                          | None                         | Setup Instructions: Stepwise credential and email config guide                                        |
| Sticky Note - OpenAI Analysis | Sticky Note                  | Notes on OpenAI usage and model choice  | None                          | None                         | OpenAI Security Analysis (shared)                                                                      |
| Sticky Note - Email Configuration | Sticky Note              | Notes on email sending setup             | None                          | None                         | Email Configuration (shared)                                                                           |
| Sticky Note - Audit Process | Sticky Note                    | Explains audit workflow logic            | None                          | None                         | Security Audit Process (shared)                                                                         |
| Sticky Note - How To Use    | Sticky Note                    | User instructions for workflow usage    | None                          | None                         | How To Use This Workflow (shared)                                                                      |
| Sticky Note - Report Formatting | Sticky Note                | Describes report design and features    | None                          | None                         | Report Formatting (shared)                                                                              |
| Sticky Note - Results Processing | Sticky Note               | Explains AI output processing and grading | None                          | None                         | Results Processing (shared)                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Name: Landing Page Url  
   - Type: Form Trigger  
   - Configure form title as "Website Security Scanner"  
   - Add one required text field labeled "Landing Page Url" with placeholder "https://example.com"  
   - Add form description: "Check your website for security vulnerabilities and get a detailed report"  

2. **Create an HTTP Request Node**  
   - Name: Scrape Website  
   - Type: HTTP Request  
   - Set URL to expression: `{{$json["Landing Page Url"]}}`  
   - Enable following redirects, max 5  
   - Set response format to full response, text body  
   - Connect input from Landing Page Url  

3. **Create a Code Node to Extract Headers**  
   - Name: Extract Headers for Debug  
   - Type: Code  
   - JavaScript: Format `items[0].json.headers` into a string and output both formatted and original headers  
   - Connect input from Scrape Website  

4. **Create Langchain Chat OpenAI Node for Headers Analysis**  
   - Name: OpenAI Headers Analysis  
   - Type: Langchain Chat OpenAI  
   - Model: GPT-4o-mini (or GPT-4o for production)  
   - Credentials: Link to OpenAI API credentials  
   - No additional options needed  

5. **Create Langchain Agent Node for Security Configuration Audit**  
   - Name: Security Configuration Audit  
   - Type: Langchain Agent  
   - Prompt: Provide detailed instructions to analyze HTTP headers, cookies, and config (see overview)  
   - Connect AI language model input from OpenAI Headers Analysis  
   - Connect main input from Extract Headers for Debug  

6. **Create Langchain Chat OpenAI Node for Content Analysis**  
   - Name: OpenAI Content Analysis  
   - Type: Langchain Chat OpenAI  
   - Model: GPT-4o-mini (or GPT-4o)  
   - Credentials: OpenAI API credentials  
   - No additional options needed  

7. **Create Langchain Agent Node for Security Vulnerabilities Audit**  
   - Name: Security Vulnerabilities Audit  
   - Type: Langchain Agent  
   - Prompt: Provide detailed instructions to analyze HTML/JS for client-side vulnerabilities (see overview)  
   - Connect AI language model input from OpenAI Content Analysis  
   - Connect main input from Scrape Website  

8. **Create Merge Node**  
   - Name: Merge Security Results  
   - Type: Merge  
   - Mode: Default (merge inputs)  
   - Connect inputs from Security Configuration Audit and Security Vulnerabilities Audit  

9. **Create Aggregate Node**  
   - Name: Aggregate Audit Results  
   - Type: Aggregate  
   - Aggregate field: output (to combine merged results into one item)  
   - Connect input from Merge Security Results  

10. **Create Code Node to Process Audit Results**  
    - Name: Process Audit Results  
    - Type: Code  
    - JavaScript: Implement logic to extract security headers, detect unsafe CSP, compute grade, count issues, and prepare auditData object (see overview)  
    - Connect input from Aggregate Audit Results  

11. **Create Code Node to Convert to HTML Report**  
    - Name: convert to HTML  
    - Type: Code  
    - JavaScript: Generate full HTML report with styling, grade badge, tables, vulnerability and configuration sections, warnings, and footer (see overview)  
    - Connect input from Process Audit Results  

12. **Create Gmail Node to Send Report**  
    - Name: Send Security Report  
    - Type: Gmail  
    - Configure recipient email address (replace default example@here.com)  
    - Subject: Use expression `"Website Security Audit - " + $json.auditData.url`  
    - Message: Use expression `$json.emailHtml`  
    - Credentials: Gmail OAuth2 credentials configured in n8n  
    - Connect input from convert to HTML  

13. **Activate Workflow**  
    - Ensure all nodes are connected as described  
    - Activate the workflow  
    - Copy the form URL from the Landing Page Url node to share or use  

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Quick Setup Guide: Add OpenAI API and Gmail OAuth2 credentials, update recipient email, activate workflow      | Sticky Note - Setup Instructions                                                                |
| OpenAI Security Analysis: Use GPT-4o models for detailed security analysis; each agent scans different aspects | Sticky Note - OpenAI Analysis                                                                   |
| Email Configuration: Gmail OAuth2 required; email sent as HTML with scanned URL in subject                     | Sticky Note - Email Configuration                                                               |
| Security Audit Process: Dual parallel analysis paths; non-invasive client-side inspection                      | Sticky Note - Audit Process                                                                     |
| How To Use: Deploy, activate, access form, submit URL, receive report via email                                | Sticky Note - How To Use                                                                        |
| Report Formatting: Professional, mobile-friendly HTML report with visual grade, color-coded sections           | Sticky Note - Report Formatting                                                                 |
| Results Processing: AI output analyzed to determine grade, count issues, extract headers, generate timestamp   | Sticky Note - Results Processing                                                                |
| OpenAI API Key Creation: https://platform.openai.com/                                                         | Setup Requirements section                                                                       |
| Gmail OAuth2 Setup: Configure via n8n Settings → Credentials → New → Gmail OAuth2 API                           | Setup Requirements section                                                                       |

---

This document fully describes the WebSecScan workflow, enabling users and AI agents to understand, reproduce, and modify the workflow with confidence.