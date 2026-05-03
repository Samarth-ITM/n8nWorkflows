Monetize Workflows with x402 Payment Protocol and 1Shot API

https://n8nworkflows.xyz/workflows/monetize-workflows-with-x402-payment-protocol-and-1shot-api-5389


# Monetize Workflows with x402 Payment Protocol and 1Shot API

### 1. Workflow Overview

This workflow, titled **"Monetize Workflows with x402 Payment Protocol and 1Shot API"**, is designed to monetize any n8n workflow by requiring and verifying stablecoin payments via the x402 payment protocol. It acts as a payment gateway that accepts an encoded payment payload via a webhook, validates it, simulates a payment authorization using the 1Shot API, and upon confirmation, allows further premium content workflows to proceed. The workflow is structured into the following logical blocks:

- **1.1 Input Reception and Header Validation:** Receive incoming HTTP requests and verify the presence of the required `X-PAYMENT` header.
- **1.2 Payment Payload Decoding and Verification:** Decode the base64-encoded payment payload, parse it into JSON, and check for required payment fields.
- **1.3 Payment Simulation:** Use the 1Shot API to simulate the payment authorization transaction to verify its validity.
- **1.4 Final Payment Submission and Response Handling:** Submit the verified payment asynchronously, respond with success or failure to the client, and provide placeholders for premium content workflow integration.
- **1.5 Error Handling and Responses:** Return appropriate HTTP error responses for missing headers, invalid payment format, or failed payment verification.
- **1.6 Documentation and Usage Notes:** Sticky notes within the workflow provide usage instructions, example commands, and protocol references.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Header Validation

- **Overview:**  
  This block receives HTTP requests on a webhook endpoint and checks whether the `X-PAYMENT` header is present in the request headers.

- **Nodes Involved:**  
  - Webhook  
  - Check for presence of X-HEADER  
  - Response: Missing or Invalid Payment Headers  

- **Node Details:**  

  - **Webhook**  
    - Type: Webhook (HTTP Entry Point)  
    - Configuration: Listens on path `92c5ca23-99a7-437d-85da-84aef8bd2a25`  
    - Role: Entry point for incoming payment requests  
    - Input: External HTTP request  
    - Output: Request data to "Check for presence of X-HEADER"  
    - Edge cases: No request received; invalid HTTP method (only GET/POST implicitly allowed)  
    - Notes: Responds based on downstream nodes with `responseMode: responseNode`  

  - **Check for presence of X-HEADER**  
    - Type: If (Conditional)  
    - Configuration: Checks if header `x-payment` exists and is non-empty (`exists` operator)  
    - Key Expression: `{{$json.headers['x-payment']}}`  
    - Input: Webhook output  
    - Output:  
      - True branch: proceeds to decode and validate payment  
      - False branch: responds with error "Missing or Invalid Payment Headers"  
    - Edge cases: Header present but empty or malformed  

  - **Response: Missing or Invalid Payment Headers**  
    - Type: Respond to Webhook  
    - Configuration: HTTP 402 response code, JSON body  
      ```json
      {
        "x402Version": "1",
        "error": "X-PAYMENT header has incorrect format"
      }
      ```  
    - Input: False branch from header check  
    - Output: HTTP response to client  

#### 1.2 Payment Payload Decoding and Verification

- **Overview:**  
  Decodes the base64-encoded `x-payment` header, parses it into JSON, and checks that the payload contains a valid payment signature.

- **Nodes Involved:**  
  - Decode & Validate X-Payment  
  - Ensure Well Formatted Payment Payload  
  - Response: Missing or Invalid Payment Headers (re-used)  

- **Node Details:**  

  - **Decode & Validate X-Payment**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Decodes `x-payment` header from base64  
      - Parses decoded string as JSON  
      - Attaches parsed JSON as `decodedXPayment` on the item  
      - On error (invalid base64 or JSON), returns an error object `{ error: "invalid token format" }`  
    - Input: Header presence confirmed data  
    - Output: To "Ensure Well Formatted Payment Payload"  
    - Edge cases: Malformed base64, invalid JSON, exceptions in parsing  

  - **Ensure Well Formatted Payment Payload**  
    - Type: If (Conditional)  
    - Configuration: Checks existence of `decodedXPayment.payload.signature` (non-empty)  
    - Input: Decoded payment data  
    - Output:  
      - True branch: proceeds to simulate payment  
      - False branch: responds with error "Missing or Invalid Payment Headers"  
    - Edge cases: Payload missing signature or empty signature field  

  - **Response: Missing or Invalid Payment Headers**  
    - Reused node for error response if signature missing  

