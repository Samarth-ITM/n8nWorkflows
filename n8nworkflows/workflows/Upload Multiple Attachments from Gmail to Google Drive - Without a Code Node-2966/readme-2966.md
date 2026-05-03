Upload Multiple Attachments from Gmail to Google Drive - Without a Code Node

https://n8nworkflows.xyz/workflows/upload-multiple-attachments-from-gmail-to-google-drive---without-a-code-node-2966


# Upload Multiple Attachments from Gmail to Google Drive - Without a Code Node

### 1. Workflow Overview

This workflow automates the extraction and conditional uploading of multiple attachments from unread Gmail messages to a specified Google Drive folder, **without using any Code nodes**. It is designed for users who want to handle multiple email attachments efficiently within n8n, leveraging native nodes and expressions.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Triggered by new unread Gmail messages from a specific sender, downloading all attachments.
- **1.2 Attachment Splitting:** Splits multiple attachments from a single email into individual items for separate processing.
- **1.3 Attachment Classification:** Uses a Switch node to categorize attachments based on file size into "Large", "Medium", or "Extra".
- **1.4 Conditional Upload or Handling:** Uploads attachments to Google Drive if they meet size criteria or routes them to no-op nodes for other cases.
- **1.5 Documentation and Reference:** Sticky notes provide detailed guidance on expression usage and workflow logic.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for new unread Gmail messages from a specified sender and downloads all attachments with a specific prefix.

- **Nodes Involved:**  
  - Gmail Trigger

- **Node Details:**

  - **Gmail Trigger**  
    - Type: Trigger node for Gmail  
    - Configuration:  
      - Filters: Sender email set to "ray.thomas@charter.com", only unread messages  
      - Options: Downloads attachments, prefixes attachment binary data keys with "attachment_"  
      - Polling: Checks every minute  
    - Expressions/Variables: None (trigger node)  
    - Input: External (Gmail inbox)  
    - Output: Email data including attachments as binary data with keys like "attachment_0", "attachment_1", etc.  
    - Version: 1.2  
    - Edge Cases:  
      - Authentication errors if OAuth2 token expires or is revoked  
      - No new emails matching filter results in no workflow execution  
      - Large attachments may cause delays or timeouts  
    - Credentials: Gmail OAuth2 account "rthomascharter"

---

#### 2.2 Attachment Splitting

- **Overview:**  
  Splits the multiple attachments contained in the binary data of a single email item into separate workflow items for individual processing.

- **Nodes Involved:**  
  - Split Out  
  - Sticky Note1 (documentation)

- **Node Details:**

  - **Split Out**  
    - Type: Utility node to split array or object fields into multiple items  
    - Configuration:  
      - Field to split out: `$binary` (literal string, not an expression)  
      - This splits each binary attachment into its own item, preserving all metadata  
    - Expressions: Uses `$binary` as a fixed field name to split  
    - Input: Output from Gmail Trigger (single item with multiple binary attachments)  
    - Output: Multiple items, each containing one attachment binary data keyed by its original name (e.g., "attachment_0")  
    - Version: 1  
    - Edge Cases:  
      - If no attachments, no items output  
      - Binary keys must exist; otherwise, node fails or outputs empty  
    - Sticky Note1 explains this splitting logic and the use of `$binary` field

  - **Sticky Note1**  
    - Content: Explains the use of `$binary` to split multiple binary files and notes that the binary keys remain non-homogenized (e.g., "attachment_0")  
    - Position: Near Split Out node for user reference

---

#### 2.3 Attachment Classification

- **Overview:**  
  Categorizes each attachment by file size into three groups: Large (>300 kB), Medium (>10 kB), and Extra (all others). This enables conditional routing for different handling strategies.

- **Nodes Involved:**  
  - Switch

- **Node Details:**

  - **Switch**  
    - Type: Conditional routing node  
    - Configuration:  
      - Rules based on attachment file size (numeric part extracted from string like "100 kB")  
      - Outputs:  
        - "Large Files" if size > 300 kB  
        - "Medium Files" if size > 10 kB  
        - "extra" fallback for all others  
      - Expression used: `{{ $binary.values()[0].fileSize.split(' ')[0].toNumber() }}` to extract numeric size  
    - Input: Items from Split Out node (each with one attachment)  
    - Output: Three branches for different size categories  
    - Version: 3.2  
    - Edge Cases:  
      - File size string format must be consistent (e.g., "100 kB") or expression may fail  
      - Attachments without fileSize metadata may cause errors or misclassification

