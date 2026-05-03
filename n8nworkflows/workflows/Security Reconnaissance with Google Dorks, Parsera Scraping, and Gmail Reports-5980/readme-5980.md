Security Reconnaissance with Google Dorks, Parsera Scraping, and Gmail Reports

https://n8nworkflows.xyz/workflows/security-reconnaissance-with-google-dorks--parsera-scraping--and-gmail-reports-5980


# Security Reconnaissance with Google Dorks, Parsera Scraping, and Gmail Reports

### 1. Workflow Overview

This workflow automates a security reconnaissance process using Google Dorks tailored to a user-provided target domain. It performs customized Google searches (dorks), scrapes the search results via the Parsera AI Scraper service, cleans and filters the output links, formats the findings into an HTML-styled report, and finally sends this report via Gmail. The workflow is structured into distinct logical blocks to handle user input, generate dork queries, scrape data, clean results, format output, and deliver email notifications.

**Logical blocks:**

- **1.1 Input Reception**  
  Handles user input of a target domain URL.

- **1.2 Google Dorks Generation**  
  Creates customized Google Dork search URLs based on the input domain.

- **1.3 Search Result Scraping**  
  Uses Parsera’s AI Scraper agent to scrape Google search results from the generated dork URLs.

- **1.4 Data Cleaning and Filtering**  
  Filters the scraped links to retain valid, relevant URLs.

- **1.5 Report Generation**  
  Converts cleaned links into a markdown-styled HTML report.

- **1.6 Email Delivery**  
  Sends the generated report to a specified email address via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures user input via a form to specify the target domain URL, which is used to customize subsequent Google Dork searches.

- **Nodes Involved:**  
  - Form Input  
  - Sticky Note (commentary)

- **Node Details:**  
  - **Form Input**  
    - Type: `formTrigger` (Webhook-based form trigger)  
    - Configuration: Presents a form titled "Enter Target Domain" with a single field labeled "URL" and placeholder "https://example.com".  
    - Input/Output: Outputs JSON containing the user-input URL.  
    - Edge Cases: Invalid or empty URL input may cause downstream logic to fail or produce incorrect dorks. No explicit validation is configured.  
    - Sticky Note: Explains purpose of the input (“This is the website that will be scraped for data”).  

#### 2.2 Google Dorks Generation

- **Overview:**  
  Generates a list of Google Dork search URLs customized to the user's input domain. The search queries target common security-sensitive file types and information leaks.

- **Nodes Involved:**  
  - Dorks Template Search (Code node)  
  - Split Dorks One-by-One (Batch splitter)  
  - Sticky Note (commentary)

- **Node Details:**  
  - **Dorks Template Search**  
    - Type: `code` (JavaScript)  
    - Configuration: Extracts domain from the input URL using regex, then substitutes `.example.com` in a set of predefined dorks with the extracted domain. Constructs Google search URLs by encoding each dork string.  
    - Key Expressions: Regex for domain extraction, URL encoding, array mapping to output structured JSON with `dork` URLs.  
    - Input: Receives the URL from the Form Input node.  
    - Output: Emits an array of objects each containing a Google search URL under the `dork` key.  
    - Edge Cases: If domain extraction fails, defaults to `example.com`, which might cause irrelevant or failed searches.  
    - Sticky Note: Describes use of a template to create customized links.  
  - **Split Dorks One-by-One**  
    - Type: `splitInBatches`  
    - Configuration: Splits the array of dork URLs into batches of 8 to manage scraping load and rate limits.  
    - Input: Receives the array of dork URLs from the previous code node.  
    - Output: Passes each batch sequentially to the scraping node.  
    - Edge Cases: Batch size too large may cause timeouts or rate limit issues; too small may reduce efficiency.

#### 2.3 Search Result Scraping

- **Overview:**  
  Uses the Parsera AI Scraper agent to scrape search results from each Google Dork URL generated.

- **Nodes Involved:**  
  - Scrape with agent (AI Scraper node)  
  - Sticky Note (commentary)

