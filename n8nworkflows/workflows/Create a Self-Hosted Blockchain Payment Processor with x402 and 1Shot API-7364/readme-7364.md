Create a Self-Hosted Blockchain Payment Processor with x402 and 1Shot API

https://n8nworkflows.xyz/workflows/create-a-self-hosted-blockchain-payment-processor-with-x402-and-1shot-api-7364


# Create a Self-Hosted Blockchain Payment Processor with x402 and 1Shot API

---

## 1. Workflow Overview

This workflow, titled **"Create a Self-Hosted Blockchain Payment Processor with x402 and 1Shot API"**, is designed to facilitate blockchain payment processing using the x402 protocol and the 1Shot API. It serves as a backend payment facilitator that supports verifying payment requests, settling payments on multiple blockchain networks, and exposing supported networks information. The workflow includes three main HTTP webhook endpoints:

- **/verify**: Validates a payment request by decoding and verifying the x402 payment header against configured payment tokens and smart contract methods.
- **/settle**: Attempts to settle (submit) a blockchain payment transaction based on the verified payment details.
- **/supported**: Returns a list of supported blockchain networks and schemes that this facilitator can process.

The logic is organized into these functional blocks:

- **1.1 Input Reception and Validation**: Receives and validates incoming HTTP POST requests at `/verify` and `/settle`, ensuring all required fields are present.
- **1.2 Payment Decoding and Verification**: Decodes the base64 encoded x402 payment header, validates the payment payload structure, and confirms that the payment token is supported.
- **1.3 Payment Configuration Lookup**: Maps the token address and network from the payment request to the internal configuration defining contract method IDs and network parameters.
- **1.4 Payment Simulation and Settlement**: For `/verify`, simulates the payment transaction with 1Shot API to check for validity; for `/settle`, submits the payment and waits for completion.
- **1.5 Response Handling**: Sends appropriate JSON responses back to the caller, including success, failure, or error messages.
- **1.6 Supported Networks Endpoint**: Responds to `/supported` requests with configured supported blockchain networks.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception and Validation

**Overview:**  
This block handles incoming HTTP POST requests on the `/verify` and `/settle` webhook endpoints. It validates the presence and format of required fields in the request body before processing further.

**Nodes Involved:**  
- `/verify` (Webhook)  
- `/settle` (Webhook)  
- Check POST Payload (IF)  
- Bad Payload or Header (IF)  
- Response: Bad POST Body (/verify) (RespondToWebhook)  
- Response: Bad POST Body (/settle) (RespondToWebhook)

**Node Details:**

- **/verify**  
  - Type: Webhook  
  - Role: Entry point for verification requests  
  - Config: HTTP POST at path `/verify`, responds via response node  
  - Inputs: External HTTP POST request  
  - Outputs: Connects to "Check POST Payload"  
  - Edge cases: Invalid or missing body fields; malformed requests

- **/settle**  
  - Type: Webhook  
  - Role: Entry point for settlement requests  
  - Config: HTTP POST at path `/settle`, responds via response node  
  - Inputs: External HTTP POST request  
  - Outputs: Connects to "Check POST Payload"  
  - Edge cases: Same as `/verify`

- **Check POST Payload**  
  - Type: IF node  
  - Role: Validates the presence of all required fields in the POST body, such as `x402Version`, `paymentHeader`, and payment requirements fields (`network`, `scheme`, etc.)  
  - Key Expressions: Checks existence of multiple fields with strict type validation  
  - Inputs: From `/verify` or `/settle` webhook node  
  - Outputs: Passes valid requests to "Decode & Validate X-Payment", invalid to "Bad Payload or Header"  
  - Edge cases: Missing fields, incorrect types

- **Bad Payload or Header**  
  - Type: IF node  
  - Role: Routes invalid requests based on webhook URL containing "verify" or "settle"  
  - Inputs: From failed "Check POST Payload" branch or decoding errors  
  - Outputs: Routes to appropriate bad payload response node  
  - Edge cases: Differentiates between `/verify` and `/settle` error responses

