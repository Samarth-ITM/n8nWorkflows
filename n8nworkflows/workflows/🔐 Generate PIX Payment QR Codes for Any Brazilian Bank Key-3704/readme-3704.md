🔐 Generate PIX Payment QR Codes for Any Brazilian Bank Key

https://n8nworkflows.xyz/workflows/---generate-pix-payment-qr-codes-for-any-brazilian-bank-key-3704


# 🔐 Generate PIX Payment QR Codes for Any Brazilian Bank Key

### 1. Workflow Overview

This workflow, titled **"PIX QR Code Generator for payments in Brazil"**, automates the creation of compliant PIX payment QR Codes for any valid Brazilian PIX key. It is designed to support multiple PIX key types (CPF, CNPJ, phone, email, or random key) and follows the official BACEN (Central Bank of Brazil) EMV QR Code standards.

**Target Use Cases:**  
- E-commerce platforms generating instant payment QR Codes  
- Chatbots or CRM systems automating payment requests  
- Subscription or billing systems requiring fast, secure payment links  
- Any business or developer needing to generate PIX payment QR Codes dynamically

**Logical Blocks:**

- **1.1 Input Reception:** Receives and sets initial payment data (PIX key, amount, receiver name, city, description).  
- **1.2 PIX Code Generation:** Processes input data to assemble a fully compliant PIX payment code with checksum.  
- **1.3 QR Code Creation:** Generates a QR Code image from the PIX payment code and prepares output data.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block collects and prepares the essential payment information needed to generate the PIX code. It initializes variables and sets default or user-provided values.

- **Nodes Involved:**  
  - Click Test (Manual Trigger)  
  - PixFieldSend (Set Node)

- **Node Details:**

  - **Click Test**  
    - *Type:* Manual Trigger  
    - *Role:* Entry point for manual workflow execution/testing.  
    - *Configuration:* No parameters; triggers workflow on manual click.  
    - *Connections:* Output connects to PixFieldSend node.  
    - *Edge Cases:* None specific; manual trigger ensures controlled execution.

  - **PixFieldSend**  
    - *Type:* Set  
    - *Role:* Initializes and sets input fields for the PIX payment, such as `pixKey`, `receiverName`, `city`, `amount`, and `description`.  
    - *Configuration:* Presumably configured to accept or set these variables (not explicitly shown in JSON but implied by workflow purpose).  
    - *Connections:* Input from Click Test; output to Generate PIX node.  
    - *Edge Cases:* Missing or invalid input values could cause downstream errors; validation is recommended.

---

#### 2.2 PIX Code Generation

- **Overview:**  
  This block constructs the PIX payment code string according to BACEN's EMV QR Code standard, including the automatic calculation and attachment of the CRC checksum.

- **Nodes Involved:**  
  - Generate PIX (Code node)

- **Node Details:**

  - **Generate PIX**  
    - *Type:* Code (JavaScript)  
    - *Role:* Core logic to assemble the PIX EMV payment code string from input fields, compute the CRC checksum, and finalize the payload.  
    - *Configuration:* Contains custom code implementing BACEN’s PIX payload formatting rules and CRC calculation.  
    - *Key Expressions:* Uses input variables such as `pixKey`, `amount`, `receiverName`, `city`, and `description` to build the code.  
    - *Connections:* Input from PixFieldSend; output to QRCodePIX node.  
    - *Edge Cases:*  
      - Incorrect input formats (e.g., invalid PIX key formats) may cause code generation errors.  
      - CRC calculation failures or code assembly bugs could invalidate the PIX code.  
      - No explicit error handling shown; adding try/catch recommended.  
    - *Version Requirements:* Requires n8n version supporting Code node v2.

---

#### 2.3 QR Code Creation and Output Preparation

- **Overview:**  
  This block generates a dynamic QR Code image from the PIX payment code and prepares the final output, including the full PIX code and a public URL to the QR Code image.

- **Nodes Involved:**  
  - QRCodePIX (HTTP Request)  
  - PIXFields (Set)

- **Node Details:**

  - **QRCodePIX**  
    - *Type:* HTTP Request  
    - *Role:* Sends the generated PIX code to an external QR Code generation service or API to create a QR Code image.  
    - *Configuration:* Likely configured to POST or GET with the PIX code as payload or query parameter to a QR Code API (exact parameters not shown).  
    - *Connections:* Input from Generate PIX; output to PIXFields.  
    - *Edge Cases:*  
      - Network errors or API downtime could prevent QR Code generation.  
      - API limits or invalid request formatting could cause failures.  
      - No authentication indicated; if API changes, credentials might be needed.  
    - *Version Requirements:* HTTP Request node v4.2 or higher.

  - **PIXFields**  
    - *Type:* Set  
    - *Role:* Finalizes the output by setting the response data structure, including the full PIX payment code, QR Code image URL, and clean metadata for frontend or bot consumption.  
    - *Configuration:* Sets output fields from previous node data.  
    - *Connections:* Input from QRCodePIX; no further outputs (end of workflow).  
    - *Edge Cases:* Missing or malformed data from QRCodePIX could cause incomplete output.

