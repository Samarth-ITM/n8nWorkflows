Multi-Chain Token Swap Relayer with Li.Fi

https://n8nworkflows.xyz/workflows/multi-chain-token-swap-relayer-with-li-fi-7206


# Multi-Chain Token Swap Relayer with Li.Fi

### 1. Workflow Overview

This workflow, titled **"Multi-Chain 1Shot Gas Station"**, enables operators to monetize token swaps from ERC-20 tokens to native gas tokens across up to 100 different EVM-compatible blockchains by leveraging the [Li.Fi](https://li.fi/) protocol. The core use case is to provide a seamless multi-chain token swap relayer service with integrated payment validation following the [x402 payment](https://www.x402.org/) protocol standard.

The workflow is organized into the following logical blocks:

- **1.1 Input Reception and Validation**: Receives incoming HTTP webhook POST requests and validates the request body and presence of required payment headers.

- **1.2 Payment Header Decoding and Validation**: Decodes the base64-encoded x-payment header, validates its structure and content against expected fields and values.

- **1.3 Payment Configuration Lookup**: Checks the token and chain against a predefined configuration of supported tokens and their contract method IDs.

- **1.4 Li.Fi Quote Fetching**: Queries the Li.Fi API to obtain swap routes and quotes to convert the user’s ERC-20 tokens into native gas tokens.

- **1.5 Payment Simulation and Submission**: Uses the 1Shot API to simulate the payment transaction and upon success, submits the transaction on-chain.

- **1.6 Response Handling**: Sends appropriate HTTP responses based on payment validation, simulation results, or errors, including detailed error messages and acceptance criteria.

Supporting this logic are several sticky notes providing documentation, usage examples, and configuration instructions.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Validation

**Overview:**  
Receives incoming HTTP POST requests on the webhook, validates the required request body parameters and checks for the presence of the x-payment header.

**Nodes Involved:**  
- Webhook  
- Ensure Request Body Parameters  
- Response: Missing or Invalid Request Body Params  
- Check for presence of X-HEADER  
- Response: No X-Payment Header

**Node Details:**

- **Webhook**  
  - Type: Webhook (HTTP POST)  
  - Configured to listen on path `/gas-station`, method POST, response handled by response nodes downstream.  
  - Outputs the full HTTP request including headers and JSON body.  
  - Edge cases: Missing payload or malformed JSON body; no authentication (relies on payment header).  

- **Ensure Request Body Parameters**  
  - Type: If Node  
  - Conditions check existence of `fromAddress`, `fromToken`, `toChain`, `fromAmount` in the request body.  
  - If any missing, routes to error response.  
  - Edge cases: Partial or empty JSON body leading to failure on condition checks.  

- **Response: Missing or Invalid Request Body Params**  
  - Type: Respond to Webhook  
  - Sends HTTP 402 with JSON error detailing missing parameters.  
  - Useful for client-side debugging.  

- **Check for presence of X-HEADER**  
  - Type: If Node  
  - Checks for existence of header `x-payment` in the HTTP request headers.  
  - If missing, routes to error response.  
  - Edge cases: Header present but empty or malformed handled downstream.  

- **Response: No X-Payment Header**  
  - Type: Respond to Webhook  
  - Sends HTTP 402 with JSON error indicating missing payment header.  
  - Includes an `estimate` object with minimal quote info from Li.Fi to assist client understanding.  

---

#### 1.2 Payment Header Decoding and Validation

**Overview:**  
Decodes the base64-encoded x-payment header, parses it into JSON, and validates key fields and logical constraints such as signature presence, authorization validity timeframe, and amount consistency.

**Nodes Involved:**  
- Decode & Validate X-Payment  
- Ensure Well Formatted Payment Payload  
- Response: Missing or Invalid X-Payment Header

**Node Details:**

- **Decode & Validate X-Payment**  
  - Type: Code (JavaScript)  
  - Decodes `x-payment` header from base64 and parses JSON.  
  - Adds parsed JSON as `decodedXPayment` in the workflow context.  
  - Error handling returns an error object on invalid format.  
  - Edge cases: Invalid base64, malformed JSON, missing header - triggers failure output.  

- **Ensure Well Formatted Payment Payload**  
  - Type: If Node  
  - Validates presence of key fields in the decoded payload: `signature`, `authorization.from`, `authorization.to`, `authorization.value`, `validAfter`, `validBefore`, `nonce`.  
  - Checks that authorization value matches `fromAmount` in webhook body.  
  - Validates that `validBefore` is at least 120 seconds in the future relative to current time.  
  - Routes invalid payloads to error response.  
  - Edge cases: Expired or future-dated signatures, mismatched amounts, missing fields leading to rejection.  

- **Response: Missing or Invalid X-Payment Header**  
  - Type: Respond to Webhook  
  - Sends HTTP 402 with JSON error indicating invalid or missing payment header.  
  - Includes acceptance criteria and token info from payment config for client correction.  

---

#### 1.3 Payment Configuration Lookup

**Overview:**  
Maps the incoming token address and chain id to a predefined configuration object that includes the token metadata and the 1Shot API contract method ID needed for the swap transaction.

**Nodes Involved:**  
- Lookup Payment Configs  
- Response: Bad Token Address

**Node Details:**

- **Lookup Payment Configs**  
  - Type: Code (JavaScript)  
  - Contains a hardcoded map of supported token addresses, chains, names, versions, and contract method IDs.  
  - Checks if the `fromToken` from the request exists in the config and matches the chain.  
  - Throws error if token or chain is unsupported.  
  - Adds `paymentConfig` object to workflow context.  
  - Edge cases: Unsupported tokens or chain mismatches cause workflow to respond with error.  

- **Response: Bad Token Address**  
  - Type: Respond to Webhook  
  - Sends HTTP 402 with message instructing to provide a valid stablecoin for swap.  

---

#### 1.4 Li.Fi Quote Fetching

**Overview:**  
Queries the Li.Fi API to fetch a swap route and quote to convert the input ERC-20 tokens to native gas tokens. It uses various parameters from the webhook body and enriched payment config.

**Nodes Involved:**  
- Fetch Li.Fi Quote  
- Response: Failed to Get Quote

**Node Details:**

- **Fetch Li.Fi Quote**  
  - Type: HTTP Request  
  - Sends GET request to `https://li.quest/v1/quote` with query parameters that include from/to chain, tokens, amounts, slippage, integrator name, and fee.  
  - Adds required API key header for authentication.  
  - Retries on failure to improve reliability.  
  - On failure routes to error response node.  
  - Edge cases: Network errors, invalid parameters, API limits, or unexpected response formats.  

- **Response: Failed to Get Quote**  
  - Type: Respond to Webhook  
  - Sends HTTP 402 with JSON error indicating bad quote input or Li.Fi failure.  

---

#### 1.5 Payment Simulation and Submission

**Overview:**  
Uses the 1Shot API to simulate the payment transaction using the decoded payment payload and Li.Fi quote data. If simulation succeeds, submits the transaction to the blockchain network.

**Nodes Involved:**  
- Simulate Payment  
- On Successful Payment Simulation  
- 1Shot API Submit & Wait  
- Response: 200 - Payment Successful  
- Response: Payment Invalid

**Node Details:**

- **Simulate Payment**  
  - Type: 1Shot API Node (operation: simulate)  
  - Parameters set dynamically from decoded payment payload and Li.Fi quote gas limits.  
  - Simulates the transaction using the configured contract method ID.  
  - Uses OAuth2 credentials for 1Shot API access.  
  - Edge cases: Simulation failure due to invalid calldata, gas estimation errors, or signature problems.  

- **On Successful Payment Simulation**  
  - Type: If Node  
  - Checks if simulation returned success status (`success == true`).  
  - Routes to submission node on success or invalid response on failure.  

- **1Shot API Submit & Wait**  
  - Type: 1Shot API Node (operation: submit and wait)  
  - Submits the actual transaction on-chain using the same parameters.  
  - Includes memo for auditing with details of receiver, token, amount, and chain.  
  - Uses OAuth2 credentials.  
  - Edge cases: Transaction failure, network errors, nonce reuse, or signature rejection.  

- **Response: 200 - Payment Successful**  
  - Type: Respond to Webhook  
  - Sends HTTP 200 with JSON containing the `transactionHash`.  

- **Response: Payment Invalid**  
  - Type: Respond to Webhook  
  - Sends HTTP 402 with error indicating payment invalidity.  
  - Includes acceptance criteria for client correction.  

---

#### 1.6 Response Handling and Documentation

**Overview:**  
Provides user-facing documentation, example curl commands, and configuration instructions via sticky notes.

**Nodes Involved:**  
- Sticky Note6  
- Sticky Note3  
- Sticky Note  
- Sticky Note1  
- Sticky Note2

**Node Details:**

- Sticky notes contain markdown-formatted information with:

  - Overview of the x402 payment endpoint and monetization purpose.  
  - Example curl command demonstrating how to invoke the webhook with a properly formatted x-payment header.  
  - Explanation of the Li.Fi quote fetching logic.  
  - Description of the 1Shot API usage for simulation and submission.  
  - Instructions on editing payment configs and setting up contract method IDs.  

---

### 3. Summary Table

| Node Name                            | Node Type                    | Functional Role                                      | Input Node(s)                      | Output Node(s)                                | Sticky Note                                                                                              |
|------------------------------------|------------------------------|-----------------------------------------------------|----------------------------------|-----------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Webhook                            | Webhook                      | Entry point, receives HTTP POST requests            |                                  | Ensure Request Body Parameters                 |                                                                                                        |
| Ensure Request Body Parameters     | If                           | Validates required body parameters                   | Webhook                          | Lookup Payment Configs, Response: Missing or Invalid Request Body Params |                                                                                                        |
| Response: Missing or Invalid Request Body Params | Respond to Webhook           | Sends error if required body params missing          | Ensure Request Body Parameters    |                                               |                                                                                                        |
| Lookup Payment Configs             | Code                         | Looks up token and chain config                       | Ensure Request Body Parameters    | Fetch Li.Fi Quote, Response: Bad Token Address | Sticky Note2: Instructions on payment configs setup                                                   |
| Response: Bad Token Address        | Respond to Webhook           | Sends error if token not supported                    | Lookup Payment Configs            |                                               |                                                                                                        |
| Fetch Li.Fi Quote                  | HTTP Request                 | Fetches swap route and quote from Li.Fi API          | Lookup Payment Configs            | Check for presence of X-HEADER, Response: Failed to Get Quote | Sticky Note: Explains Li.Fi quote fetching logic                                                      |
| Response: Failed to Get Quote      | Respond to Webhook           | Sends error on quote fetch failure                    | Fetch Li.Fi Quote                 |                                               |                                                                                                        |
| Check for presence of X-HEADER     | If                           | Checks for existence of x-payment header             | Fetch Li.Fi Quote                 | Decode & Validate X-Payment, Response: No X-Payment Header |                                                                                                        |
| Response: No X-Payment Header      | Respond to Webhook           | Sends error if x-payment header missing               | Check for presence of X-HEADER    |                                               |                                                                                                        |
| Decode & Validate X-Payment        | Code                         | Decodes base64 x-payment header and parses JSON      | Check for presence of X-HEADER    | Ensure Well Formatted Payment Payload          |                                                                                                        |
| Ensure Well Formatted Payment Payload | If                           | Validates structure and contents of decoded payment  | Decode & Validate X-Payment       | Simulate Payment, Response: Missing or Invalid X-Payment Header |                                                                                                        |
| Response: Missing or Invalid X-Payment Header | Respond to Webhook           | Sends error if x-payment header invalid               | Ensure Well Formatted Payment Payload |                                               |                                                                                                        |
| Simulate Payment                  | 1Shot API Node               | Simulates the payment transaction                      | Ensure Well Formatted Payment Payload | On Successful Payment Simulation               | Sticky Note1: 1Shot API usage explanation                                                             |
| On Successful Payment Simulation   | If                           | Checks if simulation succeeded                         | Simulate Payment                 | 1Shot API Submit & Wait, Response: Payment Invalid |                                                                                                        |
| 1Shot API Submit & Wait            | 1Shot API Node               | Submits the transaction on-chain                       | On Successful Payment Simulation  | Response: 200 - Payment Successful, Response: Payment Invalid |                                                                                                        |
| Response: 200 - Payment Successful | Respond to Webhook           | Sends HTTP 200 success with transaction hash          | 1Shot API Submit & Wait          |                                               |                                                                                                        |
| Response: Payment Invalid          | Respond to Webhook           | Sends error indicating payment invalidity             | On Successful Payment Simulation, 1Shot API Submit & Wait |                                               |                                                                                                        |
| Sticky Note6                      | Sticky Note                  | Workflow overview and external links                   |                                  |                                               | "## x402 Payment Endpoint\n\nThis workflow lets the operator monetize swaps ... See [x402 payment](https://www.x402.org/) protocol. Watch the [YouTube tutorial]() video." |
| Sticky Note3                      | Sticky Note                  | Example curl command illustrating webhook usage       |                                  |                                               | "## Example Curl Command\n\nYou can test 1Shot Gas Station ... with a command like this ..."              |
| Sticky Note                       | Sticky Note                  | Explains Li.Fi quote fetching logic                    |                                  |                                               | "## Fetch Swap Route & Quote form Li.Fi\n\nUse the input parameters to find a route..."                  |
| Sticky Note1                      | Sticky Note                  | Explains 1Shot API simulation and submission           |                                  |                                               | "## Use 1Shot API to Simulate and Submit the Swap\n\n[1Shot API](https://1shotapi.com) handles simulating..." |
| Sticky Note2                      | Sticky Note                  | Payment configuration instructions                      |                                  |                                               | "## Payment Configs\n\nYou must edit the \"Lookup Payment Configs\" node with the tokens you wish to support..." |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `gas-station`  
   - Response Mode: Response Node (to send responses downstream)  

2. **Create If Node: Ensure Request Body Parameters**  
   - Conditions: Check existence of `fromAddress`, `fromToken`, `toChain`, `fromAmount` in webhook JSON body  
   - True output: Proceed to Lookup Payment Configs  
   - False output: Connect to Response: Missing or Invalid Request Body Params  

3. **Create Respond to Webhook Node: Response: Missing or Invalid Request Body Params**  
   - HTTP Response Code: 402  
   - Respond With: JSON  
   - Response Body: JSON error message listing missing parameters  

4. **Create Code Node: Lookup Payment Configs**  
   - JS Code: Hardcoded config object mapping token addresses to chain id, name, version, and 1Shot contractMethodId  
   - Validates token exists and chain matches  
   - On success: add `paymentConfig` to JSON  
   - On failure: throw error to route to Response: Bad Token Address  

5. **Create Respond to Webhook Node: Response: Bad Token Address**  
   - HTTP Response Code: 402  
   - Respond With: JSON  
   - Response Body: error message asking to provide a valid stablecoin  

6. **Create HTTP Request Node: Fetch Li.Fi Quote**  
   - URL: `https://li.quest/v1/quote`  
   - Method: GET  
   - Query Params: fromChain, fromToken, fromAddress, toChain, toToken (zero address for native token), toAddress, slippage (0.05), integrator ("oneshot-gas-station"), fromAmount, allowBridges ("gasZipBridge"), fee (0.005)  
   - Header: `x-lifi-api-key` with your API key  
   - On error: Continue with error output  
   - True output: Connect to Check for presence of X-HEADER  
   - Error output: Connect to Response: Failed to Get Quote  

7. **Create Respond to Webhook Node: Response: Failed to Get Quote**  
   - HTTP Response Code: 402  
   - Respond With: JSON  
   - Response Body: error message about bad quote input parameters  

8. **Create If Node: Check for presence of X-HEADER**  
   - Condition: Header `x-payment` exists in webhook request headers  
   - True output: Connect to Decode & Validate X-Payment  
   - False output: Connect to Response: No X-Payment Header  

9. **Create Respond to Webhook Node: Response: No X-Payment Header**  
   - HTTP Response Code: 402  
   - Respond With: JSON  
   - Response Body: error indicating missing x-payment header with estimate and acceptance info  

10. **Create Code Node: Decode & Validate X-Payment**  
    - JS Code:  
      - Retrieve `x-payment` header from webhook headers  
      - Decode base64, parse JSON, assign to `decodedXPayment`  
      - On error (invalid format), return error object  
    - Connect output to Ensure Well Formatted Payment Payload  

11. **Create If Node: Ensure Well Formatted Payment Payload**  
    - Conditions:  
      - Fields `payload.signature`, `payload.authorization.from`, `authorization.to`, `authorization.value`, `authorization.validAfter`, `authorization.validBefore`, `authorization.nonce` exist  
      - `authorization.value` equals `fromAmount` from webhook body  
      - `validBefore` minus current time >= 120 seconds  
    - True output: Connect to Simulate Payment  
    - False output: Connect to Response: Missing or Invalid X-Payment Header  

12. **Create Respond to Webhook Node: Response: Missing or Invalid X-Payment Header**  
    - HTTP Response Code: 402  
    - Respond With: JSON  
    - Response Body: error about invalid x-payment header and acceptance criteria  

13. **Create 1Shot API Node: Simulate Payment**  
    - Operation: simulate  
    - Parameters: tokenAddress, from, value, validAfter, validBefore, nonce, signature, diamondCalldata (all from decodedXPayment and webhook body)  
    - GasLimit: parse hex gasLimit from Li.Fi quote  
    - Contract Method ID: from paymentConfig  
    - Credentials: OAuth2 for 1Shot API  
    - Output to On Successful Payment Simulation  

14. **Create If Node: On Successful Payment Simulation**  
    - Condition: Check if simulation success is true  
    - True output: Connect to 1Shot API Submit & Wait  
    - False output: Connect to Response: Payment Invalid  

15. **Create 1Shot API Node: 1Shot API Submit & Wait**  
    - Operation: Submit and wait for on-chain transaction confirmation  
    - Parameters: Same as simulation node, plus memo describing swap details  
    - GasLimit: from Li.Fi quote  
    - Contract Method ID: from paymentConfig  
    - Credentials: OAuth2 for 1Shot API  
    - True output: Connect to Response: 200 - Payment Successful  
    - False output: Connect to Response: Payment Invalid  

16. **Create Respond to Webhook Node: Response: 200 - Payment Successful**  
    - HTTP Response Code: 200  
    - Respond With: Text  
    - Response Body: JSON with `transactionHash` from 1Shot API response  

17. **Create Respond to Webhook Node: Response: Payment Invalid**  
    - HTTP Response Code: 402  
    - Respond With: JSON  
    - Response Body: error indicating payment invalidity with acceptance criteria  

18. **Add Sticky Notes**  
    - Sticky Note6: Overview of x402 payment endpoint and monetization purpose with external links  
    - Sticky Note3: Example curl command demonstrating proper webhook usage and x-payment header  
    - Sticky Note: Explanation of Li.Fi quote fetching logic  
    - Sticky Note1: Explanation of 1Shot API simulation and submission process  
    - Sticky Note2: Instructions on payment configuration node setup  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                                       |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| This workflow implements the [x402 payment](https://www.x402.org/) protocol to monetize token swaps from ERC-20 tokens to native gas tokens on multiple EVM chains using Li.Fi and 1Shot APIs.                                                                                                                 | x402 payment protocol documentation                                                                                   |
| Example curl command shows how to invoke the webhook endpoint with a base64-encoded x-payment header. The header must include a valid signature and authorization payload matching the swap parameters.                                                                                                      | Sticky Note3 content                                                                                                  |
| Li.Fi quote fetching uses the public API endpoint `https://li.quest/v1/quote` with an API key. It requests swap routes converting user tokens to native tokens with a fixed slippage and fee.                                                                                                                  | Li.Fi API documentation: https://li.fi/                                                                               |
| 1Shot API nodes require OAuth2 credentials linked to a 1Shot Gas Station business account. The contract method ID used corresponds to the imported `callDiamondWithEIP3009SignatureToNative` method for on-chain payment submission.                                                                          | 1Shot API: https://1shotapi.com                                                                                       |
| Payment configs must be maintained manually in the "Lookup Payment Configs" node, mapping token addresses and chains to their method IDs and metadata. This is critical for correct operation and must be updated when adding new supported tokens or chains.                                                 | Sticky Note2 content                                                                                                  |

---

This completes the detailed analysis and structured reference for the **Multi-Chain 1Shot Gas Station** workflow.