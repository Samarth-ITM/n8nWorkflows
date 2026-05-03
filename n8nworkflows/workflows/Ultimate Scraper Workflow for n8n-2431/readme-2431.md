Ultimate Scraper Workflow for n8n

https://n8nworkflows.xyz/workflows/ultimate-scraper-workflow-for-n8n-2431


# Ultimate Scraper Workflow for n8n

### 1. Workflow Overview

**Purpose:**  
The "Ultimate Scraper Workflow for n8n" is designed to extract structured information from any webpage, including pages requiring login via session cookies. It leverages Selenium for browser automation and screenshots, and OpenAI’s GPT-4 for AI-powered extraction of relevant data from webpage images or text. It supports cookie injection for authenticated scraping and fallback scraping without cookies. This makes it suitable for advanced web scraping scenarios, including dynamic content and sites protected by anti-bot measures.

**Target Use Cases:**  
- Scraping public or authenticated web pages for specific data points such as followers or stars on GitHub repositories.  
- Extracting information where direct HTML parsing is insufficient and AI analysis of rendered page screenshots is beneficial.  
- Handling websites that block scraping attempts, detecting such blocks, and responding accordingly.  
- Using session cookies collected externally to access authenticated content.  

**Logical Blocks:**  
1.1 Input Reception & Initial Parsing  
1.2 Google Search to Find Relevant URL (if target URL not provided)  
1.3 Selenium Session Management (Create, Configure, Delete)  
1.4 URL Navigation and Browser Interaction (with or without cookies)  
1.5 Screenshot Capture and AI Analysis  
1.6 Result Extraction and Response Handling  
1.7 Error and Edge Case Handling  
1.8 Debug and Auxiliary Nodes (IP check, Proxy notes)

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Initial Parsing  
**Overview:**  
Receives the webhook POST request containing the target subject, URL, data points to extract, and optional cookies. Parses and prepares initial fields for subsequent processing.

**Nodes Involved:**  
- Webhook  
- Edit Fields (For testing purpose)  
- If Target Url

**Node Details:**  
- **Webhook**  
  - Type: HTTP Webhook Trigger  
  - Configured to listen on POST path `67d77918-2d5b-48c1-ae73-2004b32125f0`  
  - Outputs JSON body containing subject, URL, target data, and cookies.  
  - Input: External HTTP POST request  
  - Output: JSON for further processing  
  - Edge cases: Missing or malformed input JSON

- **Edit Fields (For testing purpose)**  
  - Type: Set Node  
  - Extracts `subject` and `Url` from webhook JSON payload and assigns to `Subject` and `Website Domaine` respectively  
  - Input: Webhook output  
  - Output: JSON with simplified fields for domain and subject  
  - Edge cases: Missing fields in input JSON

- **If Target Url**  
  - Type: If Node  
  - Checks if the `Target Url` field in webhook is empty  
  - Routes workflow to Google Search block if empty, or directly to Selenium session creation if present  
  - Input: Edited fields JSON  
  - Output: Conditional branching  
  - Edge cases: Empty or malformed URL field

---

#### 1.2 Google Search to Find Relevant URL  
**Overview:**  
If no direct target URL is provided, performs a Google search limited to the target domain and subject to find relevant URLs. Extracts URLs from the search results and uses AI to select the best URL for scraping.

**Nodes Involved:**  
- Google search Query  
- Extract First Url Match  
- Information Extractor  
- Check if empty or NA  
- Error can't find url

**Node Details:**  
- **Google search Query**  
  - Type: HTTP Request  
  - Sends a Google search query restricted to `site:<Website Domaine>` and subject keyword  
  - Input: Edited fields with domain and subject  
  - Output: HTML content of search results page  
  - Edge cases: Google blocking or CAPTCHAs, network timeout, unexpected HTML structure

- **Extract First Url Match**  
  - Type: HTML Extract  
  - Extracts all anchor tags with `href` containing `https://` and matching domain filter  
  - Input: Google search results HTML  
  - Output: Array of URLs as attributes `href`  
  - Edge cases: No matching URLs found, malformed HTML

- **Information Extractor**  
  - Type: Langchain Information Extractor  
  - Uses GPT-4o chat model to select the most relevant URL from the extracted list based on subject and domain  
  - Input: Extracted URLs concatenated into text for AI analysis  
  - Output: JSON with attribute `Good_url_for_etract_information`  
  - Edge cases: GPT output invalid or returns "NA" or empty  

