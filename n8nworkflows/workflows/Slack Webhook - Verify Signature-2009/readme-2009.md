Slack Webhook - Verify Signature

https://n8nworkflows.xyz/workflows/slack-webhook---verify-signature-2009


# Slack Webhook - Verify Signature

### 1. Workflow Overview

This Sub-Workflow is designed to securely verify incoming Slack webhook requests by validating their signatures. Its primary purpose is to ensure that the requests genuinely originate from Slack, preventing unauthorized access or malicious bot interactions. This verification step is essential for any main workflow that processes Slack webhook events, adding a critical security layer.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives the Slack request object to be verified.
- **1.2 Signature Base String Construction:** Builds the canonical base string used for signature verification according to Slack’s specification.
- **1.3 HMAC SHA256 Signature Calculation:** Computes the HMAC SHA256 hash of the base string using the Slack Signing Secret.
- **1.4 Signature Verification Decision:** Compares the computed signature against the signature sent by Slack.
- **1.5 Result Handling:** Outputs a verified flag and original data on success or raises an error if verification fails.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block accepts the incoming Slack webhook request as input. It is intended to be placed immediately after a Slack Webhook node in the main workflow to capture the raw request data.

- **Nodes Involved:**  
  - Execute Workflow Trigger  
  - Merge (partial input handling)

- **Node Details:**

  - **Execute Workflow Trigger**  
    - Type: *Execute Workflow Trigger*  
    - Role: Entry point node for this Sub-Workflow, triggered by the main workflow.  
    - Configuration: No parameters; automatically receives input from the parent workflow.  
    - Inputs: Receives the Slack request JSON object (headers, body, etc.) from the main workflow.  
    - Outputs: Passes the Slack request JSON to the next node, “Make Slack Verif Token.”  
    - Failures: None expected here; relies on correct upstream data.

  - **Merge**  
    - Type: *Merge*  
    - Role: Combines outputs for final output after verification.  
    - Configuration: Mode set to “combine” with combination mode “mergeByPosition.”  
    - Inputs: Receives data from “Execute Workflow Trigger” and from “Set Verified to True” node after verification.  
    - Outputs: Passes merged data (including verification flag) to the Sub-Workflow output.  
    - Failures: Misalignment of data length could cause merging issues, but controlled here.

---

#### 2.2 Signature Base String Construction

- **Overview:**  
  Constructs the signature base string as defined by Slack’s security model. This string concatenates the Slack version prefix, the request timestamp, and the exact encoded request body to be used for HMAC signature calculation.

- **Nodes Involved:**  
  - Make Slack Verif Token

- **Node Details:**

  - **Make Slack Verif Token**  
    - Type: *Code* (JavaScript)  
    - Role: Creates the canonical signature base string from the incoming request.  
    - Configuration:  
      - Custom JS code that:  
        - Encodes the request body using a custom encoder matching Slack’s rules (handles spaces, asterisks, and tildes).  
        - Extracts the timestamp from headers `x-slack-request-timestamp`.  
        - Builds the base string as `v0:timestamp:encodedBody`.  
        - Extracts the provided Slack signature from header `x-slack-signature`.  
    - Key expressions/variables:  
      - `requestJson.headers["x-slack-request-timestamp"]`  
      - `requestJson.headers["x-slack-signature"]`  
      - `requestJson.body`  
      - `sigBaseString` (output)  
      - `requestSignature` (output)  
    - Inputs: Receives the Slack request JSON from the previous node.  
    - Outputs: JSON with `sigBaseString` and `requestSignature`.  
    - Edge Cases & Failures:  
      - Missing or malformed headers or body could cause undefined values or JS errors.  
      - Encoding errors if body format is unexpected.  
      - Slack may send requests with replay attack attempts (old timestamps), not handled here but recommended to consider.  
    - Version: Requires n8n supporting Code node v2 for full JS features like `replaceAll`.

---

#### 2.3 HMAC SHA256 Signature Calculation

- **Overview:**  
  Uses the Slack Signing Secret to compute the HMAC SHA256 hash of the signature base string, producing a candidate signature for comparison.

- **Nodes Involved:**  
  - Encode Secret String

