Create Secure Interactive Applications with WhatsApp Flows End-to-End Encryption

https://n8nworkflows.xyz/workflows/create-secure-interactive-applications-with-whatsapp-flows-end-to-end-encryption-3973


# Create Secure Interactive Applications with WhatsApp Flows End-to-End Encryption

### 1. Workflow Overview

This workflow securely manages end-to-end encrypted interactive data exchange between a business and WhatsApp users via WhatsApp Flows, adhering strictly to WhatsApp Business Encryption specifications. It uses hybrid encryption—RSA for secure AES key exchange and AES-GCM for payload encryption—to ensure confidentiality and integrity of sensitive data transmitted over WhatsApp Business API.

The workflow is logically divided into four main blocks:

- **1.1 Webhook Entry & Initial Decryption**: Receives encrypted payloads from WhatsApp, converts incoming Base64 strings into buffers, decrypts the AES key using RSA private key, then decrypts the payload with AES-GCM.

- **1.2 Payload Parsing & Preprocessing**: Parses decrypted JSON payloads to extract key parameters such as the current "screen" context and flow token for routing decisions.

- **1.3 Flow Decision & Business Logic Routing**: Routes execution based on the extracted "screen" parameter, enabling different handling for appointment scheduling or seat selection.

- **1.4 Response Generation & Encryption**: Creates tailored response data depending on the business logic branch, encrypts the response using the AES key and an inverted initialization vector (IV), and sends the encrypted Base64 response back to WhatsApp.

---

### 2. Block-by-Block Analysis

#### 2.1 Webhook Entry & Initial Decryption Block

**Overview:**  
This block handles incoming HTTP POST requests from WhatsApp, containing encrypted data components. It converts Base64-encoded strings to buffers and performs a secure two-step decryption: RSA private key decrypts AES key, which then decrypts the payload using AES-GCM.

**Nodes Involved:**  
- Webhook1  
- move to base64  
- Decryption Code  

**Node Details:**

- **Webhook1**  
  - Type: Webhook  
  - Role: Listens for incoming encrypted WhatsApp POST requests on path `/flow`.  
  - Configuration: HTTP POST, responds with output from subsequent node, sets response header `Content-Type: text/plain`.  
  - Inputs: External HTTP requests.  
  - Outputs: Passes raw JSON body with encrypted fields.  
  - Edge cases: Missing or malformed requests, HTTP errors, webhook ID must be unique.  
  - Version: 2  

- **move to base64**  
  - Type: Code (JavaScript)  
  - Role: Converts Base64-encoded strings (`encrypted_flow_data`, `encrypted_aes_key`, `initial_vector`) from the webhook payload into binary buffers required for cryptographic operations.  
  - Configuration: Uses Node.js Buffer API to decode Base64 strings, returns buffers in JSON properties.  
  - Inputs: Output of Webhook1.  
  - Outputs: JSON containing buffers for encryption fields.  
  - Edge cases: Missing fields in input JSON; logs input for debugging.  
  - Version: 2  

- **Decryption Code**  
  - Type: Code (JavaScript)  
  - Role: Decrypts the AES key with RSA private key (using PKCS1 OAEP padding and SHA-256 hash), then decrypts the encrypted payload with AES-128-GCM using the decrypted AES key and initialization vector (IV). Extracts authentication tag from payload.  
  - Configuration:  
    - RSA private key embedded in code (replace placeholder with actual key).  
    - Validates buffer sizes for AES key and IV (must be 16 bytes).  
    - Error handling for missing data or decryption failures.  
  - Inputs: Buffers from `move to base64`.  
  - Outputs: JSON containing decrypted payload string and Base64-encoded AES key.  
  - Version: 2  
  - Edge cases: Invalid key lengths, missing data, decryption errors (throws with detailed messages).  

---

#### 2.2 Payload Parsing & Preprocessing Block

**Overview:**  
Parses decrypted JSON payloads to extract critical parameters such as the current screen identifier, date, and flow token. Structures data for downstream routing and processing.

