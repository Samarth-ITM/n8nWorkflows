Monetize Your Private LLM Models with x402 & Ollama

https://n8nworkflows.xyz/workflows/monetize-your-private-llm-models-with-x402---ollama-6597


# Monetize Your Private LLM Models with x402 & Ollama

### 1. Workflow Overview

This workflow enables monetization of private large language model (LLM) inference requests hosted via Ollama, using the x402 payment protocol integrated with the 1Shot API for ERC-20 token payments on EVM-compatible chains. It is designed to receive AI query requests over a webhook, verify payment authorization conveyed through a custom `x-payment` HTTP header, simulate and submit a payment transaction for validation, then route the query to a private Ollama LLM instance only if payment verification succeeds. The response from the LLM is returned as a successful webhook response.

**Logical Blocks:**

- **1.1 Input Reception and Payment Header Check**  
  Receives incoming webhook POST requests and checks for the presence of the required `x-payment` header.

- **1.2 Payment Decoding and Validation**  
  Decodes the base64-encoded `x-payment` header, validates the JSON payload structure, and ensures required payment fields exist.

- **1.3 Payment Simulation and Execution**  
  Simulates the payment authorization on the 1Shot API, then submits the payment transaction if simulation passes.

- **1.4 Payment Outcome Handling**  
  Based on payment validation success or failure, routes to appropriate webhook response nodes.

- **1.5 Private Model Inference via Ollama**  
  Upon successful payment, forwards the user query to a private Ollama-hosted LLM for inference.

- **1.6 Webhook Response**  
  Returns either payment error responses or the successful inference result to the client.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Payment Header Check

- **Overview:**  
  Listens for incoming POST requests on a webhook, then confirms the presence of the custom `x-payment` header which carries payment authorization data.

- **Nodes Involved:**  
  - Webhook  
  - Check for presence of X-HEADER  
  - Response: Missing or Invalid Payment Headers

- **Node Details:**  
  - **Webhook**  
    - Type: Webhook  
    - Configured to accept POST requests at path `92c5ca23-99a7-437d-85da-84aef8bd2a25`.  
    - Response mode set to respond from a downstream node.  
    - Input: External HTTP request  
    - Output: Request data including headers and body  
    - Edge Cases: Missing or malformed request body; unexpected HTTP methods.

  - **Check for presence of X-HEADER**  
    - Type: If node  
    - Checks if the HTTP header `x-payment` exists and is non-empty in the request headers.  
    - Input: Output of Webhook node  
    - Output:  
      - True branch if header present → continues workflow.  
      - False branch → triggers `Response: Missing or Invalid Payment Headers` node.  
    - Edge Cases: Header case sensitivity or header name typos.

  - **Response: Missing or Invalid Payment Headers**  
    - Type: Respond to Webhook  
    - Returns HTTP 402 status with JSON error message indicating missing or malformed `x-payment` header.  
    - Input: False branch of header check.  
    - Output: HTTP response to client.

---

#### 2.2 Payment Decoding and Validation

- **Overview:**  
  Decodes the base64-encoded `x-payment` header to JSON, parses it to extract payment details, and validates presence of required fields like `signature`.

- **Nodes Involved:**  
  - Decode & Validate X-Payment  
  - Ensure Well Formatted Payment Payload  
  - Response: Missing or Invalid Payment Headers (on failure)

- **Node Details:**  
  - **Decode & Validate X-Payment**  
    - Type: Code (JavaScript) node  
    - Decodes base64 `x-payment` header to UTF-8 string, parses JSON, and appends parsed data as `decodedXPayment` in the workflow data.  
    - Returns error object if decoding or parsing fails.  
    - Inputs: True branch data from header check.  
    - Outputs: Parsed payment JSON or error.  
    - Edge Cases: Invalid base64 format, malformed JSON, missing header value.

  - **Ensure Well Formatted Payment Payload**  
    - Type: If node  
    - Checks that the decoded payment JSON contains a non-empty `signature` field, confirming payload completeness.  
    - True branch → proceeds to payment simulation.  
    - False branch → triggers error response about missing or invalid payment headers.  
    - Inputs: Output of Decode & Validate node.  
    - Outputs: Payment payload validation pass/fail.  
    - Edge Cases: Missing required fields, unexpected data types.