- **Check if empty or NA**  
  - Type: If Node  
  - Checks if the selected URL is empty or "NA"  
  - If true, triggers error response  
  - Otherwise, continues to Selenium session creation  
  - Edge cases: AI returns no relevant URL

- **Error can't find url**  
  - Type: Respond to Webhook  
  - Returns HTTP 404 with JSON error message `"Can't find url"`  
  - Input: Condition from previous node  
  - Output: Terminates workflow with error response

---

#### 1.3 Selenium Session Management  
**Overview:**  
Creates and configures a Selenium session for browser automation, including setting a user-agent and disabling automation detection. Also manages session deletion at various workflow exit points.

**Nodes Involved:**  
- Create Selenium Session  
- Resize browser window  
- Clean Webdriver  
- Delete Session (multiple variants)

**Node Details:**  
- **Create Selenium Session**  
  - Type: HTTP Request  
  - POST to Selenium Grid endpoint to create a session with Chrome browser and custom Chrome options, including disabling automation features and setting user-agent  
  - Input: Triggered after URL resolution or directly if target URL is known  
  - Output: Session ID for subsequent Selenium commands  
  - On error: Continues error output to allow graceful handling  
  - Edge cases: Selenium server unavailable, timeout, invalid capabilities

- **Resize browser window**  
  - Type: HTTP Request  
  - POST to Selenium endpoint to set browser window size to 1920x1080 for consistent rendering  
  - Input: Session ID from create session  
  - Output: Confirmation of window resizing  
  - Edge cases: Session invalidated, request failure

- **Clean Webdriver**  
  - Type: HTTP Request  
  - Executes JavaScript on the browser to remove Selenium automation traces (e.g., `navigator.webdriver`, `navigator.languages`) to evade detection  
  - Input: Session ID, triggered after window resize  
  - Output: Execution result  
  - Edge cases: Script execution failure, session lost

- **Delete Session**, **Delete Session1-8**  
  - Type: HTTP Request  
  - DELETE HTTP request to Selenium to terminate the session and free resources  
  - Multiple nodes placed at different workflow exit or error points to ensure cleanup  
  - Some have `onError` set to continue to avoid breaking the workflow if deletion fails  
  - Edge cases: Session already deleted, network errors

---

#### 1.4 URL Navigation and Browser Interaction (with or without cookies)  
**Overview:**  
Navigates the Selenium browser session to the target URL. If cookies are provided, injects them into the session before navigation. Supports both authenticated scraping (via cookie injection) and unauthenticated scraping.

**Nodes Involved:**  
- If2  
- If (cookie domain check)  
- Code (cookie conversion)  
- Inject Cookie  
- Limit  
- Refresh browser  
- Go on url, Go on url1, Go on url2, Go on url3

**Node Details:**  
- **If2**  
  - Checks if `cookies` array in webhook payload is not empty to determine whether to inject cookies  
  - Routes to cookie injection flow or direct navigation flow accordingly  
  - Input: Webhook JSON cookies field  
  - Output: Branching

- **If**  
  - Verifies that the domain of the first cookie matches the target URL domain to prevent cookie injection errors  
  - Input: Cookies and target URL domain  
  - Output: Conditional routing

- **Code (cookie conversion)**  
  - Type: Code (JavaScript)  
  - Converts `sameSite` cookie attribute values from various formats to Selenium-compatible values (`Strict`, `Lax`, `None`)  
  - Extracts cookie objects from webhook input, normalizes them, outputs array of cookie JSON objects ready for injection  
  - Input: Webhook cookies array  
  - Output: Cookie JSON items for injection  
  - Edge cases: Missing or malformed cookies, unknown `sameSite` values

- **Inject Cookie**  
  - Type: HTTP Request  
  - POSTs individual cookies to Selenium session cookie endpoint to inject them into the browser context  
  - Uses templates to fill cookie fields (name, value, domain, path, secure, httpOnly, sameSite, expirationDate)  
  - On error: continues to allow injection attempts to proceed for other cookies  
  - Input: Cookie items from Code node  
  - Output: Confirmation of injection

- **Limit**  
  - Limits the rate or concurrency before refreshing browser (likely to avoid anti-bot detection)  
  - Input: After cookie injection  
  - Output: Controls flow rate

- **Refresh browser**  
  - POST to Selenium to refresh the current page after cookie injection and limit  
  - Input: After Limit  
  - Output: Page refresh confirmation

