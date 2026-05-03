Convert HTML to PDF and Return via Webhook using CustomJS API

https://n8nworkflows.xyz/workflows/convert-html-to-pdf-and-return-via-webhook-using-customjs-api-3331


# Convert HTML to PDF and Return via Webhook using CustomJS API

### 1. Workflow Overview

This workflow is designed to convert incoming HTML content into a styled PDF document and return it directly via a webhook response. It is ideal for use cases where an external system or user submits HTML content via an HTTP request and expects a PDF file in return without intermediate storage or manual intervention.

The workflow consists of three main logical blocks:

- **1.1 Input Reception:** Receives incoming HTTP requests containing HTML content through a webhook.
- **1.2 HTML to PDF Conversion:** Processes the HTML content using the CustomJS API to generate a PDF document.
- **1.3 Response Delivery:** Sends the generated PDF back to the original requester as a binary HTTP response.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for incoming HTTP requests on a specific webhook URL. It captures the HTML content sent by the client and initiates the workflow.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**  

  **Webhook**  
  - *Type and Role:* HTTP Webhook Trigger node; entry point for the workflow.  
  - *Configuration:*  
    - Path: `060dbacf-0feb-43d4-b4ac-44011a7dd1a4` (unique webhook endpoint).  
    - Response Mode: `responseNode` (delays response until the Respond to Webhook node executes).  
    - Accepts any HTTP method by default (can be restricted if needed).  
  - *Expressions/Variables:* Receives incoming request data, including HTML content in the body or query parameters.  
  - *Input/Output:* No input; outputs the received data to the next node.  
  - *Version Requirements:* Uses version 2 of the webhook node, supporting advanced response modes.  
  - *Potential Failures:*  
    - Invalid or missing HTML content in the request.  
    - Unauthorized or malformed requests if security is not configured.  
  - *Sub-workflow:* None.

#### 1.2 HTML to PDF Conversion

- **Overview:**  
  Converts the received HTML content into a PDF document using the CustomJS API node designed for HTML-to-PDF transformation.

- **Nodes Involved:**  
  - HTML to PDF

- **Node Details:**  

  **HTML to PDF**  
  - *Type and Role:* CustomJS PDF Toolkit node (`@custom-js/n8n-nodes-pdf-toolkit.html2Pdf`); performs HTML to PDF conversion.  
  - *Configuration:*  
    - HTML Input: Static example HTML provided (`<h1>Hello CustomJS!</h1><h2>CustomJS provides the missing toolset for your no-code projects</h2>`).  
    - Credentials: Uses CustomJS API credentials configured in n8n for authentication.  
  - *Expressions/Variables:* In the current setup, the HTML content is hardcoded; for dynamic input, this should be replaced with an expression referencing webhook data (e.g., `{{$json["body"]["html"]}}`).  
  - *Input/Output:* Receives HTML content from the webhook node; outputs binary PDF data.  
  - *Version Requirements:* Version 1 of the CustomJS node; ensure compatibility with the installed node version.  
  - *Potential Failures:*  
    - API authentication errors (invalid or expired API key).  
    - HTML content errors causing conversion failure.  
    - Network timeouts or API unavailability.  
  - *Sub-workflow:* None.

#### 1.3 Response Delivery

- **Overview:**  
  Sends the generated PDF back to the original webhook requester as a binary HTTP response, completing the request-response cycle.

- **Nodes Involved:**  
  - Respond to Webhook

- **Node Details:**  

  **Respond to Webhook**  
  - *Type and Role:* Respond to Webhook node; sends HTTP response back to the client.  
  - *Configuration:*  
    - Respond With: `binary` (sends binary data, i.e., the PDF file).  
    - No additional headers or status codes configured explicitly (defaults apply).  
  - *Expressions/Variables:* Receives binary PDF data from the HTML to PDF node.  
  - *Input/Output:* Input from HTML to PDF node; no output as this ends the workflow.  
  - *Version Requirements:* Version 1.1; supports binary responses.  
  - *Potential Failures:*  
    - Missing or malformed binary data input.  
    - Client disconnects before response is sent.  
  - *Sub-workflow:* None.

---

### 3. Summary Table

| Node Name         | Node Type                             | Functional Role           | Input Node(s) | Output Node(s)    | Sticky Note                                                                                     |
|-------------------|-------------------------------------|--------------------------|---------------|-------------------|------------------------------------------------------------------------------------------------|
| Webhook           | n8n-nodes-base.webhook               | Input Reception           | -             | HTML to PDF       | Accepts incoming HTTP requests with HTML content; triggers the workflow.                        |
| HTML to PDF       | @custom-js/n8n-nodes-pdf-toolkit.html2Pdf | HTML to PDF Conversion    | Webhook       | Respond to Webhook | Uses CustomJS API to convert HTML to PDF; requires CustomJS API key credentials.                |
| Respond to Webhook| n8n-nodes-base.respondToWebhook      | Response Delivery         | HTML to PDF   | -                 | Sends generated PDF as binary response to the original webhook request.                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Add a **Webhook** node.  
   - Set the HTTP Method to accept (default is all).  
   - Set the Path to a unique identifier (e.g., `060dbacf-0feb-43d4-b4ac-44011a7dd1a4`).  
   - Set Response Mode to `responseNode` to delay response until workflow finishes.  
   - Save the node.

2. **Create HTML to PDF Node**  
   - Add a **CustomJS HTML to PDF** node (`@custom-js/n8n-nodes-pdf-toolkit.html2Pdf`).  
   - Connect the Webhook node’s output to this node’s input.  
   - In the node parameters, set the HTML input:  
     - For testing, use static HTML such as:  
       ```html
       <h1>Hello CustomJS!</h1>
       <h2>CustomJS provides the missing toolset for your no-code projects</h2>
       ```  
     - For dynamic input, replace with an expression referencing the webhook data, e.g., `{{$json["body"]["html"]}}`.  
   - Under Credentials, select or create **CustomJS API** credentials:  
     - Obtain an API key from https://www.customjs.space after signup.  
     - In n8n, create new credentials of type CustomJS API and enter the API key.  
   - Save the node.

3. **Create Respond to Webhook Node**  
   - Add a **Respond to Webhook** node.  
   - Connect the HTML to PDF node’s output to this node’s input.  
   - Set the response type to `binary` to send the PDF file.  
   - Save the node.

4. **Connect Nodes**  
   - Ensure the connections are:  
     - Webhook → HTML to PDF → Respond to Webhook.

5. **Activate Workflow**  
   - Save and activate the workflow.  
   - Test by sending an HTTP POST request to the webhook URL with HTML content in the body (if dynamic input is configured).  
   - The response should be a PDF file generated from the HTML.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                  |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------|
| CustomJS API provides a no-code toolkit for HTML to PDF conversion integrated into n8n.         | https://www.customjs.space                       |
| Workflow requires a self-hosted n8n instance to securely manage API credentials and webhook.    | n8n documentation                               |
| Example HTML input for testing: `<h1>Hello CustomJS!</h1><h2>CustomJS provides the missing toolset for your no-code projects</h2>` | Provided in workflow description                 |
| For detailed setup of CustomJS API credentials in n8n, refer to the provided screenshots.        | Attached images in original workflow description |

---

This documentation fully describes the workflow’s structure, node configurations, and operational logic, enabling reproduction, modification, and troubleshooting by advanced users or automation agents.