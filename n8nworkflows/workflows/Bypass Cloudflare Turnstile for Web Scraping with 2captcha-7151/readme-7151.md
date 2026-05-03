Bypass Cloudflare Turnstile for Web Scraping with 2captcha

https://n8nworkflows.xyz/workflows/bypass-cloudflare-turnstile-for-web-scraping-with-2captcha-7151


# Bypass Cloudflare Turnstile for Web Scraping with 2captcha

### 1. Workflow Overview

This n8n workflow automates the process of bypassing Cloudflare Turnstile captcha challenges for web scraping purposes using the 2Captcha service. It is designed to navigate a target website protected by Cloudflare Turnstile, extract the sitekey needed for captcha solving, submit a captcha solving task to 2Captcha, poll for the solution, and then proceed with web scraping after solving the captcha. The workflow logically divides into these main blocks:

- **1.1 Trigger and Initialization:** Entry points to start the workflow and set the target URL.
- **1.2 Webpage Retrieval and Sitekey Extraction:** Download the destination page and extract the Cloudflare Turnstile sitekey required by 2Captcha.
- **1.3 Captcha Task Creation and Polling:** Create a task on 2Captcha, wait for a response, and poll until the captcha solution is ready.
- **1.4 Handling Captcha Solution and Loop Control:** Evaluate captcha status, handle retry logic or errors, and loop back to awaiting the solution if necessary.
- **1.5 Final Automation Step:** Pass the solved captcha to Puppeteer for further browser automation actions.

### 2. Block-by-Block Analysis

---

#### 2.1 Trigger and Initialization

**Overview:**  
This block handles manual or automated workflow triggering and initializes key parameters like the destination URL for scraping.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Pass CF Turnstile check using 2Captcha (Execute Workflow)  
- Execute Workflow Trigger (Execute Workflow Trigger)  
- Set destination_url value (Set)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point for manual testing of the workflow  
  - *Parameters:* None  
  - *Connections:* Output to "Pass CF Turnstile check using 2Captcha"  
  - *Edge Cases:* No inputs; manual trigger ensures controlled start.

- **Pass CF Turnstile check using 2Captcha**  
  - *Type:* Execute Workflow  
  - *Role:* Invokes a sub-workflow or a separate workflow that implements the captcha bypass logic  
  - *Parameters:* None specified (assumed to handle internal logic)  
  - *Connections:* Input from manual trigger, output to "Puppeteer" node  
  - *Edge Cases:* Requires the invoked workflow to be correctly configured and available.

- **Execute Workflow Trigger**  
  - *Type:* Execute Workflow Trigger  
  - *Role:* Secondary or automated trigger point for the workflow, possibly for scheduled or event-driven starts  
  - *Parameters:* None  
  - *Connections:* Outputs to "Set destination_url value"  
  - *Edge Cases:* Must have proper triggering conditions; no parameters suggest it expects external trigger.

- **Set destination_url value**  
  - *Type:* Set  
  - *Role:* Defines the target URL to scrape; sets this value once per execution  
  - *Parameters:* No explicit values shown; expected to set a URL parameter for downstream use  
  - *Connections:* Input from execute workflow trigger, output to "Get destination web page"  
  - *Edge Cases:* If URL is not set or malformed, subsequent HTTP requests will fail.

---

#### 2.2 Webpage Retrieval and Sitekey Extraction

**Overview:**  
Downloads the target webpage and extracts the Cloudflare Turnstile sitekey from its HTML content, which is necessary for submitting captcha solving requests.

**Nodes Involved:**  
- Get destination web page (HTTP Request)  
- Extract Turnstile Sitekey (Code)

**Node Details:**

- **Get destination web page**  
  - *Type:* HTTP Request  
  - *Role:* Fetches the HTML content of the target URL set previously  
  - *Parameters:* URL dynamically set from "Set destination_url value" node  
  - *Connections:* Input from "Set destination_url value", output to "Extract Turnstile Sitekey"  
  - *Edge Cases:* Network errors, 404 or other HTTP errors, Cloudflare blocks could cause fetch failure.