- **Go on url**, **Go on url1**, **Go on url2**, **Go on url3**  
  - POST requests to Selenium session URL endpoint to navigate browser to the intended URL  
  - Go on url, Go on url1, Go on url3 navigate to the extracted "good" URL from AI or initial input.  
  - Go on url2 navigates directly to the `Target Url` if cookies are present and cookie injection is done.  
  - On error: continue error output, retry enabled on some nodes  
  - Input: Session ID and target URL  
  - Output: Navigation confirmation  
  - Edge cases: Navigation failures, invalid URLs, session timeout

---

#### 1.5 Screenshot Capture and AI Analysis  
**Overview:**  
Captures screenshots of the rendered webpage, converts them to binary files, and sends them to OpenAI models for analysis. Extracted text or detection of blocks (e.g., WAF) guides further processing or error handling.

**Nodes Involved:**  
- Get ScreenShot 1, Get ScreenShot, Get ScreenShot 2  
- Convert to File, Convert to File1, Convert to File2  
- OpenAI1 (image analysis), OpenAI (image analysis)  
- Information Extractor1, Information Extractor2, Information Extractor  
- OpenAI Chat Model, OpenAI Chat Model1, OpenAI Chat Model2  
- If Block, If Block1

**Node Details:**  
- **Get ScreenShot (1, 2)**  
  - HTTP Request to Selenium `/screenshot` endpoint to capture base64 encoded PNG image of current browser view  
  - On error: continue error output to allow fallback  
  - Input: Session ID  
  - Output: Base64 image string

- **Convert to File (all variants)**  
  - Converts base64 image strings into n8n binary file format for input to AI nodes  
  - On error: continue error output  
  - Input: Screenshot base64 string  
  - Output: Binary image file

- **OpenAI1**, **OpenAI**  
  - Uses OpenAI GPT-4o model to analyze the screenshot image and extract relevant information about the subject  
  - Prompt instructs AI to detect if page is blocked or contains no relevant info, returning "BLOCK" if so  
  - Input: Binary image file  
  - Output: Text content with analysis or "BLOCK" flag  
  - Credentials: OpenAI API Key  
  - Edge cases: API failures, response timeouts, misinterpretation by AI

- **Information Extractor (all variants)**  
  - Langchain Information Extractor nodes parse the AI text output to extract requested data attributes (e.g., Followers, Total Stars)  
  - Uses system prompt templates to extract only relevant requested data or mark unknown values as "NA"  
  - Input: AI text content  
  - Output: Structured JSON with extracted data

- **OpenAI Chat Model (all variants)**  
  - Chat models (GPT-4o or mini versions) used to further process or refine extracted information  
  - Typically chained after Information Extractor for enhanced understanding  
  - Input: Text content or extracted data  
  - Output: Refined text or data  
  - Credentials: OpenAI API Key

- **If Block, If Block1**  
  - Checks if AI analysis returned "BLOCK" indicating that scraping was blocked by the target website  
  - Routes to session deletion and error response accordingly  
  - Input: AI analysis content  
  - Output: Conditional routing

---

#### 1.6 Result Extraction and Response Handling  
**Overview:**  
Finalizes extraction results, sends HTTP responses back to the webhook caller, and handles success or failure messages.

**Nodes Involved:**  
- Success, Success with cookie  
- Respond to Webhook2, Respond to Webhook3  
- Error, Error1, Error2, Error3

**Node Details:**  
- **Success**  
  - Responds with HTTP 200 and the extracted data in the response body from the scraping without cookies flow  
  - Input: Extracted data JSON  
  - Output: HTTP response to caller

- **Success with cookie**  
  - Responds with HTTP 200 and extracted data from scraping with cookies flow  
  - Input: Extracted data JSON  
  - Output: HTTP response

- **Respond to Webhook2**  
  - Sends HTTP 200 with JSON message indicating the request was blocked by the target website (used after detecting "BLOCK")  
  - Input: Condition from block detection  
  - Output: HTTP response

- **Respond to Webhook3**  
  - Sends HTTP 200 with JSON success message after session deletion in some error flows  
  - Input: Session deletion success  
  - Output: HTTP response

- **Error, Error1, Error2, Error3**  
  - Send HTTP error responses (404 or 500) with messages indicating various failure reasons such as cookie issues, page crash, or general errors  
  - Input: Various error conditions throughout workflow  
  - Output: HTTP response to caller

---

