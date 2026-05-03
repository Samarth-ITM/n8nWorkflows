Generate e-commerce product descriptions from a form with Google Gemini

https://n8nworkflows.xyz/workflows/generate-e-commerce-product-descriptions-from-a-form-with-google-gemini-13673


# Generate e-commerce product descriptions from a form with Google Gemini

This document provides a comprehensive technical analysis of the **Product Description Generator** workflow, designed to automate the creation of SEO-optimized e-commerce content using Google Gemini.

---

### 1. Workflow Overview

The workflow automates the pipeline from product data collection to multi-language copy generation and final distribution. It allows users to submit product specifications via a web form and receive a formatted e-commerce package (short/long descriptions, SEO meta-data, and translations) via email and Google Sheets.

**The logic is organized into four functional blocks:**
*   **1.1 Input Reception:** Captures product specs and user preferences via an n8n Form.
*   **1.2 AI Content Generation:** Uses Google Gemini to generate structured marketing copy in English followed by a secondary translation step.
*   **1.3 Data Formatting:** A Code node parses the AI-generated JSON strings into a clean object structure.
*   **1.4 Storage & Delivery:** Saves the results to a Google Sheets database and dispatches an email to the requester.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception
*   **Overview:** Collects necessary product details from the user to provide context for the AI.
*   **Nodes Involved:** `Product Details Form` (Trigger).
*   **Node Details:**
    *   **Type:** Form Trigger.
    *   **Configuration:** Collects `Product Name`, `Key Features` (Textarea), `Target Audience`, `Price Range`, `Tone` (Dropdown), `Languages` (Dropdown), and `Delivery Email`.
    *   **Output:** Sends a JSON object containing these fields to the AI generation block.
    *   **Edge Cases:** Missing "Required" fields will prevent submission; invalid email formats are caught by the browser-side validation.

#### 2.2 AI Content Generation
*   **Overview:** This block uses a two-step LLM chain to first create the primary content and then translate it into multiple languages.
*   **Nodes Involved:** `Generate Description`, `Gemini Chat Model`, `Translate Descriptions`, `Gemini Translate Model`.
*   **Node Details:**
    *   **Generate Description (Chain LLM):** Uses a system prompt to act as an e-commerce specialist. It requests a specific JSON schema (short_description, long_description, seo_title, etc.).
    *   **Translate Descriptions (Chain LLM):** Takes the output of the first node and translates the specific fields into Japanese, Spanish, and French.
    *   **Gemini Models:** Both nodes use `models/gemini-1.5-flash`. The generation node is set to `Temperature 0.7` (creativity), while the translation node is set to `0.5` (accuracy).
    *   **Failure Modes:** API rate limits (Google Gemini free tier) or the AI failing to return valid JSON.

#### 2.3 Data Formatting
*   **Overview:** Ensures the data is properly structured for downstream services.
*   **Nodes Involved:** `Format Output`.
*   **Node Details:**
    *   **Type:** Code Node (JavaScript).
    *   **Role:** Extracts JSON data from the LLM's text response using Regex to handle instances where the AI might include conversational text or markdown code blocks.
    *   **Expressions:** Accesses `$input.all()`, `Product Details Form`, and `Generate Description` nodes.
    *   **Output:** A unified object containing `productName`, `tone`, `deliveryEmail`, `english` (structured), `translations` (structured), and a `generatedAt` timestamp.

#### 2.4 Storage & Delivery
*   **Overview:** Persists the generated content and notifies the stakeholder.
*   **Nodes Involved:** `Save to Description DB`, `Email Description`.
*   **Node Details:**
    *   **Save to Description DB:** Connects to Google Sheets using the `Append or Update` operation. Requires a spreadsheet with headers matching the output object.
    *   **Email Description:** Uses Gmail to send a formatted message to the email address provided in the form. It uses expressions to list bullet points with a "•" prefix and provides a summary of the SEO metadata.
    *   **Failure Modes:** Google Sheets permission issues; Gmail OAuth token expiration.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Product Details Form** | Form Trigger | User input capture | None | Generate Description | Product input: Collects product details through a simple web form. |
| **Generate Description** | Chain LLM | English copy generation | Product Details Form | Translate Descriptions | AI description generation: Gemini creates SEO-optimized copy and optional translations. |
| **Gemini Chat Model** | Google Gemini | AI engine provider | None | Generate Description | AI description generation: Gemini creates SEO-optimized copy and optional translations. |
| **Translate Descriptions** | Chain LLM | Multi-language translation | Generate Description | Format Output | AI description generation: Gemini creates SEO-optimized copy and optional translations. |
| **Gemini Translate Model** | Google Gemini | AI engine provider | None | Translate Descriptions | AI description generation: Gemini creates SEO-optimized copy and optional translations. |
| **Format Output** | Code | JSON parsing & cleanup | Translate Descriptions | Save to Description DB | Format and deliver: Structures the output, saves to Sheets, and emails the final copy. |
| **Save to Description DB** | Google Sheets | Data persistence | Format Output | Email Description | Format and deliver: Structures the output, saves to Sheets, and emails the final copy. |
| **Email Description** | Gmail | Result notification | Save to Description DB | None | Format and deliver: Structures the output, saves to Sheets, and emails the final copy. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:**
    *   Create a **Form Trigger** node. Name it `Product Details Form`.
    *   Define fields: `Product Name` (Text), `Key Features` (Textarea), `Target Audience` (Text), `Price Range` (Text), `Tone` (Dropdown: Professional, Casual, Luxury, etc.), `Languages` (Dropdown), and `Delivery Email` (Email).

2.  **AI Logic Setup:**
    *   Add a **Basic LLM Chain** node (`Generate Description`). Use an expression to pass form data into the prompt. Instruct the AI to return *only* JSON.
    *   Attach a **Google Gemini Chat Model** node to the chain. Set the model to `gemini-1.5-flash`.
    *   Add a second **Basic LLM Chain** node (`Translate Descriptions`) connected to the first. Provide a prompt for translating the specific fields into JA, ES, and FR. Attach another Gemini Chat Model node.

3.  **Data Processing:**
    *   Add a **Code Node** (`Format Output`). Use JavaScript to parse the LLM strings. Include a `try/catch` block and a `.match(/\{[\s\S]*\}/)` regex to ensure only the JSON object is extracted from the AI's response.

4.  **Integration & Delivery:**
    *   Create a **Google Sheets** node. Configure it to `Append` or `Update` rows in a sheet named "Product Descriptions". Map the JSON keys to your sheet columns.
    *   Create a **Gmail** node. Set the "To" field to `{{ $json.deliveryEmail }}`. Use a formatted template for the "Body" that includes the Title, Full Description, and Bullet Points.

5.  **Credential Configuration:**
    *   Set up **Google Gemini API** (via Google AI Studio).
    *   Authenticate **Google Sheets** and **Gmail** using OAuth2.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **How it works** | 1. Fill form -> 2. AI generates SEO copy -> 3. AI translates -> 4. Code formats -> 5. Save to Sheets -> 6. Email delivery. |
| **Setup steps** | Connect Gemini (free tier), create "Product Descriptions" Sheet, connect Gmail, and activate form URL. |
| **Customization** | Edit prompts for brand voice (Shopify/Amazon), adjust language list, or change SEO keyword density. |