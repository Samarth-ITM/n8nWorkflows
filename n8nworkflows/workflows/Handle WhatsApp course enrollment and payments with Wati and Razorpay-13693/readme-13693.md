Handle WhatsApp course enrollment and payments with Wati and Razorpay

https://n8nworkflows.xyz/workflows/handle-whatsapp-course-enrollment-and-payments-with-wati-and-razorpay-13693


# Handle WhatsApp course enrollment and payments with Wati and Razorpay

This document provides a technical breakdown of the n8n workflow designed to automate course discovery, enrollment, and payment processing via WhatsApp using WATI, Google Sheets, and Razorpay.

### 1. Workflow Overview
The workflow acts as a conversational bot on WhatsApp. It allows students to browse a course catalog, view specific course details, generate payment links for enrollment, and check their current enrollment status. All data is managed through Google Sheets, serving as a lightweight CRM and CMS.

**Functional Logic Blocks:**
*   **1.1 Trigger & Route:** Captures incoming WhatsApp messages via WATI and routes them based on specific keywords (`courses`, `enroll`, `pay`, `mystatus`).
*   **1.2 Course Catalogue:** Fetches the full list of available courses from Google Sheets and formats them into a categorized list.
*   **1.3 Enrollment Details:** Parses a specific course code, retrieves full descriptions from the database, and presents a "Detail Card" to the user.
*   **1.4 Payment Link Generation:** Integrates with Razorpay to create a unique payment URL, logs the transaction as "Pending" in the database, and sends the link to the student.
*   **1.5 Status Inquiry:** Filters the enrollment database by the student’s phone number to return a history of active and pending courses.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Route
This block identifies the user's intent from the raw text message.
*   **Nodes:** `Wati Trigger`, `Route Message`.
*   **Node Details:**
    *   **Wati Trigger:** Listens for `messageReceived` events. It captures the sender's phone number (`waId`) and the message text.
    *   **Route Message (Switch):** Uses four string-based rules (ignoring case and whitespace) to branch the workflow.