---

### 3. Summary Table

| Node Name    | Node Type           | Functional Role                         | Input Node(s)     | Output Node(s) | Sticky Note |
|--------------|---------------------|---------------------------------------|-------------------|----------------|-------------|
| Click Test   | Manual Trigger      | Workflow entry point for manual start | —                 | PixFieldSend   |             |
| PixFieldSend | Set                 | Initializes and sets input payment data | Click Test        | Generate PIX   |             |
| Generate PIX | Code                | Generates PIX EMV payment code with CRC | PixFieldSend      | QRCodePIX     |             |
| QRCodePIX    | HTTP Request        | Generates QR Code image from PIX code | Generate PIX      | PIXFields      |             |
| PIXFields    | Set                 | Prepares final output with PIX code and QR URL | QRCodePIX        | —              |             |
| Sticky Note  | Sticky Note         | —                                     | —                 | —              |             |
| Sticky Note1 | Sticky Note         | —                                     | —                 | —              |             |
| Sticky Note2 | Sticky Note         | —                                     | —                 | —              |             |
| Sticky Note3 | Sticky Note         | —                                     | —                 | —              |             |
| Sticky Note4 | Sticky Note         | —                                     | —                 | —              |             |

*Note: Sticky Notes are present but empty; no comments or links to preserve.*

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `Click Test`  
   - Type: Manual Trigger  
   - Purpose: To manually start the workflow for testing.

2. **Create Set Node for Input Fields**  
   - Name: `PixFieldSend`  
   - Type: Set  
   - Configure fields to accept or set the following variables:  
     - `pixKey` (string): The PIX key (CPF, CNPJ, phone, email, or random key)  
     - `receiverName` (string): Name of the payment receiver  
     - `city` (string): City associated with the receiver  
     - `amount` (number or string): Payment amount in BRL (e.g., 79.90)  
     - `description` (string, optional): Payment description  
   - Connect output of `Click Test` to input of `PixFieldSend`.

3. **Create Code Node to Generate PIX Code**  
   - Name: `Generate PIX`  
   - Type: Code (JavaScript)  
   - Paste or write code implementing:  
     - Assembly of PIX EMV payment code string from inputs  
     - Automatic CRC checksum calculation and attachment  
     - Output the full PIX code string for QR generation  
   - Connect output of `PixFieldSend` to input of `Generate PIX`.

4. **Create HTTP Request Node to Generate QR Code**  
   - Name: `QRCodePIX`  
   - Type: HTTP Request  
   - Configure to send the PIX code to a QR Code generation API (e.g., a public QR Code service or your own)  
   - Method: POST or GET depending on API  
   - Parameters: Include the PIX code as required by the API  
   - Connect output of `Generate PIX` to input of `QRCodePIX`.

5. **Create Set Node for Final Output**  
   - Name: `PIXFields`  
   - Type: Set  
   - Configure to set output fields:  
     - `pixCode`: The full PIX payment code string from `Generate PIX`  
     - `qrCodeUrl`: Public URL of the generated QR Code image from `QRCodePIX` response  
     - Additional metadata fields as needed for frontend or bot consumption  
   - Connect output of `QRCodePIX` to input of `PIXFields`.

6. **Activate and Test Workflow**  
   - Save and activate the workflow.  
   - Trigger manually via `Click Test`.  
   - Verify outputs: PIX code string and QR Code image URL.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                           |
|----------------------------------------------------------------------------------------------------|------------------------------------------|
| PIX is a real-time payment system by BACEN, enabling instant, fee-free transfers in Brazil.        | Workflow description section              |
| Workflow supports all valid PIX key types: CPF, CNPJ, phone, email, or random keys.                 | Workflow description section              |
| No external paid APIs required; QR Code generation uses open or free services.                      | Workflow description section              |
| Ready-to-use workflows available at: [https://iloveflows.com](https://iloveflows.com)              | Purchase and additional workflows         |
| Try n8n Cloud with partner link: [https://n8n.partnerlinks.io/amanda](https://n8n.partnerlinks.io/amanda) | n8n Cloud trial and hosting                |

---

This documentation provides a complete understanding of the PIX QR Code Generator workflow, enabling reproduction, modification, and troubleshooting by advanced users or AI agents.