---

#### 2.3 Payment Simulation and Execution

- **Overview:**  
  Simulates the payment authorization on 1Shot API to pre-validate the payment, then if valid, actually submits the payment transaction synchronously.

- **Nodes Involved:**  
  - Simulate Payment  
  - On Successful Payment Simulation  
  - 1Shot API Submit & Wait  
  - Response: Payment Invalid (on failure)

- **Node Details:**  
  - **Simulate Payment**  
    - Type: 1Shot API node (simulate operation)  
    - Sends payment parameters extracted from `decodedXPayment` to simulate payment authorization.  
    - Inputs: Payment JSON from validation node.  
    - Outputs: Simulation result JSON including `success` boolean.  
    - Credentials: 1Shot OAuth2 API account.  
    - Edge Cases: API rate limits, network timeouts, invalid parameters.

  - **On Successful Payment Simulation**  
    - Type: If node  
    - Checks if simulation response indicates success (`success == true`).  
    - True branch → proceeds to submit payment synchronously.  
    - False branch → routes to payment invalid response.  
    - Inputs: Simulate Payment output.  
    - Outputs: Payment success/fail branch.

  - **1Shot API Submit & Wait**  
    - Type: 1Shot API synchronous node  
    - Submits the actual payment transaction with parameters from decoded payment JSON.  
    - Inputs: True branch from payment simulation success.  
    - Outputs: Payment submission result, which triggers the private model inference flow on success or payment invalid response on failure.  
    - Credentials: Same 1Shot OAuth2 API account.  
    - Edge Cases: Transaction failures, network issues, signature verification errors.

  - **Response: Payment Invalid**  
    - Type: Respond to Webhook  
    - Returns HTTP 402 with JSON error indicating payment verification failure.  
    - Input from failed payment simulation or submission branches.

---

#### 2.4 Private Model Inference via Ollama

- **Overview:**  
  After payment verification, forwards the user’s query to the private Ollama LLM inference engine and returns the generated response.

- **Nodes Involved:**  
  - Private Model Inference  
  - Ollama Engine  
  - Response: 200 - Payment Successful

- **Node Details:**  
  - **Private Model Inference**  
    - Type: LangChain LLM Chain node  
    - Receives the query string from the webhook body (`body.query`) and sends it for inference.  
    - Uses `Ollama Engine` as the language model.  
    - Inputs: Success output from payment submission node.  
    - Outputs: LLM response text.  
    - Edge Cases: Empty query, malformed input, inference timeout.

  - **Ollama Engine**  
    - Type: LangChain Ollama LLM node  
    - Configured to connect to a private Ollama inference instance, possibly via ngrok tunnel as described in notes.  
    - Credentials: Ollama API credentials.  
    - Inputs: Query text from the Private Model Inference node.  
    - Outputs: LLM text completion result.  
    - Edge Cases: Network errors, Ollama service downtime, authentication failure.

  - **Response: 200 - Payment Successful**  
    - Type: Respond to Webhook  
    - Returns HTTP 200 with the text response from the LLM.  
    - Input: Output from Private Model Inference.  
    - Outputs: Final HTTP response to client.

---

### 3. Summary Table

