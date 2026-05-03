QR Code Generator via Webhook

https://n8nworkflows.xyz/workflows/qr-code-generator-via-webhook-4596


# QR Code Generator via Webhook

---

### 1. Workflow Overview

The **QR Code Generator via Webhook** workflow is designed to generate a QR code image dynamically based on input data received through a webhook POST request. It is primarily targeted for use cases where external systems or users need to generate QR codes on-demand by sending JSON data to a designated API endpoint.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception:** Listens for incoming HTTP POST requests containing the data to encode into a QR code.
- **1.2 QR Code Generation:** Calls an external QR code generation API to create the QR code image based on the received data.
- **1.3 Response Delivery:** Sends the generated QR code image back to the original requester as the webhook response.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Listens for incoming HTTP POST requests on a configurable webhook path. It expects the request body to contain a JSON object with a `data` property, which holds the string or information to be encoded into the QR code.

- **Nodes Involved:**  
  - Receive Data Webhook  
  - Note for Webhook Trigger (sticky note)

- **Node Details:**

  - **Receive Data Webhook**  
    - *Type:* Webhook Trigger  
    - *Role:* Entry point to the workflow, receives POST requests.  
    - *Configuration:*  
      - HTTP Method: POST  
      - Path: `generate-qr` (customizable)  
      - Response Mode: `responseNode` (defers response to a dedicated respond node)  
    - *Key Expressions:* None; expects JSON body with `data` property accessible as `$json.body.data`.  
    - *Inputs:* External HTTP POST requests.  
    - *Outputs:* Passes received JSON data downstream.  
    - *Edge Cases / Potential Failures:*  
      - Missing or malformed JSON body.  
      - Missing `data` property in the JSON payload.  
      - Unauthorized or unexpected HTTP methods (only POST allowed).  
    - *Sub-workflow:* None.

  - **Note for Webhook Trigger**  
    - *Type:* Sticky Note  
    - *Role:* Documentation within the workflow for users.  
    - *Content Summary:* Explains the webhook’s purpose, expected JSON payload structure, and that the webhook path is adjustable.  
    - *Sticky Note Coverage:* Positioned next to the webhook node for contextual help.

#### 1.2 QR Code Generation

- **Overview:**  
  Uses an HTTP GET request to call the external QR Server API, generating a QR code image. The API URL includes query parameters for the size of the QR code and the data to encode, the latter fetched dynamically from the webhook input.

- **Nodes Involved:**  
  - Generate QR Code  
  - Note for QR Code Generation (sticky note)

- **Node Details:**

  - **Generate QR Code**  
    - *Type:* HTTP Request  
    - *Role:* Fetches the QR code image from a public API.  
    - *Configuration:*  
      - Method: GET  
      - URL: `https://api.qrserver.com/v1/create-qr-code/?size=150x150&data={{ $json.body.data }}`  
      - No authentication required.  
    - *Key Expressions:*  
      - URL uses expression `{{ $json.body.data }}` to dynamically insert input data.  
    - *Inputs:* Receives JSON data from the webhook node.  
    - *Outputs:* Returns raw image data (PNG) of the QR code from the API.  
    - *Edge Cases / Potential Failures:*  
      - Invalid or empty `data` parameter causing API to generate an invalid or empty QR code.  
      - API downtime or network errors causing request failure or timeouts.  
      - URL encoding issues if `data` contains special characters.  
    - *Version-specific Requirements:* HTTP Request node version 4.2 or higher to support expression syntax used.

  - **Note for QR Code Generation**  
    - *Type:* Sticky Note  
    - *Role:* Provides contextual information about the API call, including how the `data` and `size` parameters are set and that size can be adjusted.  
    - *Sticky Note Coverage:* Positioned near the Generate QR Code node.

#### 1.3 Response Delivery

- **Overview:**  
  Sends the QR code image data received from the external API back to the original webhook caller as the HTTP response.

- **Nodes Involved:**  
  - Respond with QR Code  
  - Note for Webhook Response (sticky note)

- **Node Details:**

  - **Respond with QR Code**  
    - *Type:* Respond to Webhook  
    - *Role:* Final node that returns the QR code image data to the webhook client.  
    - *Configuration:*  
      - Respond with: All incoming items (passes the full data from the previous node)  
      - No additional headers or transformations configured by default.  
    - *Inputs:* Receives the HTTP response from the QR code API node.  
    - *Outputs:* Sends HTTP response to webhook caller.  
    - *Edge Cases / Potential Failures:*  
      - If the previous node fails or returns no data, the response may be empty or invalid.  
      - Content-Type header may need to be set explicitly if the client expects image data; default behavior depends on n8n version.  
    - *Version-specific Requirements:* Respond to Webhook node version 1.2 or higher to support `respondWith` options.  

  - **Note for Webhook Response**  
    - *Type:* Sticky Note  
    - *Role:* Explains that this node returns the QR code image or URL back to the caller and that additional nodes can be inserted before it to extend functionality (e.g., saving or emailing the image).  
    - *Sticky Note Coverage:* Positioned near the Respond node.

