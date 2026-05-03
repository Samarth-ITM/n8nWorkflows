Send Organized Security CVE Digests from NVD with AI-Polished Summaries to Gmail

https://n8nworkflows.xyz/workflows/send-organized-security-cve-digests-from-nvd-with-ai-polished-summaries-to-gmail-8097


# Send Organized Security CVE Digests from NVD with AI-Polished Summaries to Gmail

### 1. Workflow Overview

This workflow automates the retrieval, processing, summarization, and emailing of recent security vulnerability information (CVEs) from the National Vulnerability Database (NVD). It is designed for cybersecurity teams or analysts who want to receive timely, concise, and professionally formatted digests of the latest CVEs with AI-enhanced summaries directly into their Gmail inbox.

The workflow is structured into the following logical blocks:

- **1.1 Scheduled Trigger:** Periodically initiates the workflow (every 30 minutes by default).
- **1.2 Data Retrieval from NVD:** Fetches recent CVE data from the NVD API using a configured API key.
- **1.3 CVE Data Parsing and Extraction:** Processes raw JSON data to extract relevant CVE details such as vendor, product, severity, CVSS score, exploit presence, and brief summaries.
- **1.4 Digest Construction:** Organizes and formats the extracted CVEs into an HTML and plain-text digest, sorting by severity and CVSS.
- **1.5 AI-Powered Email Polishing:** Uses OpenAI GPT-4o-mini model to rewrite the digest into a concise, professional email format, returning only JSON with subject, HTML, and text.
- **1.6 Email Sending:** Sends the polished digest email through Gmail using OAuth2 credentials.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:** Periodically triggers the entire workflow to run every 30 minutes by default, enabling near-real-time CVE monitoring.
- **Nodes Involved:** Schedule Trigger

**Node: Schedule Trigger**  
- Type: Schedule Trigger  
- Role: Initiates the workflow execution on a fixed interval  
- Configuration: Interval set to every 30 minutes  
- Input: None (trigger node)  
- Output: Triggers the next node (HTTP Request)  
- Version-specific: None  
- Potential Failures: None (internal scheduler)  
- Sticky Note: "Workflow is triggered to execute every 30 min. You can change this to run every day, etc."

#### 2.2 Data Retrieval from NVD

- **Overview:** Queries the NVD API for CVEs published within the last day, requesting up to 200 results, using an API key for authentication.
- **Nodes Involved:** HTTP Request

**Node: HTTP Request**  
- Type: HTTP Request  
- Role: Fetches CVE data JSON from NVD API  
- Configuration:  
  - URL: `https://services.nvd.nist.gov/rest/json/cves/2.0`  
  - Query parameters sent as JSON:  
    - `pubStartDate`: 24 hours ago ISO timestamp  
    - `pubEndDate`: current ISO timestamp  
    - `resultsPerPage`: 200  
  - Headers include API key (must be replaced with user’s key)  
  - Sends query and headers as JSON  
- Input: Trigger from Schedule Trigger  
- Output: Raw CVE JSON data to Parse NVD node  
- Version-specific: Using n8n v0.190+ recommended due to HTTP node version 4.2 features  
- Edge Cases:  
  - API key missing or invalid → authentication failure  
  - Network timeout or NVD service downtime  
  - Rate limiting by NVD API  
- Sticky Note: "Current data source is set to NVD and it uses an API key from NVD. Make sure to update the HTTP Request Node with your API Key."

#### 2.3 CVE Data Parsing and Extraction

- **Overview:** Processes the raw CVE data to extract the top 20 newest CVEs with enriched metadata including vendor, product, version, severity, CVSS score, exploit indicators, and brief English summaries.
- **Nodes Involved:** Parse NVD

**Node: Parse NVD**  
- Type: Code (JavaScript)  
- Role: Parses input JSON, extracts and normalizes CVE information, applies heuristics to infer vendor/product/version from CPE and text, detects exploit references  
- Configuration: Custom JavaScript code with helper functions covering:  
  - CVSS score and severity extraction (v3.1, v3.0, v2 metrics)  
  - Parsing CPE 2.3 URIs to vendor/product/version triples  
  - Heuristics for vendor/product extraction from English description text  
  - Extraction of version strings from description text  
  - Detection of public exploit presence from references tagged as "exploit" or containing known exploit URLs  
  - Sorting and slicing top 20 newest CVEs  