- **Response: Bad POST Body (/verify)**  
  - Type: RespondToWebhook  
  - Role: Returns JSON response with HTTP 402 and message indicating invalid or missing payload arguments for `/verify`  
  - Inputs: From "Bad Payload or Header" when on `/verify` path

- **Response: Bad POST Body (/settle)**  
  - Type: RespondToWebhook  
  - Role: Returns JSON response with HTTP 402 and error message for bad payload on `/settle`  
  - Inputs: From "Bad Payload or Header" when on `/settle` path

---

### 2.2 Payment Decoding and Verification

**Overview:**  
Decodes the `paymentHeader` from base64 to JSON, validates the presence of required authorization fields in the decoded payment, and ensures the payment token is supported.

**Nodes Involved:**  
- Decode & Validate X-Payment (Code)  
- Ensure Payment Payload (IF)  
- Lookup Payment Configs (Code)  
- Unsupported Token (IF)  
- Response: Unsupported Token (/verify) (RespondToWebhook)  
- Response: Unsupported Token (/settle) (RespondToWebhook)

**Node Details:**

- **Decode & Validate X-Payment**  
  - Type: Code node (JavaScript)  
  - Role:  
    - Decodes base64 `paymentHeader` to UTF-8 string  
    - Parses the JSON of the decoded string and attaches it as `decodedXPayment` to the JSON data  
    - On error, returns an error object signaling invalid token format  
  - Inputs: Validated POST body from "Check POST Payload"  
  - Outputs: Passes to "Ensure Payment Payload" on success, or "Bad Payload or Header" on failure  
  - Edge cases: Malformed base64, invalid JSON payload

- **Ensure Payment Payload**  
  - Type: IF node  
  - Role: Checks presence of all required fields within the decoded payment payload, such as signature, authorization details (`from`, `to`, `value`, `validAfter`, `validBefore`, `nonce`)  
  - Inputs: From "Decode & Validate X-Payment"  
  - Outputs: Passes valid to "Lookup Payment Configs", invalid to "Bad Payload or Header"  
  - Edge cases: Missing authorization fields, empty or null values

- **Lookup Payment Configs**  
  - Type: Code node (JavaScript)  
  - Role:  
    - Contains a hardcoded configuration map of supported payment tokens keyed by token address (lowercase)  
    - Validates if the token from the payment request exists in config  
    - Validates the associated network chain matches the request  
    - Attaches the token config object as `paymentConfig` to JSON data for downstream use  
  - Inputs: Validated payment with decoded payload from "Ensure Payment Payload"  
  - Outputs: Passes supported tokens to "verify or settle" IF node, unsupported to "Unsupported Token" IF node  
  - Edge cases: Unsupported token address, network mismatch

- **Unsupported Token**  
  - Type: IF node  
  - Role: Branches based on webhook URL containing "verify" to route to the relevant unsupported token response node  
  - Inputs: From "Lookup Payment Configs" on unsupported token path  
  - Outputs: To either "Response: Unsupported Token (/verify)" or "Response: Unsupported Token (/settle)"

- **Response: Unsupported Token (/verify)**  
  - Type: RespondToWebhook  
  - Role: Returns JSON response with HTTP 402 and message indicating unsupported payment token for `/verify` endpoint

- **Response: Unsupported Token (/settle)**  
  - Type: RespondToWebhook  
  - Role: Returns JSON response with HTTP 402 and message indicating unsupported payment token for `/settle` endpoint  
  - Includes networkId extraction from error message if available

---

### 2.3 Payment Simulation and Settlement

**Overview:**  
Handles simulating the payment transaction for `/verify` and submitting the actual payment transaction for `/settle` using the 1Shot API via OAuth2 credentials.

**Nodes Involved:**  
- verify or settle (IF)  
- Simulate Payment (1Shot API Node)  
- 1Shot API Submit & Wait (1Shot Synch node)  
- Verify Response (RespondToWebhook)  
- Settlement Response (RespondToWebhook)

**Node Details:**

