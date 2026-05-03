Capture vCard QR code contacts with AllCodeRelay and add them to KlickTipp

https://n8nworkflows.xyz/workflows/capture-vcard-qr-code-contacts-with-allcoderelay-and-add-them-to-klicktipp-13621


# Capture vCard QR code contacts with AllCodeRelay and add them to KlickTipp

# Reference Document: Transfer scanned vCard QR codes data to KlickTipp

### 1. Workflow Overview

This workflow automates the process of capturing contact information from physical business cards via QR codes and synchronizing that data with KlickTipp. It is designed specifically for event lead capture, enabling frictionless digital networking.

The logic is divided into four functional stages:
1.  **Input Reception:** A webhook waits for incoming data from the AllCodeRelay mobile app.
2.  **Validation:** A filter ensures the received payload contains a valid static vCard string.
3.  **Data Extraction:** A custom JavaScript engine parses the raw vCard text into structured JSON fields (names, addresses, phones, etc.).
4.  **Integration:** The structured data is pushed to KlickTipp to create or update a subscriber using the Double Opt-In (DOI) process.

---

### 2. Block-by-Block Analysis

#### 2.1 Block 1: Get data
-   **Overview:** Acts as the entry point for the workflow, receiving HTTP POST requests from the AllCodeRelay application.
-   **Nodes Involved:** `AllCodeRelay: Scan Webhook`
-   **Node Details:**
    -   **Node Name:** AllCodeRelay: Scan Webhook
    -   **Type:** Webhook
    -   **Configuration:** Set to `POST` method with the path `qr-code-scanner`.
    -   **Input:** External HTTP request from the mobile app.
    -   **Output:** JSON object containing the scan payload (typically in `$json.body.code`).
    -   **Potential Failures:** Incorrect Webhook URL in the app; networking/firewall issues preventing the POST request.

#### 2.2 Block 2: Filter data
-   **Overview:** Ensures the workflow only processes valid vCard data, ignoring non-vCard QR scans (like simple URLs).
-   **Nodes Involved:** `Filter: vCard Only`
-   **Node Details:**
    -   **Node Name:** Filter: vCard Only
    -   **Type:** Filter
    -   **Configuration:** Logical `AND` condition.
    -   **Key Expressions:** 
        -   `{{ $json.body.code }}` starts with `BEGIN:VCARD`
        -   `{{ $json.body.code }}` ends with `END:VCARD`
    -   **Output:** Passes the data only if both conditions are met.
    -   **Edge Cases:** Dynamic QR codes that only contain a URL will be filtered out.

#### 2.3 Block 3: Parse data
-   **Overview:** Transforms the raw, multi-line vCard string into a clean, flat JSON object compatible with CRM systems.
-   **Nodes Involved:** `Parse: vCard → Contact Fields`
-   **Node Details:**
    -   **Node Name:** Parse: vCard → Contact Fields
    -   **Type:** Code (JavaScript)
    -   **Technical Role:** Normalizes line breaks, "unfolds" wrapped lines, and splits vCard attributes (N, ADR, TEL, EMAIL, ORG, TITLE, URL).
    -   **Key Logic:** 
        -   Extracts First and Last name from the `N` property (fallback to `FN`).
        -   Parses the semi-colon delimited `ADR` property into Street, City, State, Zip, and Country.
        -   Prioritizes `CELL`/`MOBILE` for mobile numbers and `WORK` for emails.
    -   **Output:** A JSON object containing fields like `firstName`, `lastName`, `email`, `company`, etc.

#### 2.4 Block 4: Save data
-   **Overview:** Connects to KlickTipp to register the contact and trigger the automated marketing sequence.
-   **Nodes Involved:** `KlickTipp: Add Contact with DOI`
-   **Node Details:**
    -   **Node Name:** KlickTipp: Add Contact with DOI
    -   **Type:** KlickTipp (Community Node)
    -   **Resource/Operation:** `subscriber` / `subscribe`
    -   **Configuration:** 
        -   Uses `listId: 358895` and `tagId: 14145915`.
        -   Maps 12 fields including Email, Names, Company, Phone, and Full Address.
    -   **Potential Failures:** Invalid email address format; KlickTipp API credential expiration; duplicate subscriber conflicts (depending on KlickTipp settings).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| AllCodeRelay: Scan Webhook | Webhook | Entry Point | (None) | Filter: vCard Only | ## 1. Get data |
| Filter: vCard Only | Filter | Validation | AllCodeRelay: Scan Webhook | Parse: vCard → Contact Fields | ## 2. Filter data |
| Parse: vCard → Contact Fields | Code | Data Transformation | Filter: vCard Only | KlickTipp: Add Contact with DOI | ## 3. Parse data |
| KlickTipp: Add Contact with DOI | KlickTipp | CRM Integration | Parse: vCard → Contact Fields | (None) | ## 4. Save data |

---

### 4. Reproducing the Workflow from Scratch

1.  **Setup the Webhook:**
    -   Create a **Webhook** node named `AllCodeRelay: Scan Webhook`.
    -   Set HTTP Method to `POST` and path to `qr-code-scanner`.
    -   *Credential:* None required for the node itself.

2.  **Setup the Filter:**
    -   Create a **Filter** node.
    -   Add two string conditions:
        -   Value 1: `{{ $json.body.code }}` **Starts With** `BEGIN:VCARD`
        -   Value 2: `{{ $json.body.code }}` **Ends With** `END:VCARD`
    -   Set the combinator to `AND`.

3.  **Setup the Parsing Logic:**
    -   Create a **Code** node (Language: JavaScript).
    -   Implement logic to split the `{{ $json.body.code }}` by newlines.
    -   Map the standard vCard keys (N, FN, ADR, TEL, EMAIL, ORG, TITLE, URL) to specific JSON keys (e.g., `firstName`, `lastName`, `email`).
    -   *Tip:* Use `.split(';')` for the `N` and `ADR` fields to extract specific components like Zip Code or City.

4.  **Setup KlickTipp Integration:**
    -   Install the **KlickTipp community node** if not present.
    -   Create a **KlickTipp** node.
    -   Operation: `Subscribe`.
    -   Configure **Credentials** using your KlickTipp Username and Password.
    -   Map the fields using expressions from the Code node (e.g., Email: `{{ $json.email }}`).
    -   Enter your specific `List ID` and `Tag ID` (ensure these exist in your KlickTipp account).

5.  **Connect the Nodes:**
    -   Webhook -> Filter -> Code -> KlickTipp.

6.  **Mobile App Setup:**
    -   Install **AllCodeRelay** on a smartphone.
    -   Configure the app's target URL to the n8n **Production URL** of the Webhook node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Community Node Disclaimer | This workflow uses KlickTipp community nodes. |
| Supported vCard Types | Supports static/offline vCard QR codes. Dynamic QRs (URLs) are not supported. |
| Setup Instructions & Use Case | Detailed in the largest sticky note within the workflow. |