- **Node Details:**

  - **Encode Secret String**  
    - Type: *Crypto*  
    - Role: Calculates the HMAC SHA256 of the signature base string using the Slack Signing Secret as the key.  
    - Configuration:  
      - Action: HMAC  
      - Algorithm: SHA256  
      - Value: Expression `={{ $json.sigBaseString }}` (the base string from previous node)  
      - Secret: Slack Signing Secret (set via credential or parameter, must be configured in main workflow)  
      - Output property: `candidateSignature`  
    - Inputs: Receives JSON containing `sigBaseString` from “Make Slack Verif Token.”  
    - Outputs: JSON with calculated `candidateSignature`.  
    - Edge Cases & Failures:  
      - Missing or incorrect Slack Signing Secret will cause wrong signatures.  
      - Timeout or errors in crypto node are rare but possible.  
    - Credential: Requires Slack Signing Secret securely stored in n8n credentials.

---

#### 2.4 Signature Verification Decision

- **Overview:**  
  Compares the Slack-provided signature against the calculated one. If they match, the signature is verified; otherwise, the workflow triggers an error.

- **Nodes Involved:**  
  - IF  
  - Set Verified to True  
  - Stop and Error

- **Node Details:**

  - **IF**  
    - Type: *If*  
    - Role: Conditional check node to compare signatures.  
    - Configuration:  
      - Condition: String comparison  
      - Compares `$json.requestSignature` vs. `"v0="+$json.candidateSignature`  
      - Note: Slack signatures are prefixed with `v0=` in header, so this prefix is included in comparison.  
    - Inputs: Takes JSON with both signatures from “Encode Secret String.”  
    - Outputs:  
      - True branch: signature matches  
      - False branch: signature mismatch  
    - Edge Cases:  
      - If either signature is null or malformed, comparison will fail.  
      - Timing attacks not prevented here (if critical, implement constant-time compare in custom code).  

  - **Set Verified to True**  
    - Type: *Set*  
    - Role: On success, sets a flag `signature_verified` to `true` in output JSON.  
    - Configuration:  
      - Adds boolean field `signature_verified` = true  
    - Inputs: True branch from IF node.  
    - Outputs: Data with verification flag for downstream processing.  

  - **Stop and Error**  
    - Type: *Stop and Error*  
    - Role: Terminates workflow execution with an error when signature verification fails.  
    - Configuration:  
      - Error message: “Could not verify Slack Webhook signature”  
    - Inputs: False branch from IF node.  
    - Outputs: Workflow error raised.  
    - Handling: Main workflow can catch and handle this error using Error Workflow or node error handling settings.

---

#### 2.5 Result Handling and Output

- **Overview:**  
  Combines the original Slack request data with the verification status for output back to the main workflow.

- **Nodes Involved:**  
  - Merge

- **Node Details:**

  - **Merge** (continuation from 2.1)  
    - Type: *Merge*  
    - Role: Combines the original Slack request data with the verification flag to form the final verified output.  
    - Configuration: See above.  
    - Inputs:  
      - Main input: Original Slack request (from “Execute Workflow Trigger”)  
      - Secondary input: Verified flag data (from “Set Verified to True”)  
    - Outputs: Single JSON object containing both original request and `signature_verified: true`.  
    - Edge Cases: If verification fails, this node is not reached due to error stop.

---

### 3. Summary Table

| Node Name               | Node Type                  | Functional Role                              | Input Node(s)               | Output Node(s)           | Sticky Note                                                                                                                         |
|-------------------------|----------------------------|----------------------------------------------|-----------------------------|--------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| Execute Workflow Trigger | Execute Workflow Trigger    | Entry point for Sub-Workflow, receives Slack request | —                           | Make Slack Verif Token, Merge (main input) | ## Input The input should be the received Slack request. Place this workflow directly after the Slack Webhook.                      |
| Make Slack Verif Token   | Code                       | Builds Slack signature base string            | Execute Workflow Trigger     | Encode Secret String      | ## Slack Webhook - Verify Signature When receiving a message from a Slack Webhook, it is much more secure to verify that the message comes from Slack and not from bots or unknown services. This small template is designed to validate the received signature (See [this URL](https://api.slack.com/authentication/verifying-requests-from-slack)). ### Colors - **Blue** areas are **areas to edit** - **Yellow** areas are **explanations** |
| Encode Secret String     | Crypto                     | Calculates HMAC SHA256 candidate signature   | Make Slack Verif Token       | IF                       | ## TO EDIT Set your Slack Signing Secret. You can obtain it by visiting your Slack App dashboard in the general tab: https://api.slack.com/apps/[SLACK_APP_ID]/general |
| IF                      | If                         | Checks if calculated signature matches Slack signature | Encode Secret String         | Set Verified to True (true branch), Stop and Error (false branch) | ## Error Output If the signature cannot be verified, an error will be raised. You can manage this scenario in your main workflow by either using an Error Workflow or by modifying your node settings and selecting appropriate actions in the event of an error. |
| Set Verified to True     | Set                        | Sets verification flag to true on success    | IF (true branch)             | Merge                    | ## Success Output If the signature is successfully verified, we return a key `verified_signature` set to `true` along with the data from the Slack request itself. |
| Stop and Error           | Stop and Error              | Raises error when signature verification fails | IF (false branch)            | —                        | See above (Error Output)                                                                                                            |
| Merge                   | Merge                      | Combines original request and verification flag | Execute Workflow Trigger, Set Verified to True | —                        | See above (Input and Success Output)                                                                                               |
| Sticky Note             | Sticky Note                | Informational notes                           | —                           | —                        | Multiple sticky notes covering input, output, error handling, and editing instructions (see individual notes in table rows)        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new Workflow in n8n** and set it as a Sub-Workflow.

