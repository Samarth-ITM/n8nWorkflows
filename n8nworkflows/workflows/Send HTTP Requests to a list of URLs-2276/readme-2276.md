Send HTTP Requests to a list of URLs

https://n8nworkflows.xyz/workflows/send-http-requests-to-a-list-of-urls-2276


# Send HTTP Requests to a list of URLs

### 1. Workflow Overview

This workflow, titled **"URL Pinger"**, is designed to periodically send HTTP requests to a predefined list of URLs. Its primary use case is for monitoring or pinging multiple web endpoints at regular intervals (default every 15 minutes) to check their responsiveness or trigger downstream processes.

The workflow is logically divided into three main blocks:

- **1.1 Schedule Trigger:** Initiates the workflow execution on a recurring schedule.
- **1.2 URLs List Preparation:** Stores and prepares the list of URLs to be pinged.
- **1.3 URL Splitting and HTTP Requests:** Splits the list into individual URLs and sends HTTP requests to each one sequentially, handling errors gracefully.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger

- **Overview:**  
  This block triggers the workflow automatically at a fixed interval, defaulting to every 15 minutes. This ensures that the URL list is pinged regularly without manual initiation.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**

  - **Schedule Trigger**  
    - Type and Technical Role: `scheduleTrigger` node; initiates workflow runs on a time-based schedule.  
    - Configuration: Set to trigger every 15 minutes (`minutesInterval: 15`).  
    - Key Expressions/Variables: None.  
    - Input Connections: None (start node).  
    - Output Connections: Connected to the "URLs List" node.  
    - Version Requirements: Node version 1.2 or higher recommended for interval scheduling support.  
    - Failure Modes: Misconfiguration can cause no trigger; n8n instance downtime affects scheduling; no direct authentication issues.  
    - Sub-workflows: None.

#### 1.2 URLs List Preparation

- **Overview:**  
  This block defines the list of URLs to be pinged. It holds the URL array in a single field, preparing it for splitting.

- **Nodes Involved:**  
  - URLs List

- **Node Details:**

  - **URLs List**  
    - Type and Technical Role: `set` node; prepares and assigns static data for use downstream.  
    - Configuration: Defines a single field `urls` assigned an array of URLs: `['http://firsturl.com', 'https://secondurl.com', 'https://thirdurl.com']`. URLs must be enclosed in single quotes and comma-separated (per setup instructions).  
    - Key Expressions/Variables: Expression used to assign array: `={{ ['http://firsturl.com', 'https://secondurl.com', 'https://thirdurl.com'] }}`  
    - Input Connections: Receives trigger from Schedule Trigger node.  
    - Output Connections: Connected to the "Split Out" node.  
    - Version Requirements: Node version 3.3 or higher recommended to support array assignment with expressions.  
    - Failure Modes: Incorrect array syntax or formatting may cause expression evaluation errors; empty array results in no URLs to process.  
    - Sub-workflows: None.

#### 1.3 URL Splitting and HTTP Requests

- **Overview:**  
  This block splits the URLs array into individual items, then sends an HTTP request to each URL separately. It handles possible HTTP request errors by continuing workflow execution.

- **Nodes Involved:**  
  - Split Out  
  - HTTP Request

- **Node Details:**

  - **Split Out**  
    - Type and Technical Role: `splitOut` node; splits an array field into multiple individual items for separate processing.  
    - Configuration:  
      - Splits the `urls` field.  
      - Uses the `destinationFieldName` parameter set to `"url"` to assign each split URL to the new field `url`.  
    - Key Expressions/Variables: None, but relies on incoming `urls` array.  
    - Input Connections: Receives from "URLs List" node.  
    - Output Connections: Connected to "HTTP Request" node.  
    - Version Requirements: Version 1 or higher; ensure compatibility with field splitting features.  
    - Failure Modes: If `urls` field missing or empty, no output rows produced.  
    - Sub-workflows: None.

  - **HTTP Request**  
    - Type and Technical Role: `httpRequest` node; sends HTTP requests to a specified URL.  
    - Configuration:  
      - URL dynamically set via expression to `{{$json.url}}`, using the field created by "Split Out".  
      - HTTP method defaults to GET (not explicitly changed).  
      - No additional headers or body configured (customizable).  
      - On error option set to `continueRegularOutput` to prevent workflow failure on HTTP errors.  
    - Key Expressions/Variables: URL expression: `{{$json.url}}`.  
    - Input Connections: Receives each URL item from "Split Out".  
    - Output Connections: None (end node).  
    - Version Requirements: Version 4.2 or higher recommended for advanced HTTP request features and error handling.  
    - Failure Modes: Network issues, invalid URLs, timeouts, or HTTP error responses (e.g., 4xx, 5xx) occur but do not stop workflow due to error handling setup.  
    - Sub-workflows: None.