**Nodes Involved:**  
- Json Parser  

**Node Details:**

- **Json Parser**  
  - Type: Code (JavaScript)  
  - Role: Parses decrypted JSON string in `decryptedPayload` to extract:  
    - `screen`: Current UI screen or interaction step.  
    - `date`: Relevant date from data object.  
    - `flow_token`: Token for session continuity.  
  - Configuration:  
    - Tries-catch block to handle JSON parse errors gracefully.  
    - Passes error info as JSON if parsing fails.  
  - Inputs: Output from `Decryption Code` (contains decrypted JSON string).  
  - Outputs: Structured JSON with extracted fields and original payload for debugging.  
  - Version: 2  
  - Edge cases: Malformed JSON, missing expected properties, parsing exceptions.  

---

#### 2.3 Flow Decision & Business Logic Routing Block

**Overview:**  
Routes workflow execution based on the `screen` value extracted from the decrypted payload, allowing distinct processing logic for different interactive screens in the WhatsApp Flow.

**Nodes Involved:**  
- Switch  

**Node Details:**

- **Switch**  
  - Type: Switch  
  - Role: Branches execution based on exact string match of `$json.screen`.  
  - Configuration:  
    - Two main conditions handled: `"APPOINTMENT"` and `"DATE_SELECTION_SCREEN"`.  
    - Unhandled screens implicitly ignored or can be extended.  
  - Inputs: Output from `Json Parser`.  
  - Outputs:  
    - Output 1: For `"APPOINTMENT"` screen, connects to appointment data processing.  
    - Output 2: For `"DATE_SELECTION_SCREEN"` screen, connects to seat selection processing.  
  - Version: 3.2  
  - Edge cases: Unknown or missing screen values lead to no path; no default path configured.  

---

#### 2.4 Appointment Data Handling Block

**Overview:**  
Processes appointment-related data by grouping appointment times per date, then prepares and encrypts a response back to WhatsApp Flow.

**Nodes Involved:**  
- Data Extraction Code  
- Encrypt Return  
- Respond to Webhook1  

**Node Details:**

- **Data Extraction Code**  
  - Type: Code (JavaScript)  
  - Role: Groups appointment slots by date from input items, producing a summarized structure of dates and associated start times.  
  - Configuration: Uses JavaScript `reduce` to aggregate start times indexed by ISO date string.  
  - Inputs: Output from `Switch` for `"APPOINTMENT"` screen.  
  - Outputs: Array of objects with `appointment_date` and `start_times` array.  
  - Version: 2  
  - Edge cases: Empty input, inconsistent date/time formats.  

- **Encrypt Return**  
  - Type: Code (JavaScript)  
  - Role: Encrypts the response payload for the appointment screen.  
  - Configuration:  
    - Reads the initial IV and AES key from upstream nodes.  
    - Inverts IV bits according to WhatsApp encryption protocol.  
    - Prepares response JSON with status, times, dates, flow token, and next screen info (`"APPOINTMENT"`).  
    - Encrypts with AES-128-GCM; appends auth tag; encodes as Base64.  
  - Inputs: Output from `Data Extraction Code`.  
  - Outputs: JSON with encrypted Base64 response body.  
  - Version: 2  
  - Edge cases: Missing or malformed input data, missing flow token or keys, encryption errors.  

- **Respond to Webhook1**  
  - Type: Respond to Webhook  
  - Role: Finalizes the HTTP response to WhatsApp by sending the encrypted Base64 string with HTTP 200 and `text/plain` content-type.  
  - Inputs: Output from `Encrypt Return`.  
  - Outputs: HTTP response.  
  - Version: 1.1  
  - Edge cases: Network or response errors.  

---

#### 2.5 Seat Selection Handling Block

**Overview:**  
Processes seat selection data from decrypted payload, formats it, encrypts the response, and returns it to WhatsApp.

**Nodes Involved:**  
- Data Extraction Code1  
- Encrypt Return1  
- Respond to Webhook2  

**Node Details:**