- Input: JSON from HTTP Request node  
- Output: Array of CVE objects with normalized fields:  
  - cveId, link, vendor, product, versions (up to 5), severity, cvss, cvssVector, summary, hasExploit, exploitUrls, published date  
- Version-specific: Requires n8n that supports JavaScript code nodes with ES6 features  
- Edge Cases:  
  - Missing CVE fields or descriptions  
  - Variability in CPE data structures  
  - Descriptions in languages other than English ignored  
  - Empty or malformed input JSON  
- Sub-workflow: None

#### 2.4 Digest Construction

- **Overview:** Builds a human-readable digest from the parsed CVEs, sorting by severity and CVSS score, generating both HTML and plain text versions, and prepares email subject and flags for sending.
- **Nodes Involved:** Build Digest

**Node: Build Digest**  
- Type: Code (JavaScript)  
- Role: Sorts CVEs by severity (CRITICAL to UNKNOWN) and CVSS descending, counts severities, builds an HTML table and plain-text fallback list, constructs subject line  
- Configuration: Custom JavaScript code that:  
  - Escapes HTML entities for safe display  
  - Generates an HTML table with columns: CVE ID (with link), Vendor, Product, Versions, Severity, CVSS, Exploit presence, Brief description  
  - Constructs a plain-text summary list for fallback  
  - Sets `send` flag false if no items found, with appropriate subject and message  
- Input: CVE array from Parse NVD node  
- Output: JSON with keys: send (boolean), subject, html, text  
- Version-specific: Requires JavaScript code node with array and string methods  
- Edge Cases:  
  - No CVEs → sends no email with appropriate message  
  - Some fields missing → shows placeholders or defaults  
- Sub-workflow: None

#### 2.5 AI-Powered Email Polishing

- **Overview:** Sends the constructed digest to OpenAI GPT-4o-mini model to rewrite it into a concise, professional email format, strictly returning JSON with subject, HTML, and text.
- **Nodes Involved:** OpenAI Email Crafter

**Node: OpenAI Email Crafter**  
- Type: OpenAI (LangChain) node  
- Role: Uses GPT-4o-mini to rewrite the email content professionally  
- Configuration:  
  - Model: GPT-4o-mini (specified in node)  
  - Temperature: 0 (deterministic output)  
  - Messages:  
    - User message contains subject, HTML, and text from Build Digest node  
    - System message instructs to rewrite digest and return ONLY valid JSON with keys subject, html, text  
  - Output: JSON parsed from AI response replacing original message content  
- Input: Digest JSON from Build Digest node  
- Output: AI-polished email JSON with subject, html, text  
- Credential: OpenAI API key configured in credentials  
- Version-specific: Requires n8n version supporting @n8n/n8n-nodes-langchain.openAi node v1.8+  
- Edge Cases:  
  - API key invalid or rate limits exceeded  
  - AI response malformed JSON (node expects strict JSON)  
  - Network issues to OpenAI  
- Sub-workflow: None

#### 2.6 Email Sending

- **Overview:** Sends the AI-polished email digest via Gmail using OAuth2 authentication.
- **Nodes Involved:** Send a message (Gmail node)

**Node: Send a message**  
- Type: Gmail (Gmail OAuth2)  
- Role: Sends the final email to a configured recipient  
- Configuration:  
  - Recipient email: hardcoded to "test@gmail.com" (should be replaced with actual target)  
  - Subject, HTML body, and options populated from AI-polished JSON content  
  - OAuth2 credentials for Gmail account set up in n8n  
- Input: AI-polished email JSON from OpenAI Email Crafter  
- Output: Email sent event  
- Version-specific: Requires Gmail OAuth2 credentials properly configured in n8n  
- Edge Cases:  
  - OAuth2 token expired or revoked  
  - Gmail API quota exceeded  
  - Invalid recipient address  
- Sub-workflow: None

---

### 3. Summary Table