---

### 3. Summary Table

| Node Name      | Node Type         | Functional Role                 | Input Node(s)     | Output Node(s) | Sticky Note                                                                                                    |
|----------------|-------------------|--------------------------------|-------------------|----------------|---------------------------------------------------------------------------------------------------------------|
| Schedule Trigger | scheduleTrigger   | Initiates workflow on schedule | None              | URLs List      |                                                                                                               |
| URLs List      | set               | Defines array of URLs to ping   | Schedule Trigger  | Split Out      | *Setup tip:* Add URLs in an array with single quotes and commas; e.g., ['http://firsturl.com','https://secondurl.com'] |
| Split Out     | splitOut           | Splits URL array into items     | URLs List         | HTTP Request   |                                                                                                               |
| HTTP Request  | httpRequest        | Sends HTTP request to each URL  | Split Out         | None           | *Error handling:* Continues workflow on request errors                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n** and name it “URL Pinger”.

2. **Add a Schedule Trigger node:**
   - Set node name to `Schedule Trigger`.
   - Configure it to run every 15 minutes:
     - Under `Rule`, add interval rule: Field = minutes, MinutesInterval = 15.
   - This node has no input and triggers the workflow automatically.

3. **Add a Set node:**
   - Name it `URLs List`.
   - Connect the output of `Schedule Trigger` to this node.
   - In Parameters, add a new field with:
     - Name: `urls`
     - Type: `Array`
     - Value: Use an expression to define the URLs array, e.g.:  
       `={{ ['http://firsturl.com', 'https://secondurl.com', 'https://thirdurl.com'] }}`
   - Make sure each URL is enclosed in single quotes and separated by commas.

4. **Add a Split Out node:**
   - Name it `Split Out`.
   - Connect the output of `URLs List` to `Split Out`.
   - Configure it to split the field `urls`.
   - Set `Destination Field Name` to `url`.

5. **Add an HTTP Request node:**
   - Name it `HTTP Request`.
   - Connect the output of `Split Out` to `HTTP Request`.
   - Configure the URL field with the expression: `{{$json.url}}`.
   - Leave HTTP Method as GET (default) or customize as needed.
   - Under Error Handling, set “On Error” to `Continue` or `continueRegularOutput` to prevent workflow failure on request errors.
   - Optionally, add headers or body as per your customization needs.

6. **Activate the workflow** by switching it ON.

7. **Optional customization:**
   - Modify HTTP Request node to change method, add headers or body.
   - Adjust schedule in Schedule Trigger node as required.

---

### 5. General Notes & Resources

| Note Content                                                                                                                 | Context or Link                   |
|------------------------------------------------------------------------------------------------------------------------------|---------------------------------|
| URLs must be formatted as a JavaScript array with single quotes around each URL and commas separating them.                  | See the "URLs List" node setup. |
| Customize the HTTP Request node to change HTTP method, add headers, or include a request body for different use cases.       | Workflow description section.   |
| Schedule Trigger node supports flexible scheduling, adjust interval as needed for your monitoring frequency.                 | n8n documentation on scheduleTrigger node. |
| The HTTP Request node is configured to continue on error to ensure all URLs are processed even if some fail or time out.     | HTTP Request node configuration.|
| ![urlslist.png](fileId:796) referenced in workflow description shows example of URL list formatting.                          | Workflow description.            |