---

#### 2.4 Conditional Upload or Handling

- **Overview:**  
  Uploads attachments classified as Medium or Large to Google Drive, while routing others to no-op nodes (placeholders for alternative handling).

- **Nodes Involved:**  
  - Google Drive  
  - Send "Too Big" Notification (for example)  
  - Ignore Little Graphics / Icons (for example)  
  - Sticky Note (documentation)

- **Node Details:**

  - **Google Drive**  
    - Type: Google Drive node to upload files  
    - Configuration:  
      - File name: Expression `{{ $binary.values()[0].fileName }}` to dynamically name the file  
      - Drive: "My Drive" selected  
      - Folder ID: Fixed folder `"0BwqhgrfUUaOuM2x1NXhxLUlGVEE"` (misc folder)  
      - Input Data Field Name: Expression `{{ $binary.keys()[0] }}` to reference the binary data key dynamically  
    - Input: Items from Switch node's "Medium Files" output  
    - Output: Uploaded file metadata  
    - Version: 3  
    - Edge Cases:  
      - Authentication errors if Google Drive OAuth2 token expires  
      - Folder ID must be valid and accessible  
      - Large files may cause timeouts or API limits  
    - Credentials: Google Drive OAuth2 account "rthomascharter"

  - **Send "Too Big" Notification (for example)**  
    - Type: No Operation (NoOp) node, placeholder for notification or other handling of large files  
    - Input: Switch node's "Large Files" output  
    - Output: None  
    - Version: 1

  - **Ignore Little Graphics / Icons (for example)**  
    - Type: No Operation (NoOp) node, placeholder for ignoring small files or icons  
    - Input: Switch node's "extra" output  
    - Output: None  
    - Version: 1

  - **Sticky Note**  
    - Content: Provides reference examples of expressions to access single binary data properties regardless of key name, such as fileName, fileExtension, fileSize, mimeType, etc.  
    - Position: Near Google Drive node for user reference

---

### 3. Summary Table

| Node Name                              | Node Type               | Functional Role                          | Input Node(s)           | Output Node(s)                        | Sticky Note                                                                                                  |
|--------------------------------------|-------------------------|----------------------------------------|-------------------------|-------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Gmail Trigger                        | Gmail Trigger           | Trigger on unread Gmail from sender, download attachments | External (Gmail inbox)   | Split Out                           |                                                                                                              |
| Split Out                           | Split Out               | Split multiple attachments into individual items | Gmail Trigger           | Switch                             | ## Split Multiple Binary Files: This uses the `$binary` name (not expression var) to split attachments.       |
| Switch                             | Switch                  | Classify attachments by file size into Large, Medium, Extra | Split Out               | Send " Too Big" Notification, Google Drive, Ignore Little Graphics |                                                                                                              |
| Send " Too Big" Notification (for example) | No Operation (NoOp)     | Placeholder for handling large files (e.g., notification) | Switch (Large Files)     | None                               |                                                                                                              |
| Google Drive                       | Google Drive            | Upload attachments to Google Drive folder | Switch (Medium Files)    | None                               | ## Reference "Single" Binary Using Expressions: Examples of referencing single binary data regardless of key. |
| Ignore Little Graphics / Icons (for example) | No Operation (NoOp)     | Placeholder to ignore small files/icons | Switch (extra)           | None                               |                                                                                                              |
| Sticky Note1                       | Sticky Note             | Documentation on splitting multiple binaries | None                    | None                               | ## Split Multiple Binary Files: Explains `$binary` splitting and binary key naming.                           |
| Sticky Note                       | Sticky Note             | Documentation on referencing single binary data | None                    | None                               | ## Reference "Single" Binary Using Expressions: Examples of expressions for fileName, fileSize, mimeType, etc.|

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**  
   - Type: Gmail Trigger  
   - Configure:  
     - Filters: Sender = "ray.thomas@charter.com", Read Status = "unread"  
     - Options: Enable "Download Attachments"  
     - Attachments Prefix: "attachment_"  
     - Polling: Every minute  
   - Credentials: Connect Gmail OAuth2 account with appropriate scopes for reading emails and attachments.