- **verify or settle**  
  - Type: IF node  
  - Role: Determines if the incoming request is for `/verify` (true) or `/settle` (false) based on webhook URL  
  - Inputs: From "Lookup Payment Configs"  
  - Outputs: True path to "Simulate Payment" (for verification), False path to "1Shot API Submit & Wait" (for settlement)

- **Simulate Payment**  
  - Type: 1Shot API Node (Operation: simulate)  
  - Role: Simulates the payment transaction with the provided authorization details and signature without actually submitting it to the blockchain—used to verify if the payment would succeed  
  - Parameters: Constructs `params` JSON object dynamically from decodedXPayment payload fields (from, to, value, validAfter, validBefore, nonce, signature)  
  - Uses contractMethodId from paymentConfig  
  - Credentials: OAuth2 credentials named "x402 Facilitator"  
  - Inputs: From "verify or settle" IF node (true branch)  
  - Outputs: Passes results to "Verify Response" node  
  - Edge cases: API authentication failure, network issues, invalid params, 1Shot API errors

- **1Shot API Submit & Wait**  
  - Type: 1Shot API Synchronous Node (Operation: submit and wait)  
  - Role: Submits the payment transaction to the blockchain and waits for confirmation  
  - Parameters: Same dynamic `params` JSON as Simulate Payment, includes optional memo field (empty here)  
  - Uses contractMethodId from paymentConfig  
  - Credentials: OAuth2 credentials named "x402 Facilitator"  
  - Inputs: From "verify or settle" IF node (false branch)  
  - Outputs: Forwards response to "Settlement Response" node  
  - Edge cases: Transaction failure, timeouts, API credential issues

- **Verify Response**  
  - Type: RespondToWebhook  
  - Role: Returns JSON response to `/verify` requests with fields: `isValid` (boolean), `invalidReason` (string) extracted from simulation response  
  - Response code 402 if invalid

- **Settlement Response**  
  - Type: RespondToWebhook  
  - Role: Returns JSON response to `/settle` requests with fields: `success` (boolean), `error` (string or null), `txHash` (transaction hash), `networkId`  
  - Response code 402 if failure

---

### 2.4 Supported Networks Endpoint

**Overview:**  
Handles requests to `/supported` endpoint by returning a JSON list of supported blockchain network schemes and names.

**Nodes Involved:**  
- /supported (Webhook)  
- Supported Networks Config (Code)  
- Response: Return Supported Networks (RespondToWebhook)

**Node Details:**

- **/supported**  
  - Type: Webhook  
  - Role: Entry point for supported networks requests  
  - Config: HTTP GET or POST at path `/supported`, response via node  
  - Inputs: External HTTP request

- **Supported Networks Config**  
  - Type: Code node (JavaScript)  
  - Role: Returns a hardcoded array of supported network objects with `scheme` and `network` properties, e.g., `base-mainnet`, `avalanche`, `arbitrum-mainnet`  
  - Attaches this array as `kinds` property in JSON data

- **Response: Return Supported Networks**  
  - Type: RespondToWebhook  
  - Role: Sends JSON response with supported network kinds to the caller  
  - Response code 402 (default; presumably can be overridden)

---

### 2.5 Sticky Notes and Documentation Nodes

These are informational sticky notes embedded in the workflow to assist users with understanding and configuring the workflow.

- **Sticky Note (top-left)**: Explains `/verify` endpoint behavior and response JSON structure.
- **Sticky Note1 (mid-left)**: Explains `/settle` endpoint behavior and response JSON structure.
- **Sticky Note2 (near start)**: Details about editing the accepted tokens configuration in the "Lookup Payment Configs" node.
- **Sticky Note3 (bottom-left)**: Explains `/supported` endpoint and response format.
- **Sticky Note4 (bottom-left)**: Reminder to update supported networks configuration to match 1Shot API account setup.
- **Sticky Note5 (far left)**: Provides an example cURL command for testing the `/settle` endpoint.
- **Sticky Note6 (near start)**: Instructions for creating 1Shot API OAuth2 credentials for authentication.

