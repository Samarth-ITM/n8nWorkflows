Extract Clean Web Content with Anti-Bot Fallback for AI Agents & Workflows

https://n8nworkflows.xyz/workflows/extract-clean-web-content-with-anti-bot-fallback-for-ai-agents---workflows-5392


# Extract Clean Web Content with Anti-Bot Fallback for AI Agents & Workflows

### 1. Workflow Overview

This workflow, titled **"Extract Clean Web Content with Anti-Bot Fallback for AI Agents & Workflows"**, is designed to reliably scrape and extract clean textual content from any public webpage given a URL. It is particularly tailored for integration into AI agents and automated workflows where clean, readable web content is necessary.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Initial HTTP Request:** Receives the URL and optional full-text flag, performs a simple HTTP GET request with browser-like headers.
- **1.2 HTTP Error Handling & Anti-Bot Evasion:** Checks HTTP response status and error codes. If anti-bot protection is detected or request times out, it falls back to using Scrape.do, a third-party scraping API.
- **1.3 Content Type Validation:** Determines if the content is binary (e.g., PDF) and stops with an error if unsupported.
- **1.4 Content Extraction:** Extracts clean text content from the HTML using the community node `webpage-content-extractor`.
- **1.5 Output Formatting:** Depending on the full-text flag, formats the output as either full text (title and full text content) or a summarized excerpt with title and URL.

The workflow also includes detailed error handling to stop execution with meaningful messages on HTTP 404 errors, unsupported content types, or server errors.

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception & Initial HTTP Request

- **Overview:**  
  Accepts input parameters (`url` and `fulltext`) and performs an HTTP GET request to fetch the webpage content using a browser User-Agent header to reduce blocking.

- **Nodes Involved:**  
  - Workflow Call  
  - Simple Scraper

- **Node Details:**

  - **Workflow Call**  
    - *Type:* Execute Workflow Trigger  
    - *Role:* Entry point of the workflow, receives input parameters (`url`, `fulltext`) from calling workflows or AI agents.  
    - *Configuration:* Defines workflowInputs with `url` (string) and `fulltext` (boolean).  
    - *Connections:* Outputs to `Simple Scraper`.  
    - *Edge Cases:* Invalid or missing URL input would cause downstream failures.

  - **Simple Scraper**  
    - *Type:* HTTP Request  
    - *Role:* Performs HTTP GET request on the input URL with a User-Agent header imitating a modern browser, to bypass some naive bot protections.  
    - *Configuration:*  
      - URL set dynamically from `$json.url`.  
      - Timeout set to 10 seconds; retries enabled with 5-second wait between tries.  
      - Allows redirects and unauthorized certificates.  
      - Sends "User-Agent" header: `"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/137.0.0.0 Safari/537.36"`.  
      - On error, continues with error output so downstream nodes can handle it.  
    - *Connections:* Outputs to `Is Binary` and `Not 404` nodes.  
    - *Edge Cases:*  
      - Timeouts or connection aborts trigger fallback logic.  
      - HTTP errors (like 404) are handled downstream.

---

#### 2.2 HTTP Error Handling & Anti-Bot Evasion

- **Overview:**  
  Validates HTTP response status and error codes. If the page is not found (404), stops with an error. If connection errors or anti-bot blocks occur, attempts to fetch content via the Scrape.do API as a fallback.

- **Nodes Involved:**  
  - Not 404  
  - Not Found  
  - Try Antibot Evasion  
  - Scrape.do  
  - Server Error

