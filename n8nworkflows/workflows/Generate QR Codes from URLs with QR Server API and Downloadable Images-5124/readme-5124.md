Generate QR Codes from URLs with QR Server API and Downloadable Images

https://n8nworkflows.xyz/workflows/generate-qr-codes-from-urls-with-qr-server-api-and-downloadable-images-5124


# Generate QR Codes from URLs with QR Server API and Downloadable Images

### 1. Workflow Overview

This workflow provides an automated process to generate QR codes from URLs submitted via a web form, using the QR Server API. It is designed for scenarios where end users need to quickly convert URLs into downloadable QR code images with customizable dimensions.

**Logical Blocks:**

- **1.1 Input Reception:** Captures URL and size parameters from the user via a web form.
- **1.2 QR Code Generation:** Calls the QR Server API to generate the QR code image based on the submitted parameters.
- **1.3 Download Delivery:** Returns the generated QR code image back to the user for download.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Collects user input including the URL to encode and the desired width and height of the QR code image through a form submission trigger.

**Nodes Involved:**  
- On form submission

**Node Details:**  

- **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point; listens for form submissions from an end-user interface titled "QR Code Generator".  
  - Configuration:  
    - Form title: "QR Code Generator"  
    - Fields:  
      - URL (required text input)  
      - width (required number input, placeholder 1000)  
      - height (required number input, placeholder 1000)  
    - Form description: "This mini application is to generate QR code for the provided URL."  
  - Inputs: None (trigger node)  
  - Outputs: Passes JSON containing submitted fields to the next node  
  - Edge cases:  
    - Missing or invalid URL (should be caught by required field constraint)  
    - Non-numeric or missing width/height (required number fields enforce type)  
  - Version: 2.2

---

#### 1.2 QR Code Generation

**Overview:**  
Generates a QR code image by calling the external QR Server API with user-provided URL and size parameters.

**Nodes Involved:**  
- HTTP Request

**Node Details:**  

- **HTTP Request**  
  - Type: HTTP Request  
  - Role: Sends a GET request to the QR Server API to generate the QR code image dynamically.  
  - Configuration:  
    - URL constructed via expression:  
      `http://api.qrserver.com/v1/create-qr-code/?data={{ $json.URL }}&size={{ $json.width }}x{{ $json.height }}`  
    - No additional options or authentication required  
  - Inputs: Receives JSON data from "On form submission" node  
  - Outputs: Returns the QR code image as binary data  
  - Edge cases:  
    - API downtime or network errors  
    - Invalid URL leading to unexpected API response or error image  
    - Very large sizes potentially causing timeouts or slow responses  
  - Version: 4.2

---

#### 1.3 Download Delivery

**Overview:**  
Delivers the generated QR code image back to the user as a downloadable file, concluding the interaction.

**Nodes Involved:**  
- Form

**Node Details:**  

- **Form**  
  - Type: Form Response  
  - Role: Sends back the QR code image in the HTTP response, allowing the user to download it directly.  
  - Configuration:  
    - Operation: completion (indicating workflow end)  
    - Respond with: returnBinary (sending binary file data)  
    - Completion title: "QR code generation"  
    - Completion message: "QR code of {{ $json.URL }} should have been downloaded to your device." (dynamic message referencing the submitted URL)  
  - Inputs: Receives binary image data from "HTTP Request" node  
  - Outputs: None (terminal node)  
  - Edge cases:  
    - Browser incompatibility handling binary downloads  
    - Large image size causing slow download or timeouts  
  - Version: 1

---

### 3. Summary Table

| Node Name         | Node Type          | Functional Role                | Input Node(s)     | Output Node(s) | Sticky Note                                                  |
|-------------------|--------------------|-------------------------------|-------------------|----------------|--------------------------------------------------------------|
| On form submission | Form Trigger       | Captures user input (URL, size) | None              | HTTP Request   | ### Form submission  This is a form submission node for end user inputting the URL to be converted to QR code. |
| HTTP Request      | HTTP Request       | Calls QR Server API for QR code | On form submission | Form           | ### QR code generation  This is a node to call API from QR Server API (https://goqr.me/api/).                 |
| Form              | Form Response      | Returns QR code image to user  | HTTP Request      | None           | ### Form ending  This is a form ending node for end user downloading the QR code.                             |
| Sticky Note       | Sticky Note        | Documentation note             | None              | None           |                                                              |
| Sticky Note1      | Sticky Note        | Documentation note             | None              | None           |                                                              |
| Sticky Note2      | Sticky Note        | Documentation note             | None              | None           |                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a "Form Trigger" node:**
   - Name: `On form submission`
   - Set the form title to "QR Code Generator".
   - Add the following form fields:  
     - Text field, label: `URL`, required  
     - Number field, label: `width`, placeholder: `1000`, required  
     - Number field, label: `height`, placeholder: `1000`, required  
   - Add a form description: "This mini application is to generate QR code for the provided URL."
   - Save the webhook URL generated by this node for testing.

3. **Add an "HTTP Request" node:**
   - Name: `HTTP Request`
   - Set HTTP Method to `GET`.
   - In the URL field, enter an expression:  
     ```
     http://api.qrserver.com/v1/create-qr-code/?data={{ $json.URL }}&size={{ $json.width }}x{{ $json.height }}
     ```
   - No authentication or additional headers are required.
   - Connect the output of `On form submission` node to this node.

4. **Add a "Form" node:**
   - Name: `Form`
   - Set operation to `completion`.
   - Set "Respond With" to `returnBinary`.
   - Set "Completion Title" to "QR code generation".
   - Set "Completion Message" to:  
     ```
     QR code of {{ $json.URL }} should have been downloaded to your device.
     ```
   - Connect the output of `HTTP Request` node to this node.

5. **Activate the workflow** and test by submitting the form URL with valid inputs.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                          |
|----------------------------------------------------------------------------------------------------|-----------------------------------------|
| QR Server API documentation and usage details: https://goqr.me/api/                               | External API documentation               |
| This workflow demonstrates simple, no-auth API integration to generate QR codes dynamically.       | Workflow use case                        |
| Consider adding input validation or sanitization for URLs to improve robustness.                   | Enhancement suggestion                   |
| Binary response handling requires compatible client browsers for smooth downloading experience.    | User experience consideration            |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.