Create Jira Tickets from Streamlit Forms with Webhook & REST API

https://n8nworkflows.xyz/workflows/create-jira-tickets-from-streamlit-forms-with-webhook---rest-api-8156


# Create Jira Tickets from Streamlit Forms with Webhook & REST API

### 1. Workflow Overview

This workflow automates the creation of Jira tickets based on form submissions from a Streamlit app. It is designed for scenarios where users submit issue reports or task requests through a Streamlit interface, and these requests need to be reliably transformed into Jira issues via the Jira REST API.

The workflow consists of five main logical blocks:

- **1.1 Trigger & Input Reception:** Captures incoming HTTP POST requests from the Streamlit app via a webhook, representing ticket creation requests.
- **1.2 De-duplication and Validation:** Filters out empty, duplicate, or invalid requests to prevent unnecessary or erroneous Jira ticket creation.
- **1.3 Data Normalization and Payload Construction:** Transforms and validates the incoming data into a properly structured JSON payload matching Jira's API requirements.
- **1.4 Jira Ticket Creation:** Sends the prepared JSON payload to Jira’s REST API to create the actual ticket.
- **1.5 Response Handling:** Processes Jira’s response and sends back a user-friendly message with the newly created ticket’s key and URL to the Streamlit app.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Input Reception

- **Overview:**  
Receives and logs raw ticket creation requests from the Streamlit app via an HTTP webhook. It ensures the workflow is triggered only by legitimate POST requests containing ticket data.

- **Nodes Involved:**  
  - Webhook streamlit  
  - raw data from streamlit

- **Node Details:**

1. **Webhook streamlit**  
   - Type: Webhook  
   - Role: Entry point that listens for POST requests at the configured path (placeholder "your path here").  
   - Configuration: HTTP POST method, response mode set to "lastNode" so the final node's response is sent back to Streamlit.  
   - Inputs: External HTTP POST request from Streamlit app.  
   - Outputs: Passes received JSON payload downstream.  
   - Edge cases: Invalid HTTP methods, missing payload, or incorrect data structure may cause failures or empty downstream data.  
   - Notes: Use production webhook URL in Streamlit app; do not trigger Jira HTTP node manually.

2. **raw data from streamlit**  
   - Type: Set  
   - Role: Pass-through node to capture and optionally inspect raw incoming JSON data for debugging and transparency.  
   - Configuration: Outputs the entire incoming JSON as-is.  
   - Inputs: From Webhook streamlit node.  
   - Outputs: Passes raw JSON forward.  
   - Edge cases: None; purely pass-through.

---

#### 2.2 De-duplication and Validation

- **Overview:**  
Prevents empty or duplicate ticket creation requests by checking for essential fields and recent duplicates before allowing the workflow to proceed.

- **Nodes Involved:**  
  - anti double  
  - if for doubles  
  - blocked request

- **Node Details:**

1. **anti double**  
   - Type: Code  
   - Role: Performs initial checks to block empty or invalid requests quickly.  
   - Configuration: JavaScript code that checks if the summary field exists and is non-empty in any expected JSON location; returns a blocked flag if invalid.  
   - Inputs: Raw data from Streamlit.  
   - Outputs: JSON with either the original data or a blocked reason.  
   - Edge cases: Empty requests, missing summary field.  

2. **if for doubles**  
   - Type: If  
   - Role: Branches workflow based on whether the request is blocked or valid.  
   - Configuration: Checks if `blocked` flag is false; passes valid requests forward, sends blocked ones to rejection.  
   - Inputs: From anti double node.  
   - Outputs:  
     - True branch: for valid (not blocked) requests.  
     - False branch: for blocked/duplicate requests.  
   - Edge cases: Misclassification if key fields are missing or malformed.  

3. **blocked request**  
   - Type: Code  
   - Role: Returns a standardized JSON response indicating that the request was blocked due to duplication or invalidity.  
   - Configuration: Returns `{ ok: false, message: "Empty request blocked" }`.  
   - Inputs: From False branch of if for doubles.  
   - Outputs: Ends workflow for blocked requests.  
   - Edge cases: None; terminal node for blocked inputs.

---

#### 2.3 Data Normalization and Payload Construction

- **Overview:**  
Transforms the incoming ticket data into a Jira-compatible JSON payload, validating required fields and enriching with optional data such as priority and story points.

- **Nodes Involved:**  
  - Process streamlit data  
  - Processed data

- **Node Details:**

1. **Process streamlit data**  
   - Type: Code  
   - Role: Extracts, validates, and normalizes incoming ticket fields into a Jira REST API payload structure.  
   - Configuration:  
     - Extracts fields like projectKey, summary, issue type, description, priority, story points, and assignee from various possible JSON paths.  
     - Validates critical fields (summary, description) and blocks if missing or default placeholder values detected.  
     - Constructs the Jira fields object, including Atlassian document format for description.  
     - Handles optional fields: priority, custom story points field (customfield_10016).  
     - Omits assignee due to Jira Cloud API restrictions (needs accountId, not username).  
   - Inputs: JSON from raw data node after de-duplication.  
   - Outputs: JSON payload with `fields` object ready for Jira API.  
   - Edge cases: Missing or empty critical fields stops workflow early; invalid story points or priority handled gracefully.