---

## 3. Summary Table

| Node Name                         | Node Type                  | Functional Role                      | Input Node(s)                  | Output Node(s)                                            | Sticky Note                                                                                   |
|----------------------------------|----------------------------|------------------------------------|-------------------------------|-----------------------------------------------------------|----------------------------------------------------------------------------------------------|
| /verify                          | Webhook                    | Entry point for verification       | External HTTP POST             | Check POST Payload                                         | /verify endpoint behavior and response structure explained                                  |
| /settle                          | Webhook                    | Entry point for settlement         | External HTTP POST             | Check POST Payload                                         | /settle endpoint behavior and response structure explained                                  |
| Check POST Payload               | IF                         | Validates required POST fields     | /verify, /settle              | Decode & Validate X-Payment (valid), Bad Payload or Header (invalid) |                                                                                              |
| Decode & Validate X-Payment     | Code                       | Decodes base64 payment header      | Check POST Payload             | Ensure Payment Payload (valid), Bad Payload or Header (error) |                                                                                              |
| Ensure Payment Payload          | IF                         | Validates decoded payment contents | Decode & Validate X-Payment   | Lookup Payment Configs (valid), Bad Payload or Header (invalid) | Accepted Tokens Config explained                                                             |
| Lookup Payment Configs          | Code                       | Maps token/network to configs      | Ensure Payment Payload         | verify or settle (valid), Unsupported Token (unsupported)  |                                                                                              |
| verify or settle               | IF                         | Routes for verify or settle logic  | Lookup Payment Configs         | Simulate Payment (verify), 1Shot API Submit & Wait (settle) |                                                                                              |
| Simulate Payment               | 1Shot API Node             | Simulates payment transaction      | verify or settle (verify path) | Verify Response                                           | Create your 1Shot API Credential explained                                                   |
| 1Shot API Submit & Wait        | 1Shot Synch API Node       | Submits and waits for settlement   | verify or settle (settle path) | Settlement Response                                      | Create your 1Shot API Credential explained                                                   |
| Verify Response                | RespondToWebhook           | Responds to `/verify` requests      | Simulate Payment              | None                                                     |                                                                                              |
| Settlement Response            | RespondToWebhook           | Responds to `/settle` requests      | 1Shot API Submit & Wait       | None                                                     |                                                                                              |
| Bad Payload or Header          | IF                         | Routes bad payload errors           | Check POST Payload, Decode & Validate X-Payment | Response: Bad POST Body (/verify), Response: Bad POST Body (/settle) |                                                                                              |
| Response: Bad POST Body (/verify) | RespondToWebhook           | Responds with error for bad `/verify` payload | Bad Payload or Header         | None                                                     |                                                                                              |
| Response: Bad POST Body (/settle) | RespondToWebhook           | Responds with error for bad `/settle` payload | Bad Payload or Header         | None                                                     |                                                                                              |
| Unsupported Token              | IF                         | Routes unsupported token errors    | Lookup Payment Configs         | Response: Unsupported Token (/verify), Response: Unsupported Token (/settle) |                                                                                              |
| Response: Unsupported Token (/verify) | RespondToWebhook           | Responds with unsupported token error `/verify` | Unsupported Token             | None                                                     |                                                                                              |
| Response: Unsupported Token (/settle) | RespondToWebhook           | Responds with unsupported token error `/settle` | Unsupported Token             | None                                                     |                                                                                              |
| /supported                    | Webhook                    | Entry point for supported networks | External HTTP request          | Supported Networks Config                                 | /supported endpoint and response format explained                                           |
| Supported Networks Config      | Code                       | Returns supported blockchain networks | /supported                   | Response: Return Supported Networks                        | Supported Networks Config reminder                                                          |
| Response: Return Supported Networks | RespondToWebhook           | Responds with supported networks   | Supported Networks Config      | None                                                     |                                                                                              |
| Sticky Note                   | StickyNote                 | Info on /verify endpoint details   | None                          | None                                                     | /verify endpoint detailed explanation                                                      |
| Sticky Note1                  | StickyNote                 | Info on /settle endpoint details   | None                          | None                                                     | /settle endpoint detailed explanation                                                      |
| Sticky Note2                  | StickyNote                 | Info on accepted token configs     | None                          | None                                                     | Accepted Tokens Config explanation                                                          |
| Sticky Note3                  | StickyNote                 | Info on /supported endpoint details | None                          | None                                                     | /supported endpoint detailed explanation                                                   |
| Sticky Note4                  | StickyNote                 | Reminder on Supported Networks config | None                          | None                                                     | Supported Networks Config reminder                                                         |
| Sticky Note5                  | StickyNote                 | Example curl request for /settle   | None                          | None                                                     | Example Facilitator Curl Request                                                           |
| Sticky Note6                  | StickyNote                 | Instructions for 1Shot API credentials | None                          | None                                                     | Create your 1Shot API Credential explained                                                 |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Webhook Nodes**  
   - Create a webhook node named `/verify` with HTTP Method POST, Path `/verify`, response mode set to "Response Node".  
   - Create a webhook node named `/settle` with HTTP Method POST, Path `/settle`, response mode set to "Response Node".  
   - Create a webhook node named `/supported` with Path `/supported`, response mode set to "Response Node".