- **Extract Turnstile Sitekey**  
  - *Type:* Code (JavaScript)  
  - *Role:* Parses the fetched HTML content to find and extract the Turnstile sitekey (usually a token in the page source)  
  - *Parameters:* Custom JavaScript code to locate the sitekey string  
  - *Connections:* Input from HTTP request node, output to "Create Captcha Task"  
  - *Edge Cases:* Sitekey missing or changed page structure could cause extraction failure or empty output.

---

#### 2.3 Captcha Task Creation and Polling

**Overview:**  
Submits the extracted sitekey and other required parameters to 2Captcha API to create a captcha-solving task, waits for some time, then polls repeatedly for the solution.

**Nodes Involved:**  
- Create Captcha Task (HTTP Request)  
- Wait 30 seconds (Wait)  
- Get Captcha Solution (HTTP Request)  
- Check Captcha Status (Switch)

**Node Details:**

- **Create Captcha Task**  
  - *Type:* HTTP Request  
  - *Role:* Sends a request to 2Captcha API to create a new Turnstile captcha solving task with sitekey and page URL  
  - *Parameters:* Includes 2Captcha API key, sitekey, page URL, and task type  
  - *Connections:* Input from "Extract Turnstile Sitekey", output to "Wait 30 seconds"  
  - *Edge Cases:* API key invalid, rate limits, network failure, malformed request.

- **Wait 30 seconds**  
  - *Type:* Wait  
  - *Role:* Pauses workflow execution for 30 seconds to allow 2Captcha time to solve the captcha  
  - *Parameters:* Fixed 30 seconds delay  
  - *Connections:* Input from "Create Captcha Task", output to "Get Captcha Solution"  
  - *Edge Cases:* Long wait periods increase latency; workflow timeout if exceeding limits.

- **Get Captcha Solution**  
  - *Type:* HTTP Request  
  - *Role:* Polls 2Captcha API for the solution to the previously created captcha task  
  - *Parameters:* Uses task ID from prior node, includes API credentials  
  - *Connections:* Input from "Wait 30 seconds", output to "Check Captcha Status"  
  - *Edge Cases:* API timeouts, no solution yet, task failure.

- **Check Captcha Status**  
  - *Type:* Switch  
  - *Role:* Evaluates the response from 2Captcha to determine if captcha is solved, pending, or failed  
  - *Parameters:* Conditions based on the status field in API response (e.g., "ready", "pending", "error")  
  - *Connections:*  
    - On success: Pass to next steps or "Puppeteer" node (not directly connected here)  
    - On pending: Loop back to "No operation - return loop" for another wait cycle  
    - On error: Send to "Raise runIndex error" node  
  - *Edge Cases:* Incorrect status parsing, infinite loops, failure handling.

---

#### 2.4 Handling Captcha Solution and Loop Control

**Overview:**  
Manages retry logic by looping the wait and polling until a solution is received or an error is raised.

**Nodes Involved:**  
- No operation - return loop (No Operation)  
- Raise runIndex error (Stop and Error)

**Node Details:**

- **No operation - return loop**  
  - *Type:* No Operation  
  - *Role:* Acts as a placeholder to loop execution back to "Wait 30 seconds" node, facilitating repeated polling  
  - *Parameters:* None  
  - *Connections:* Output loops back to "Wait 30 seconds"  
  - *Edge Cases:* Risk of infinite loop if captcha never solves; no direct timeout inside this node.

- **Raise runIndex error**  
  - *Type:* Stop and Error  
  - *Role:* Stops workflow execution with an error when captcha solving fails or an unexpected state occurs  
  - *Parameters:* Standard error raising without custom message shown  
  - *Connections:* Input from "Check Captcha Status" on error condition  
  - *Edge Cases:* Workflow terminates; no graceful recovery unless manually restarted.

---

#### 2.5 Final Automation Step