2. **Processed data**  
   - Type: Set  
   - Role: Pass-through node to allow inspection of the final payload before sending to Jira.  
   - Configuration: Outputs the JSON payload unchanged.  
   - Inputs: From Process streamlit data.  
   - Outputs: Forward payload to Jira HTTP request node.  
   - Edge cases: None.

---

#### 2.4 Jira Ticket Creation

- **Overview:**  
Sends the constructed JSON payload to Jira’s REST API to create the ticket and captures the response.

- **Nodes Involved:**  
  - Jira HTTP request  
  - jira response

- **Node Details:**

1. **Jira HTTP request**  
   - Type: HTTP Request  
   - Role: Performs POST request to Jira API endpoint `/rest/api/3/issue` to create the ticket.  
   - Configuration:  
     - Method: POST  
     - URL: `https://<your-domain>.atlassian.net/rest/api/3/issue` (replace `<your-domain>` with actual Jira domain)  
     - Authentication: Jira Software Cloud API credentials (email + API token) configured in n8n credentials.  
     - Headers: Content-Type: application/json, Accept: application/json  
     - Body: Raw JSON stringified from the previous node’s payload.  
   - Inputs: Payload from Processed data node.  
   - Outputs: Jira API JSON response with created issue key and details.  
   - Edge cases: Authentication failure, API rate limits, invalid payload causing Jira errors, network timeouts.

2. **jira response**  
   - Type: Set  
   - Role: Pass-through node to hold Jira response JSON.  
   - Configuration: Outputs Jira response JSON as-is for transparency and downstream processing.  
   - Inputs: From Jira HTTP request.  
   - Outputs: Forward response to final result node.  
   - Edge cases: None.

---

#### 2.5 Response Handling

- **Overview:**  
Formats and returns a user-friendly response back to the Streamlit app, including the Jira issue key and direct URL.

- **Nodes Involved:**  
  - Result

- **Node Details:**

1. **Result**  
   - Type: Code  
   - Role: Constructs a simple JSON response containing a success flag, Jira issue key, and a clickable URL to the created ticket.  
   - Configuration:  
     - Extracts `key` from Jira response JSON.  
     - Builds URL: `https://YOURJIRAURL.atlassian.net/browse/<KEY>` (replace YOURJIRAURL with actual Jira domain).  
     - Returns `{ ok: true, jiraKey: <KEY>, url: <URL> }`.  
   - Inputs: From jira response node.  
   - Outputs: Final JSON response sent back via Webhook node to Streamlit.  
   - Edge cases: Missing or malformed Jira response key will cause broken URLs.

---

### 3. Summary Table

| Node Name            | Node Type         | Functional Role                            | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                         |
|----------------------|-------------------|-------------------------------------------|------------------------|--------------------------|---------------------------------------------------------------------------------------------------|
| Webhook streamlit    | Webhook           | Entry point, receives Streamlit requests | -                      | raw data from streamlit  | ## 1) Trigger & intake (Streamlit → n8n)\nPurpose: receive the ticket from the app and hand it to the workflow. |
| raw data from streamlit | Set              | Pass-through raw JSON payload             | Webhook streamlit      | Process streamlit data   | ## 1) Trigger & intake (Streamlit → n8n) (shared)                                                |
| anti double          | Code              | Blocks empty or invalid requests          | raw data from streamlit | if for doubles           | ## 2) De-dup guard & branching (count/IF)\nPurpose: prevent empty/invalid calls and duplicates.   |
| if for doubles       | If                | Branches on blocked flag                   | anti double            | Processed data (true) / blocked request (false) | ## 2) De-dup guard & branching (count/IF) (shared)                                               |
| blocked request      | Code              | Returns blocked response to stop workflow | if for doubles (false) | -                        | ## 2) De-dup guard & branching (count/IF) (shared)                                               |
| Process streamlit data | Code              | Normalizes and validates input for Jira   | if for doubles (true)  | Processed data           | ## 3) Normalize & build Jira payload\nPurpose: transform app fields into a valid Jira JSON.       |
| Processed data       | Set               | Pass-through validated Jira payload       | Process streamlit data | Jira HTTP request        | ## 4) Create issue in Jira\nPurpose: call Jira REST API and create the real ticket.               |
| Jira HTTP request    | HTTP Request      | Sends ticket creation request to Jira API | Processed data         | jira response            | ## 4) Create issue in Jira (shared)                                                               |
| jira response        | Set               | Captures Jira API response                  | Jira HTTP request      | Result                   | ## 5) Return result to the app\nPurpose: send a friendly response back to Streamlit.               |
| Result               | Code              | Prepares final user-friendly response      | jira response          | -                        | ## 5) Return result to the app (shared)                                                           |
| Sticky Note1         | Sticky Note       | Notes prerequisites                        | -                      | -                        | ## Required\n- Streamlit creation ticket app\n- Jira account                                     |
| Sticky Note          | Sticky Note       | Describes Trigger & Intake block           | -                      | -                        | ## 1) Trigger & intake (Streamlit → n8n) (detailed explanation)                                  |
| Sticky Note2         | Sticky Note       | Describes Normalize & Payload block        | -                      | -                        | ## 3) Normalize & build Jira payload (detailed explanation)                                     |
| Sticky Note3         | Sticky Note       | Describes Jira ticket creation block        | -                      | -                        | ## 4) Create issue in Jira (detailed explanation)                                               |
| Sticky Note4         | Sticky Note       | Describes De-dup & branching block          | -                      | -                        | ## 2) De-dup guard & branching (count/IF) (detailed explanation)                                |
| Sticky Note5         | Sticky Note       | Describes final response block               | -                      | -                        | ## 5) Return result to the app (detailed explanation)                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook node ("Webhook streamlit")**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Set your desired webhook path (e.g., "create-ticket")  
   - Response Mode: lastNode  
   - Save credentials if required (none needed here).  

