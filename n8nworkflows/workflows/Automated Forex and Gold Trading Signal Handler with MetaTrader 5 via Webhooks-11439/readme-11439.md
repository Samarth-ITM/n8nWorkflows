Automated Forex and Gold Trading Signal Handler with MetaTrader 5 via Webhooks

https://n8nworkflows.xyz/workflows/automated-forex-and-gold-trading-signal-handler-with-metatrader-5-via-webhooks-11439


# Automated Forex and Gold Trading Signal Handler with MetaTrader 5 via Webhooks

### 1. Workflow Overview

This workflow automates the handling of Forex and Gold trading signals received via MetaTrader 5 (MT5) webhooks. It supports multiple operations including receiving new trading signals, retrieving pending signals, confirming processed signals, executing market and limit orders, and clearing all stored signals. The workflow is designed to interact with MT5 by exposing RESTful webhook endpoints to receive and respond to trading-related requests.

Logical blocks:

- **1.1 Signal Reception and Storage**: Receives incoming trading signals via POST, stores them internally, and responds to the sender.
- **1.2 Pending Signals Retrieval**: Provides a GET endpoint to fetch all currently pending signals.
- **1.3 Signal Confirmation**: Accepts POST requests to confirm that a signal has been processed, marking it accordingly.
- **1.4 Market Order Execution**: Handles POST requests to execute market orders based on received instructions.
- **1.5 Limit Order Execution**: Handles POST requests to execute limit orders.
- **1.6 Clearing All Signals**: Provides a POST endpoint to clear all stored signals.
- **1.7 Response Handling**: Respond nodes are used in each block to send appropriate HTTP responses back to the webhook callers.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Signal Reception and Storage

- **Overview:**  
  This block exposes a POST webhook endpoint to receive trading signals from MT5. Upon receiving a signal, it stores the data, then sends an acknowledgment response.

- **Nodes Involved:**  
  - Receive Signal (POST)  
  - Store Signal (Code)  
  - Respond to POST

- **Node Details:**

  - **Receive Signal (POST)**  
    - Type: Webhook  
    - Role: Entry point for receiving trading signals through HTTP POST requests.  
    - Configuration: Webhook ID set to "mt5-signal-receive", listens for POST method.  
    - Inputs: External HTTP POST request with signal data.  
    - Outputs: Passes received data to "Store Signal" node.  
    - Edge cases: Invalid payload format, network or connectivity issues.

  - **Store Signal (Code)**  
    - Type: Code  
    - Role: Processes and stores the incoming signal data internally (likely in-memory or external database, implementation details are coded).  
    - Configuration: Custom JavaScript code for signal parsing and storage.  
    - Inputs: Receives JSON payload from webhook node.  
    - Outputs: Passes control to "Respond to POST".  
    - Edge cases: Parsing errors, storage failures, data validation issues.

  - **Respond to POST**  
    - Type: Respond to Webhook  
    - Role: Sends HTTP response confirming receipt of the signal.  
    - Configuration: Default response configuration, likely HTTP 200 OK.  
    - Inputs: Triggered by "Store Signal" node after successful processing.  
    - Outputs: HTTP response back to the webhook caller.  
    - Edge cases: Response failures or timeouts.

---

#### 1.2 Pending Signals Retrieval

- **Overview:**  
  Provides an HTTP GET endpoint for clients to retrieve all pending (unprocessed) trading signals stored in the system.

- **Nodes Involved:**  
  - Get Pending Signals (GET)  
  - Fetch Pending Signals (Code)  
  - Return Signals

- **Node Details:**

  - **Get Pending Signals (GET)**  
    - Type: Webhook  
    - Role: Entry point for GET requests to retrieve pending signals.  
    - Configuration: Webhook ID "mt5-get-signals", listens for GET method.  
    - Inputs: External HTTP GET request.  
    - Outputs: Passes control to "Fetch Pending Signals".  
    - Edge cases: Unauthorized access if security not configured, request format errors.

  - **Fetch Pending Signals (Code)**  
    - Type: Code  
    - Role: Retrieves all pending signals from storage.  
    - Configuration: Custom code querying the stored signals, filtering for unprocessed.  
    - Inputs: Request trigger from webhook.  
    - Outputs: Data payload of pending signals forwarded to "Return Signals".  
    - Edge cases: Storage access errors, empty datasets.

  - **Return Signals**  
    - Type: Respond to Webhook  
    - Role: Returns the list of pending signals in the HTTP response.  
    - Configuration: Sends JSON response with signal data.  
    - Inputs: Receives data from "Fetch Pending Signals".  
    - Outputs: HTTP response to the GET caller.  
    - Edge cases: Serialization errors, response timeout.