#### 1.3 Payment Simulation

- **Overview:**  
  Simulates the payment authorization using the 1Shot API to verify payment details before actual submission.

- **Nodes Involved:**  
  - Simulate Payment  
  - On Successful Payment Simulation  
  - Response: Payment Invalid  

- **Node Details:**  

  - **Simulate Payment**  
    - Type: 1Shot API node (simulate operation)  
    - Configuration:  
      - Uses payment authorization fields extracted from `decodedXPayment` JSON (fields: from, to, value, validAfter, validBefore, nonce, signature)  
      - Contract method ID specified for simulation  
      - Credentials: 1Shot OAuth2 API (named "1Shot account")  
    - Input: Validated payment payload  
    - Output: Simulation result to "On Successful Payment Simulation"  
    - Edge cases: API call failure, invalid credentials, network timeout, malformed parameters  

  - **On Successful Payment Simulation**  
    - Type: If (Conditional)  
    - Configuration: Checks if simulation result `success` field equals `"true"` (string)  
    - Input: Simulation result  
    - Output:  
      - True branch: proceeds to final submission  
      - False branch: responds with payment invalid error  
    - Edge cases: unexpected simulation response shape, missing `success` field  

  - **Response: Payment Invalid**  
    - Type: Respond to Webhook  
    - Configuration: HTTP 402, JSON body  
      ```json
      {
        "x402Version": "1",
        "error": "X-PAYMENT header did not verify"
      }
      ```  
    - Input: False branch from simulation success check  
    - Output: HTTP response  

#### 1.4 Final Payment Submission and Response Handling

- **Overview:**  
  Submits and waits for the actual payment transaction confirmation via the 1Shot API, then returns success or failure responses. This is the point where premium workflows can be integrated in place of the success response.

- **Nodes Involved:**  
  - 1Shot API Submit & Wait  
  - Response: 200 - Payment Successful  
  - Response: Payment Invalid (reused)  

- **Node Details:**  

  - **1Shot API Submit & Wait**  
    - Type: 1Shot API synchronous node  
    - Configuration:  
      - Uses the same payment authorization fields from the validated payment JSON  
      - Contract method ID same as simulation  
      - Credentials: 1Shot OAuth2 API (named "1Shot account")  
    - Input: Successful simulation branch  
    - Output:  
      - True branch: response 200 success  
      - False branch: response payment invalid  
    - Edge cases: API submission failure, network errors, invalid signature rejection  

  - **Response: 200 - Payment Successful**  
    - Type: Respond to Webhook  
    - Configuration: HTTP 200 with plain text body: `"Payment Received!"`  
    - Input: Success branch from final payment submission  
    - Output: HTTP response to client  
    - Notes: Placeholder for replacing with premium content workflow response  

  - **Response: Payment Invalid**  
    - Reused for failure response on submission failure  

#### 1.5 Error Handling and Responses

- **Overview:**  
  Covers the error responses returned during the workflow when payment headers are missing, payload malformed, or payment fails verification.

- **Nodes Involved:**  
  - Response: Missing or Invalid Payment Headers  
  - Response: Payment Invalid  

- **Node Details:**  
  Both nodes return HTTP 402 with JSON error messages describing the specific failure cause.

#### 1.6 Documentation and Usage Notes (Sticky Notes)

- **Nodes Involved:**  
  - Sticky Note (x402 Payment Endpoint)  
  - Sticky Note1 (Any ERC-20 on any EVM)  
  - Sticky Note2 (Put your workflow down here)  
  - Sticky Note3 (Example Curl Command)  
  - Sticky Note4 (Ensure Required Payment Details)  

- **Content Summary:**  
  - Explains the x402 payment endpoint functionality with links to protocol and tutorial video.  
  - Notes that any ERC-20 token with transferWithAuthorization can be accepted via 1Shot API.  
  - Suggests replacing the success response node with actual premium content workflow logic.  
  - Provides a sample curl command to test the webhook with a valid base64 payment payload header.  
  - Highlights the importance of validating required payment details such as minimum amounts.

---

### 3. Summary Table