- **Node Details:**  
  - **Scrape with agent**  
    - Type: `aiScraper` (Parsera AI Scraper integration)  
    - Configuration: Uses the `google` agent (pre-configured in Parsera dashboard) to scrape the URL provided in each dork. The URL is dynamically read from the incoming JSON key `dork`.  
    - Credentials: Requires valid Parsera API credentials linked to the account and agent.  
    - Input: Receives batches of dork URLs from the Split Dorks node.  
    - Output: Produces scraped data including links and other metadata from Google search results.  
    - Edge Cases:  
      - API authentication errors if credentials expire or are invalid.  
      - Scraping failures due to Google blocking, captchas, or network issues.  
      - Agent misconfiguration may cause incomplete or no data.  
    - Sticky Note: Instructs setup of Parsera agent named "Google" with API key and URL `https://google.com`.

#### 2.4 Data Cleaning and Filtering

- **Overview:**  
  Filters and sanitizes the scraped results to retain only valid and relevant URLs, removing Google internal links or irrelevant entries.

- **Nodes Involved:**  
  - Clean Output (Code node)

- **Node Details:**  
  - **Clean Output**  
    - Type: `code` (JavaScript)  
    - Configuration: Iterates over all scraped items, checks if the `Link` property exists and meets criteria: starts with `http`, excludes Google internal URLs, excludes relative URLs starting with `/search` or anchors `#`. Returns only valid URLs as `cleanedLink`.  
    - Input: Receives raw scraped data from the Scrape with agent node.  
    - Output: Array of cleaned link objects.  
    - Edge Cases:  
      - Missing or malformed `Link` properties may be filtered out.  
      - If no valid links remain, downstream nodes receive empty arrays which may produce an empty report.  
    - No explicit error handling for unexpected data formats.

#### 2.5 Report Generation

- **Overview:**  
  Formats the cleaned links into a markdown-styled HTML report with a numbered list for readability.

- **Nodes Involved:**  
  - Generate HTML (Code node)

- **Node Details:**  
  - **Generate HTML**  
    - Type: `code` (JavaScript)  
    - Configuration: Collects all `cleanedLink` values, constructs a markdown string with a heading and numbered list with links formatted as `[link](url)`. Uses double newlines between items for spacing.  
    - Input: Receives cleaned links array from Clean Output node.  
    - Output: JSON with a `markdown` key containing the formatted report string.  
    - Edge Cases: Empty input results in a report with heading but no links.  
    - No HTML sanitization needed as content is generated internally.

#### 2.6 Email Delivery

- **Overview:**  
  Sends the markdown report via Gmail to a predefined email recipient.

- **Nodes Involved:**  
  - Send a message (Gmail node)  
  - Sticky Note (commentary)

- **Node Details:**  
  - **Send a message**  
    - Type: `gmail`  
    - Configuration: Sends an email with subject "Google Dorks Report" to the configured recipient email address (default "example@gmail.com"). The message body is the markdown report generated earlier.  
    - Credentials: Requires Gmail OAuth2 credentials configured with send email permissions.  
    - Input: Receives markdown report from Generate HTML node.  
    - Edge Cases:  
      - Authentication failure if OAuth token expires.  
      - Email sending failure due to network or quota limits.  
    - Sticky Note: Instructs user to double-click and configure Gmail credentials and recipient address.

---

### 3. Summary Table

| Node Name            | Node Type         | Functional Role                | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                                          |
|----------------------|-------------------|-------------------------------|-----------------------|--------------------------|----------------------------------------------------------------------------------------------------------------------|
| Form Input           | formTrigger       | Receive target domain input    | —                     | Dorks Template Search    | ## Enter URL This is the website that will be scraped for data                                                       |
| Dorks Template Search| code              | Generate Google Dork URLs      | Form Input            | Split Dorks One-by-One   | ## Google Dorks A template is used to create customized links based off the URL input.                               |
| Split Dorks One-by-One| splitInBatches    | Batch dork URLs for scraping   | Dorks Template Search | Scrape with agent        |                                                                                                                      |
| Scrape with agent    | aiScraper         | Scrape Google search results   | Split Dorks One-by-One| Clean Output             | ## Parsera Node An agent must be created in Parsera before using this node. Create a new agent called "Google" with https://google.com and enter your API key. |
| Clean Output         | code              | Filter and clean scraped links | Scrape with agent     | Generate HTML            |                                                                                                                      |
| Generate HTML        | code              | Format links into markdown     | Clean Output          | Send a message           |                                                                                                                      |
| Send a message       | gmail             | Send report email              | Generate HTML         | —                        | ## Setup Gmail Account **Double click** to edit this node and configure your Gmail account and enter the destination email address. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create “Form Input” node (formTrigger):**  
   - Set webhook path to a unique identifier (e.g., `f6a3f761-54de-4b3f-93bf-063c4181b5ca`).  
   - Add a form field labeled “URL” with placeholder “https://example.com”.  
   - Title the form “Enter Target Domain”.  
   - This node initiates the workflow upon form submission.