---

#### 1.3 Signal Confirmation

- **Overview:**  
  Accepts POST requests to confirm that a particular trading signal has been processed, marking it as completed or acknowledged.

- **Nodes Involved:**  
  - Confirm Signal Processed (POST)  
  - Mark as Processed (Code)  
  - Confirm Response

- **Node Details:**

  - **Confirm Signal Processed (POST)**  
    - Type: Webhook  
    - Role: Receives confirmation requests via POST to mark signals processed.  
    - Configuration: Webhook ID "mt5-confirm", listens for POST method.  
    - Inputs: POST payload specifying signal ID or criteria.  
    - Outputs: Passes to "Mark as Processed".  
    - Edge cases: Missing or invalid signal identifier, unauthorized access.

  - **Mark as Processed (Code)**  
    - Type: Code  
    - Role: Updates the internal storage to mark the specified signal as processed.  
    - Configuration: Custom JavaScript to locate and update signal status.  
    - Inputs: Confirmation data from webhook node.  
    - Outputs: Passes control to "Confirm Response".  
    - Edge cases: Signal not found, update failures.

  - **Confirm Response**  
    - Type: Respond to Webhook  
    - Role: Sends back an HTTP confirmation response after marking signal processed.  
    - Configuration: Default success response.  
    - Inputs: Triggered after successful update.  
    - Outputs: HTTP response to confirmation sender.  
    - Edge cases: Response delivery issues.

---

#### 1.4 Market Order Execution

- **Overview:**  
  Handles POST requests to execute market orders based on signal instructions.

- **Nodes Involved:**  
  - Market Order (POST)  
  - market order code (Code)  
  - Respond to Webhook

- **Node Details:**

  - **Market Order (POST)**  
    - Type: Webhook  
    - Role: Entry point for market order execution requests.  
    - Configuration: Webhook ID "bee7e24f-efa9-41b2-8761-1871c15479da" (same as Clear All Signals POST, likely differentiated by payload or internal routing), listens for POST.  
    - Inputs: Incoming order instructions via POST.  
    - Outputs: Passes to "market order code".  
    - Edge cases: Invalid order parameters, authentication issues.

  - **market order code (Code)**  
    - Type: Code  
    - Role: Processes the market order instructions and interacts with trading backend accordingly.  
    - Configuration: Custom JavaScript implementing order validation and execution logic.  
    - Inputs: Market order data from webhook.  
    - Outputs: Passes to "Respond to Webhook".  
    - Edge cases: Trading API errors, order rejection, network failures.

  - **Respond to Webhook**  
    - Type: Respond to Webhook  
    - Role: Sends HTTP response confirming order reception and/or execution status.  
    - Configuration: Default response.  
    - Inputs: Triggered after code node completes.  
    - Outputs: HTTP response to caller.  
    - Edge cases: Response timeouts.

---

#### 1.5 Limit Order Execution

- **Overview:**  
  Handles POST requests to execute limit orders.

- **Nodes Involved:**  
  - Limit Order (POST)  
  - limit order code (Code)  
  - Respond to Limit Order

- **Node Details:**

  - **Limit Order (POST)**  
    - Type: Webhook  
    - Role: Entry point for limit order requests.  
    - Configuration: Webhook ID "mt5-limit-order-webhook", listens for POST.  
    - Inputs: HTTP POST with limit order details.  
    - Outputs: Passes to "limit order code".  
    - Edge cases: Invalid parameters, missing required fields.

  - **limit order code (Code)**  
    - Type: Code  
    - Role: Validates and processes limit order instructions.  
    - Configuration: Custom JavaScript for order logic.  
    - Inputs: Order data from webhook.  
    - Outputs: Passes to "Respond to Limit Order".  
    - Edge cases: Trading API rejection, logic errors.

  - **Respond to Limit Order**  
    - Type: Respond to Webhook  
    - Role: Sends back confirmation or error response.  
    - Configuration: Standard response node.  
    - Inputs: Data from code node.  
    - Outputs: HTTP response.  
    - Edge cases: Response failures.

---

#### 1.6 Clearing All Signals

- **Overview:**  
  Provides a POST webhook to clear all stored signals, useful for resetting the state.

