Validate TOTP Token (without creating a credential)

https://n8nworkflows.xyz/workflows/validate-totp-token--without-creating-a-credential--2379


# Validate TOTP Token (without creating a credential)

### 1. Workflow Overview

This workflow provides a template for validating a 6-digit Time-based One-Time Password (TOTP) code against a given base32-encoded TOTP secret without the need to create or store credentials within n8n. It is primarily designed for authentication systems that require verifying a user-provided 2FA code.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Manual trigger and example input setup nodes to provide TOTP secret and code for testing.
- **1.2 TOTP Validation:** A Python function node that decodes the secret, generates the current valid TOTP code, and compares it with the input code.
- **1.3 Validation Result Routing:** An IF node that branches the flow based on whether the provided code matches the generated TOTP code.
- **1.4 Documentation:** A sticky note node explaining the workflow’s purpose, usage, and setup instructions.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** This block captures the initiation of the workflow via manual trigger and sets example input data for testing the TOTP validation.
- **Nodes Involved:**  
  - When clicking ‘Test workflow’  
  - EXAMPLE FIELDS

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow on manual user action.  
  - Configuration: Default manual trigger; no parameters needed.  
  - Inputs: None  
  - Outputs: Connected to EXAMPLE FIELDS node.  
  - Edge cases: None; user must manually trigger for execution.

- **EXAMPLE FIELDS**  
  - Type: Set  
  - Role: Sets example data for TOTP secret and code to verify.  
  - Configuration:  
    - `totp_secret_example`: String with base32 TOTP secret (e.g., "CNSUKUMZLQJEZJ3")  
    - `code_to_verify_example`: String with 6-digit code to verify (e.g., "516620")  
  - Inputs: From Manual Trigger node  
  - Outputs: Connected to TOTP VALIDATION node  
  - Edge cases: The code must be provided as a string to preserve leading zeros. Numeric input could cause validation errors.

#### 1.2 TOTP Validation

- **Overview:** This block runs a Python script that decodes the base32 TOTP secret, generates the current valid TOTP code based on the current time, and compares it with the user-supplied code to determine validity.  
- **Nodes Involved:**  
  - TOTP VALIDATION

**Node Details:**

- **TOTP VALIDATION**  
  - Type: Code (Python)  
  - Role: Implements the logic to generate TOTP and verify the user code.  
  - Configuration (interpreted):  
    - Decodes base32 secret with padding correction.  
    - Calculates the TOTP code using HMAC-SHA1 with a 30-second interval, per RFC 6238.  
    - Compares generated code with input code.  
    - Returns a JSON object with `status: 1` if valid, `status: 0` otherwise.  
  - Key expressions/variables:  
    - `secret = _input.item.json.totp_secret_example` (line 39) — expects base32 secret string input.  
    - `code = _input.item.json.code_to_verify_example` (line 40) — expects 6-digit code string.  
  - Inputs: From EXAMPLE FIELDS node  
  - Outputs: To IF CODE IS VALID node  
  - Version-specific: Python 3 environment required in n8n (version 2 of the node).  
  - Edge cases and potential failures:  
    - Incorrect base32 secret format or invalid characters can cause decoding errors.  
    - If the code is treated as a number (losing leading zeros), validation will fail.  
    - Time synchronization issues on the server may cause unexpected invalid results.  
    - No credential required, simplifying usage but increasing responsibility for input correctness.  
  - Sub-workflow: None

#### 1.3 Validation Result Routing

- **Overview:** This block routes the flow based on whether the TOTP code is valid (`status` equals 1) or invalid (`status` equals 0). This differentiation is essential to trigger subsequent conditional flows in a larger system.  
- **Nodes Involved:**  
  - IF CODE IS VALID

**Node Details:**

- **IF CODE IS VALID**  
  - Type: IF  
  - Role: Checks if the `status` output from TOTP VALIDATION equals 1.  
  - Configuration:  
    - Condition: Number equals 1 on expression `{{$json.status}}`.  
  - Inputs: From TOTP VALIDATION node  
  - Outputs: Two branches — `true` branch if code is valid, `false` branch if invalid  
  - Version-specific: Node version 2.1 used, supporting expression evaluation and number comparison.  
  - Edge cases:  
    - If the TOTP VALIDATION node fails or returns unexpected output, this node will not match the condition and default to `false` branch.  
    - If `status` field is missing, the condition fails.

#### 1.4 Documentation

- **Overview:** Provides in-workflow documentation explaining the template’s purpose, usage instructions, and testing guidelines for users modifying or deploying this workflow.  
- **Nodes Involved:**  
  - Sticky Note

**Node Details:**

- **Sticky Note**  
  - Type: Sticky Note  
  - Role: Displays detailed information about the workflow’s function, example usage, setup, and testing instructions.  
  - Configuration: Multi-line markdown content describing the workflow.  
  - Inputs: None  
  - Outputs: None  
  - Edge cases: None; purely informational.

