Generate Google Maps lead lists with SerpApi, Google Gemini and Sheets

https://n8nworkflows.xyz/workflows/generate-google-maps-lead-lists-with-serpapi--google-gemini-and-sheets-13678


# Generate Google Maps lead lists with SerpApi, Google Gemini and Sheets

# Workflow Reference: Google Maps Lead Generation & AI Sales Strategy

This document provides a technical analysis of the n8n workflow designed to automate lead discovery using Google Maps data, filtered by performance metrics, and enriched with personalized sales strategies via Google Gemini AI.

---

### 1. Workflow Overview

The workflow functions as a **Market Intelligence AI Agent**. It identifies local businesses with a weak digital presence (low ratings or missing info) to facilitate targeted B2B sales. The logic is divided into four main functional blocks:

*   **1.1 Data Acquisition:** Captures search criteria through a user form and retrieves real-time local business data from Google Maps via the SerpApi.
*   **1.2 Lead Filtering & Selection:** Geographically filters results to a specific region (Peru), sorts them by rating, and selects the most "critical" cases for intervention.
*   **1.3 AI Intelligence & Strategy:** Batches the selected leads and utilizes a Google Gemini-powered agent to perform a gap analysis, assign priority scores, and draft personalized outreach content.
*   **1.4 Output & Storage:** Standardizes the AI's response, formats the data for CRM-like tracking, and records all information into a Google Spreadsheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Acquisition
**Overview:** This block handles the entry point and external data fetching. It transforms a search query into a raw list of business entities.

*   **Lead Search Trigger (Form Trigger):**
    *   **Role:** Entry point.
    *   **Configuration:** A web form asking for "Type of business and zone" (e.g., "restaurants in Lima").
    *   **Output:** The search string used in subsequent nodes.
*   **Fetch Google Maps Data (SerpApi) (HTTP Request):**
    *   **Role:** Integration with SerpApi.
    *   **Configuration:** Calls `https://serpapi.com/search.json` using the `google_maps` engine. The query `q` is dynamically mapped from the form input.
    *   **Failure Modes:** API limit reached, invalid API key, or timeout.
*   **Extract Business List (Split Out):**
    *   **Role:** Data normalization.
    *   **Configuration:** Splits the `local_results` array from the SerpApi response into individual n8n items.

#### 2.2 Lead Filtering & Selection
**Overview:** This block refines the raw data into a high-intent target list by applying geographic and performance-based logic.

*   **Filter by Region (Peru) (Filter):**
    *   **Role:** Geographic validation.
    *   **Configuration:** Checks if the `address` field contains the string "Peru".
*   **Prioritize Low-Rating Leads (Sort):**
    *   **Role:** Lead prioritization.
    *   **Configuration:** Sorts items based on the `rating` field (ascending) to surface businesses with poor reviews.
*   **Select Top 5 Critical Cases (Limit):**
    *   **Role:** Resource management.
    *   **Configuration:** Passes only the first 5 items to prevent excessive AI costs and focus on the highest-need leads.

#### 2.3 AI Intelligence & Strategy
**Overview:** The "brain" of the workflow. It uses Generative AI to transform raw business data into actionable sales intelligence.

*   **Batch Leads for AI Processing (Aggregate):**
    *   **Role:** Efficiency.
    *   **Configuration:** Combines all 5 lead items into a single array under the field `lista_de_negocios` so the AI can process them in one prompt.
*   **Market Intelligence Strategy Agent (AI Agent):**
    *   **Role:** Analytical Consultant.
    *   **Configuration:** Uses a comprehensive "System" prompt. It is instructed to identify 2 specific weaknesses, a "hook" phrase, and an email template for each business.
    *   **Key Expression:** `JSON.stringify($json.lista_de_negocios)` passes the data to the LLM.
    *   **Constraint:** Forced JSON output format without Markdown decorators.
*   **Google Gemini Chat Model (Google Gemini Model):**
    *   **Role:** LLM Provider.
    *   **Configuration:** Temperature set to 0.2 for high consistency and low "hallucination" in data formatting.

#### 2.4 Output & Storage
**Overview:** Final processing to ensure the data is clean, structured, and saved in a persistent database.

*   **Clean AI Raw Response (Set):**
    *   **Role:** Capture output.
    *   **Configuration:** Stores the AI's string output into a variable `datos_limpios`.