2. **Add an Execute Workflow Trigger node:**  
   - Type: *Execute Workflow Trigger*  
   - Purpose: Receives input from the main workflow (Slack webhook request).  
   - No special configuration needed.

3. **Add a Code node named “Make Slack Verif Token”:**  
   - Paste the JavaScript code that:  
     - Defines `encodeFormData` to encode the request body parameters according to Slack’s spec (spaces replaced by "+", "*" and "~" percent-encoded).  
     - Defines `buildSigBaseString` that extracts `x-slack-request-timestamp` and encodes the body, then constructs `v0:timestamp:encodedBody`.  
     - Extracts `x-slack-signature` from headers.  
     - Outputs JSON with `sigBaseString` and `requestSignature`.  
   - Connect **Execute Workflow Trigger → Make Slack Verif Token**.

4. **Add a Crypto node named “Encode Secret String”:**  
   - Type: *Crypto*  
   - Action: HMAC  
   - Algorithm: SHA256  
   - Value: Expression `={{ $json.sigBaseString }}`  
   - Secret: Set your Slack Signing Secret here (use credentials or direct parameter).  
   - Output property: `candidateSignature`  
   - Connect **Make Slack Verif Token → Encode Secret String**.

5. **Add an If node named “IF”:**  
   - Condition Type: String  
   - Compare: `{{$json.requestSignature}}` equals `=v0={{$json.candidateSignature}}`  
   - Connect **Encode Secret String → IF**.

6. **Add a Set node named “Set Verified to True”:**  
   - Add boolean field named `signature_verified` with value `true`.  
   - Connect **IF (true branch) → Set Verified to True**.

7. **Add a Stop and Error node named “Stop and Error”:**  
   - Error Message: “Could not verify Slack Webhook signature”  
   - Connect **IF (false branch) → Stop and Error**.

8. **Add a Merge node named “Merge”:**  
   - Mode: Combine  
   - Combination Mode: mergeByPosition  
   - Connect:  
     - **Execute Workflow Trigger → Merge (main input)**  
     - **Set Verified to True → Merge (secondary input)**

9. **Configure the workflow output:**  
   - Set the Merge node as the final output node to return the combined data including verification status.

10. **Add Sticky Notes for documentation:**  
    - Include notes on:  
      - Where to set Slack Signing Secret (link to Slack App dashboard).  
      - Input data expectations (must come right after Slack Webhook).  
      - Success output details.  
      - Error handling instructions.  
      - General security explanation and link to Slack’s verification docs.

11. **Save and activate the Sub-Workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                          |
|------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow is a Sub-Workflow and should be called immediately after a Slack Webhook node to verify incoming Slack requests.    | Core integration instruction.                                                                            |
| Slack Signing Secret must be obtained from your Slack App dashboard under the General tab: https://api.slack.com/apps/[APP_ID]/general | Official Slack documentation.                                                                            |
| Verification follows Slack’s official guide: https://api.slack.com/authentication/verifying-requests-from-slack                    | Slack security best practices.                                                                           |
| On verification failure, the workflow raises an error. Handle errors in main workflow via Error Workflow or node error settings. | Recommended error handling strategy.                                                                     |
| Encoding uses custom replacements beyond `encodeURIComponent` for full Slack compatibility (spaces to "+", "*" and "~" encoding). | Important for accurate signature matching.                                                               |

---

This document provides a detailed and structured reference for understanding, reproducing, and modifying the “Slack Webhook - Verify Signature” workflow in n8n, ensuring secure Slack webhook integrations.