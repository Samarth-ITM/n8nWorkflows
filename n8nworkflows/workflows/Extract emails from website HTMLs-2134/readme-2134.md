Extract emails from website HTMLs

https://n8nworkflows.xyz/workflows/extract-emails-from-website-htmls-2134


# Extract emails from website HTMLs

### 1. Workflow Overview

This workflow is designed to scrape and extract email addresses from any given website URL via an API built in n8n. It targets marketing professionals or developers needing bulk email contact harvesting from websites for outreach or lead generation. The workflow receives a website URL through a webhook, fetches the HTML content, extracts email addresses using regex, removes duplicates, and returns the unique email list as a response.

Logical blocks:

- **1.1 Input Reception:** Accepts the website URL via webhook query parameter.
- **1.2 Website Data Retrieval:** Fetches the HTML content of the specified website.
- **1.3 Email Extraction:** Uses regex to extract email addresses from the HTML content.
- **1.4 Data Processing:** Splits extracted emails into individual items and filters duplicates.
- **1.5 Response Handling:** Returns the processed email list to the webhook caller, or a fallback success message if no emails found.
- **1.6 Documentation Notes:** Sticky notes explain usage, API calling, and general instructions.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Receives an HTTP request containing a website URL as a query parameter. This URL is the target for email scraping.

**Nodes Involved:**  
- Webhook

**Node Details:**  
- **Webhook**  
  - Type: `Webhook` (Entry point node)  
  - Role: Listens for incoming HTTP GET requests at a specified path (`ea568868-5770-4b2a-8893-700b344c995e`).  
  - Configuration: Expects a query parameter named `Website` containing the URL to scrape.  
  - Response Mode: Delegates response to a downstream node (`Respond to Webhook`).  
  - Inputs: External HTTP request  
  - Outputs: Passes the request data forward containing the `Website` query parameter  
  - Edge Cases: Missing or malformed `Website` URLs may cause HTTP request failures downstream; no explicit validation is implemented here.  
  - Version: 1.1

#### 2.2 Website Data Retrieval

**Overview:**  
Fetches the entire HTML content from the provided website URL for analysis.

**Nodes Involved:**  
- Get the website data

**Node Details:**  
- **Get the website data**  
  - Type: `HTTP Request`  
  - Role: Performs a GET request to the URL provided in the webhook query parameter.  
  - Configuration: URL dynamically set via expression `={{ $json.query['Website'] }}`. No additional authentication or headers.  
  - Inputs: Receives URL from the webhook node.  
  - Outputs: Returns the raw HTML content in the `data` field of the JSON output.  
  - Retry on Fail: Enabled (will retry on failure).  
  - Edge Cases: Invalid URLs, non-HTTP(S) schemes, timeout, or website blocking request could cause failures or empty responses.  
  - Version: 4.1

#### 2.3 Email Extraction

**Overview:**  
Extracts email addresses from the website's HTML content using a regex pattern.

**Nodes Involved:**  
- Extract the emails found

**Node Details:**  
- **Extract the emails found**  
  - Type: `Set`  
  - Role: Uses JavaScript regex to find all email addresses in the HTML content string.  
  - Configuration: Assigns a new field `Email` as an array of all matched email addresses extracted by regex: `{{$json.data.match(/(?:[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,})/g)}}`  
  - Inputs: HTML content from the HTTP Request node.  
  - Outputs: JSON containing an `Email` array field with extracted emails or `null` if none found.  
  - Edge Cases: If no emails found, the `Email` field will be `null` or empty, which downstream nodes must handle. Regex might miss obfuscated or unusual email formats.  
  - Version: 3.3

#### 2.4 Data Processing

**Overview:**  
Splits the array of emails into individual items and removes duplicates for clean output.

**Nodes Involved:**  
- Split Out  
- If contains email  
- Remove Duplicates