*   **Parse AI Output to JSON (Code):**
    *   **Role:** Type conversion.
    *   **Technical Role:** Executes `JSON.parse()` on the AI string to return a standard n8n item list.
*   **Final Data Formatting (Set):**
    *   **Role:** Enrichment.
    *   **Configuration:** Adds a default status (`Estado: Por contactar`) and a processing timestamp (`Fecha_Procesamiento`).
*   **Save Strategies to Google Sheets (Google Sheets):**
    *   **Role:** Persistence.
    *   **Configuration:** Maps all AI-generated fields and original Google Maps data into specific spreadsheet columns. Includes a Regex cleanup for phone numbers `replace(/[^0-9]/g, '')`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Lead Search Trigger | Form Trigger | User Input | None | Fetch Google Maps Data | Data Acquisition |
| Fetch Google Maps Data | HTTP Request | API Integration | Lead Search Trigger | Extract Business List | Data Acquisition |
| Extract Business List | Split Out | Data Parsing | Fetch Google Maps Data | Filter by Region (Peru) | Data Acquisition |
| Filter by Region (Peru) | Filter | Geo-Validation | Extract Business List | Prioritize Low-Rating Leads | Lead Filtering & Selection |
| Prioritize Low-Rating Leads | Sort | Prioritization | Filter by Region | Select Top 5 Critical Cases | Lead Filtering & Selection |
| Select Top 5 Critical Cases | Limit | Lead Selection | Prioritize Leads | Batch Leads for AI | Lead Filtering & Selection |
| Batch Leads for AI | Aggregate | Data Bundling | Select Top 5 | Strategy Agent | Lead Filtering & Selection |
| Market Intelligence Agent | AI Agent | Analysis | Batch Leads | Clean AI Response | AI Intelligence & Strategy |
| Google Gemini Model | Chat Model | LLM Provider | Market Intel Agent | None | AI Intelligence & Strategy |
| Clean AI Raw Response | Set | Field Prep | Market Intel Agent | Parse AI Output | Output & Storage |
| Parse AI Output to JSON | Code | JSON Parsing | Clean AI Response | Final Formatting | Output & Storage |
| Final Data Formatting | Set | Meta-data | Parse AI Output | Save to Sheets | Output & Storage |
| Save to Sheets | Google Sheets | Data Storage | Final Formatting | None | Output & Storage |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:** Create a **Form Trigger** node. Add one required text field for the search query.
2.  **API Connection:** Add an **HTTP Request** node.
    *   URL: `https://serpapi.com/search.json`
    *   Method: GET
    *   Auth: Query Auth (Header/Param name `api_key`).
    *   Parameters: `engine=google_maps`, `q={{$json["Field_Name"]}}`.
3.  **Data Extraction:** Add a **Split Out** node. Set "Field to Split Out" to `local_results`.
4.  **Cleaning Logic:**
    *   Add a **Filter** node: `address` CONTAINS "Peru".
    *   Add a **Sort** node: Field `rating`, Order `Ascending`.
    *   Add a **Limit** node: Max items `5`.
5.  **AI Aggregation:** Add an **Aggregate** node. Select "Aggregate All Item Data" to create a single JSON list.
6.  **AI Integration:**
    *   Add an **AI Agent** node. Set "Prompt Type" to "Define below".
    *   In the prompt, define the structure (JSON array) and pass the input via `{{ JSON.stringify($json.lista_de_negocios) }}`.
    *   Connect a **Google Gemini Chat Model** node to the AI Agent. Use Temperature `0.2`.
7.  **Data Re-Parsing:**
    *   Add a **Set** node to map `{{ $json.output }}` to a new field.
    *   Add a **Code** node. Use `return JSON.parse($input.item.json.your_field_name);` to turn the string back into individual items.
8.  **Final Formatting:** Add a **Set** node to define columns for the spreadsheet (e.g., Status, Date, cleaned Phone numbers).
9.  **Storage:** Add a **Google Sheets** node. Select "Append" as the operation. Map the JSON fields to your spreadsheet column names.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Developed by JESUS ANTONIO PACAHUALA ARROYO | Project Credit |
| Requires SerpApi Registration | [https://serpapi.com](https://serpapi.com) |
| Requires Google AI Studio API Key | [https://aistudio.google.com/](https://aistudio.google.com/) |
| Recommended Spreadsheet Columns | Company Name, Rating, Address, AI Score, Sales Strategy, Email Template |