**Overview:**  
After the captcha is solved, this step uses Puppeteer to operate a headless browser session to pass the Turnstile check or continue web scraping.

**Nodes Involved:**  
- Puppeteer

**Node Details:**

- **Puppeteer**  
  - *Type:* Puppeteer Node  
  - *Role:* Automates browser actions to submit the captcha solution token, navigate the protected webpage, or scrape content  
  - *Parameters:* Not explicitly detailed; typically includes browser context, scripts, and URL  
  - *Connections:* Input from "Pass CF Turnstile check using 2Captcha" node  
  - *Edge Cases:* Browser execution errors, Puppeteer version compatibility, JavaScript errors on page, session expiration.

---

### 3. Summary Table

| Node Name                    | Node Type                     | Functional Role                            | Input Node(s)                         | Output Node(s)                      | Sticky Note               |
|------------------------------|-------------------------------|--------------------------------------------|-------------------------------------|-----------------------------------|---------------------------|
| When clicking ‘Test workflow’ | Manual Trigger               | Manual entry point to start workflow        | -                                   | Pass CF Turnstile check using 2Captcha |                           |
| Pass CF Turnstile check using 2Captcha | Execute Workflow           | Executes sub-workflow for captcha bypass    | When clicking ‘Test workflow’       | Puppeteer                          |                           |
| Execute Workflow Trigger      | Execute Workflow Trigger     | Automated/secondary trigger                  | -                                   | Set destination_url value          |                           |
| Set destination_url value     | Set                         | Defines the URL to scrape                    | Execute Workflow Trigger            | Get destination web page           |                           |
| Get destination web page      | HTTP Request                | Fetches webpage HTML                         | Set destination_url value           | Extract Turnstile Sitekey          |                           |
| Extract Turnstile Sitekey     | Code                        | Extracts Turnstile sitekey from HTML        | Get destination web page            | Create Captcha Task                |                           |
| Create Captcha Task           | HTTP Request                | Creates 2Captcha solving task                 | Extract Turnstile Sitekey           | Wait 30 seconds                   |                           |
| Wait 30 seconds              | Wait                        | Waits to allow captcha solving               | Create Captcha Task / No operation - return loop | Get Captcha Solution              |                           |
| Get Captcha Solution          | HTTP Request                | Polls 2Captcha for captcha solution          | Wait 30 seconds                    | Check Captcha Status               |                           |
| Check Captcha Status          | Switch                      | Checks captcha status and controls flow      | Get Captcha Solution               | No operation - return loop / Raise runIndex error |                           |
| No operation - return loop    | No Operation                | Loop control to retry polling                 | Check Captcha Status               | Wait 30 seconds                   |                           |
| Raise runIndex error          | Stop and Error              | Stops workflow on captcha failure             | Check Captcha Status               | -                                 |                           |
| Puppeteer                    | Puppeteer                   | Automates browser actions after captcha solved | Pass CF Turnstile check using 2Captcha | -                                 |                           |
| Sticky Note1                 | Sticky Note                 | (Empty content)                               | -                                   | -                                 |                           |
| Sticky Note2                 | Sticky Note                 | (Empty content)                               | -                                   | -                                 |                           |
| Sticky Note3                 | Sticky Note                 | (Empty content)                               | -                                   | -                                 |                           |
| Sticky Note                  | Sticky Note                 | (Empty content)                               | -                                   | -                                 |                           |
| Sticky Note9                 | Sticky Note                 | (Empty content)                               | -                                   | -                                 |                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Manual Trigger** node named "When clicking ‘Test workflow’". No parameters needed.  
   - Add an **Execute Workflow Trigger** node named "Execute Workflow Trigger". No parameters needed.

2. **Create Initialization Node:**  
   - Add a **Set** node named "Set destination_url value". Configure to set a string field `destination_url` with the target website URL for scraping. Set this node to execute only once per run.

3. **Create Webpage Retrieval Node:**  
   - Add an **HTTP Request** node named "Get destination web page". Configure its URL parameter to reference the `destination_url` from the previous Set node (`{{$node["Set destination_url value"].json["destination_url"]}}`).  
   - Set method to GET and ensure full response body is returned.