- **Node Details:**

  - **Not 404**  
    - *Type:* If  
    - *Role:* Checks if the error status is NOT 404.  
    - *Configuration:* Checks `$json.error.status !== 404`.  
    - *Connections:*  
      - If true, continues to `Try Antibot Evasion`.  
      - If false, goes to `Not Found`.  
    - *Edge Cases:* If error object is missing or malformed, condition might fail.

  - **Not Found**  
    - *Type:* Stop and Error  
    - *Role:* Stops workflow with an error message "Error requesting website (404)" indicating the page was not found.  
    - *Connections:* None (terminates workflow).

  - **Try Antibot Evasion**  
    - *Type:* If  
    - *Role:* Checks if the error code from the HTTP request indicates a possible anti-bot or connection issue requiring fallback.  
    - *Configuration:*  
      Checks if `$json.error.code` equals any of: `"ECONNABORTED"`, `"ETIMEDOUT"`, `"ERR_CANCELED"`, or `"ERR_BAD_REQUEST"`.  
    - *Connections:*  
      - If true, routes to `Scrape.do` node (fallback scraper).  
      - If false, routes to `Server Error` node (stops with error).  
    - *Edge Cases:* If error object or code is undefined, condition may fail.

  - **Scrape.do**  
    - *Type:* HTTP Request  
    - *Role:* Calls the external Scrape.do API to fetch webpage content bypassing anti-bot protections.  
    - *Configuration:*  
      - URL: `http://api.scrape.do` with query param `url` set from `$json.url`.  
      - Timeout: 120 seconds (longer to accommodate API delays).  
      - Uses HTTP Query Auth with credentials storing API token from Scrape.do account.  
      - Retries enabled with 5-second intervals.  
    - *Connections:* Outputs to `Is Binary`.  
    - *Version:* Requires credentials setup as per setup notes.  
    - *Edge Cases:*  
      - API quota limits, invalid token, or network errors could cause failure.  
      - Timeout longer due to third-party API latency.

  - **Server Error**  
    - *Type:* Stop and Error  
    - *Role:* Stops workflow with a generic error message including the error code from the failed request.  
    - *Connections:* None (terminates workflow).

---

#### 2.3 Content Type Validation

- **Overview:**  
  Checks if the fetched content is binary (e.g., PDF), which is unsupported by the workflow, and stops with an error if so. Otherwise, proceeds to extract readable text.

- **Nodes Involved:**  
  - Is Binary  
  - ContentType Error  
  - Content Extractor

- **Node Details:**

  - **Is Binary**  
    - *Type:* If  
    - *Role:* Determines if the response contains binary data indicating unsupported formats (e.g., PDFs).  
    - *Configuration:* Checks if `$binary.data` exists and contains ".pdf".  
    - *Connections:*  
      - If true, goes to `ContentType Error`.  
      - If false, goes to `Content Extractor`.  
    - *Edge Cases:*  
      - Binary data detection relies on presence of `.pdf` string, may not detect all binary types.

  - **ContentType Error**  
    - *Type:* Stop and Error  
    - *Role:* Stops workflow with message "Unsupported content-type" when binary content is detected.  
    - *Connections:* None (terminates workflow).

  - **Content Extractor**  
    - *Type:* Webpage Content Extractor (community node)  
    - *Role:* Extracts clean textual content from raw HTML input.  
    - *Configuration:*  
      - Takes HTML content from `$json.data` (output of HTTP request).  
      - Uses the `n8n-nodes-webpage-content-extractor` node which parses HTML and extracts title, textContent, excerpt, etc.  
    - *Connections:* Outputs to `Full Text` node.  
    - *Version:* Requires installation of community node `n8n-nodes-webpage-content-extractor`.  
    - *Edge Cases:*  
      - Extraction accuracy depends on HTML quality and node capabilities.  
      - Node only works on self-hosted n8n.

---

#### 2.4 Output Formatting

- **Overview:**  
  Formats the extracted content based on the input `fulltext` flag, outputting either the full text content or a summarized excerpt.

- **Nodes Involved:**  
  - Full Text  
  - Fulltext Output  
  - Summary Output

- **Node Details:**

  - **Full Text**  
    - *Type:* If  
    - *Role:* Checks if the `fulltext` flag passed to the workflow is `true`.  
    - *Configuration:* Expression checks `$('Workflow Call').item.json.fulltext === true`.  
    - *Connections:*  
      - If true, goes to `Fulltext Output`.  
      - If false, goes to `Summary Output`.  
    - *Edge Cases:*  
      - Missing `fulltext` parameter defaults to false (summary).

  - **Fulltext Output**  
    - *Type:* Set  
    - *Role:* Prepares output with full page content: `title` and `text`.  
    - *Configuration:*  
      - `title`: extracted title with all emoji/pictographic characters removed (regex `/\p{Extended_Pictographic}/gu`).  
      - `text`: full page text content cleaned by removing emojis, line breaks, extra spaces.  
    - *Connections:* Output node for full text mode.

  - **Summary Output**  
    - *Type:* Set  
    - *Role:* Prepares output with summarized content: `title`, `url`, and short `content` excerpt.  
    - *Configuration:*  
      - `title`: cleaned extracted title without emojis.  
      - `url`: the original URL from workflow input.  
      - `content`: excerpt cleaned similarly to full text.  
    - *Connections:* Output node for summary mode.

---

### 3. Summary Table

