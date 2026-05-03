Scan WhatsApp receipt photos and track expenses using Wati and GPT-4o

https://n8nworkflows.xyz/workflows/scan-whatsapp-receipt-photos-and-track-expenses-using-wati-and-gpt-4o-13654


# Scan WhatsApp receipt photos and track expenses using Wati and GPT-4o

# Reference Document: WhatsApp AI Receipt Scanner & Expense Tracker

## 1. Workflow Overview
This workflow automates the process of tracking expenses via WhatsApp using WATI, OpenAI (GPT-4o), and Google Sheets. It allows users to simply snap a photo of a receipt and send it to a WhatsApp number to have the details automatically extracted and logged. Additionally, users can request a visual monthly spending report by sending a text command.

The logic is organized into four primary functional blocks:
*   **1.1 Input Reception & Routing:** Captures the WhatsApp message and determines if the user is submitting a receipt (image) or requesting a report (text).
*   **1.2 AI Receipt Scanning:** Downloads the image, prepares it for AI processing, and uses GPT-4o Vision to extract structured data (vendor, amount, date, etc.).
*   **1.3 Data Logging & Confirmation:** Saves the extracted data to a Google Sheet and sends a confirmation message back to the user.
*   **1.4 Monthly Reporting:** Aggregates all expenses for the current month, calculates category-wise totals, and sends a formatted summary with visual progress bars.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Routing
**Overview:** This block acts as the entry point, receiving webhooks from WATI and routing the execution based on the message type.
*   **Nodes Involved:** `Wati Trigger`, `Route Message`.
*   **Node Details:**
    *   **Wati Trigger:** Monitors the `messageReceived` event. It captures the sender's ID (`waId`), name, and the message content (text or media URL).
    *   **Route Message (Switch):** 
        *   *Output 1 (Image Receipt):* Triggered if `$json.type` equals "image".
        *   *Output 2 (Monthly Report):* Triggered if `$json.text` matches the specific report command (configured to check for the presence of text).
        *   *Fallback:* Handles unexpected inputs.

### 2.2 AI Receipt Scanning
**Overview:** Downloads the physical image from WATI’s servers and uses Vision AI to "read" the receipt.
*   **Nodes Involved:** `Get a media file1`, `Prepare Image for OpenAI1`, `OpenAI – Extract Receipt Data2`.
*   **Node Details:**
    *   **Get a media file1 (Wati):** Uses the `mediaFileUrl` from the trigger to download the binary image file.
    *   **Prepare Image for OpenAI1 (Code):** Converts the binary buffer into a Base64 data URL. This is necessary because OpenAI's API requires the image as a URL or a base64 string.
    *   **OpenAI – Extract Receipt Data2 (HTTP Request):** Sends the image to the `gpt-4o` model. It uses a strict system prompt to ensure the AI returns **only** raw JSON containing `vendor`, `amount`, `currency`, `date`, `category`, and `description`.
    *   **Edge Cases:** If the image is blurry or not a receipt, GPT-4o is instructed to return `null` for fields, which is handled in the next block.

### 2.3 Data Logging & Confirmation
**Overview:** Processes the AI's response and commits it to permanent storage.
*   **Nodes Involved:** `Parse & Validate Expense`, `Google Sheets – Log Expense`, `Send Expenditure message`.
*   **Node Details:**
    *   **Parse & Validate Expense (Code):** Parses the JSON string from OpenAI. It also enriches the data with the sender's phone number, name, and a timestamp. It defaults missing values (e.g., "Unknown Vendor") to prevent sheet errors.
    *   **Google Sheets – Log Expense:** Appends a new row to the spreadsheet. Columns include `date`, `month`, `phone`, `amount`, `vendor`, `category`, `currency`, `timestamp`, and `description`.
    *   **Send Expenditure message (Wati):** Sends a WhatsApp session message to the user confirming the log was successful, echoing back the vendor and amount for verification.

