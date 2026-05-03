Scrape Google Maps business data with BrowserAct and Google Sheets

https://n8nworkflows.xyz/workflows/scrape-google-maps-business-data-with-browseract-and-google-sheets-12937


# Scrape Google Maps business data with BrowserAct and Google Sheets

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Scrape Google Maps business data with BrowserAct and Google Sheets

**Purpose:**  
Collect business listings from Google Maps based on a user-submitted **Location** and **Business Category**, run a BrowserAct scraping workflow, download the produced data file, extract structured records, then **append or update** rows in a Google Sheet.

**Typical use cases:** lead generation, market research, competitor analysis, building business directories.

### 1.1 Input Reception (Form)
Captures user input (location + category) via an n8n Form Trigger and starts the automation.

### 1.2 Remote Scraping Execution (BrowserAct)
Invokes a BrowserAct workflow (hosted in BrowserAct) using the form inputs, waits for completion, and returns an output payload containing a file URL.

### 1.3 File Retrieval & Parsing
Downloads the generated file from BrowserAct, then extracts items/rows into JSON.

### 1.4 Data Persistence (Google Sheets)
Writes each extracted business entry to Google Sheets using an “append or update” operation keyed on the business name.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (Form)

**Overview:**  
Provides a simple UI form so users can enter scraping parameters. The submitted values become the workflow’s initial JSON payload.

**Nodes involved:**  
- **On form submission**

#### Node: On form submission
- **Type / role:** `Form Trigger` — entry point that starts the workflow when a form is submitted.
- **Key configuration:**
  - Form title: **“Scrapping Maps”**
  - Fields:
    - `Location` (label “Lokasi”), required, placeholder “Jakarta”
    - `Business_Category` (label “Kategori Bisnis”), required, placeholder “Restaurant”
- **Key variables produced (output JSON):**
  - `$json.Location`
  - `$json.Business_Category`
- **Connections:**
  - **Output →** Run a workflow and wait for its result
- **Version-specific notes:** Node typeVersion **2.5**; behavior depends on n8n form trigger implementation for your n8n version.
- **Edge cases / failure modes:**
  - Missing required fields will prevent submission.
  - If the form URL is not accessible (network / auth), users can’t start the workflow.

---

### Block 2 — Remote Scraping Execution (BrowserAct)

**Overview:**  
Calls a BrowserAct workflow template that performs the Google Maps scraping. n8n waits up to 900 seconds for BrowserAct to finish and return output, including a downloadable file link.

**Nodes involved:**  
- **Run a workflow and wait for its result**

#### Node: Run a workflow and wait for its result
- **Type / role:** `browserAct` (community node) — executes a BrowserAct workflow and waits for completion.
- **Key configuration choices:**
  - BrowserAct workflow ID: **`75450516286513963`** (must match the workflow you imported in BrowserAct)
  - Max wait time: **900 seconds**
  - Input parameters passed to BrowserAct:
    - `Location` = `{{$json.Location}}`
    - `Business_Category` = `{{$json.Business_Category}}`
- **Credentials:**
  - Uses BrowserAct API credential: **“BrowserAct account 2”**
- **Output expectations:**
  - The next node assumes BrowserAct returns something like:  
    `output.files[0]` = a URL to the exported results file
- **Connections:**
  - **Input ←** On form submission
  - **Output →** Get Data
- **Version-specific requirements:**
  - Node typeVersion **1**
  - Requires installing community package: `n8n-nodes-browseract-workflows`
- **Edge cases / failure modes:**
  - **Auth/permission failure** if BrowserAct API key is invalid or lacks access to the workflow.
  - **Timeout** if BrowserAct run exceeds 900s (large result sets, slow browsing, captchas).
  - **Output shape mismatch** if the BrowserAct template returns a different structure (e.g., `output.file` instead of `output.files[0]`), causing downstream expressions to fail.
  - **Rate limiting / scraping disruptions** (Google Maps blocking, captchas) can cause BrowserAct run failures or empty outputs.
- **Sub-workflow reference:**
  - This node invokes a **BrowserAct-hosted workflow**, not an n8n sub-workflow. You must import the referenced BrowserAct template and update the workflow ID if different.

---

### Block 3 — File Retrieval & Parsing

**Overview:**  
Downloads the file created by BrowserAct and extracts rows/records into individual n8n items for processing.

**Nodes involved:**  
- **Get Data**
- **Extract from File**