**Node Details:**  
- **Split Out**  
  - Type: `Split Out`  
  - Role: Transforms the array in `Email` field into individual JSON items, each containing a single email address.  
  - Config: Splits on field `Email`.  
  - Inputs: JSON with `Email` array.  
  - Outputs: Multiple JSON items each with one email.  
  - Edge Cases: If `Email` is empty or null, no output items are produced.

- **If contains email**  
  - Type: `If`  
  - Role: Checks if the `Email` field is non-empty (not empty string).  
  - Config: Condition: `Email` field is not empty.  
  - Inputs: Items from Split Out.  
  - Outputs: Passes items to next node if emails exist; otherwise, workflow ends or bypasses downstream nodes.  
  - Edge Cases: If no emails exist, workflow skips duplicate removal and response with emails.

- **Remove Duplicates**  
  - Type: `Remove Duplicates`  
  - Role: Removes duplicate email entries from the split list.  
  - Config: No additional configuration; defaults to removing duplicates based on entire item content.  
  - Inputs: Items passing the `If contains email` check.  
  - Outputs: Cleaned list of unique emails.  
  - On Error: Continues regular output (does not fail workflow).  
  - Retry: Enabled.  
  - Edge Cases: Duplicate emails with minor variations (case differences) may or may not be detected depending on n8n default behavior.

#### 2.5 Response Handling

**Overview:**  
Sends the final email list back to the caller of the webhook or a success message if no emails found.

**Nodes Involved:**  
- Respond to Webhook

**Node Details:**  
- **Respond to Webhook**  
  - Type: `Respond to Webhook`  
  - Role: Returns the extracted unique emails as HTTP response to the original API call.  
  - Config: Default output; responds with incoming data from `Remove Duplicates`.  
  - Inputs: Unique emails from `Remove Duplicates`.  
  - Outputs: HTTP response to the API user.  
  - Edge Cases: If no emails found (empty input), the webhook returns `"workflow successfully executed."` as default message.  
  - Version: 1

#### 2.6 Documentation Notes

**Overview:**  
Sticky notes provide user instructions, usage details, and contextual info for the workflow.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2

**Node Details:**  
- **Sticky Note**  
  - Content: "* Scraping emails from websites using an api"  
  - Role: High-level description.

- **Sticky Note1**  
  - Content:  
    ```
    * Call the webhook using a query parameter eg 

    http://localhost:5678/webhook/ea568868-5770-4b2a-8893-7e?Website=https://mailsafi.com

    HTTP request rest the query Website and gets the emails therein
    ```  
  - Role: Explains how to call the API with example URL.

- **Sticky Note2**  
  - Content:  
    ```
    # How to scrap emails from websites

    This workflow shows how you can quickly build an Email scraping API using n8n.
    Usage
    Copy the webhook URL to your browser and add a query parameter eg {{$n8nhosteingurl/webhook/ea568868-5770-4b2a-8893-700b344c995e?Website=https://mailsafi.com
    This will return the email address on the website or if there is no email, the response will be "workflow successfully executed"

    # Make sure to use HTTP:// for your domains

    Otherwise, you may get an error. 
    ```  
  - Role: Provides detailed usage notes and warnings about URL formatting.

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                    | Input Node(s)          | Output Node(s)        | Sticky Note                                                                                                          |
|---------------------|---------------------|----------------------------------|-----------------------|-----------------------|----------------------------------------------------------------------------------------------------------------------|
| Webhook             | Webhook             | Entry point; receives website URL| -                     | Get the website data   | * Call the webhook using a query parameter eg http://localhost:5678/webhook/ea568868-5770-4b2a-8893-7e?Website=https://mailsafi.com |
| Get the website data | HTTP Request        | Fetches website HTML content     | Webhook               | Extract the emails found| * Call the webhook using a query parameter eg http://localhost:5678/webhook/ea568868-5770-4b2a-8893-7e?Website=https://mailsafi.com |
| Extract the emails found | Set              | Extracts emails using regex      | Get the website data   | Split Out             |                                                                                                                      |
| Split Out           | Split Out           | Splits email array into items    | Extract the emails found| If contains email      |                                                                                                                      |
| If contains email   | If                  | Checks if emails exist            | Split Out             | Remove Duplicates      |                                                                                                                      |
| Remove Duplicates    | Remove Duplicates    | Removes duplicate emails          | If contains email     | Respond to Webhook     |                                                                                                                      |
| Respond to Webhook  | Respond to Webhook  | Returns final email list to caller| Remove Duplicates      | -                     |                                                                                                                      |
| Sticky Note         | Sticky Note         | Usage description                 | -                     | -                     | * Scraping emails from websites using an api                                                                         |
| Sticky Note1        | Sticky Note         | API calling example and usage    | -                     | -                     | * Call the webhook using a query parameter eg http://localhost:5678/webhook/ea568868-5770-4b2a-8893-7e?Website=https://mailsafi.com |
| Sticky Note2        | Sticky Note         | Detailed usage instructions      | -                     | -                     | # How to scrap emails from websites ... Make sure to use HTTP:// for your domains Otherwise, you may get an error.     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook node:**  
   - Type: Webhook  
   - Set path to `ea568868-5770-4b2a-8893-700b344c995e`  
   - Response Mode: `Response Node`  
   - No authentication needed  
   - This node will accept HTTP GET requests with a query parameter `Website`.

