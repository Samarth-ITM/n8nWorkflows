Set alert on the website changes

https://n8nworkflows.xyz/workflows/set-alert-on-the-website-changes-426


# Set alert on the website changes

### 1. Workflow Overview

This workflow monitors changes on a specific webpage by checking for the presence or absence of the phrase "Out Of Stock" in the page content. It is designed primarily to alert when an item comes back in stock by detecting that the phrase "Out Of Stock" is no longer found on the page.

The workflow is organized into the following logical blocks:

- **1.1 Scheduled Trigger:** Regularly initiates the workflow on an hourly basis.
- **1.2 Webpage Content Retrieval:** Fetches the webpage content via an HTTP request.
- **1.3 Content Evaluation:** Checks the retrieved content for the presence of the string "Out Of Stock".
- **1.4 Alert Notifications:** Sends a Discord message depending on whether the phrase is found or not.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:** This block triggers the workflow automatically every hour to perform the webpage check without manual intervention.
- **Nodes Involved:** `Cron`
- **Node Details:**

  - **Cron**
    - **Type and Role:** Scheduler node; triggers the workflow at set intervals.
    - **Configuration:** Set to trigger every hour (`mode: everyHour`).
    - **Expressions/Variables:** None.
    - **Input/Output:** No input; outputs to the `HTTP Request` node.
    - **Version:** v1.
    - **Potential Failures:** Rare; possible issues with system clock or timezone misalignment.
    - **Other:** Timezone set globally for the workflow to `America/Los_Angeles`.

#### 2.2 Webpage Content Retrieval

- **Overview:** Fetches the raw content of the target webpage as a string to be analyzed.
- **Nodes Involved:** `HTTP Request`
- **Node Details:**

  - **HTTP Request**
    - **Type and Role:** Makes an HTTP GET request to the URL specified by the user.
    - **Configuration:** 
      - URL field is empty by default; user must set the target webpage URL.
      - Response format set to `string` to obtain raw HTML or text content.
      - No additional HTTP options configured.
    - **Expressions/Variables:** The response body is accessed as `{{$node["HTTP Request"].json["data"]}}` in the subsequent IF node.
    - **Input/Output:** Receives input from `Cron`; outputs to `IF`.
    - **Version:** v1.
    - **Potential Failures:** Network issues, invalid URL, timeouts, or unexpected response formats.
    - **Notes:** URL must be set by the user before activation.

#### 2.3 Content Evaluation

- **Overview:** Determines if the webpage content contains the phrase "Out Of Stock", which indicates the item is currently unavailable.
- **Nodes Involved:** `IF`
- **Node Details:**

  - **IF**
    - **Type and Role:** Conditional node; routes workflow based on string containment test.
    - **Configuration:** Checks if the string `"Out Of Stock"` is contained in the HTTP response content.
      - Condition: `{{$node["HTTP Request"].json["data"]}}` contains `"Out Of Stock"`.
    - **Expressions/Variables:** Uses the output from the `HTTP Request` node.
    - **Input/Output:** Input from `HTTP Request`; two outputs:
      - **True Branch:** (contains "Out Of Stock") — no downstream node connected.
      - **False Branch:** (does not contain "Out Of Stock") — connects to `Discord1`.
    - **Version:** v1.
    - **Potential Failures:** Expression errors if response format changes; empty or null response.
    - **Notes:** The true branch currently has no connected node, meaning no alert is sent if the phrase is found.

#### 2.4 Alert Notifications

- **Overview:** Sends Discord webhook messages to notify about the stock status detected on the webpage.
- **Nodes Involved:** `Discord`, `Discord1`
- **Node Details:**

  - **Discord**
    - **Type and Role:** Sends a Discord message when the "Out Of Stock" phrase is found.
    - **Configuration:**
      - Message text: `"value found"`.
      - Webhook URL: empty by default; user must provide their Discord webhook URL.
    - **Input/Output:** Not connected in the current workflow (no input).
    - **Version:** v1.
    - **Potential Failures:** Invalid or missing webhook URL, network issues.
    - **Notes:** User must provide webhook URL and optionally customize the message.

  - **Discord1**
    - **Type and Role:** Sends a Discord message when the "Out Of Stock" phrase is not found (item back in stock).
    - **Configuration:**
      - Message text: `"value not found"`.
      - Webhook URL: empty by default; user must provide their Discord webhook URL.
    - **Input/Output:** Receives input from the `IF` node's false output.
    - **Version:** v1.
    - **Potential Failures:** Invalid or missing webhook URL, network problems.
    - **Notes:** This is the primary alert node for stock availability changes.

---

### 3. Summary Table

| Node Name    | Node Type           | Functional Role               | Input Node(s)      | Output Node(s) | Sticky Note                            |
|--------------|---------------------|------------------------------|--------------------|----------------|--------------------------------------|
| Cron         | Cron                | Scheduled hourly trigger     | -                  | HTTP Request   |                                      |
| HTTP Request | HTTP Request        | Fetch webpage content        | Cron               | IF             | Set the URL for the HTTP Request node |
| IF           | If                  | Check if "Out Of Stock" text | HTTP Request       | Discord1       |                                      |
| Discord      | Discord             | Send alert if phrase found   | - (not connected)  | -              | Set your Webhook URL and Messages for the Discord nodes |
| Discord1     | Discord             | Send alert if phrase missing | IF (false output)  | -              | Same as Discord node note            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** and name it appropriately (e.g., "Website check").

2. **Add a `Cron` node:**
   - Set the mode to `Every Hour`.
   - This node will trigger the workflow hourly.
   - No credentials needed.
   - Position optionally: leftmost.

3. **Add an `HTTP Request` node:**
   - Connect the `Cron` node output to this node input.
   - Set the `HTTP Request` node parameters:
     - HTTP Method: GET (default).
     - URL: Set to the target webpage URL you want to monitor.
     - Response Format: `String`.
     - No authentication unless necessary.
   - This node fetches the raw webpage content.

4. **Add an `IF` node:**
   - Connect the `HTTP Request` node output to the `IF` node input.
   - Configure the condition:
     - Field: `{{$node["HTTP Request"].json["data"]}}`
     - Operation: `Contains`
     - Value: `Out Of Stock`
   - This node checks if the phrase is present in the page content.

5. **Add two `Discord` nodes:**

   - **Discord node (for "value found"):**
     - No connection from the IF node in this workflow, but you may connect the IF node's true output here if you want notification when the phrase is found.
     - Set the message text to `"value found"`.
     - Provide your Discord webhook URL in the `Webhook URL` field.
     - Ensure valid credentials or authentication if required by Discord.

   - **Discord1 node (for "value not found"):**
     - Connect the IF node’s false output (meaning phrase not found) to this node input.
     - Set the message text to `"value not found"`.
     - Provide your Discord webhook URL.
     - This sends alert when the item is back in stock.

6. **Set the workflow timezone:**
   - In workflow settings, set timezone to `America/Los_Angeles` or your preferred timezone.

7. **Activate the workflow:**
   - Make sure URLs and webhook URLs are set correctly.
   - Test the workflow manually or wait for scheduled trigger.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                      |
|----------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Set the URL for the HTTP Request node and your Webhook URL and Messages for the Discord nodes.    | Sticky note applies to HTTP Request, Discord nodes  |
| This workflow is useful to monitor stock availability changes by detecting textual changes on pages.| Description provided with the workflow               |

---

This documentation provides a complete, stepwise understanding of the "Website check" workflow, enabling users or automation agents to reproduce, modify, and maintain it effectively.