- **Data Extraction Code1**  
  - Type: Code (JavaScript)  
  - Role: Parses seat selection data from the decrypted payload's `data.seats` array, outputting individual seat objects.  
  - Configuration: Parses original decrypted payload JSON, extracts seats array, maps to output format.  
  - Inputs: Output from `Switch` for `"DATE_SELECTION_SCREEN"`.  
  - Outputs: Array of seat objects for encryption.  
  - Version: 2  
  - Edge cases: Missing or empty seats array, malformed JSON.  

- **Encrypt Return1**  
  - Type: Code (JavaScript)  
  - Role: Encrypts the seat selection response.  
  - Configuration:  
    - Similar to `Encrypt Return` node but formats seat data with id and title.  
    - Uses inverted IV and AES key.  
    - Sets next screen to `"SUMMARY"`.  
  - Inputs: Output from `Data Extraction Code1`.  
  - Outputs: Encrypted Base64 response body.  
  - Version: 2  
  - Edge cases: Missing seats or flow token, encryption failures.  

- **Respond to Webhook2**  
  - Type: Respond to Webhook  
  - Role: Sends encrypted seat selection response back to WhatsApp.  
  - Inputs: Output from `Encrypt Return1`.  
  - Outputs: HTTP response.  
  - Version: 1.1  
  - Edge cases: Network or response failures.  

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                              | Input Node(s)        | Output Node(s)         | Sticky Note                                                                                                      |
|---------------------|---------------------|----------------------------------------------|----------------------|------------------------|------------------------------------------------------------------------------------------------------------------|
| Webhook1            | Webhook             | Entry point for encrypted WhatsApp requests | —                    | move to base64          | Described in Sticky Note as part of Webhook Entry & Initial Decryption Block                                    |
| move to base64       | Code                | Converts Base64 strings to binary buffers    | Webhook1             | Decryption Code         | See above                                                                                                        |
| Decryption Code      | Code                | Decrypts AES key with RSA, decrypts payload | move to base64       | Json Parser             | See above                                                                                                        |
| Json Parser          | Code                | Parses decrypted JSON payload                 | Decryption Code       | Switch                  | See above                                                                                                        |
| Switch               | Switch              | Routes flow based on screen parameter         | Json Parser           | Data Extraction Code, Data Extraction Code1 | See above                                                                                                        |
| Data Extraction Code | Code                | Groups appointment slots by date              | Switch (APPOINTMENT)  | Encrypt Return          | See above                                                                                                        |
| Encrypt Return       | Code                | Encrypts appointment response                  | Data Extraction Code  | Respond to Webhook1     | See above                                                                                                        |
| Respond to Webhook1  | Respond to Webhook   | Sends encrypted appointment response          | Encrypt Return        | —                       | See above                                                                                                        |
| Data Extraction Code1| Code                | Extracts seat selection data                   | Switch (DATE_SELECTION_SCREEN) | Encrypt Return1         | See above                                                                                                        |
| Encrypt Return1      | Code                | Encrypts seat selection response               | Data Extraction Code1 | Respond to Webhook2     | See above                                                                                                        |
| Respond to Webhook2  | Respond to Webhook   | Sends encrypted seat selection response       | Encrypt Return1       | —                       | See above                                                                                                        |
| Sticky Note          | Sticky Note          | Documentation and usage instructions          | —                    | —                       | Detailed explanation of the workflow blocks and usage provided                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Name: `Webhook1`  
   - HTTP Method: POST  
   - Path: `flow`  
   - Response Mode: `responseNode`  
   - Response Headers: `Content-Type: text/plain`  
   - Save and activate webhook.

2. **Add Code Node to Convert Base64 to Buffers**  
   - Type: Code  
   - Name: `move to base64`  
   - JavaScript: Decode `encrypted_flow_data`, `encrypted_aes_key`, `initial_vector` from Base64 strings to Buffers using `Buffer.from()`.  
   - Connect output of `Webhook1` to this node.