2. **Add Split Out Node**  
   - Type: Split Out  
   - Configure:  
     - Field to Split Out: Enter literal string `$binary` (do not use expression syntax)  
   - Connect Gmail Trigger output to Split Out input.

3. **Add Switch Node**  
   - Type: Switch  
   - Configure rules:  
     - Rule 1: Name "Large Files"  
       - Condition: Numeric value of attachment file size > 300  
       - Expression: `{{ $binary.values()[0].fileSize.split(' ')[0].toNumber() }}`  
     - Rule 2: Name "Medium Files"  
       - Condition: Numeric value of attachment file size > 10  
       - Expression: same as above  
     - Fallback output: "extra"  
   - Connect Split Out output to Switch input.

4. **Add Google Drive Node**  
   - Type: Google Drive  
   - Configure:  
     - Operation: Upload file  
     - File Name: Expression `{{ $binary.values()[0].fileName }}`  
     - Drive: Select "My Drive"  
     - Folder ID: Enter `"0BwqhgrfUUaOuM2x1NXhxLUlGVEE"` (or your target folder ID)  
     - Input Data Field Name: Expression `{{ $binary.keys()[0] }}`  
   - Credentials: Connect Google Drive OAuth2 account with file upload permissions.  
   - Connect Switch output "Medium Files" to Google Drive input.

5. **Add No Operation (NoOp) Nodes**  
   - Create two NoOp nodes:  
     - "Send \" Too Big\" Notification (for example)"  
     - "Ignore Little Graphics / Icons (for example)"  
   - Connect Switch output "Large Files" to "Send \" Too Big\" Notification" node.  
   - Connect Switch output "extra" to "Ignore Little Graphics / Icons" node.

6. **Add Sticky Notes for Documentation (Optional but Recommended)**  
   - Add a sticky note near Split Out node explaining the use of `$binary` to split multiple attachments.  
   - Add a sticky note near Google Drive node with example expressions for referencing single binary data properties:  
     - Input Data Field Name: `{{ $binary.keys()[0] }}`  
     - File Name: `{{ $binary.values()[0].fileName }}`  
     - File Extension: `{{ $binary.values()[0].fileExtension }}`  
     - File Size (string): `{{ $binary.values()[0].fileSize }}`  
     - File Size (numeric): `{{ $binary.values()[0].fileSize.split(' ')[0].toNumber() }}`  
     - Mime Type: `{{ $binary.values()[0].mimeType }}`  
     - Attachment ID: `{{ $binary.values()[0].id }}`

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                          | Context or Link                                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow demonstrates how to handle multiple email attachments in n8n without using any Code nodes, relying on native nodes and expression syntax for dynamic referencing.                                                      | Workflow description and design rationale                                                                       |
| The key technique is using the literal `$binary` field in the Split Out node to split multiple binary attachments into separate items, enabling individual processing.                                                               | Workflow description                                                                                             |
| Expressions to dynamically access attachment metadata regardless of the binary key name are essential for generic handling of multiple attachments.                                                                                | Sticky Note near Google Drive node                                                                               |
| For alternative workflows using Code nodes for similar functionality, see: [Get Multiple Attachments from Gmail and upload them to GDrive](https://n8n.io/workflows/2348-get-multiple-attachments-from-gmail-and-upload-them-to-gdrive/) | External workflow link                                                                                           |
| Ensure OAuth2 credentials for Gmail and Google Drive are properly configured with required scopes to avoid authentication errors.                                                                                                   | Credentials setup                                                                                                 |
| Folder ID for Google Drive must be valid and accessible by the connected account; replace `"0BwqhgrfUUaOuM2x1NXhxLUlGVEE"` with your own folder ID as needed.                                                                      | Google Drive folder configuration                                                                                 |

---

This structured reference document provides a comprehensive understanding of the workflow’s logic, node configurations, and reproduction steps, enabling advanced users and automation agents to maintain, modify, or extend the workflow confidently.