| Node Name                         | Node Type                | Functional Role                           | Input Node(s)                   | Output Node(s)                                 | Sticky Note                                                                                                                   |
|----------------------------------|--------------------------|-----------------------------------------|--------------------------------|-----------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Webhook                          | Webhook                  | Entry point for payment requests        | (external HTTP request)         | Check for presence of X-HEADER                 | Sticky Note3: Example Curl Command                                                                                            |
| Check for presence of X-HEADER   | If                       | Validates presence of X-PAYMENT header  | Webhook                        | Decode & Validate X-Payment (true) <br> Response: Missing or Invalid Payment Headers (false) |                                                                                                                              |
| Decode & Validate X-Payment      | Code                     | Decodes and parses payment payload      | Check for presence of X-HEADER | Ensure Well Formatted Payment Payload           | Sticky Note4: Ensure Required Payment Details                                                                                 |
| Ensure Well Formatted Payment Payload | If                   | Checks payment payload signature exists | Decode & Validate X-Payment    | Simulate Payment (true) <br> Response: Missing or Invalid Payment Headers (false)              |                                                                                                                              |
| Simulate Payment                 | 1Shot API (simulate)     | Simulates payment authorization         | Ensure Well Formatted Payment Payload | On Successful Payment Simulation               | Sticky Note1: Any ERC-20 on any EVM                                                                                          |
| On Successful Payment Simulation | If                       | Checks simulation success                | Simulate Payment               | 1Shot API Submit & Wait (true) <br> Response: Payment Invalid (false)                           |                                                                                                                              |
| 1Shot API Submit & Wait          | 1Shot API synchronous    | Submits and waits for actual payment    | On Successful Payment Simulation | Response: 200 - Payment Successful (true) <br> Response: Payment Invalid (false)                | Sticky Note2: Put your workflow down here                                                                                     |
| Response: Missing or Invalid Payment Headers | Respond to Webhook      | Responds with HTTP 402 for header errors | Check for presence of X-HEADER (false) <br> Ensure Well Formatted Payment Payload (false) | (HTTP response)                                |                                                                                                                              |
| Response: Payment Invalid        | Respond to Webhook        | Responds with HTTP 402 for payment failure | On Successful Payment Simulation (false) <br> 1Shot API Submit & Wait (false) | (HTTP response)                                |                                                                                                                              |
| Response: 200 - Payment Successful | Respond to Webhook      | Responds with HTTP 200 on success       | 1Shot API Submit & Wait (true) | (HTTP response)                                | Sticky Note2: Put your workflow down here                                                                                     |
| Sticky Note                     | Sticky Note               | Documentation - x402 Payment Endpoint   |                                |                                               | Content about x402 protocol and tutorial link                                                                                |
| Sticky Note1                    | Sticky Note               | Documentation - 1Shot API ERC-20 tokens |                                |                                               | Notes on accepting any ERC-20 token via 1Shot API                                                                            |
| Sticky Note2                    | Sticky Note               | Documentation - Integration instructions |                                |                                               | Instructions to replace success response with premium content workflow                                                       |
| Sticky Note3                    | Sticky Note               | Documentation - Example curl command    |                                |                                               | Provides a curl command example for testing webhook                                                                         |
| Sticky Note4                    | Sticky Note               | Documentation - Payment detail validation |                                |                                               | Notes on ensuring required payment details such as minimum amounts                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node:**
   - Type: Webhook  
   - Set path to `92c5ca23-99a7-437d-85da-84aef8bd2a25` (or your preferred unique identifier)  
   - Set Response Mode to `responseNode`  
   - No authentication required (or configure as needed)  

2. **Create an If Node "Check for presence of X-HEADER":**
   - Connect Webhook main output to this node input  
   - Set condition: Check if header `x-payment` exists and is not empty  
   - Expression for left value: `{{$json.headers['x-payment']}}`  
   - Operator: Exists (string operation)  
   - If true: proceed to decoding  
   - If false: send error response  

3. **Create a Code Node "Decode & Validate X-Payment":**
   - Connect true output of previous If node  
   - JavaScript code:  
     ```js
     try {
         const xPaymentHeader = $input.first().json.headers['x-payment'];
         const decodedXPayment = Buffer.from(xPaymentHeader, 'base64').toString('utf-8');
         const decodedXPaymentJson = JSON.parse(decodedXPayment);
         $input.first().json.decodedXPayment = decodedXPaymentJson;
         return $input.all();
     } catch (error) {
         return { error: "invalid token format" };
     }
     ```  
   - Output passes to next validation node  

4. **Create an If Node "Ensure Well Formatted Payment Payload":**
   - Connect output of Code node  
   - Condition: Check that `decodedXPayment.payload.signature` exists and is not empty  
   - Expression: `{{$json.decodedXPayment.payload.signature}}`  
   - Operator: Exists (string operation)  
   - True: proceed to simulate payment  
   - False: send error response  