#### Node: Get Data
- **Type / role:** `HTTP Request` — downloads the BrowserAct-generated file.
- **Key configuration choices:**
  - URL expression: `{{$json.output.files[0]}}`
- **Connections:**
  - **Input ←** Run a workflow and wait for its result
  - **Output →** Extract from File
- **Version-specific notes:** typeVersion **4.3**
- **Edge cases / failure modes:**
  - **Expression failure** if `output.files[0]` is missing/null.
  - **403/404** if the file URL is expired, private, or requires headers/auth not provided.
  - **Large file** may cause memory/time constraints depending on n8n settings.

#### Node: Extract from File
- **Type / role:** `Extract From File` — parses the downloaded file into structured JSON items.
- **Key configuration choices:**
  - Uses default options (auto-detection depends on file type and node behavior).
- **Connections:**
  - **Input ←** Get Data
  - **Output →** Append or update row in sheet
- **Version-specific notes:** typeVersion **1.1**
- **Edge cases / failure modes:**
  - If the downloaded content is not a supported/expected format (CSV/XLSX/JSON, etc.), extraction can fail.
  - If the file has unusual encoding/BOM headers, extracted field names may include invisible characters (this workflow already hints at this; see `﻿Name` below).

---

### Block 4 — Data Persistence (Google Sheets)

**Overview:**  
Upserts (append or update) each extracted business record into a Google Sheet. Matching is performed on the “Nama” column (Business Name).

**Nodes involved:**  
- **Append or update row in sheet**

#### Node: Append or update row in sheet
- **Type / role:** `Google Sheets` — writes extracted data to a spreadsheet using an upsert strategy.
- **Operation:** `appendOrUpdate`
- **Document / sheet targeting:**
  - Spreadsheet URL points to: `https://docs.google.com/spreadsheets/d/1_B3-9dUA0XNCirVcA9qr-zeSXK_zkZRoW8xXt7vQflA/edit?usp=sharing`
  - Sheet tab: **Task** (`gid=0`)
- **Matching logic (upsert):**
  - `matchingColumns`: **["Nama"]**
  - If a row with the same “Nama” exists → update; otherwise → append.
- **Column mappings (from extracted JSON → sheet columns):**
  - **Nama** = `{{$json['﻿Name']}}`  
    *Note the key contains a leading invisible character (likely BOM).*
  - **Alamat** = `{{$json.Address}}`
  - **Rating** = `{{$json.Rating}}`
  - **Telepon** = `='{{ $json.Phone }}`  
    (Prepends `=` and wraps in quotes so Sheets treats it as text; intended to preserve leading `+` or zeros.)
  - **Website** = `{{$json.Website}}`
  - **Kategori** = `{{$json.Category}}`
  - **Ulasan Terakhir** = `{{$json.LastSummary}}`
- **Credentials:**
  - Google Sheets OAuth2 credential: **“Google Sheets account”**
- **Connections:**
  - **Input ←** Extract from File
  - **Output:** none (end of main flow)