2. **Create “Dorks Template Search” node (code):**  
   - Connect input from “Form Input”.  
   - Use JavaScript code to extract the domain from the input URL using regex.  
   - Define a template string of Google Dork queries with `.example.com` placeholders.  
   - Replace `.example.com` with the extracted domain.  
   - Encode each dork query as a Google search URL.  
   - Output an array of objects containing these URLs under the key `dork`.  

3. **Create “Split Dorks One-by-One” node (splitInBatches):**  
   - Connect input from “Dorks Template Search”.  
   - Set batch size to 8 to process dork URLs in manageable chunks.

4. **Create “Scrape with agent” node (aiScraper):**  
   - Connect input from “Split Dorks One-by-One”.  
   - Configure resource as “agent”.  
   - Set `agentName` to `google`.  
   - Set URL parameter to `={{ $json.dork }}` to dynamically scrape each Google Dork URL.  
   - Provide valid Parsera AI Scraper API credentials linked to an agent named “Google” configured with base URL `https://google.com`.  

5. **Create “Clean Output” node (code):**  
   - Connect input from “Scrape with agent”.  
   - Use JavaScript code to filter out links that are invalid, internal to Google, or relative paths.  
   - Emit only cleaned absolute URLs as `cleanedLink`.

6. **Create “Generate HTML” node (code):**  
   - Connect input from “Clean Output”.  
   - Gather all cleaned links and generate a markdown string: a heading plus a numbered list of clickable links.  
   - Output the markdown string in a JSON property `markdown`.  

7. **Create “Send a message” node (gmail):**  
   - Connect input from “Generate HTML”.  
   - Set recipient email address (e.g., `example@gmail.com`).  
   - Set subject to “Google Dorks Report”.  
   - Set message body to `={{ $json.markdown }}`.  
   - Configure Gmail OAuth2 credentials with proper permissions to send emails.

8. **Connect nodes according to the following flow:**  
   Form Input → Dorks Template Search → Split Dorks One-by-One → Scrape with agent → Clean Output → Generate HTML → Send a message.

9. **Add Sticky Notes for clarity as follows:**  
   - Near “Form Input”: Note about entering the URL.  
   - Near “Dorks Template Search”: Note explaining the use of a dork template.  
   - Near “Scrape with agent”: Note about setting up Parsera agent named “Google” with API key.  
   - Near “Send a message”: Note about configuring Gmail account and recipient email.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                  | Context or Link                                                                                           |
|-----------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| An agent must be created in Parsera before using the Scrape with agent node. Create a new agent in your Parsera dashboard named “Google” and use https://google.com. Enter your API key and use ‘google’ as the agent name. | Parsera AI Scraper setup instructions.                                                                   |
| Double click the “Send a message” node to configure your Gmail OAuth2 credentials and set the destination email address for report delivery. | Gmail OAuth2 credential setup for sending emails.                                                        |
| Google Dorks are powerful search queries used to uncover sensitive information or vulnerabilities by targeting specific file types or URL patterns. | Security reconnaissance context.                                                                         |
| The batch process in “Split Dorks One-by-One” controls concurrency and rate limits to avoid scraping blocks or API overuse. Adjust the batch size accordingly. | Performance and rate limiting considerations.                                                            |

---

**Disclaimer:** The provided content is extracted exclusively from an automated n8n workflow. It complies fully with content policies and contains no illegal or protected elements. All data processed is publicly available and legal.