#### 1.7 Error and Edge Case Handling  
**Overview:**  
Various nodes handle errors gracefully by returning appropriate HTTP codes and messages, deleting sessions even on errors, and continuing workflow execution where possible to avoid hanging.

**Nodes Involved:**  
- All Delete Session nodes (with onError continue)  
- Error response nodes (Error, Error1, Error2, Error3)  
- Error can't find url  
- If nodes checking conditions leading to error responses

---

#### 1.8 Debug and Auxiliary Nodes  
**Overview:**  
Additional nodes for debugging proxy/IP and notes for users.

**Nodes Involved:**  
- Go on ip-api.com  
- Get ScreenShot 2  
- Convert to File2  
- Delete Session8  
- Sticky Notes (multiple)

**Node Details:**  
- **Go on ip-api.com**  
  - Navigates Selenium session to `https://ip-api.com/` to check current IP address used in requests  
  - Used for debugging proxy setups

- **Sticky Notes**  
  - Contain detailed instructions, warnings about proxy setup, usage notes, example requests, and project references  
  - Provide valuable context for users of the workflow  
  - Link to GitHub setup: https://github.com/Touxan/n8n-ultimate-scraper  
  - Proxy whitelist warning for GeoNode  

---

### 3. Summary Table

| Node Name                 | Node Type                       | Functional Role                                  | Input Node(s)                    | Output Node(s)                   | Sticky Note                                                                                                             |
|---------------------------|--------------------------------|-------------------------------------------------|---------------------------------|---------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| Webhook                   | Webhook                        | Receive external HTTP POST request               | -                               | Edit Fields                     |                                                                                                                         |
| Edit Fields (For testing prupose ) | Set                          | Prepare simplified fields from webhook JSON      | Webhook                         | If Target Url                   |                                                                                                                         |
| If Target Url             | If                             | Check if target URL is provided                   | Edit Fields                    | Google search Query / Create Selenium Session |                                                                                                                         |
| Google search Query       | HTTP Request                   | Perform Google search for relevant URLs           | If Target Url                  | Extract First Url Match          |                                                                                                                         |
| Extract First Url Match   | HTML Extract                   | Extract URLs from Google search results            | Google search Query             | Information Extractor            |                                                                                                                         |
| Information Extractor     | Langchain Extractor            | AI selection of best URL from extracted URLs       | Extract First Url Match         | Check if empty of NA             |                                                                                                                         |
| Check if empty of NA      | If                             | Verify if AI returned a valid URL                  | Information Extractor           | Error can't find url / Create Selenium Session |                                                                                                                         |
| Error can't find url      | Respond to Webhook             | Return 404 error when no URL found                  | Check if empty of NA            | -                               |                                                                                                                         |
| Create Selenium Session   | HTTP Request                   | Create Selenium session with custom Chrome options | If Target Url / Check if empty of NA | Resize browser window / Delete Session7 | Proxy configuration note in sticky note                                                                                 |
| Resize browser window     | HTTP Request                   | Set browser window size for consistent rendering   | Create Selenium Session         | Clean Webdriver                 |                                                                                                                         |
| Clean Webdriver           | HTTP Request                   | Remove Selenium detection traces from browser      | Resize browser window           | If2                            |                                                                                                                         |
| If2                      | If                             | Check if cookies are provided                       | Clean Webdriver                 | If / If1                       |                                                                                                                         |
| If                       | If                             | Verify cookie domain matches target domain         | If2                            | Code / If1                     |                                                                                                                         |
| Code                     | Code                          | Convert cookies to Selenium-compatible format       | Go on url2 / Go on url3         | Inject Cookie                  |                                                                                                                         |
| Inject Cookie            | HTTP Request                   | Inject cookies into Selenium browser session         | Code                           | Limit                         |                                                                                                                         |
| Limit                    | Limit                         | Control rate before page refresh                      | Inject Cookie                  | Refresh browser                |                                                                                                                         |
| Refresh browser          | HTTP Request                   | Refresh page in Selenium session                      | Limit                         | Get ScreenShot                |                                                                                                                         |
| Go on url                | HTTP Request                   | Navigate Selenium to AI-selected URL                  | If1                           | Get ScreenShot 1 / Delete Session6 |                                                                                                                         |
| Go on url1               | HTTP Request                   | Navigate Selenium to AI-selected URL (alternative)    | If1                           | Get ScreenShot 1 / Delete Session6 |                                                                                                                         |
| Go on url2               | HTTP Request                   | Navigate Selenium to target URL (with cookies)         | If3                           | Code / Delete Session4          |                                                                                                                         |
| Go on url3               | HTTP Request                   | Navigate Selenium to AI-selected URL (no cookies)      | If3                           | Code / Delete Session4          |                                                                                                                         |
| Get ScreenShot 1         | HTTP Request                   | Take screenshot of current page in Selenium           | Go on url / Go on url1          | Convert to File1               |                                                                                                                         |
| Get ScreenShot            | HTTP Request                   | Take screenshot (used in Refresh browser flow)        | Refresh browser                | Convert to File                |                                                                                                                         |
| Get ScreenShot 2         | HTTP Request                   | Screenshot for IP debug flow                            | Go on ip-api.com               | Convert to File2               |                                                                                                                         |
| Convert to File          | Convert to File               | Convert base64 image to binary for AI nodes            | Get ScreenShot                 | OpenAI / Delete Session4       |                                                                                                                         |
| Convert to File1         | Convert to File               | Convert base64 image to binary for AI nodes            | Get ScreenShot 1               | OpenAI1 / Delete Session6      |                                                                                                                         |
| Convert to File2         | Convert to File               | Convert base64 image to binary for IP debug              | Get ScreenShot 2               | Delete Session8               |                                                                                                                         |
| OpenAI                   | Langchain OpenAI Image Analysis | Analyze screenshot to extract relevant info            | Convert to File                | If Block1                     |                                                                                                                         |
| OpenAI1                  | Langchain OpenAI Image Analysis | Analyze screenshot (with cookies)                        | Convert to File1               | If Block / Delete Session6     |                                                                                                                         |
| Information Extractor1   | Langchain Extractor           | Extract structured data from AI text (with cookies)     | Delete Session                | Success                      |                                                                                                                         |
| Information Extractor2   | Langchain Extractor           | Extract structured data from AI text                      | Delete Session3               | Success with cookie           |                                                                                                                         |
| Information Extractor    | Langchain Extractor           | Extract structured data from AI text                      | OpenAI Chat Model             | Check if empty of NA          |                                                                                                                         |
| OpenAI Chat Model        | Langchain Chat Model          | Assist in URL extraction AI task                          | Extract First Url Match       | Information Extractor         |                                                                                                                         |
| OpenAI Chat Model1       | Langchain Chat Model          | Assist AI extraction with cookies                          | OpenAI1                      | Information Extractor1        |                                                                                                                         |
| OpenAI Chat Model2       | Langchain Chat Model          | Assist AI extraction without cookies                       | Information Extractor2        |                             |                                                                                                                         |
| If Block                 | If                            | Check if AI analysis indicates a blocked page            | OpenAI1                      | Delete Session1 / Delete Session |                                                                                                                         |
| If Block1                | If                            | Check if AI analysis indicates a blocked page            | OpenAI                      | Delete Session2 / Delete Session3 |                                                                                                                         |
| Delete Session           | HTTP Request                  | Delete Selenium session                                  | If Block                    | Information Extractor1        |                                                                                                                         |
| Delete Session1          | HTTP Request                  | Delete Selenium session                                  | If Block                    | Respond to Webhook3           |                                                                                                                         |
| Delete Session2          | HTTP Request                  | Delete Selenium session                                  | If Block1                   | Respond to Webhook2           |                                                                                                                         |
| Delete Session3          | HTTP Request                  | Delete Selenium session                                  | If Block1                   | Information Extractor2        |                                                                                                                         |
| Delete Session4          | HTTP Request                  | Delete Selenium session                                  | Go on url2 / Go on url3      | Error1                       |                                                                                                                         |
| Delete Session5          | HTTP Request                  | Delete Selenium session                                  | If Target Url               | Error                       |                                                                                                                         |
| Delete Session6          | HTTP Request                  | Delete Selenium session                                  | OpenAI1 / Go on url          | Error3                      |                                                                                                                         |
| Delete Session7          | HTTP Request                  | Delete Selenium session                                  | Create Selenium Session      | Error2                      |                                                                                                                         |
| Delete Session8          | HTTP Request                  | Delete Selenium session                                  | Get ScreenShot 2 / Go on ip-api.com |                         |                                                                                                                         |
| Success                  | Respond to Webhook            | Return successful extraction response                     | Information Extractor1       | -                           |                                                                                                                         |
| Success with cookie      | Respond to Webhook            | Return successful extraction response (with cookies)      | Information Extractor2       | -                           |                                                                                                                         |
| Respond to Webhook2      | Respond to Webhook            | Return 200 with "Request has been block by website"       | Delete Session2             | -                           |                                                                                                                         |
| Respond to Webhook3      | Respond to Webhook            | Return 200 success message after session deletion          | Delete Session1             | -                           |                                                                                                                         |
| Error                    | Respond to Webhook            | Return 404 error "Cookies are not for the targeted URL"    | Delete Session5             | -                           |                                                                                                                         |
| Error1                   | Respond to Webhook            | Return 500 error general                                     | Delete Session4             | -                           |                                                                                                                         |
| Error2                   | Respond to Webhook            | Return 500 error page crash                                  | Delete Session7             | -                           |                                                                                                                         |
| Error3                   | Respond to Webhook            | Return 500 error general                                     | Delete Session6             | -                           |                                                                                                                         |
| Limit                    | Limit                        | Control rate before refresh                                  | Inject Cookie              | Refresh browser             |                                                                                                                         |
| Sticky Note (multiple)   | Sticky Note                  | Documentation, warnings, setup instructions                  | -                           | -                           | See section 5 for details                                                                                                |
| Go on ip-api.com         | HTTP Request                 | Navigate to IP check site for proxy debugging                 | -                           | Get ScreenShot 2            |                                                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: unique identifier (e.g., `67d77918-2d5b-48c1-ae73-2004b32125f0`)  
   - Response Mode: Response Node  
   - No credentials needed