- **Version-specific notes:** typeVersion **4.7**
- **Edge cases / failure modes:**
  - **Auth errors** if OAuth token expired/revoked, or scopes insufficient.
  - **Sheet structure mismatch** if the target sheet doesn’t contain the expected headers/columns.
  - **Upsert collisions**: matching on business name can overwrite distinct businesses with identical names (common for franchises).
  - **Field name issues**: if extracted data uses `Name` (without BOM) but mapping expects `﻿Name`, “Nama” may become blank and upsert matching fails.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On form submission | Form Trigger | Collect Location + Business Category from user and start workflow | — | Run a workflow and wait for its result | ## 🛠️ Quick Setup Guide<br>**Created by:** [Kristian Ekachandra](https://yapp.ink/aichandre)<br><br>### ✅ Step 1: Install BrowserAct Community Node<br>Install [n8n BrowserAct Community Node](https://www.npmjs.com/package/n8n-nodes-browseract-workflows) in your n8n instance.<br><br>### ✅ Step 2: Add BrowserAct Credentials<br>Set up your [BrowserAct](https://browseract.ai/kristian) API credentials in n8n.<br><br>### ✅ Step 3: Import BrowserAct Workflow Template<br>Import the [Google Maps Detail Scraper Template](https://www.browseract.com/template-share/template-of-googlemaps-detail-scraper-2026-01-20/626dfc97051f46dc9ee0d0f33642387a) to your BrowserAct account.<br><br>### ✅ Step 4: Add Google Sheets Credentials<br>Set up OAuth2 credentials from [Google Cloud Console](https://console.cloud.google.com/) in n8n.<br><br>### ✅ Step 5: Duplicate Template & Update Links<br>1. [Duplicate this Google Sheets template](https://docs.google.com/spreadsheets/d/1_B3-9dUA0XNCirVcA9qr-zeSXK_zkZRoW8xXt7vQflA/copy)<br>2. Update the Google Sheets node URL to your cloned spreadsheet<br><br>### ✅ Step 6: Configure BrowserAct Workflow ID<br>Update the workflow ID in the "Run a workflow and wait for its result" node to match your imported BrowserAct workflow.<br><br>### 💡 Step 7: Test It!<br>1. Open the form URL (from "On form submission" node)<br>2. Enter Location (e.g., "Jakarta") and Business Category (e.g., "Restaurant")<br>3. Submit the form<br>4. Check your Google Sheets for scraped results<br><br>***<br>## 🚀 What This Does<br>**Scrapes Google Maps** business data based on location and category, then **automatically saves** to Google Sheets.<br><br>**Perfect for:** Lead generation, market research, competitor analysis, business directories.<br><br>**Extracted Data:**<br>- ✅ Business Name<br>- ✅ Phone Number<br>- ✅ Category<br>- ✅ Rating<br>- ✅ Address<br>- ✅ Website<br>- ✅ Latest Review Summary<br>*** |
| Run a workflow and wait for its result | BrowserAct (community) | Run BrowserAct Google Maps scraper with input params, wait for result | On form submission | Get Data | ## 🛠️ Quick Setup Guide<br>**Created by:** [Kristian Ekachandra](https://yapp.ink/aichandre)<br>… (same note content as above) |
| Get Data | HTTP Request | Download exported result file from BrowserAct output URL | Run a workflow and wait for its result | Extract from File | ## 🛠️ Quick Setup Guide<br>**Created by:** [Kristian Ekachandra](https://yapp.ink/aichandre)<br>… (same note content as above) |
| Extract from File | Extract From File | Parse downloaded file into individual JSON records | Get Data | Append or update row in sheet | ## 🛠️ Quick Setup Guide<br>**Created by:** [Kristian Ekachandra](https://yapp.ink/aichandre)<br>… (same note content as above) |
| Append or update row in sheet | Google Sheets | Upsert parsed business rows into Google Sheet (match on Nama) | Extract from File | — | ## 🛠️ Quick Setup Guide<br>**Created by:** [Kristian Ekachandra](https://yapp.ink/aichandre)<br>… (same note content as above) |
| Sticky Note | Sticky Note | Documentation / links | — | — | ## 🔥 Follow My Channels<br>Don't miss out on **AI and Automation** content! Follow me on:<br>🎥 **YouTube:** [@aichandre](https://www.youtube.com/@aichandre)<br>📸 **Instagram:** [@aichandre](https://www.instagram.com/aichandre)<br>🎵 **TikTok:** [@aichandre](https://www.tiktok.com/@aichandre)<br>***<br>## 📺 Full Tutorial<br>[![Watch the video on YouTube](https://img.youtube.com/vi/6M3HyfOYzVI/0.jpg)](https://youtu.be/6M3HyfOYzVI)<br>*** |
| Sticky Note1 | Sticky Note | Setup guide / prerequisites | — | — | ## 🛠️ Quick Setup Guide<br>**Created by:** [Kristian Ekachandra](https://yapp.ink/aichandre)<br>… (full content as in node) |

> Note: The large “Quick Setup Guide” sticky note is positioned to cover the main working area in many templates; this document applies it to the operational nodes for traceability.

---

## 4. Reproducing the Workflow from Scratch

1. **Install prerequisite community node (BrowserAct)**
   - In your n8n instance, install:  
     `n8n-nodes-browseract-workflows`  
     Link: https://www.npmjs.com/package/n8n-nodes-browseract-workflows

2. **Create a new workflow** in n8n named:  
   **“Scrape Google Maps business data with BrowserAct and Google Sheets”**

3. **Add node: “On form submission”**
   - Node type: **Form Trigger**
   - Form title: **Scrapping Maps**
   - Add fields:
     - Field name: `Location` (Label: “Lokasi”), Required, Placeholder: “Jakarta”
     - Field name: `Business_Category` (Label: “Kategori Bisnis”), Required, Placeholder: “Restaurant”