2. **Create Set node ("raw data from streamlit")**  
   - Connect from "Webhook streamlit".  
   - Mode: Raw  
   - Output: Pass through incoming JSON without changes.  

3. **Create Code node ("anti double")**  
   - Connect from "raw data from streamlit".  
   - Paste the JavaScript code that:  
     - Checks if the summary field exists and is non-empty in the request body or root.  
     - Returns `{blocked: true, reason: "empty_request"}` if invalid, or passes the original item if valid.  

4. **Create If node ("if for doubles")**  
   - Connect from "anti double".  
   - Condition: Check if `$json.blocked` is false (boolean false).  
   - True output: valid requests  
   - False output: blocked requests  

5. **Create Code node ("blocked request")**  
   - Connect from "if for doubles" False output.  
   - JavaScript returns `{ok: false, message: "Empty request blocked"}`.  
   - This node ends the workflow for invalid requests.  

6. **Create Code node ("Process streamlit data")**  
   - Connect from "if for doubles" True output.  
   - Paste the JavaScript code that:  
     - Extracts fields like projectKey, summary, type, description, priority, story points, assignee from the JSON with fallback keys.  
     - Validates summary and description are non-empty and not default placeholders, returning blocked if invalid.  
     - Constructs a Jira API compliant payload with fields including the Atlassian document format for description.  
     - Adds optional priority and story points fields if present.  
     - Omits assignee due to Jira Cloud API requirements.  

7. **Create Set node ("Processed data")**  
   - Connect from "Process streamlit data".  
   - Mode: Raw  
   - Output: Pass through the normalized Jira payload JSON.  

8. **Create HTTP Request node ("Jira HTTP request")**  
   - Connect from "Processed data".  
   - HTTP Method: POST  
   - URL: `https://<your-domain>.atlassian.net/rest/api/3/issue` (replace `<your-domain>` with your Jira URL)  
   - Authentication: Use Jira Software Cloud API credentials (email + API token) configured in n8n credentials manager.  
   - Headers:  
     - Content-Type: application/json  
     - Accept: application/json (optional)  
   - Body Content Type: Raw  
   - Body: Use expression to pass JSON stringified payload (e.g., `={{ JSON.stringify($json) }}`)  
   - Send Headers: true  

9. **Create Set node ("jira response")**  
   - Connect from "Jira HTTP request".  
   - Mode: Raw  
   - Output: Pass through Jira API response JSON.  

10. **Create Code node ("Result")**  
    - Connect from "jira response".  
    - JavaScript:  
      ```js
      return [{
        json: {
          ok: true,
          jiraKey: $json.key,
          url: `https://YOURJIRAURL.atlassian.net/browse/${$json.key}`
        }
      }];
      ```  
    - Replace `YOURJIRAURL` with your actual Jira domain.  

11. **Connect "Result" node output back to "Webhook streamlit" response**  
    - This enables the HTTP webhook to return the final JSON response to Streamlit.  

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                     |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------|
| Required prerequisites: Streamlit app for ticket creation and a Jira account with API token for authentication. | Sticky Note1                                        |
| Use production webhook URL in Streamlit app, not the n8n test URL, to avoid issues.                            | Sticky Note (Trigger & Intake)                      |
| Jira Cloud requires `assignee.accountId` instead of `assignee.name`; the current workflow excludes assignee.   | Sticky Note2 (Normalize & Payload)                  |
| Jira API endpoint: `https://<your-domain>.atlassian.net/rest/api/3/issue` with POST method creates tickets.    | Sticky Note3 (Create issue in Jira)                 |
| Final response includes ticket key and direct URL to improve user experience in Streamlit UI.                  | Sticky Note5 (Return result to the app)             |

---

This documentation provides a detailed, structured reference to understand, reproduce, and extend this n8n workflow for automating Jira ticket creation from Streamlit form submissions.