---

### 3. Summary Table

| Node Name                  | Node Type          | Functional Role                   | Input Node(s)             | Output Node(s)        | Sticky Note                                                                                                                          |
|----------------------------|--------------------|---------------------------------|---------------------------|-----------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger     | Initiates workflow manually      | —                         | EXAMPLE FIELDS         |                                                                                                                                      |
| EXAMPLE FIELDS             | Set                | Provides example TOTP secret and code for testing | When clicking ‘Test workflow’ | TOTP VALIDATION       |                                                                                                                                      |
| TOTP VALIDATION            | Code (Python)      | Validates the TOTP code against secret | EXAMPLE FIELDS             | IF CODE IS VALID       |                                                                                                                                      |
| IF CODE IS VALID           | IF                 | Branches flow based on validation result | TOTP VALIDATION            | — (true/false branches) |                                                                                                                                      |
| Sticky Note                | Sticky Note        | Documentation and usage instructions | —                         | —                     | ## TOTP Validation with Function Node. This template allows you to verify if a 6-digit TOTP code is valid using the corresponding TOTP secret. It can be used in an authentication system. Setup guidelines and testing instructions included. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `When clicking ‘Test workflow’`  
   - Default settings; no parameters required.

2. **Create a Set node**  
   - Name: `EXAMPLE FIELDS`  
   - Add two fields:  
     - `totp_secret_example` (String): enter a base32 TOTP secret (e.g., `"CNSUKUMZLQJEZJ3"`)  
     - `code_to_verify_example` (String): enter a 6-digit code as text (e.g., `"516620"`)  
   - Connect output of `When clicking ‘Test workflow’` to input of this node.

3. **Create a Code node**  
   - Name: `TOTP VALIDATION`  
   - Language: Python  
   - Paste the following logic (modify lines 39-40 to match input field names if different):  
   ```python
   import hmac
   import hashlib
   import time
   import base64

   def base32_decode(key):
       key += '=' * (-len(key) % 8)
       return base64.b32decode(key.upper(), casefold=True)

   def generate_totp(secret, interval=30, digits=6):
       interval_count = int(time.time() // interval)
       interval_bytes = interval_count.to_bytes(8, byteorder='big')

       hmac_hash = hmac.new(secret, interval_bytes, hashlib.sha1).digest()

       offset = hmac_hash[-1] & 0x0F
       binary_code = ((hmac_hash[offset] & 0x7F) << 24 |
                      (hmac_hash[offset + 1] & 0xFF) << 16 |
                      (hmac_hash[offset + 2] & 0xFF) << 8 |
                      (hmac_hash[offset + 3] & 0xFF))

       otp_code = binary_code % (10 ** digits)
       otp_code_str = str(otp_code).zfill(digits)

       return otp_code_str

   def verify_totp(secret, code, interval=30, digits=6):
       secret_bytes = base32_decode(secret)
       generated_code = generate_totp(secret_bytes, interval, digits)

       return generated_code == code

   secret = _input.item.json.totp_secret_example
   code = _input.item.json.code_to_verify_example

   if verify_totp(secret, code):
       return [{"status": 1}]
   else:
       return [{"status": 0}]
   ```  
   - Connect output of `EXAMPLE FIELDS` to input of `TOTP VALIDATION`.

4. **Create an IF node**  
   - Name: `IF CODE IS VALID`  
   - Condition:  
     - Type: Number  
     - Operation: Equals  
     - Left Value: `{{$json.status}}` (expression)  
     - Right Value: `1`  
   - Connect output of `TOTP VALIDATION` to input of this node.  
   - This node will have two outputs: true (code valid) and false (code invalid).

5. **Create a Sticky Note node**  
   - Name: `Sticky Note`  
   - Content: Add detailed explanation of the workflow purpose, usage, and testing instructions.  
   - Position it visually apart for clarity.  
   - No connections needed.

6. **Arrange nodes visually** for clarity: manual trigger → set example fields → code validation → IF node.

7. **Run manual test:**  
   - Click `Execute Workflow` on the manual trigger.  
   - Based on the example fields, the validation will branch accordingly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                         | Context or Link                              |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------|
| The 6-digit TOTP code **must be provided as a string** to preserve leading zeros, which are critical for correct validation. Numeric values may lead to incorrect results due to truncation of leading zeros.                                          | Setup Guidelines section                      |
| RFC 6238 standard underlies the TOTP generation algorithm used in this workflow.                                                                                                                                                                      | General knowledge                             |
| The workflow does not require creating or storing credentials in n8n, simplifying deployment in scenarios where secrets are managed externally.                                                                                                   | Workflow overview                             |
| This workflow can be integrated downstream into broader authentication or user verification workflows by branching on the IF node outputs.                                                                                                         | Possible integration context                   |

---

This structured documentation enables developers and automation agents to fully understand, reproduce, and adapt the TOTP validation workflow within n8n.