4. **Create Sitekey Extraction Node:**  
   - Add a **Code** node named "Extract Turnstile Sitekey".  
   - Write JavaScript code to parse the HTML response from "Get destination web page" and extract the Cloudflare Turnstile sitekey from the page source. For example, use regex or DOM parsing to find the `data-sitekey` or equivalent attribute.

5. **Create Captcha Task Submission Node:**  
   - Add an **HTTP Request** node named "Create Captcha Task".  
   - Configure to POST to 2Captcha API endpoint to create a Turnstile captcha task. Include parameters:  
     - API key (from credentials)  
     - `sitekey` (from "Extract Turnstile Sitekey")  
     - `pageurl` (destination URL)  
     - Task type set to Turnstile  
   - Set to execute once per run.

6. **Add Wait Node:**  
   - Add a **Wait** node named "Wait 30 seconds" configured to pause workflow execution for 30 seconds.

7. **Create Captcha Solution Polling Node:**  
   - Add an **HTTP Request** node named "Get Captcha Solution".  
   - Configure to GET the captcha solution from 2Captcha API using task ID returned by the previous node. Use API credentials.  
   - Set to execute once per run.

8. **Add Status Checking Switch Node:**  
   - Add a **Switch** node named "Check Captcha Status".  
   - Configure conditional branches:  
     - If status is "ready", proceed to next steps.  
     - If status is "pending", loop back to wait node.  
     - If status is error or other, direct to error node.

9. **Create Loop Control Nodes:**  
   - Add a **No Operation** node named "No operation - return loop". Connect it to loop back to the "Wait 30 seconds" node for polling.  
   - Add a **Stop and Error** node named "Raise runIndex error" to terminate workflow on failure.

10. **Add Puppeteer Automation Node:**  
    - Add a **Puppeteer** node named "Puppeteer" to automate browser actions after captcha is solved. Configure browser context, navigation steps, and pass the captcha solution token as needed.

11. **Create Execute Workflow Node:**  
    - Add an **Execute Workflow** node named "Pass CF Turnstile check using 2Captcha" to invoke the sub-workflow handling the captcha bypass logic (this may be the main workflow or a separate one). Connect the manual trigger node output to this node.

12. **Connect Nodes in Order:**  
    - Manual trigger → Execute Workflow (captcha bypass) → Puppeteer  
    - Execute Workflow Trigger → Set destination URL → Get destination web page → Extract Turnstile Sitekey → Create Captcha Task → Wait 30 seconds → Get Captcha Solution → Check Captcha Status  
    - Check Captcha Status outputs:  
      - On ready: proceed to Puppeteer or end  
      - On pending: No operation → Wait 30 seconds (loop)  
      - On error: Raise runIndex error (stop workflow)

13. **Credential Setup:**  
    - Configure HTTP Request nodes calling 2Captcha API with valid 2Captcha API key credentials.  
    - Configure Puppeteer node with appropriate browser environment and any required authentication.  
    - Ensure HTTP Request nodes fetching target webpage handle headers/cookies if necessary.

14. **Default Values and Constraints:**  
    - Wait node fixed at 30 seconds delay.  
    - Set node defines only one URL.  
    - Switch node must accurately parse status values from 2Captcha API response.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                 |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------|
| This workflow is specifically tailored for bypassing Cloudflare Turnstile captchas, which differ from standard reCAPTCHA. | Workflow focus                                  |
| 2Captcha API documentation: https://2captcha.com/api                                                          | For configuring captcha task creation & polling |
| Puppeteer node in n8n requires proper browser environment setup; consult n8n Puppeteer docs for details.        | https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-puppeteer/ |
| Ensure API keys and credentials are stored securely in n8n credentials manager.                                 | Security best practice                           |
| The looping mechanism based on polling may increase execution time; consider adding maximum retry limits.      | Performance and reliability note                 |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and publicly available.