2. **Add “Edit Fields (For testing purpose)” Node**  
   - Type: Set  
   - Assign two string fields:  
     - `Subject` = `{{$json["body"]["subject"]}}`  
     - `Website Domaine` = `{{$json["body"]["Url"]}}`  
   - Connect Webhook → Edit Fields

3. **Add “If Target Url” Node**  
   - Type: If  
   - Condition: Check if `Target Url` in webhook body is empty (string empty)  
   - Connect Edit Fields → If Target Url

4. **Google Search Query Node**  
   - Type: HTTP Request  
   - Method: GET (or default)  
   - URL: `https://www.google.com/search?q=site:{{$json["Website Domaine"]}}+{{$json.Subject}}&...` (use the full Google query from the original)  
   - Connect If Target Url (True) → Google Search Query

5. **Extract First Url Match Node**  
   - Type: HTML Extract  
   - Operation: extractHtmlContent  
   - CSS Selector: `=a[href*="https://"][href*="{{$json["Website Domaine"]}}"]`  
   - Return attribute: href, return as array  
   - Connect Google Search Query → Extract First Url Match

6. **OpenAI Chat Model Node (for URL selection)**  
   - Type: Langchain LM Chat OpenAI  
   - Model: gpt-4o  
   - Prompt: Select relevant URL from list for subject/domain  
   - Credentials: OpenAI API Key  
   - Connect Extract First Url Match → OpenAI Chat Model

