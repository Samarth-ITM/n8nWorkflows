Compare LINE palm images and log Gemini health insights to Google Sheets

https://n8nworkflows.xyz/workflows/compare-line-palm-images-and-log-gemini-health-insights-to-google-sheets-13646


# Compare LINE palm images and log Gemini health insights to Google Sheets

# Workflow Reference: AI Palm Health Tracker – LINE Image Comparison & Gemini Analysis

This document provides a technical breakdown of the n8n workflow designed to receive palm images via the LINE Messaging API, analyze them using Google Gemini AI, and log health-related insights into Google Sheets while maintaining a historical comparison for returning users.

---

### 1. Workflow Overview

The workflow automates the process of health tracking through palmistry and visual analysis. It handles image reception, cloud storage, historical data retrieval, and AI-driven comparative analysis.

**Logical Blocks:**
- **1.1 Input Reception & Configuration:** Receives the webhook from LINE and sets up session variables.
- **1.2 Media Handling & Storage:** Downloads the image from LINE servers and uploads it to Google Drive.
- **1.3 Historical Data Management:** Queries Google Sheets to determine if the user is a first-time or returning participant.
- **1.4 AI Analysis (Branching Path):**
    - *Path A (First-time):* Performs a general "Palm Fortune" analysis.
    - *Path B (Returning):* Compares the current palm image with the most recent previous image to detect changes in color, dryness, or lines.
- **1.5 Data Logging & User Response:** Records the AI output in Google Sheets and sends the final analysis back to the user via LINE.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
Processes the incoming LINE webhook and filters for valid image messages.
*   **Nodes Involved:** `LINE_trigger`, `config`, `PIC_check`, `Reply_Text_Instructions`.
*   **Node Details:**
    *   **LINE_trigger (Webhook):** Listens for POST requests from LINE. 
    *   **config (Set):** Extracts the `userId` and holds the `LINE_ACCESS_TOKEN`.
    *   **PIC_check (If):** Checks if the incoming message type is "image". If false, it routes to `Reply_Text_Instructions`.
    *   **Reply_Text_Instructions (HTTP Request):** Sends a text message back to the user via the LINE Reply API asking for a palm photo.

#### 2.2 Media Handling & Storage
Retrieves the binary image and stores it for future reference.
*   **Nodes Involved:** `LINE_input`, `PIC_upload`.
*   **Node Details:**
    *   **LINE_input (HTTP Request):** Downloads the binary content of the image from the LINE data API using the message ID and Bearer token.
    *   **PIC_upload (Google Drive):** Uploads the binary file to a specific Google Drive folder (`LINE_PIC`) naming it `{messageId}.jpg`.

#### 2.3 Historical Data Management
Determines user history to decide which AI analysis path to take.
*   **Nodes Involved:** `DATA_download`, `DATA_counting`, `Check Previous Record Exists`, `Get latest DATA`, `PIC_Search`.
*   **Node Details:**
    *   **DATA_download (Google Sheets):** Reads all existing records from the tracking spreadsheet.
    *   **DATA_counting (Code):** Counts the number of valid rows to identify if records exist.
    *   **Check Previous Record Exists (If):** Routes the workflow: True (Previous records exist) or False (First-time user).
    *   **Get latest DATA (Code):** If records exist, identifies the last entry's file ID.
    *   **PIC_Search (Google Drive):** Downloads the *previous* palm image binary from Google Drive for comparison.

#### 2.4 AI Analysis (Branching Path)
The core logic where Gemini analyzes the images.
*   **Nodes Involved:** `Validate Image Pair`, `image_counting`, `Analyze`, `First_Analyze`, `Google Gemini Chat Model`.
*   **Node Details:**
    *   **First_Analyze (AI Chain):** Used for new users. Analyzes one image and outputs a "Fortune" style report (Life Line, Heart Line, etc.).
    *   **Validate Image Pair (Merge):** Combines the current image and the retrieved historical image.
    *   **Analyze (AI Chain):** Used for returning users. Takes two images as input. It is prompted to compare dryness, color tone, and lines within 200 characters.
    *   **Edge Case:** If the AI cannot identify a palm in the image, it is instructed to output "Unable to detect a palm."