| Node Name           | Node Type                      | Functional Role                             | Input Node(s)        | Output Node(s)           | Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
|---------------------|--------------------------------|---------------------------------------------|----------------------|--------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Workflow Call       | Execute Workflow Trigger        | Entry point; receives input params          | —                    | Simple Scraper            | # WebPage Reader for AI Agents & Workflows<br><br>This sub-workflow enables reliable and clean scraping of any public webpage by simply passing a **url** parameter. It is designed to be embedded into other workflows or used as a tool for AI agents.<br><br>🧩 This template requires the [n8n-nodes-webpage-content-extractor](https://www.npmjs.com/package/n8n-nodes-webpage-content-extractor) community node, so it only works in self-hosted n8n environments.<br><br>💡 If the site is protected by anti-bot systems (like Cloudflare), it will automatically fallback to [Scrape.do](https://scrape.do/), a scraping API with a generous free plan. You only need to provide your API Token.<br><br>## Input Parameters:<br>- **url** (string): the webpage URL to scrape<br>- **fulltext** (boolean): set true for full page content, false for summarized output<br><br>## Output Modes:<br>- **fulltext: true** — returns *{ title, text }* with full page content<br>- **fulltext: false** — returns *{ title, url, content }* with a short excerpt<br><br>## Usage:<br>In your workflows you can invoke this workflow using the **Execute Sub-workflow** or **Call n8n Workflow Tool** nodes.<br>Remember to pass the url as a parameter and configure the fulltext option when invoking it.<br><br>*(See the Setup note for instructions on how to set up this workflow.)* |
| Simple Scraper      | HTTP Request                   | Fetches webpage with browser headers        | Workflow Call        | Is Binary, Not 404        |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| Not 404             | If                            | Checks HTTP status not 404                   | Simple Scraper       | Try Antibot Evasion, Not Found |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| Not Found           | Stop and Error                | Stops on 404 error                           | Not 404              | —                        |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| Try Antibot Evasion | If                            | Detects anti-bot or connection errors       | Not 404              | Scrape.do, Server Error  |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| Scrape.do           | HTTP Request                  | Fetches webpage via Scrape.do API fallback  | Try Antibot Evasion  | Is Binary                 | # Setup<br>To set up this workflow you will need to install the [n8n-nodes-webpage-content-extractor](https://www.npmjs.com/package/n8n-nodes-webpage-content-extractor) community node and API Token from your [Scrape.do](https://scrape.do/) account. Then you will just need to configure the `Scrape.do` node here with your credentials.<br><br>## Community Node Installation<br>Before importing this workflow you first need to install this node on your n8n.<br>- Go to your n8n's settings page. By clicking on the three dots next to your username in the bottom left corner of the screen.<br>- In the left side menu click on Community Nodes.<br>- Now click on the Install button.<br>- In the npm Package Name field enter **n8n-nodes-webpage-content-extractor**, check the box that says *I understand the risks of installing unverified code from a public source*, and then click the Install button.<br><br>## `Scrape.do` Node Setup<br>Before configuring the node, create your account on [Scrape.do](https://scrape.do/) and save your API Token<br>- Open the Node `Scrape.do` configuration window by double-clicking on it.<br>- In the **Authentication** field, select the **Generic Credential Type** option.<br>- In the **Generic Credential Type** field below select the **Query Auth** option<br>- In the **Query Auth** field below select **Create new credential**. To save your token to a new n8n credential.<br>- In the window that opens, in the **Name** field, enter the word **token** only. In the **Value** field, paste your **API Token**. Then click the Save button.<br><br>**Your workflow is ready! You can now use it in your workflows and in your AI Agents!** |
| Server Error        | Stop and Error                | Stops on other HTTP errors                   | Try Antibot Evasion  | —                        |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| Is Binary           | If                            | Checks if response is binary (e.g., PDF)    | Simple Scraper, Scrape.do | ContentType Error, Content Extractor |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| ContentType Error   | Stop and Error                | Stops when unsupported binary content found | Is Binary            | —                        |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| Content Extractor   | Webpage Content Extractor     | Extracts clean text content from HTML        | Is Binary (false)    | Full Text                 |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| Full Text           | If                            | Checks fulltext flag to decide output format | Content Extractor    | Fulltext Output, Summary Output |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| Fulltext Output     | Set                           | Formats and cleans full text output          | Full Text            | —                        |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| Summary Output      | Set                           | Formats and cleans summarized output         | Full Text            | —                        |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| Sticky Note         | Sticky Note                   | Overview and usage instructions               | —                    | —                        | # WebPage Reader for AI Agents & Workflows<br><br>This sub-workflow enables reliable and clean scraping of any public webpage by simply passing a **url** parameter. It is designed to be embedded into other workflows or used as a tool for AI agents.<br><br>🧩 This template requires the [n8n-nodes-webpage-content-extractor](https://www.npmjs.com/package/n8n-nodes-webpage-content-extractor) community node, so it only works in self-hosted n8n environments.<br><br>💡 If the site is protected by anti-bot systems (like Cloudflare), it will automatically fallback to [Scrape.do](https://scrape.do/), a scraping API with a generous free plan. You only need to provide your API Token.<br><br>## Input Parameters:<br>- **url** (string): the webpage URL to scrape<br>- **fulltext** (boolean): set true for full page content, false for summarized output<br><br>## Output Modes:<br>- **fulltext: true** — returns *{ title, text }* with full page content<br>- **fulltext: false** — returns *{ title, url, content }* with a short excerpt<br><br>## Usage:<br>In your workflows you can invoke this workflow using the **Execute Sub-workflow** or **Call n8n Workflow Tool** nodes.<br>Remember to pass the url as a parameter and configure the fulltext option when invoking it.<br><br>*(See the Setup note for instructions on how to set up this workflow.)* |
| Sticky Note1        | Sticky Note                   | Setup instructions for required nodes and credentials | —                    | —                        | # Setup<br>To set up this workflow you will need to install the [n8n-nodes-webpage-content-extractor](https://www.npmjs.com/package/n8n-nodes-webpage-content-extractor) community node and API Token from your [Scrape.do](https://scrape.do/) account. Then you will just need to configure the `Scrape.do` node here with your credentials.<br><br>## Community Node Installation<br>Before importing this workflow you first need to install this node on your n8n.<br>- Go to your n8n's settings page. By clicking on the three dots next to your username in the bottom left corner of the screen.<br>- In the left side menu click on Community Nodes.<br>- Now click on the Install button.<br>- In the npm Package Name field enter **n8n-nodes-webpage-content-extractor**, check the box that says *I understand the risks of installing unverified code from a public source*, and then click the Install button.<br><br>## `Scrape.do` Node Setup<br>Before configuring the node, create your account on [Scrape.do](https://scrape.do/) and save your API Token<br>- Open the Node `Scrape.do` configuration window by double-clicking on it.<br>- In the **Authentication** field, select the **Generic Credential Type** option.<br>- In the **Generic Credential Type** field below select the **Query Auth** option<br>- In the **Query Auth** field below select **Create new credential**. To save your token to a new n8n credential.<br>- In the window that opens, in the **Name** field, enter the word **token** only. In the **Value** field, paste your **API Token**. Then click the Save button.<br><br>**Your workflow is ready! You can now use it in your workflows and in your AI Agents!** |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow and name it** (e.g., "WebPage-Reader").