2. **Create an HTTP Request node:**  
   - Name: Get the website data  
   - Type: HTTP Request  
   - Method: GET  
   - URL: Expression `={{ $json.query["Website"] }}` to dynamically get the website URL from the webhook query parameter.  
   - Enable "Retry on Fail" for robustness.  
   - Connect Webhook node output to this node.

3. **Create a Set node:**  
   - Name: Extract the emails found  
   - Type: Set  
   - Add a new field:  
     - Name: `Email`  
     - Type: Array  
     - Value (Expression): `{{$json.data.match(/(?:[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,})/g)}}`  
   - Connect HTTP Request node output to this node.

4. **Create a Split Out node:**  
   - Name: Split Out  
   - Type: Split Out  
   - Field to Split Out: `Email`  
   - Connect Set node output to this node.

5. **Create an If node:**  
   - Name: If contains email  
   - Type: If  
   - Condition: `Email` field is not empty (use "String -> not empty")  
   - Connect Split Out node output to this node.

6. **Create a Remove Duplicates node:**  
   - Name: Remove Duplicates  
   - Type: Remove Duplicates  
   - No special configuration needed.  
   - Set "On Error" to continue regular output.  
   - Enable "Retry on Fail".  
   - Connect If node's true output to this node.

7. **Create a Respond to Webhook node:**  
   - Name: Respond to Webhook  
   - Type: Respond to Webhook  
   - Connect Remove Duplicates node output to this node.  
   - This node sends the final result back to the original webhook caller.

8. **Connect the workflow for the no-email scenario:**  
   - Connect the If node's false output directly to Respond to Webhook node with a message like:  
     - Add a Set node before Respond to Webhook to set JSON `{ "message": "workflow successfully executed." }`  
     - Connect If node's false output to this Set node, then to Respond to Webhook.

9. **Add Sticky Notes for documentation:**  
   - Add notes explaining workflow purpose, usage instructions, and API call examples.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                   |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| Make sure to use HTTP:// for your domains; HTTPS is not guaranteed to work due to certificate or redirects     | Sticky Note2 warning in workflow                                 |
| This workflow enables building a custom email scraping API for marketing or sales lead generation with n8n      | Usage description in Sticky Notes and workflow overview         |
| Example API call: `http://localhost:5678/webhook/ea568868-5770-4b2a-8893-700b344c995e?Website=https://mailsafi.com` | Sticky Note1 provides example usage URL                          |

---

This completes the comprehensive documentation and analysis of the "Extract emails from website HTMLs" workflow.