4. **Set up BrowserAct**
   - Create BrowserAct account / API key (if not done).
   - Import the BrowserAct template:  
     https://www.browseract.com/template-share/template-of-googlemaps-detail-scraper-2026-01-20/626dfc97051f46dc9ee0d0f33642387a
   - Note the imported **BrowserAct workflow ID**.

5. **Add node: “Run a workflow and wait for its result”**
   - Node type: **BrowserAct**
   - Credentials: add **BrowserAct API** credentials in n8n (API key)
   - Configuration:
     - Workflow ID: set to your BrowserAct workflow ID (template uses `75450516286513963`)
     - Max wait time: `900` seconds
     - Input parameters:
       - `Location` = `{{$json.Location}}`
       - `Business_Category` = `{{$json.Business_Category}}`

6. **Add node: “Get Data”**
   - Node type: **HTTP Request**
   - Method: GET (default)
   - URL: `{{$json.output.files[0]}}`
   - (Optional but recommended) enable “Download”/binary response settings if your n8n version requires it for Extract From File compatibility; validate by running once and confirming the next node can parse the content.

7. **Add node: “Extract from File”**
   - Node type: **Extract From File**
   - Keep defaults initially.
   - Run once to confirm it outputs one item per business record (or per row).

8. **Prepare Google Sheets**
   - Duplicate the provided sheet template:  
     https://docs.google.com/spreadsheets/d/1_B3-9dUA0XNCirVcA9qr-zeSXK_zkZRoW8xXt7vQflA/copy
   - Ensure the destination tab exists (template uses sheet name **Task**).

9. **Add Google Sheets credentials**
   - In n8n, create **Google Sheets OAuth2** credentials.
   - Configure in Google Cloud Console (OAuth consent + client ID/secret) as needed:  
     https://console.cloud.google.com/
   - Ensure the credential has access to the duplicated spreadsheet.

10. **Add node: “Append or update row in sheet”**
   - Node type: **Google Sheets**
   - Operation: **Append or update**
   - Document: paste your duplicated sheet URL
   - Sheet: choose the **Task** tab (or your chosen tab)
   - Matching columns: set to **Nama**
   - Map columns:
     - Nama → `{{$json['﻿Name']}}` (adjust to `{{$json.Name}}` if your extracted data does not include the BOM-prefixed key)
     - Alamat → `{{$json.Address}}`
     - Rating → `{{$json.Rating}}`
     - Telepon → `='{{ $json.Phone }}` (or map directly to `{{$json.Phone}}` if you prefer)
     - Website → `{{$json.Website}}`
     - Kategori → `{{$json.Category}}`
     - Ulasan Terakhir → `{{$json.LastSummary}}`

11. **Connect nodes in this order**
   - On form submission → Run a workflow and wait for its result → Get Data → Extract from File → Append or update row in sheet

12. **Test**
   - Open the form URL from the Form Trigger node.
   - Submit example values (e.g., Location: “Jakarta”, Business_Category: “Restaurant”).
   - Verify:
     - BrowserAct run completes within 900s
     - `output.files[0]` exists
     - Extracted items have the expected fields
     - Rows are appended/updated in Google Sheets

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Created by: Kristian Ekachandra | https://yapp.ink/aichandre |
| Install BrowserAct Community Node | https://www.npmjs.com/package/n8n-nodes-browseract-workflows |
| BrowserAct signup / API | https://browseract.ai/kristian |
| BrowserAct template to import (Google Maps Detail Scraper) | https://www.browseract.com/template-share/template-of-googlemaps-detail-scraper-2026-01-20/626dfc97051f46dc9ee0d0f33642387a |
| Google Cloud Console (OAuth2 setup for Sheets) | https://console.cloud.google.com/ |
| Google Sheets template copy link | https://docs.google.com/spreadsheets/d/1_B3-9dUA0XNCirVcA9qr-zeSXK_zkZRoW8xXt7vQflA/copy |
| YouTube channel @aichandre | https://www.youtube.com/@aichandre |
| Instagram @aichandre | https://www.instagram.com/aichandre |
| TikTok @aichandre | https://www.tiktok.com/@aichandre |
| Video link | https://youtu.be/6M3HyfOYzVI |