2. **Add POST Payload Validation (IF Node)**  
   - Create an IF node named "Check POST Payload".  
   - Configure conditions to strictly check existence of the following in the request body:  
     - `x402Version` (number exists)  
     - `paymentHeader` (string exists)  
     - `paymentRequirements.network` (string exists)  
     - `paymentRequirements.scheme` (string exists)  
     - `paymentRequirements.maxAmountRequired` (string exists)  
     - `paymentRequirements.resource` (string exists)  
     - `paymentRequirements.payTo` (string exists)  
     - `paymentRequirements.maxTimeoutSeconds` (number exists)  
     - `paymentRequirements.asset` (string exists)

3. **Create Error Routing IF Node for Bad Payloads**  
   - Create an IF node named "Bad Payload or Header".  
   - Condition: If the webhook URL contains "verify".  
   - Use this node to route invalid or error outputs to appropriate response nodes for `/verify` or `/settle`.

4. **Create Response Nodes for Bad Payloads**  
   - Create two RespondToWebhook nodes:  
     - "Response: Bad POST Body (/verify)" with HTTP code 402 and JSON body `{ "isValid": false, "invalidReason": "Incorrect or missing payload arguments" }`.  
     - "Response: Bad POST Body (/settle)" with HTTP code 402 and JSON body containing `{ success: false, error: "Missing or incorrect POST body", txHash: null, networkId: <dynamic from request> }`.

5. **Connect `/verify` and `/settle` Webhook nodes to "Check POST Payload" node**.

6. **Create Code Node "Decode & Validate X-Payment"**  
   - Add JavaScript to:  
     - Decode `paymentHeader` from base64 to UTF-8 string.  
     - Parse JSON and attach as `decodedXPayment`.  
     - On failure, return error object.  
   - Connect "Check POST Payload" (true branch) to this node.

7. **Create IF Node "Ensure Payment Payload"**  
   - Check that the following exist and are strings in `decodedXPayment.payload.authorization`:  
     - `signature`  
     - `from`  
     - `to`  
     - `value`  
     - `validAfter`  
     - `validBefore`  
     - `nonce` (check existence)  
   - Connect "Decode & Validate X-Payment" (success branch) to this node.  
   - Connect failure branch to "Bad Payload or Header".

8. **Create Code Node "Lookup Payment Configs"**  
   - Hardcode a configuration object with token addresses mapped to their token details, chain IDs, contract method IDs, and network names.  
   - Validate that the payment token exists and matches the expected network.  
   - Attach the matched config as `paymentConfig`.  
   - Connect "Ensure Payment Payload" (true branch) to this node.

9. **Create IF Node "Unsupported Token"**  
   - Condition: webhook URL contains "verify".  
   - Connect "Lookup Payment Configs" node:  
     - Valid tokens to next logic branch.  
     - Unsupported tokens to this node.