### 2.4 Monthly Reporting
**Overview:** Triggered when the user sends a "report" command; it summarizes the user's financial activity for the current month.
*   **Nodes Involved:** `Google Sheets – Read This Month`, `Build Monthly Report`, `Send Expenditure Report - month`.
*   **Node Details:**
    *   **Google Sheets – Read This Month:** Fetches all records from the "Expenses" sheet.
    *   **Build Monthly Report (Code):** Filters the records by the current month (YYYY-MM) and the specific user's phone number. It calculates category totals and generates a text-based visual bar chart (using `█` and `░` characters) for the WhatsApp message.
    *   **Send Expenditure Report - month (Wati):** Delivers the final formatted report and grand total to the user.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Wati Trigger | Wati Trigger | Webhook Entry | None | Route Message | 1️⃣ Webhook + Route |
| Route Message | Switch | Logical Routing | Wati Trigger | Get a media file1, Google Sheets – Read | 1️⃣ Webhook + Route |
| Get a media file1 | Wati | Media Download | Route Message | Prepare Image for OpenAI1 | 2️⃣ Download & Scan Receipt |
| Prepare Image for OpenAI1 | Code | Data Formatting | Get a media file1 | OpenAI – Extract Receipt Data2 | 2️⃣ Download & Scan Receipt |
| OpenAI – Extract Receipt Data2 | HTTP Request | Vision AI Analysis | Prepare Image for OpenAI1 | Parse & Validate Expense | 2️⃣ Download & Scan Receipt |
| Parse & Validate Expense | Code | Data Validation | OpenAI – Extract Receipt Data2 | Google Sheets – Log Expense | 3️⃣ Log to Sheets + Confirm |
| Google Sheets – Log Expense | Google Sheets | Data Storage | Parse & Validate Expense | Send Expenditure message | 3️⃣ Log to Sheets + Confirm |
| Send Expenditure message | Wati | User Confirmation | Google Sheets – Log Expense | None | 3️⃣ Log to Sheets + Confirm |
| Google Sheets – Read This Month | Google Sheets | Data Retrieval | Route Message | Build Monthly Report | 4️⃣ Monthly Report |
| Build Monthly Report | Code | Report Generation | Google Sheets – Read This Month | Send Expenditure Report - month | 4️⃣ Monthly Report |
| Send Expenditure Report - month | Wati | User Reporting | Build Monthly Report | None | 4️⃣ Monthly Report |

---

## 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:**
    *   Add a **Wati Trigger** node. Connect your WATI account and set the event to `messageReceived`.
2.  **Routing:**
    *   Add a **Switch** node. Set rule 1: If `type` equals `image`. Set rule 2: If `text` contains `report`.
3.  **Receipt Processing Path:**
    *   Add a **Wati** node (Action: `getMedia`). Use the expression `{{ $json.data }}` (the media URL from the trigger).
    *   Add a **Code** node to convert the binary to Base64. Use `this.helpers.getBinaryDataBuffer` to fetch the image and return a `dataUrl` string.
    *   Add an **HTTP Request** node. Set to `POST` to `https://api.openai.com/v1/chat/completions`. Use your OpenAI API Key.
    *   **Body Setup:** Use model `gpt-4o`. Prompt the AI to return JSON and pass the `dataUrl` in the `image_url` property.
4.  **Logging Path:**
    *   Add a **Code** node to use `JSON.parse()` on the OpenAI response and map it to your Sheet column headers.
    *   Add a **Google Sheets** node (Action: `Append`). Map the JSON fields to your columns (Date, Amount, Vendor, etc.).
    *   Add a **Wati** node (Action: `sendMessage`). Send a success message to `{{ $waId }}`.
5.  **Reporting Path:**
    *   Add a **Google Sheets** node (Action: `Read`). Select the same spreadsheet used in the logging path.
    *   Add a **Code** node. Filter the rows using JavaScript to match the current month and the user's ID. Calculate totals per category.
    *   Add a **Wati** node to send the generated `reportText` back to the user.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Requires WATI Account | Essential for WhatsApp connectivity. |
| OpenAI Model: GPT-4o | Ensure your API tier has access to GPT-4o for vision capabilities. |
| Google Sheets Structure | Ensure columns `timestamp`, `phone`, `vendor`, `amount`, `currency`, `date`, `category`, `description`, `month` exist in "Sheet1". |
| Data Privacy | Note that receipt images are processed by OpenAI; ensure compliance with your organization's data policy. |