2. **Add an "Execute Workflow Trigger" node** (name: `Workflow Call`):
   - Configure it to accept input parameters:
     - `url` (string)
     - `fulltext` (boolean)
   - This node serves as the entry point for external workflows or AI agents.

3. **Add an HTTP Request node** (name: `Simple Scraper`):
   - Set URL to `={{ $json.url }}` (dynamic from input).
   - Method: GET.
   - Set Headers → Add header:
     - Name: `User-Agent`
     - Value: `Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/137.0.0.0 Safari/537.36`
   - Options:
     - Timeout: 10000 ms
     - Follow Redirects: enabled
     - Allow Unauthorized Certificates: true
   - Enable "Retry on Fail" with 5 seconds between tries.
   - Set "On Error" to "Continue on Error" to allow downstream error handling.
   - Connect `Workflow Call` output to this node.

4. **Add an If node** (name: `Not 404`):
   - Condition: Check if `{{$json.error.status}}` NOT equals `404`.
   - Connect `Simple Scraper` output to this node.

5. **Add a Stop and Error node** (name: `Not Found`):
   - Error message: `Error requesting website (404)`
   - Connect `Not 404` node's false output to this node.

6. **Add an If node** (name: `Try Antibot Evasion`):
   - Condition: Check if `$json.error.code` equals any of:
     - `ECONNABORTED`
     - `ETIMEDOUT`
     - `ERR_CANCELED`
     - `ERR_BAD_REQUEST`
   - Use OR combinator.
   - Connect `Not 404` true output to this node.