---

### 3. Summary Table

| Node Name             | Node Type               | Functional Role                        | Input Node(s)        | Output Node(s)         | Sticky Note                                                                                                      |
|-----------------------|-------------------------|-------------------------------------|----------------------|------------------------|------------------------------------------------------------------------------------------------------------------|
| Note for Webhook Trigger | Sticky Note             | Documentation for webhook trigger   |                      | Receive Data Webhook    | This node listens for incoming POST requests. It expects a JSON body with a 'data' property (or 'sampleData' as currently configured) which will be encoded into the QR code. You can easily adjust the webhook path. |
| Receive Data Webhook   | Webhook                 | Receives POST requests with data    | Note for Webhook Trigger | Generate QR Code        | This node listens for incoming POST requests. It expects a JSON body with a 'data' property (or 'sampleData' as currently configured) which will be encoded into the QR code. You can easily adjust the webhook path. |
| Note for QR Code Generation | Sticky Note             | Documentation for QR code generation | Generate QR Code      |                        | This node makes an HTTP GET request to the QR Server API to generate the QR code image. The 'data' parameter in the URL is populated from the incoming webhook. The 'size' parameter can be adjusted here.           |
| Generate QR Code       | HTTP Request            | Calls external API to generate QR   | Receive Data Webhook  | Respond with QR Code    | This node makes an HTTP GET request to the QR Server API to generate the QR code image. The 'data' parameter in the URL is populated from the incoming webhook. The 'size' parameter can be adjusted here.           |
| Note for Webhook Response | Sticky Note             | Documentation for webhook response   | Respond with QR Code  |                        | This node sends the response from the QR Server API (which is typically the QR code image data itself, or a URL to it if you change the API call) back to the original caller of the webhook. You can insert other nodes before this to save the image, send it via email, etc. |
| Respond with QR Code   | Respond to Webhook      | Sends QR code image back to caller  | Generate QR Code      |                        | This node sends the response from the QR Server API (which is typically the QR code image data itself, or a URL to it if you change the API call) back to the original caller of the webhook. You can insert other nodes before this to save the image, send it via email, etc. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node**  
   - Type: Webhook  
   - Name: `Receive Data Webhook`  
   - HTTP Method: POST  
   - Path: `generate-qr` (or customize as desired)  
   - Response Mode: `Response Node` (defers response to a dedicated node)  
   - No authentication needed.  
   - Purpose: Accept JSON POST requests with a body containing a `data` property.

2. **Add an HTTP Request Node**  
   - Type: HTTP Request  
   - Name: `Generate QR Code`  
   - Method: GET  
   - URL: `https://api.qrserver.com/v1/create-qr-code/?size=150x150&data={{ $json.body.data }}`  
     - Use expression editor to insert `{{ $json.body.data }}` from the webhook JSON body.  
   - No authentication required.  
   - Purpose: Request the QR code image from the external API based on incoming data.

3. **Add a Respond to Webhook Node**  
   - Type: Respond to Webhook  
   - Name: `Respond with QR Code`  
   - Configuration:  
     - Respond With: All Incoming Items (passes through the HTTP response from previous node)  
   - Purpose: Send the QR code image back to the original webhook caller.

4. **Connect the Nodes**  
   - Connect `Receive Data Webhook` → `Generate QR Code`  
   - Connect `Generate QR Code` → `Respond with QR Code`

5. **Optional: Add Sticky Notes for Documentation**  
   - Add sticky notes near each main node to describe its purpose and any configuration notes.  
   - Examples:  
     - Near `Receive Data Webhook`: Explain expected JSON input and webhook path.  
     - Near `Generate QR Code`: Explain URL construction and size parameter.  
     - Near `Respond with QR Code`: Explain response behavior and extensibility.

6. **Save and Activate the Workflow**  
   - Confirm all nodes are properly configured.  
   - Activate the workflow to start listening for requests.

---

### 5. General Notes & Resources

| Note Content                                                                                                              | Context or Link                                    |
|---------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------|
| This workflow uses the public QR Server API (https://goqr.me/api/) to generate QR codes dynamically based on input data. | QR Code API documentation                          |
| You can customize the webhook path and the QR code size by modifying the respective node parameters.                      | Workflow configuration options                     |
| Additional processing (e.g., saving the QR code image or emailing it) can be inserted between QR Code generation and response nodes. | Workflow extensibility suggestions                 |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.