| Node Name                          | Node Type                     | Functional Role                            | Input Node(s)                      | Output Node(s)                        | Sticky Note                                                                                                                   |
|-----------------------------------|-------------------------------|--------------------------------------------|----------------------------------|-------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| Webhook                           | Webhook                       | Receive POST request                       | External HTTP request             | Check for presence of X-HEADER      | ## x402 Payment Endpoint This workflow lets you monetize inference on private models hosted with Ollama in n8n. Learn more about the [x402 payment](https://www.x402.org/) protocol. Watch the [YouTube tutorial](https://youtu.be/m3ThthLtj3g) video. |
| Check for presence of X-HEADER    | If                            | Verify presence of x-payment header        | Webhook                         | Decode & Validate X-Payment, Response: Missing or Invalid Payment Headers |                                                                                                                               |
| Decode & Validate X-Payment       | Code (JavaScript)             | Decode and parse x-payment header          | Check for presence of X-HEADER   | Ensure Well Formatted Payment Payload |                                                                                                                               |
| Ensure Well Formatted Payment Payload | If                         | Validate payment payload fields             | Decode & Validate X-Payment      | Simulate Payment, Response: Missing or Invalid Payment Headers | ## Ensure Required Payment Details Use this block to add/check for payment detail requirements like minimum payment amount. You might also want to add a condition in the block to ensure payments are sent to the correct address. |
| Response: Missing or Invalid Payment Headers | Respond to Webhook      | Respond 402 for missing/invalid payment header | Check for presence of X-HEADER (false branch), Ensure Well Formatted Payment Payload (false branch) |                                     |                                                                                                                               |
| Simulate Payment                  | 1Shot API (simulate)          | Simulate payment authorization             | Ensure Well Formatted Payment Payload | On Successful Payment Simulation    | ## Any ERC-20 on any EVM With the 1Shot API node, you can accept any ERC-20 with the appropriate `transferWithAuthorization` methods. |
| On Successful Payment Simulation  | If                            | Check simulation success                    | Simulate Payment                | 1Shot API Submit & Wait, Response: Payment Invalid |                                                                                                                               |
| 1Shot API Submit & Wait           | 1Shot API (submit sync)       | Submit payment transaction                  | On Successful Payment Simulation | Private Model Inference, Response: Payment Invalid |                                                                                                                               |
| Response: Payment Invalid         | Respond to Webhook            | Respond 402 for invalid payment             | On Successful Payment Simulation (fail), 1Shot API Submit & Wait (fail) |                                     |                                                                                                                               |
| Private Model Inference           | LangChain Chain LLM           | Perform inference on private Ollama model  | 1Shot API Submit & Wait          | Response: 200 - Payment Successful   | ## x402 + Ollama Inference Set up the LLM block to point at your Ollama inference engine. This will let you charge for agents to request queries to your private models. |
| Ollama Engine                    | LangChain Ollama LLM          | Connect to private Ollama inference engine | Private Model Inference (ai_languageModel input) | Private Model Inference (ai_languageModel output) | ## Ollama + ngrok Docker stack This example assumes your n8n session is hosted separately from your Ollama instance. You can use ngrok to create a secure tunnel between your n8n workflow and your Ollama instance. See sticky note for setup details. |
| Response: 200 - Payment Successful | Respond to Webhook            | Return LLM inference success response       | Private Model Inference          |                                     |                                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node**  
   - Type: Webhook (n8n-nodes-base.webhook)  
   - HTTP Method: POST  
   - Path: `92c5ca23-99a7-437d-85da-84aef8bd2a25` (or your custom path)  
   - Response Mode: Response Node (to respond downstream)  
   - Connect this node as the entry point.

2. **Add an If Node to Check for `x-payment` Header**  
   - Name: "Check for presence of X-HEADER"  
   - Condition type: String operation "exists"  
   - Expression: `{{$json.headers['x-payment']}}`  
   - True branch: continue to decode payment  
   - False branch: connect to a Respond to Webhook node returning HTTP 402 and JSON error `{ "x402Version": "1", "error": "X-PAYMENT header has incorrect format" }`

3. **Add a Code Node to Decode & Validate `x-payment` Header**  
   - Name: "Decode & Validate X-Payment"  
   - Code (JavaScript):  
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
   - Connect true branch of header check to this node.

4. **Add an If Node to Ensure Well-Formed Payment Payload**  
   - Name: "Ensure Well Formatted Payment Payload"  
   - Condition: Check that `decodedXPayment.signature` exists and is non-empty.  
   - True branch: proceed to payment simulation  
   - False branch: respond with 402 error as in step 2.