- **Nodes Involved:**  
  - Clear all signals (POST)  
  - clear all signals (Code)  
  - confirm signals are cleared

- **Node Details:**

  - **Clear all signals (POST)**  
    - Type: Webhook  
    - Role: Entry point for clearing the signal store.  
    - Configuration: Webhook ID "bee7e24f-efa9-41b2-8761-1871c15479da" (shared with Market Order POST, likely differentiated internally).  
    - Inputs: POST request to trigger clearing.  
    - Outputs: To "clear all signals" code node.  
    - Edge cases: Unauthorized access, concurrent modification issues.

  - **clear all signals (Code)**  
    - Type: Code  
    - Role: Executes logic to remove all stored signals from memory or database.  
    - Configuration: Custom JavaScript for clearing storage.  
    - Inputs: Trigger from webhook.  
    - Outputs: To "confirm signals are cleared".  
    - Edge cases: Storage access errors.

  - **confirm signals are cleared**  
    - Type: Respond to Webhook  
    - Role: Sends HTTP confirmation that signals have been cleared.  
    - Configuration: Default success response.  
    - Inputs: From clearing code node.  
    - Outputs: HTTP response.  
    - Edge cases: Response delivery failures.

---

### 3. Summary Table

| Node Name                | Node Type             | Functional Role                         | Input Node(s)           | Output Node(s)          | Sticky Note                         |
|--------------------------|-----------------------|---------------------------------------|------------------------|-------------------------|-----------------------------------|
| Receive Signal (POST)     | Webhook               | Receive trading signals (POST)         | -                      | Store Signal            |                                   |
| Store Signal             | Code                  | Store received signals                  | Receive Signal (POST)   | Respond to POST         |                                   |
| Respond to POST          | Respond to Webhook    | Respond to signal POST request          | Store Signal           | -                       |                                   |
| Get Pending Signals (GET)| Webhook               | Receive GET request for pending signals| -                      | Fetch Pending Signals   |                                   |
| Fetch Pending Signals    | Code                  | Retrieve pending signals                | Get Pending Signals (GET)| Return Signals          |                                   |
| Return Signals           | Respond to Webhook    | Return pending signals in response      | Fetch Pending Signals   | -                       |                                   |
| Confirm Signal Processed (POST)| Webhook         | Receive confirmation of processed signal| -                      | Mark as Processed       |                                   |
| Mark as Processed        | Code                  | Mark signal as processed                 | Confirm Signal Processed (POST)| Confirm Response |                                   |
| Confirm Response         | Respond to Webhook    | Respond to confirmation POST             | Mark as Processed       | -                       |                                   |
| Market Order (POST)      | Webhook               | Receive market order requests            | -                      | market order code       |                                   |
| market order code        | Code                  | Process market order instructions        | Market Order (POST)     | Respond to Webhook      |                                   |
| Respond to Webhook       | Respond to Webhook    | Respond to market order POST              | market order code       | -                       |                                   |
| Limit Order (POST)       | Webhook               | Receive limit order requests              | -                      | limit order code        |                                   |
| limit order code         | Code                  | Process limit order instructions          | Limit Order (POST)      | Respond to Limit Order  |                                   |
| Respond to Limit Order   | Respond to Webhook    | Respond to limit order POST                | limit order code        | -                       |                                   |
| Clear all signals (POST) | Webhook               | Receive request to clear all signals      | -                      | clear all signals       |                                   |
| clear all signals        | Code                  | Clear all stored signals                   | Clear all signals (POST)| confirm signals are cleared |                                |
| confirm signals are cleared| Respond to Webhook   | Confirm clearing of signals                | clear all signals       | -                       |                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "Receive Signal (POST)"**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Webhook ID: "mt5-signal-receive"  
   - Purpose: Receive incoming trading signals.

2. **Create Code Node: "Store Signal"**  
   - Type: Code  
   - Script: Implement logic to parse and store the received signal data into internal storage or database.  
   - Connect "Receive Signal (POST)" output to this node.

3. **Create Respond to Webhook Node: "Respond to POST"**  
   - Type: Respond to Webhook  
   - Default HTTP 200 response to confirm receipt.  
   - Connect "Store Signal" output to this node.

4. **Create Webhook Node: "Get Pending Signals (GET)"**  
   - Type: Webhook  
   - HTTP Method: GET  
   - Webhook ID: "mt5-get-signals"  
   - Purpose: Provide endpoint to fetch pending signals.