10. **Create Response Nodes for Unsupported Tokens**  
    - "Response: Unsupported Token (/verify)" returns `{ isValid: false, invalidReason: "Unsupported payment token" }`.  
    - "Response: Unsupported Token (/settle)" returns `{ success: false, error: "Unsupported payment token", txHash: null, networkId: <extracted if available> }`.

11. **Create IF Node "verify or settle"**  
    - Condition: webhook URL contains "verify".  
    - Connect "Lookup Payment Configs" (valid tokens path) to this node.

12. **Add 1Shot API Nodes**  
    - Create a 1Shot API node named "Simulate Payment" with Operation `simulate`.  
    - Parameters: dynamically set `params` JSON with decodedXPayment authorization fields (`from`, `to`, `value`, `validAfter`, `validBefore`, `nonce`, `signature`) and contractMethodId from paymentConfig.  
    - Use OAuth2 credentials configured for your 1Shot API business.  
    - Connect "verify or settle" true branch to this node.

    - Create a 1Shot Synch node named "1Shot API Submit & Wait" with Operation `submit and wait`.  
    - Same parameters and credentials as above.  
    - Connect "verify or settle" false branch to this node.

13. **Create Response Nodes**  
    - "Verify Response" returns JSON for `/verify` with `{ isValid, invalidReason }` extracted from simulation result.  
    - "Settlement Response" returns JSON for `/settle` with `{ success, error, txHash, networkId }`.

14. **Connect 1Shot API nodes to respective response nodes**.

15. **Supported Networks Endpoint Setup**  
    - Connect `/supported` webhook node to a "Supported Networks Config" code node, which returns a hardcoded array of supported networks (with `scheme` and `network` fields).  
    - Connect this code node to "Response: Return Supported Networks" RespondToWebhook node, returning `{ kinds }`.

16. **Credential Setup**  
    - Create an OAuth2 credential in n8n named "x402 Facilitator" with your 1Shot API key, secret, and business ID.  
    - Assign this credential to both 1Shot API nodes.

17. **Test and Validate**  
    - Use the example cURL command provided in sticky notes to test `/settle`.  
    - Adjust token configs and supported networks as necessary.

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                     | Context or Link                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| The verify endpoint will check that the given `x-payment` header and `paymentRequirements` are valid, returning `{ isValid: true }` or `{ isValid: false, invalidReason: "<Smart Contract Revert Error>" }`.                                                                                       | Sticky Note on `/verify` endpoint                                                                       |
| The settle endpoint attempts to settle a payment on the blockchain and returns success or failure JSON with transaction hash and network info.                                                                                                                                                  | Sticky Note on `/settle` endpoint                                                                       |
| You must edit the payment tokens `config` object in the `Lookup Payment Configs` node to include all tokens you want to support. Import corresponding contract methods into your 1Shot API account accordingly.                                                                                   | Sticky Note near "Lookup Payment Configs"                                                              |
| The `/supported` endpoint returns a list of supported blockchain networks as an array of `{ scheme, network }` objects.                                                                                                                                                                         | Sticky Note on `/supported` endpoint                                                                   |
| Update the supported networks list to match those configured in your 1Shot API account.                                                                                                                                                                                                         | Sticky Note near "Supported Networks Config"                                                          |
| Example cURL request provided for testing the `/settle` endpoint, including all necessary headers and a sample payment payload.                                                                                                                                                               | Sticky Note with example cURL request                                                                  |
| Important: Create an OAuth2 credential in n8n for your 1Shot API business, supplying API key, secret, and business ID, and assign it to the 1Shot API nodes.                                                                                                                                   | Sticky Note near "Simulate Payment" and "1Shot API Submit & Wait" nodes                                |
| The workflow is designed for use with the x402 payment protocol and the 1Shot API, which provides blockchain transaction simulation and submission services.                                                                                                                                   | Workflow overview context                                                                               |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All processed data is lawful and public.

---