7. **Add an HTTP Request node** (name: `Scrape.do`):
   - URL: `http://api.scrape.do`
   - Method: GET
   - Add Query Parameter:
     - Name: `url`
     - Value: `={{ $json.url }}`
   - Timeout: 120000 ms
   - Enable "Retry on Fail" with 5 seconds interval.
   - Authentication:
     - Generic Credential Type → HTTP Query Auth
     - Create a new credential with name "token" and paste your Scrape.do API token.
   - Connect `Try Antibot Evasion` true output to this node.

8. **Add a Stop and Error node** (name: `Server Error`):
   - Error message: `Error requesting website ({{$json.error.code}})`
   - Connect `Try Antibot Evasion` false output to this node.

9. **Add an If node** (name: `Is Binary`):
   - Condition: Check if `$binary.data` exists and contains `.pdf` (detect PDF).
   - Connect `Simple Scraper` true output to this node.
   - Connect `Scrape.do` output to this node as well.

10. **Add a Stop and Error node** (name: `ContentType Error`):
    - Error message: `Unsupported content-type`
    - Connect `Is Binary` true output to this node.

11. **Add a Webpage Content Extractor node** (name: `Content Extractor`):
    - Input parameter `html`: set to `={{ $json.data }}` (raw HTML from HTTP request).
    - Requires installation of the community node `n8n-nodes-webpage-content-extractor`.
    - Connect `Is Binary` false output to this node.

12. **Add an If node** (name: `Full Text`):
    - Condition: Check if the input parameter `fulltext` is `true`.
    - Expression: `={{ $('Workflow Call').item.json.fulltext }} === true`
    - Connect `Content Extractor` output to this node.

13. **Add a Set node** (name: `Fulltext Output`):
    - Fields:
      - `title`: `={{ $json.title.replace(/\p{Extended_Pictographic}/gu, '') }}`
      - `text`: Cleaned full text content:
        ```js
        ={{ ($json.textContent || '').replace(/\p{Extended_Pictographic}/gu, '').replace(/[\r\n]+/g, ' ').replace(/\s+/g, ' ').trim() }}
        ```
    - Connect `Full Text` true output to this node.

14. **Add a Set node** (name: `Summary Output`):
    - Fields:
      - `title`: same cleaning as above.
      - `url`: `={{ $('Workflow Call').item.json.url }}`
      - `content`: cleaned excerpt:
        ```js
        ={{ ($json.excerpt || '').replace(/\p{Extended_Pictographic}/gu, '').replace(/[\r\n]+/g, ' ').replace(/\s+/g, ' ').trim() }}
        ```
    - Connect `Full Text` false output to this node.

15. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Context or Link                                                                                                                                                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This sub-workflow enables reliable and clean scraping of any public webpage by simply passing a **url** parameter. It is designed to be embedded into other workflows or used as a tool for AI agents.<br>Requires the [n8n-nodes-webpage-content-extractor](https://www.npmjs.com/package/n8n-nodes-webpage-content-extractor) community node, so it only works in self-hosted n8n environments.<br><br>If the site is protected by anti-bot systems (like Cloudflare), it will automatically fallback to [Scrape.do](https://scrape.do/), a scraping API with a generous free plan. You only need to provide your API Token.<br><br>Input Parameters:<br>- **url** (string): the webpage URL to scrape<br>- **fulltext** (boolean): set true for full page content, false for summarized output<br><br>Output Modes:<br>- **fulltext: true** returns *{ title, text }*<br>- **fulltext: false** returns *{ title, url, content }*<br><br>Usage:<br>Invoke this workflow using **Execute Sub-workflow** or **Call n8n Workflow Tool** nodes, passing the url and fulltext parameters. | [webpage-content-extractor Node](https://www.npmjs.com/package/n8n-nodes-webpage-content-extractor)<br>[Scrape.do API](https://scrape.do/)<br>Workflow Sticky Note content                                                          |
| Setup Instructions:<br>- Install `n8n-nodes-webpage-content-extractor` community node via n8n Settings → Community Nodes → Install.<br>- Obtain Scrape.do API Token by creating an account at [Scrape.do](https://scrape.do/).<br>- Configure the `Scrape.do` HTTP Request node’s authentication as Generic Credential Type → HTTP Query Auth with the token.<br><br>Once setup, this workflow can be used seamlessly inside other workflows and AI agents.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Workflow Sticky Note with detailed installation and credential setup instructions.                                                                                                         |

---

**Disclaimer:** The text provided originates exclusively from an automated n8n workflow. It complies strictly with content policies, contains no illegal or offensive elements, and only processes legal and publicly available data.