5. **Create Code Node: "Fetch Pending Signals"**  
   - Type: Code  
   - Script: Retrieve list of pending signals from storage.  
   - Connect "Get Pending Signals (GET)" output to this node.

6. **Create Respond to Webhook Node: "Return Signals"**  
   - Type: Respond to Webhook  
   - Purpose: Return pending signals as JSON in response.  
   - Connect "Fetch Pending Signals" output to this node.

7. **Create Webhook Node: "Confirm Signal Processed (POST)"**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Webhook ID: "mt5-confirm"  
   - Purpose: Receive confirmation that signal was processed.

8. **Create Code Node: "Mark as Processed"**  
   - Type: Code  
   - Script: Update the storage to mark the signal as processed based on input data.  
   - Connect "Confirm Signal Processed (POST)" output to this node.

9. **Create Respond to Webhook Node: "Confirm Response"**  
   - Type: Respond to Webhook  
   - Purpose: Send confirmation response to POST request.  
   - Connect "Mark as Processed" output to this node.

10. **Create Webhook Node: "Market Order (POST)"**  
    - Type: Webhook  
    - HTTP Method: POST  
    - Webhook ID: "bee7e24f-efa9-41b2-8761-1871c15479da"  
    - Purpose: Receive market order requests.

11. **Create Code Node: "market order code"**  
    - Type: Code  
    - Script: Process market order instructions, validate and execute trade via MT5 API or backend.  
    - Connect "Market Order (POST)" output to this node.

12. **Create Respond to Webhook Node: "Respond to Webhook"**  
    - Type: Respond to Webhook  
    - Purpose: Confirm market order request reception/execution.  
    - Connect "market order code" output to this node.

13. **Create Webhook Node: "Limit Order (POST)"**  
    - Type: Webhook  
    - HTTP Method: POST  
    - Webhook ID: "mt5-limit-order-webhook"  
    - Purpose: Receive limit order requests.

14. **Create Code Node: "limit order code"**  
    - Type: Code  
    - Script: Validate and process limit order instructions.  
    - Connect "Limit Order (POST)" output to this node.

15. **Create Respond to Webhook Node: "Respond to Limit Order"**  
    - Type: Respond to Webhook  
    - Purpose: Confirm limit order request.  
    - Connect "limit order code" output to this node.

16. **Create Webhook Node: "Clear all signals (POST)"**  
    - Type: Webhook  
    - HTTP Method: POST  
    - Webhook ID: "bee7e24f-efa9-41b2-8761-1871c15479da" (same as Market Order, distinguish internally)  
    - Purpose: Receive request to clear all stored signals.

17. **Create Code Node: "clear all signals"**  
    - Type: Code  
    - Script: Clear all stored signals from database or memory.  
    - Connect "Clear all signals (POST)" output to this node.

18. **Create Respond to Webhook Node: "confirm signals are cleared"**  
    - Type: Respond to Webhook  
    - Purpose: Send confirmation of clearing signals.  
    - Connect "clear all signals" output to this node.

**Credential Configuration:**  
- No explicit credentials are visible in this workflow. If interaction with MT5 API requires authentication, configure credentials accordingly (e.g., API keys or OAuth2).  
- Webhooks may require secure access control (e.g., API keys or IP whitelisting) to prevent unauthorized access.

**Default Values / Constraints:**  
- Webhook nodes listen on specified IDs with HTTP methods POST or GET as indicated.  
- Code nodes require scripting for storage and processing logic; ensure robust error handling in code.  
- Respond nodes provide HTTP responses; customize status codes and messages as needed.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow serves as a backend signal handler for automated Forex and Gold trading with MetaTrader 5 via webhooks, enabling integration into trading bots or dashboards. | Workflow description |
| Webhook IDs are critical for external MT5 integration; ensure these match those configured in MT5 or client applications. | Integration detail |
| Careful handling of data validation and error cases in code nodes is essential to avoid processing invalid or malicious signals. | Best practice |
| Consider securing webhook endpoints using authentication or IP filtering to prevent unauthorized access. | Security best practice |
| For more on n8n Webhook nodes and Respond to Webhook nodes, visit: https://docs.n8n.io/nodes/n8n-nodes-base.webhook/ | Official n8n Documentation |

---

This document provides a detailed, structured reference for the "Automated Forex and Gold Trading Signal Handler with MetaTrader 5 via Webhooks" workflow, facilitating understanding, reproduction, and modification.