5. **Create a 1Shot API Node "Simulate Payment":**
   - Connect true output from previous If node  
   - Operation: `simulate`  
   - Contract Method ID: `b63aaaa1-059d-4c38-928a-33ad17d66827` (specific to payment contract)  
   - Parameters (use expressions):  
     ```json
     {
       "from": "{{ $json.decodedXPayment.payload.authorization.from }}",
       "to": "{{ $json.decodedXPayment.payload.authorization.to }}",
       "value": "{{ $json.decodedXPayment.payload.authorization.value }}",
       "validAfter": "{{ $json.decodedXPayment.payload.authorization.validAfter }}",
       "validBefore": "{{ $json.decodedXPayment.payload.authorization.validBefore }}",
       "nonce": "{{ $json.decodedXPayment.payload.authorization.nonce }}",
       "signature": "{{ $json.decodedXPayment.payload.signature }}"
     }
     ```  
   - Credentials: Set 1Shot OAuth2 API credentials (create and link your 1Shot account OAuth2 credential)  

6. **Create an If Node "On Successful Payment Simulation":**
   - Connect output of Simulate Payment node  
   - Condition: Check if `$json.success` equals string `"true"`  
   - True: proceed to actual payment submission  
   - False: respond with payment invalid error  

7. **Create a 1Shot API synchronous Node "1Shot API Submit & Wait":**
   - Connect true output from simulation success check  
   - Operation: Submit and wait (sync)  
   - Contract Method ID: same as simulation  
   - Parameters: same as simulation, use expressions from prior decoded payment JSON but referencing the appropriate node (in example, `Ensure Well Formatted Payment Payload`)  
   - Credentials: Same 1Shot OAuth2 API credentials  

8. **Connect the output of "1Shot API Submit & Wait":**
   - True branch: Connect to Respond to Webhook node returning HTTP 200 with body `"Payment Received!"`  
   - False branch: Connect to Respond to Webhook node returning HTTP 402 with JSON error `"X-PAYMENT header did not verify"`  

9. **Create Respond to Webhook Node "Response: Missing or Invalid Payment Headers":**
   - Connect false outputs from the header presence check and well-formatted payload check  
   - Response code: 402  
   - Response body (JSON):  
     ```json
     {
       "x402Version": "1",
       "error": "X-PAYMENT header has incorrect format"
     }
     ```  

10. **Create Respond to Webhook Node "Response: Payment Invalid":**
    - Connect false outputs from simulation success and payment submission  
    - Response code: 402  
    - Response body (JSON):  
      ```json
      {
        "x402Version": "1",
        "error": "X-PAYMENT header did not verify"
      }
      ```  

11. **Create Respond to Webhook Node "Response: 200 - Payment Successful":**
    - Connect true output of final payment submission node  
    - Response code: 200  
    - Response body (Text): `"Payment Received!"`  
    - This node can be replaced with your premium workflow logic as needed  

12. **Optional: Add Sticky Notes:**
    - Add notes with documentation, example curl commands, and instructions to guide users and maintainers.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                               |
|----------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Learn more about the [x402 payment protocol](https://www.x402.org/).                               | Official x402 protocol website                                |
| Watch the [YouTube tutorial video](https://youtu.be/m3ThthLtj3g) explaining the workflow.          | Video tutorial for setup and usage                            |
| You can accept any ERC-20 token on any EVM-compatible chain using the 1Shot API's transferWithAuthorization methods. | Sticky Note1 content                                          |
| Replace the success response node with your own premium content workflow to deliver paid content. | Sticky Note2 content                                          |
| Example curl command to test the webhook with a base64-encoded payment header:                      | Sticky Note3 content; example curl command below:             |
| ```sh                                                                                              |                                                               |
| curl -X GET \                                                                                      |                                                               |
|   https://<your-webhook-url>/92c5ca23-99a7-437d-85da-84aef8bd2a25 \                               |                                                               |
|   -H "x-payment: YOUR-BASE64-ENCODED-PAYMENT-PAYLOAD" \                                           |                                                               |
|   -H "User-Agent: CustomUserAgent/1.0" \                                                         |                                                               |
|   -H "Accept: application/json"                                                                   |                                                               |
| ```                                                                                              |                                                               |
| Use this block to validate required payment details like minimum payment amount before accepting payment. | Sticky Note4 content                                         |

---

**Disclaimer:** This document and the workflow it describes are produced from an automated n8n workflow export. All data handled is legal and public, with strict adherence to content policies.