*   **Edge Cases:** If no keyword is matched, it triggers a fallback output (though not explicitly connected in this JSON, it's configured for "extra").

#### 2.2 Course Catalogue
Retrieves and displays the "storefront."
*   **Nodes:** `Google Sheets – Read Courses`, `Build Course Catalogue`, `Send a text message`.
*   **Node Details:**
    *   **Google Sheets – Read Courses:** Reads the `Courses` sheet.
    *   **Build Course Catalogue (Code):** A JavaScript snippet that groups courses by category and calculates currency formatting (INR).
*   **Variable Use:** Uses `$('Wati Trigger').first().json.waId` to ensure the reply is sent to the correct requester.

#### 2.3 Enrollment Intent
Provides deep-dive information for a single course.
*   **Nodes:** `Parse Enroll Intent`, `Google Sheets – Read Course Detail`, `Build Enroll Detail Card`, `Send a text message1`.
*   **Node Details:**
    *   **Parse Enroll Intent (Code):** Regex-style parsing to extract the code after the word "enroll" (e.g., "enroll PY101" -> "PY101").
    *   **Build Enroll Detail Card (Code):** Searches the Google Sheets data for the specific `courseCode`. If not found, it returns an error message.

#### 2.4 Payment Generation
Handles financial integration and logging.
*   **Nodes:** `Parse Pay Command`, `Google Sheets – Read Course for Payment`, `Prepare Razorpay Payload`, `Razorpay – Create Payment Link`, `Google Sheets – Log Pending Enrollment`, `Send a text message2`.
*   **Node Details:**
    *   **Prepare Razorpay Payload (Code):** Converts the price into **Paise** (required by Razorpay API) and structures the `customer` object.
    *   **Razorpay – Create Payment Link (HTTP Request):** Performs a POST request to `https://api.razorpay.com/v1/payment_links` using Basic Auth.
    *   **Google Sheets – Log Pending Enrollment:** Appends a row to the `Enrollments` sheet with a "Pending" status.
*   **Potential Failures:** Invalid course code or Razorpay API timeout/auth failure.

#### 2.5 Enrollment Status
Provides user-specific account history.
*   **Nodes:** `Google Sheets – Read Enrollments`, `Build Enrollment Status`, `Send a text message3`.
*   **Node Details:**
    *   **Build Enrollment Status (Code):** Filters the entire enrollment list based on the user's phone number and splits the output into "Active" and "Pending" sections.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Wati Trigger | Wati Trigger | Webhook Entry | None | Route Message | 1️⃣ Trigger & Route |
| Route Message | Switch | Intent Router | Wati Trigger | GS - Read Courses, Parse Enroll, Parse Pay, GS - Read Enrollments | 1️⃣ Trigger & Route |
| Google Sheets – Read Courses | Google Sheets | Data Fetching | Route Message | Build Course Catalogue | 2️⃣ Course Catalogue & Enroll Intent |
| Build Course Catalogue | Code | Data Formatting | GS - Read Courses | Send a text message | 2️⃣ Course Catalogue & Enroll Intent |
| Parse Enroll Intent | Code | String Parsing | Route Message | GS - Read Course Detail | 2️⃣ Course Catalogue & Enroll Intent |
| Google Sheets – Read Course Detail | Google Sheets | Data Fetching | Parse Enroll Intent | Build Enroll Detail Card | 2️⃣ Course Catalogue & Enroll Intent |
| Build Enroll Detail Card | Code | UI Formatting | GS - Read Course Detail | Send a text message1 | 2️⃣ Course Catalogue & Enroll Intent |
| Parse Pay Command | Code | String Parsing | Route Message | GS - Read Course for Payment | 3️⃣ Payment Link Generation |
| Google Sheets – Read Course for Payment | Google Sheets | Data Fetching | Parse Pay Command | Prepare Razorpay Payload | 3️⃣ Payment Link Generation |
| Prepare Razorpay Payload | Code | API Preparation | GS - Read Course for Payment | Razorpay – Create Payment Link | 3️⃣ Payment Link Generation |
| Razorpay – Create Payment Link | HTTP Request | Payment Processing | Prepare Razorpay Payload | GS - Log Pending Enrollment | 3️⃣ Payment Link Generation |
| Google Sheets – Log Pending Enrollment | Google Sheets | Transaction Logging | Razorpay – Create Payment Link | Send a text message2 | 3️⃣ Payment Link Generation |
| Google Sheets – Read Enrollments | Google Sheets | Data Fetching | Route Message | Build Enrollment Status | 5️⃣ Enrollment Status |
| Build Enrollment Status | Code | Data Filtering | GS - Read Enrollments | Send a text message3 | 5️⃣ Enrollment Status |
| Send a text message | Wati | WhatsApp Reply | Build Course Catalogue | None | 2️⃣ Course Catalogue & Enroll Intent |
| Send a text message1 | Wati | WhatsApp Reply | Build Enroll Detail Card | None | 2️⃣ Course Catalogue & Enroll Intent |
| Send a text message2 | Wati | WhatsApp Reply | GS - Log Pending Enrollment | None | 3️⃣ Payment Link Generation |
| Send a text message3 | Wati | WhatsApp Reply | Build Enrollment Status | None | 5️⃣ Enrollment Status |

---

### 4. Reproducing the Workflow from Scratch

1.  **Preparation:** Create a Google Sheet with two tabs: `Courses` (Columns: code, name, price, category, description, duration, instructor) and `Enrollments` (Columns: timestamp, phone, courseCode, courseName, amount, status).
2.  **Trigger Setup:** Add a **Wati Trigger** node. Connect your WATI account and set the event to `messageReceived`.
3.  **Routing Logic:** Add a **Switch** node. Create four outputs using "String starts with" or "equals" for keywords: `courses`, `enroll `, `pay `, and `mystatus`.
4.  **Catalogue Branch:** 
    *   Connect a **Google Sheets** node (Download/Read) to the `courses` output.
    *   Add a **Code** node to iterate through the rows and build a single string message.
    *   Add a **Wati (Send Text)** node to push the string back to the user.
5.  **Enrollment Branch:**
    *   Connect a **Code** node to the `enroll` output to extract the course ID using `.replace(/^enroll\s+/i, '')`.
    *   Fetch course details from Google Sheets based on that ID.
    *   Format a message in a **Code** node and send via **WATI**.
6.  **Payment Branch:**
    *   Extract the course ID from the `pay` command.
    *   Fetch the price from Google Sheets.
    *   Use a **Code** node to build the Razorpay JSON (Amount must be `Price * 100`).
    *   Add an **HTTP Request** node (POST) to `https://api.razorpay.com/v1/payment_links`. Use Header/Basic Auth with your Razorpay Key and Secret.
    *   Append a row to the Google Sheets `Enrollments` tab.
    *   Send the `short_url` from the Razorpay response via **WATI**.
7.  **Status Branch:**
    *   Connect a **Google Sheets** node to the `mystatus` output to read all enrollments.
    *   Add a **Code** node to filter `row.phone === senderPhone`. 
    *   Format and send via **WATI**.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **WATI Setup** | Ensure the Webhook URL from n8n is pasted into the WATI Dashboard under Webhooks. |
| **Razorpay API** | Requires the "Payment Links" feature enabled in the dashboard. |
| **Callback URL** | In the "Prepare Razorpay Payload" node, update the `callback_url` to your own n8n webhook to handle payment confirmations. |
| **Currency Handling** | This workflow is currently hardcoded for `INR` and uses Indian numbering formats (`en-IN`). |