| Node Name           | Node Type                 | Functional Role                      | Input Node(s)      | Output Node(s)          | Sticky Note                                                                                  |
|---------------------|---------------------------|------------------------------------|--------------------|-------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger     | Schedule Trigger           | Periodic workflow trigger          | None               | HTTP Request            | Workflow is triggered to execute every 30 min. You can change this to run every day, etc.    |
| HTTP Request        | HTTP Request               | Fetch CVEs JSON from NVD API       | Schedule Trigger   | Parse NVD               | Current data source is set to NVD and it uses an API key from NVD. Update HTTP Request Node with your API Key |
| Parse NVD           | Code                      | Parse and extract CVE details      | HTTP Request       | Build Digest            |                                                                                              |
| Build Digest        | Code                      | Build HTML and text CVE digest     | Parse NVD          | OpenAI Email Crafter    |                                                                                              |
| OpenAI Email Crafter| OpenAI (LangChain)         | Rewrite digest into polished email | Build Digest       | Send a message          |                                                                                              |
| Send a message      | Gmail                      | Send email via Gmail OAuth2        | OpenAI Email Crafter| None                    |                                                                                              |
| Sticky Note         | Sticky Note                | Workflow trigger info note         | None               | None                    | Workflow is triggered to execute every 30 min. You can change this to run every day, etc.    |
| Sticky Note1        | Sticky Note                | HTTP Request info note             | None               | None                    | Current data source is set to NVD and it uses an API key from NVD. Update HTTP Request Node with your API Key |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**
   - Set to run every 30 minutes (or adjust interval as needed).

3. **Add HTTP Request node:**
   - Connect from Schedule Trigger output.
   - Set URL to `https://services.nvd.nist.gov/rest/json/cves/2.0`.
   - Enable sending query parameters as JSON:
     - `pubStartDate`: Expression → `{{ new Date(Date.now() - 24*60*60*1000).toISOString() }}`
     - `pubEndDate`: Expression → `{{ new Date().toISOString() }}`
     - `resultsPerPage`: `200`
   - Add header parameters:
     - `apiKey`: Your NVD API key (replace placeholder).
     - `pubStartDate`, `pubEndDate`, and `resultsPerPage` matching the query params.
   - Set Method to GET (default).
   - Enable `Send Query` and `Send Headers`.
   - Set Response Format to JSON.

4. **Add Code node "Parse NVD":**
   - Connect from HTTP Request output.
   - Paste the provided JavaScript code for parsing CVE data (includes helper functions for CVSS, CPE parsing, vendor/product heuristics, exploit detection).
   - Set to output multiple items (one per CVE).
   - Limit output to top 20 newest CVEs.

5. **Add Code node "Build Digest":**
   - Connect from Parse NVD output.
   - Paste the JavaScript code that:
     - Sorts CVEs by severity and CVSS.
     - Counts severities.
     - Builds an HTML table and a plain text fallback.
     - Produces a subject line.
     - Sets a `send` flag (false if no items).
   - Output JSON object with `send`, `subject`, `html`, and `text`.

6. **Add OpenAI node "OpenAI Email Crafter":**
   - Connect from Build Digest output.
   - Select model `gpt-4o-mini`.
   - Set temperature to 0.
   - Configure messages:
     - User message includes subject, HTML, text from Build Digest (via expressions).
     - System message instructs to rewrite digest and return ONLY valid JSON with keys `subject`, `html`, `text`.
   - Enable JSON output.
   - Provide OpenAI API credentials.

7. **Add Gmail node "Send a message":**
   - Connect from OpenAI Email Crafter output.
   - Set recipient email (replace `test@gmail.com` with actual target).
   - Configure subject and message body (HTML) by expressions pulling from AI node JSON output.
   - Authenticate with Gmail OAuth2 credentials.
   - Use typeVersion 2.1 or higher for best compatibility.

8. **Optional: Add Sticky Notes to document workflow trigger and HTTP Request node API key requirement.**

9. **Activate and test the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                                 |
|----------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Workflow is triggered every 30 minutes by default; can be adjusted to daily or other intervals via the Schedule node. | Sticky Note near Schedule Trigger node                                                                         |
| NVD API requires a valid API key; update the HTTP Request node header with your key before running the workflow.     | Sticky Note near HTTP Request node                                                                              |
| AI summarization uses OpenAI GPT-4o-mini model with temperature 0 for deterministic professional results.             | Node configuration in OpenAI Email Crafter node                                                                |
| Output email includes both HTML and plain text fallback for compatibility with different email clients.               | Build Digest node builds both HTML and text versions                                                          |
| Official NVD API documentation: https://nvd.nist.gov/developers/vulnerabilities                                        | Useful for extending or modifying API queries and parameters                                                   |
| Gmail OAuth2 setup requires creating credentials in Google Cloud Console and linking them in n8n credentials manager. | Gmail node configuration                                                                                        |

---

**Disclaimer:**  
The provided text is solely from an automated workflow created with n8n, respecting all applicable content policies and containing no illegal or protected data. All data processed is public and legal.