5. **Add a 1Shot API Node for Payment Simulation**  
   - Name: "Simulate Payment"  
   - Operation: `simulate`  
   - Contract Method ID: `b63aaaa1-059d-4c38-928a-33ad17d66827` (transferWithAuthorization method for ERC-20)  
   - Parameters: Map all fields from `decodedXPayment` object (`from`, `to`, `value`, `validAfter`, `validBefore`, `nonce`, `signature`) via expressions like `={{ $json.decodedXPayment.from }}`  
   - Credentials: Connect your configured 1Shot OAuth2 API credentials.  
   - Connect true branch of well-formed payment payload node here.

6. **Add an If Node to Check Simulation Success**  
   - Name: "On Successful Payment Simulation"  
   - Condition: Check if `success` field in simulation result equals `"true"` (string comparison)  
   - True branch: proceed to submit payment synchronously  
   - False branch: respond with 402 payment invalid error.

7. **Add a 1Shot API Synchronous Submit Node**  
   - Name: "1Shot API Submit & Wait"  
   - Operation: `submit` or similar synchronous payment method  
   - Contract Method ID: same as simulation node  
   - Parameters: Map all fields from `decodedXPayment` as above  
   - Credentials: Same 1Shot OAuth2 API credentials  
   - True branch connects to Private Model Inference  
   - False branch connects to Response: Payment Invalid.

8. **Add a LangChain Ollama Node for Private Model Inference**  
   - Name: "Ollama Engine"  
   - Configure credentials with your Ollama API access  
   - Connect input from Private Model Inference node ai_languageModel input  
   - (Optional) Setup ngrok tunnel as per sticky note instructions to expose your private Ollama instance securely.

9. **Add a LangChain Chain LLM Node**  
   - Name: "Private Model Inference"  
   - Text input: Bind to webhook body query: `={{ $('Webhook').item.json.body.query }}`  
   - Prompt type: Define or default  
   - Connect ai_languageModel input to Ollama Engine output  
   - Connect main output to Response: 200 - Payment Successful node.

10. **Add Respond to Webhook Nodes for Error and Success Responses**  
    - "Response: Missing or Invalid Payment Headers" → HTTP 402, JSON error about malformed payment header  
    - "Response: Payment Invalid" → HTTP 402, JSON error about payment verification failure  
    - "Response: 200 - Payment Successful" → HTTP 200, respond with `{ "response": "<LLM output text>" }` where text is from Private Model Inference node.

11. **Connect all nodes according to logical flow:**  
    - Webhook → Check for presence of X-HEADER  
    - True → Decode & Validate X-Payment → Ensure Well Formatted Payment Payload  
    - True → Simulate Payment → On Successful Payment Simulation  
    - True → 1Shot API Submit & Wait → Private Model Inference → Response 200  
    - False branches to appropriate respond nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Context or Link                                                                                           |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow lets you monetize inference on private models hosted with Ollama in n8n. The x402 payment scheme lets user agents pass payments in the x-payment header field. Learn more about the [x402 payment](https://www.x402.org/) protocol. Watch the [YouTube tutorial](https://youtu.be/m3ThthLtj3g) video.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Sticky Note on Webhook node                                                                                |
| You can test the webhook with a cURL command by sending a properly formatted base64-encoded `x-payment` header and a JSON request body containing the query. Example provided in sticky note.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Sticky Note near Webhook node                                                                              |
| With the 1Shot API node, you can accept any ERC-20 token on any EVM-compatible blockchain that supports the `transferWithAuthorization` method required by the x402 protocol.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Sticky Note near Simulate Payment node                                                                    |
| Use the payment validation block to add further checks such as minimum payment amount or verifying the payment recipient address to secure your monetization logic.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Sticky Note near Ensure Well Formatted Payment Payload node                                               |
| The Ollama + ngrok Docker stack allows you to securely expose your private Ollama instance to n8n when hosted separately. It requires a free ngrok account, a static URL, and an auth token. Docker-compose YAML example is provided in sticky note, including GPU device reservation.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Sticky Note near Ollama Engine node                                                                        |

---

**Disclaimer:**  
The provided text exclusively originates from an automated workflow created with n8n, an integration and automation tool. This processing fully complies with applicable content policies and contains no illegal, offensive, or protected material. All handled data is legal and public.