3. **Add Code Node for Decryption**  
   - Type: Code  
   - Name: `Decryption Code`  
   - JavaScript:  
     - Import Node.js crypto library.  
     - Paste your 2048-bit RSA private key in PEM format, replacing placeholder.  
     - Convert inputs to buffers.  
     - Decrypt AES key with `crypto.privateDecrypt` (RSA_PKCS1_OAEP_PADDING, SHA256).  
     - Use first 16 bytes of decrypted key as AES-128 key.  
     - Extract auth tag from last 16 bytes of encrypted payload.  
     - Decrypt payload using AES-128-GCM with extracted IV and auth tag.  
     - Return decrypted UTF-8 string and AES key as Base64.  
   - Connect output of `move to base64` to this node.

4. **Add Code Node to Parse Decrypted JSON**  
   - Type: Code  
   - Name: `Json Parser`  
   - JavaScript:  
     - Parse JSON string from `decryptedPayload`.  
     - Extract `screen`, `date`, `flow_token`.  
     - Handle JSON parse errors gracefully.  
   - Connect output of `Decryption Code` to this node.

5. **Add Switch Node for Screen Routing**  
   - Type: Switch  
   - Name: `Switch`  
   - Mode: String  
   - Conditions:  
     - If `screen` equals `"APPOINTMENT"` → output 1  
     - If `screen` equals `"DATE_SELECTION_SCREEN"` → output 2  
   - Connect output of `Json Parser` to this node.

6. **Create Appointment Data Processing Nodes**  
   - **Data Extraction Code** (Code node):  
     - Group appointment slots by date from input.  
   - **Encrypt Return** (Code node):  
     - Encrypt response with AES-GCM using inverted IV and AES key.  
     - Format response with status, time/date slots, flow token, and next screen `"APPOINTMENT"`.  
   - **Respond to Webhook1** (Respond to Webhook node):  
     - Send encrypted Base64 response with HTTP 200.  
   - Connect Switch output 1 → Data Extraction Code → Encrypt Return → Respond to Webhook1.

7. **Create Seat Selection Processing Nodes**  
   - **Data Extraction Code1** (Code node):  
     - Extract seats array from decrypted payload.  
   - **Encrypt Return1** (Code node):  
     - Encrypt response similarly, with seat data and next screen `"SUMMARY"`.  
   - **Respond to Webhook2** (Respond to Webhook node):  
     - Send encrypted Base64 response.  
   - Connect Switch output 2 → Data Extraction Code1 → Encrypt Return1 → Respond to Webhook2.

8. **Credentials Setup**  
   - No explicit external credentials used beyond the RSA private key embedded in `Decryption Code`.  
   - Ensure the RSA private key is securely stored and replaced in the code node.  
   - WhatsApp Business API must be configured to use the matching public key and encryption protocol.

9. **Validation and Testing**  
   - Deploy the workflow.  
   - Configure WhatsApp Business API to send encrypted payloads to the webhook URL.  
   - Send test encrypted messages matching the expected JSON structure.  
   - Monitor logs and responses for correct encrypted replies.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                             |
|----------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| This workflow implements the WhatsApp Business Encryption protocol using RSA for key exchange and AES-GCM for payload encryption. | Official WhatsApp Business Encryption documentation and Cloud API encryption guides.                        |
| RSA Key setup requires generation of a 2048-bit RSA key pair; private key embedded here; public key must be registered with WhatsApp. | RSA key generation tools such as OpenSSL; WhatsApp Cloud API endpoint `/v17.0/{WhatsApp-Business-Account-ID}/whatsapp_business_encryption`. |
| Encryption uses AES-128-GCM with inverted IV for response, per WhatsApp protocol specification.                     | WhatsApp Business API encryption specifications.                                                           |
| The workflow assumes JSON payload includes `screen` and `flow_token` fields to manage user interaction steps.       | WhatsApp Flows interactive application design.                                                             |

---

This document fully describes the workflow structure, logic, node configurations, edge cases, and reproduction steps to facilitate secure encrypted communication with WhatsApp Flows using n8n.