#### 2.5 Data Logging & User Response
Finalizes the cycle by saving results and notifying the user.
*   **Nodes Involved:** `Massege_check`, `DATA_upload`, `LINE_output`.
*   **Node Details:**
    *   **Massege_check (Code):** Cleans the AI text output (escaping newlines and quotes) for safe JSON transmission.
    *   **DATA_upload (Google Sheets):** Appends the date, image ID, Drive URL, and AI message to the sheet.
    *   **LINE_output (HTTP Request):** Uses the Push Message API to send the final analysis back to the user's LINE account.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| LINE_trigger | Webhook | Entry point | None | config | LINE Webhook Trigger |
| config | Set | Configuration | LINE_trigger | PIC_check | LINE Webhook Trigger |
| PIC_check | If | Input Filter | config | LINE_input, Reply_Text... | LINE Webhook Trigger |
| LINE_input | HTTP Request | Download Media | PIC_check | PIC_upload, Validate... | Image Storage Layer |
| PIC_upload | Google Drive | Save Image | LINE_input | DATA_download | Image Storage Layer |
| DATA_download | Google Sheets | Fetch History | PIC_upload | DATA_counting | Image Storage Layer |
| DATA_counting | Code | Logic Router | DATA_download | Check Prev Record... | Historical Record Retrieval Logic |
| Check Prev Record... | If | Branching | DATA_counting | Get latest DATA, First... | Historical Record Retrieval Logic |
| Get latest DATA | Code | Select Image | Check Prev Record... | PIC_Search | Historical Record Retrieval Logic |
| PIC_Search | Google Drive | Download History | Get latest DATA | Validate Image Pair | Historical Record Retrieval Logic |
| Validate Image Pair | Merge | Data Grouping | LINE_input, PIC_Search | image_counting | AI Comparison Analysis |
| image_counting | Code | Validation | Validate Image Pair | Analyze | AI Comparison Analysis |
| Analyze | Chain LLM | Comparison | image_counting | Massege_check | AI Comparison Analysis |
| First_Analyze | Chain LLM | Initial Analysis | Check Prev Record... | Massege_check | First-Time Palm Fortune Analysis |
| Massege_check | Code | Data Cleanup | Analyze, First_Analyze | DATA_upload | Health Log Recording |
| DATA_upload | Google Sheets | Archiving | Massege_check | LINE_output | Health Log Recording |
| LINE_output | HTTP Request | Notify User | DATA_upload | None | Health Log Recording |

---

### 4. Reproducing the Workflow from Scratch

1.  **Environment Setup:**
    *   Create a **LINE Messaging API** channel. Save the "Channel Access Token" and "User ID".
    *   Create a **Google Drive folder** and a **Google Sheet** with columns: `days`, `pic`, `url`, `message`.
2.  **Trigger & Config:**
    *   Add a **Webhook** node (POST). Set the URL in the LINE Developer Console.
    *   Add a **Set** node (`config`) to store your `LINE_ACCESS_TOKEN`.
3.  **Validation Logic:**
    *   Add an **If** node (`PIC_check`) to check if `body.events[0].message.type` is "image".
    *   Connect an **HTTP Request** node to the 'False' branch to reply with instructions.
4.  **Storage Setup:**
    *   Add an **HTTP Request** node (`LINE_input`) to GET the image using the LINE Message ID.
    *   Add a **Google Drive** node (`PIC_upload`) to upload the file.
5.  **History Logic:**
    *   Add a **Google Sheets** node (`DATA_download`) to fetch all rows.
    *   Add a **Code** node to count rows and an **If** node to branch based on count > 0.
6.  **Comparison Setup (Branch A):**
    *   If count > 0, use a **Code** node to get the last row's Drive ID.
    *   Use **Google Drive** (`PIC_Search`) to download that file.
    *   Use a **Merge** node and a **Code** node to ensure you have two binary images.
7.  **AI Integration:**
    *   Add **Chain LLM** nodes for both "First_Analyze" and "Analyze".
    *   Connect **Google Gemini Chat Model** nodes to both.
    *   Paste the prompts (provided in node details) focusing on medical disclaimers and character limits.
8.  **Output Logic:**
    *   Use a **Code** node (`Massege_check`) to sanitize the AI text (replace newlines).
    *   Add a **Google Sheets** node (`DATA_upload`) to append the new result.
    *   Add an **HTTP Request** node (`LINE_output`) to POST the message to the LINE Push API.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Non-Medical Disclaimer** | The AI output is for awareness and entertainment only; it is not a medical diagnosis. |
| **LINE API Rate Limits** | Ensure the usage of Push messages fits within your LINE account plan limits. |
| **Gemini Vision** | This workflow requires the Gemini model version that supports multi-modal (image) inputs. |