7. **Information Extractor Node**  
   - Type: Langchain Information Extractor  
   - Text: Concatenate extracted URLs  
   - System prompt: Expert extraction of relevant URL or “NA” if unknown  
   - Attribute: `Good_url_for_etract_information` required  
   - Connect OpenAI Chat Model → Information Extractor

8. **Check if empty or NA Node**  
   - Type: If  
   - Condition: `Good_url_for_etract_information` is empty or equals "NA"  
   - Connect Information Extractor → Check if empty or NA

9. **Error can't find url Node**  
   - Type: Respond to Webhook  
   - HTTP 404 response with JSON error message  
   - Connect Check if empty or NA (True) → Error can't find url

10. **Create Selenium Session Node**  
    - Type: HTTP Request POST  
    - URL: `http://selenium_chrome:4444/wd/hub/session`  
    - JSON Body: capabilities with Chrome and chromeOptions disabling automation controlled flags and user-agent set  
    - Retry on fail enabled, timeout 5000ms  
    - Connect Check if empty or NA (False) and If Target Url (False) → Create Selenium Session

11. **Resize Browser Window Node**  
    - HTTP Request POST to `/window/rect` with width=1920, height=1080, x=0, y=0  
    - Connect Create Selenium Session → Resize browser window

12. **Clean Webdriver Node**  
    - HTTP Request POST to `/execute/sync` running JS to remove webdriver traces  
    - Connect Resize browser window → Clean Webdriver

