Summarize invoices with AWS Textract, Google Gemini, and send to Slack

https://n8nworkflows.xyz/workflows/summarize-invoices-with-aws-textract--google-gemini--and-send-to-slack-13803


# Summarize invoices with AWS Textract, Google Gemini, and send to Slack

# Workflow Reference: Summarize Invoices with AWS Textract, Google Gemini, and Slack

This document provides a comprehensive technical analysis of the n8n workflow designed to automate the extraction, summarization, and distribution of invoice data using cloud-native AI and storage services.

---

### 1. Workflow Overview

The workflow automates the transition from a manual file upload to a structured AI summary delivered via Slack. It is designed for finance and operations teams to streamline invoice auditing and record-keeping without manual data entry.

**Logical Blocks:**
- **1.1 Input Reception:** Captures a file via an n8n-hosted form.
- **1.2 Storage & OCR:** Simultaneously archives the file in AWS S3 and extracts text content using AWS Textract.
- **1.3 AI Processing:** Summarizes the raw text extraction into key financial highlights using Google Gemini.
- **1.4 Notification:** Delivers the final summary to a specific Slack recipient.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception
*   **Overview:** Acts as the entry point, providing a public-facing interface for users to submit documents.
*   **Nodes Involved:** `On form submission`.
*   **Node Details:**
    *   **On form submission (n8n-nodes-base.formTrigger):**
        *   **Type:** Trigger.
        *   **Configuration:** Title set to "invoice extractor". It defines a single required field named `data` of type `file`.
        *   **Output:** Binary data object containing the uploaded file.
        *   **Failure Modes:** Webhook URL misconfiguration; file size exceeding n8n instance limits.

#### 2.2 Storage & OCR
*   **Overview:** This block performs two parallel actions: persistent storage and optical character recognition (OCR).
*   **Nodes Involved:** `Upload a file`, `AWS Textract`.
*   **Node Details:**
    *   **Upload a file (n8n-nodes-base.awsS3):**
        *   **Type:** Action (AWS S3).
        *   **Configuration:** Set to "Upload". It uses the bucket named "test".
        *   **Key Expressions:** `{{ $json.data[0].filename }}` is used to name the file in S3.
        *   **Edge Cases:** Bucket permissions (IAM) errors; bucket name mismatch; duplicate filename conflicts.
    *   **AWS Textract (n8n-nodes-base.awsTextract):**
        *   **Type:** Action.
        *   **Configuration:** Default extraction settings to process the binary data from the trigger.
        *   **Output:** JSON structure containing lines, words, and tables extracted from the image/PDF.
        *   **Failure Modes:** Unsupported file format (e.g., non-image/PDF); AWS Textract service limits or region unavailability.

#### 2.3 AI Processing
*   **Overview:** Transforms the raw, often messy OCR text into a human-readable summary.
*   **Nodes Involved:** `Message a model`.
*   **Node Details:**
    *   **Message a model (@n8n/n8n-nodes-langchain.googleGemini):**
        *   **Type:** AI Model Interaction.
        *   **Configuration:** Uses `gemini-2.5-flash`.
        *   **System Prompt:** "you will be given invoice data you have to summarize that invoice with key important things".
        *   **Input:** Receives extracted text from the AWS Textract node.
        *   **Failure Modes:** API Quota exhaustion (Google AI Studio); context window limits if the invoice is exceptionally long.

#### 2.4 Notification
*   **Overview:** Final delivery of the summarized information.
*   **Nodes Involved:** `Send a message`.
*   **Node Details:**
    *   **Send a message (n8n-nodes-base.slack):**
        *   **Type:** Action (Slack).
        *   **Configuration:** Sends a direct message to a specific user (identified in the workflow as "asana" with ID `U07G370G5MW`).
        *   **Input:** Text output from the Google Gemini node.
        *   **Failure Modes:** Slack OAuth2 token expiration; User ID no longer valid in the workspace.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| On form submission | Form Trigger | Entry Point/File Upload | (None) | Upload a file, AWS Textract | Automate Invoice Data Extraction and Summarization |
| Upload a file | AWS S3 | File Archiving | On form submission | (None) | Upload to AWS and extract text |
| AWS Textract | AWS Textract | OCR/Text Extraction | On form submission | Message a model | Upload to AWS and extract text |
| Message a model | Google Gemini | AI Summarization | AWS Textract | Send a message | summarize and send to slack |
| Send a message | Slack | Notification | Message a model | (None) | summarize and send to slack |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:**
    *   Add an **n8n Form Trigger** node.
    *   Set the title to "invoice extractor" and add a required "File" field named `data`.
2.  **S3 Integration:**
    *   Create an **AWS S3** node. Set the operation to "Upload".
    *   Connect the Form Trigger to this node.
    *   Use the expression `{{ $json.data[0].filename }}` for the File Name.
    *   Set the Bucket Name to your preferred bucket (e.g., "test").
3.  **OCR Setup:**
    *   Create an **AWS Textract** node and connect it to the **Form Trigger**.
    *   Ensure your AWS Credentials have `Textract:DetectDocumentText` and `Textract:AnalyzeDocument` permissions.
4.  **AI Integration:**
    *   Create a **Google Gemini** node (Language Model).
    *   Connect the **AWS Textract** node output to the Gemini input.
    *   Select model `gemini-2.5-flash` (or current equivalent).
    *   Set the prompt to request a summary of the key invoice details.
5.  **Slack Integration:**
    *   Create a **Slack** node. Set the operation to "Message" and "Post to User".
    *   Connect the **Google Gemini** node to this node.
    *   Select the target user or channel to receive the results.
6.  **Credentials:**
    *   Configure **AWS Credentials** (Access Key/Secret) for S3 and Textract.
    *   Configure **Google Gemini API Key**.
    *   Configure **Slack OAuth2 API**.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **How it works:** 1. User submits via form. 2. Stored in S3. 3. Textract extracts data. 4. Gemini summarizes. 5. Slack delivers. | Overall Workflow Logic |
| **Setup Prerequisites:** Connect S3, Gemini, and Slack accounts; ensure AWS IAM permissions for S3 and Textract. | Pre-deployment Checklist |
| **S3 Storage:** The file name is dynamically resolved from the form submission data. | Configuration Detail |