13. **If2 Node (Check cookies presence)**  
    - Type: If  
    - Condition: `cookies` array in webhook body not empty  
    - Connect Clean Webdriver → If2

14. **If Node (Check cookie domain matches target URL domain)**  
    - Condition: first cookie domain contains target URL domain  
    - Connect If2 (True) → If  
    - Connect If2 (False) → If1 (next block)

15. **Code Node (Convert Cookies)**  
    - JavaScript to convert cookies sameSite attribute to Selenium-compatible strings and output cookie array  
    - Connect If (True) → Code

16. **Inject Cookie Node**  
    - HTTP Request POST to `/cookie` endpoint per cookie JSON from Code node  
    - Continue on error enabled  
    - Connect Code → Inject Cookie

17. **Limit Node**  
    - Type: Limit (default settings)  
    - Connect Inject Cookie → Limit

18. **Refresh Browser Node**  
    - HTTP Request POST to `/refresh` endpoint with empty JSON body  
    - Connect Limit → Refresh browser

19. **Go on url Node**  
    - HTTP Request POST to `/url` endpoint with URL from AI-extracted good URL or webhook target URL depending on flow  
    - Connect Refresh browser → Go on url

20. **Screenshot Capture Nodes**  
    - HTTP Request GET `/screenshot` endpoint, multiple variants (Get ScreenShot, Get ScreenShot 1, 2)  
    - Convert base64 to binary using Convert to File nodes  
    - Connect Go on url → Get ScreenShot 1 → Convert to File1  
    - Connect Go on ip-api.com → Get ScreenShot 2 → Convert to File2 (for IP debug)

21. **OpenAI Image Analysis Nodes**  
    - Langchain OpenAI image analysis with GPT-4o, analyzing the screenshot binary  
    - Connect Convert to File nodes → OpenAI / OpenAI1 nodes

22. **Information Extractor Nodes (Final Data)**  
    - Extract requested target data fields from AI text output, set unknown fields to "NA"  
    - Connect OpenAI Chat Model / OpenAI nodes → Information Extractor1 / Information Extractor2

23. **If Block Nodes**  
    - Check if AI returned "BLOCK" content indicating scraping blocked  
    - Connect OpenAI Chat Model / OpenAI nodes → If Block1 / If Block

24. **Delete Session Nodes**  
    - HTTP DELETE to Selenium session endpoint at various exit points (success, error, block)  
    - On error continue enabled for graceful cleanup  
    - Connect If Block nodes to Delete Session nodes accordingly  
    - Connect Success and Error responses after session deletion

25. **Respond to Webhook Nodes (Success and Error)**  
    - Respond with HTTP 200 and extracted data or appropriate error messages  
    - Connect final extraction or error conditions to Respond nodes

26. **Add Sticky Notes**  
    - Add detailed documentation sticky notes at relevant workflow sections for user guidance

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                               | Context or Link                                                                                       |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| This workflow's objective is to collect data from any website page, whether it requires login or not. Example use: collect number of stars of a GitHub project. Requires Selenium container and OpenAI API key.                                             | Sticky Note on workflow start                                                                        |
| Selenium container setup instructions available at GitHub: https://github.com/Touxan/n8n-ultimate-scraper/tree/main. Docker Compose file provided.                                                                                                         | https://github.com/Touxan/n8n-ultimate-scraper/tree/main                                             |
| Residential proxy recommended for scale scraping with GeoNode: https://geonode.com/invite/98895. Note Selenium does not support proxy authentication; whitelist your server IP on GeoNode.                                                                 | Sticky Note on proxy configuration                                                                   |
| To use login cookies, collect session cookies with the dedicated browser extension at GitHub project and provide cookies in webhook request.                                                                                                             | https://github.com/Touxan/n8n-ultimate-scraper/tree/main#cookies-collection                          |
| Example curl requests for scraping with and without cookies are documented in sticky notes. Maximum of 5 target data fields supported.                                                                                                                    | Sticky Notes                                                                                         |
| Debug IP Block: A small flow navigates to https://ip-api.com/ to verify the IP address used in the Selenium session, useful when using proxies.                                                                                                          | Sticky Note near IP debug nodes                                                                      |

---

This comprehensive reference document provides a detailed understanding of the workflow’s structure, logic, and configuration details, enabling advanced users and AI agents to reproduce, modify, and troubleshoot the Ultimate Scraper Workflow for n8n effectively.