Generate ASO Reports from Google Play Apps with Gemini AI & Google Docs

https://n8nworkflows.xyz/workflows/generate-aso-reports-from-google-play-apps-with-gemini-ai---google-docs-7460


# Generate ASO Reports from Google Play Apps with Gemini AI & Google Docs

### 1. Workflow Overview

This workflow automates the generation of a professional App Store Optimization (ASO) report for Android apps listed on the Google Play Store. It targets marketing analysts, ASO specialists, and product managers seeking automated, AI-enhanced insights about an app’s market performance, user feedback, and competitive positioning.

**Logical blocks:**

- **1.1 Input Reception:** Triggered by user submission of a Google Play Store URL via a form.
- **1.2 Package Name Extraction:** Parses the submitted URL to extract the app’s package name.
- **1.3 App Data Retrieval:** Uses the package name to fetch detailed app intelligence from an external API (SensorTower or similar).
- **1.4 Data Parsing & Formatting:** Processes the raw JSON data to structure key app information, reviews, competitors, and market metrics into formatted text.
- **1.5 AI-Powered ASO Report Generation:** Sends the formatted data to a Large Language Model (Gemini via OpenRouter) to generate a structured, plain-text ASO report.
- **1.6 Google Docs Integration:** Creates a new Google Doc named after the app and inserts the generated report.
- **1.7 Notification:** Sends a Telegram message with a direct link to the Google Doc for immediate access.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Starts the workflow when a user submits a form containing a Google Play Store URL for app analysis.
- **Nodes Involved:** `On form submission`
- **Node Details:**
  - **Type:** Form Trigger
  - **Configuration:** 
    - Form titled "ASO Report"
    - Single required input field labeled "Play Store URL"
  - **Input/Output:** 
    - Input: User submits form
    - Output: JSON containing the submitted URL
  - **Potential Failures:** User submits invalid or empty URL; no URL submitted.
  - **Notes:** This node is the exclusive entry point.

#### 1.2 Package Name Extraction

- **Overview:** Extracts the app’s package name (the `id` parameter) from the submitted Play Store URL.
- **Nodes Involved:** `Code1`
- **Node Details:**
  - **Type:** Code node (JavaScript)
  - **Configuration:**
    - Parses URL safely using `URL` class
    - Fallback regex extraction if URL parsing fails
    - Returns extracted `packageName` or null if not found
  - **Inputs:** JSON with `Play Store URL`
  - **Outputs:** JSON with `packageName`
  - **Edge Cases:**
    - Malformed URLs
    - Missing or malformed `id` parameter
  - **Failure Modes:** If package name extraction fails, subsequent HTTP request will fail or return invalid data.

#### 1.3 App Data Retrieval

- **Overview:** Calls an external app intelligence API (e.g., SensorTower) using the package name to fetch detailed app data.
- **Nodes Involved:** `HTTP Request`
- **Node Details:**
  - **Type:** HTTP Request
  - **Configuration:**
    - GET request to `https://app.sensortower.com/api/android/apps/{{ $json.packageName }}?country=US`
    - No additional options specified (e.g., headers or authentication details presumably set via credentials)
  - **Inputs:** JSON with `packageName`
  - **Outputs:** JSON raw app data including app info, reviews, competitors, and market data
  - **Potential Failures:**
    - Network errors or timeouts
    - API authentication errors
    - Invalid or missing package name causing 404 or empty response
    - Rate limiting from the API provider

#### 1.4 Data Parsing & Formatting

- **Overview:** Processes raw app data JSON into structured, human-readable text blocks for input to the AI model.
- **Nodes Involved:** `Code`
- **Node Details:**
  - **Type:** Code node (JavaScript)
  - **Configuration:**
    - Extracts and formats:
      - Basic app info: name, app ID, publisher, category, installs, rating, rating count, in-app purchases, last update version
      - Top 3 featured reviews with user, date, rating, content, and tags
      - Top 3 competitors with name, publisher, rating, installs
      - Market insights: last month downloads, revenue, currency
    - Outputs a JSON object with fields: `appInfo`, `reviewsText`, `competitorsText`, `market`
  - **Inputs:** Raw JSON app data
  - **Outputs:** Formatted JSON data for LLM consumption
  - **Edge Cases:**
    - Missing or incomplete fields in API response
    - Empty reviews or competitors arrays
    - Null or zero values for market data

#### 1.5 AI-Powered ASO Report Generation

- **Overview:** Uses a Large Language Model (Gemini 2.0 via OpenRouter) to generate a polished ASO report based on structured app data.
- **Nodes Involved:** `Basic LLM Chain`, `OpenRouter Chat Model`
- **Node Details:**
  - **Basic LLM Chain:**
    - **Type:** LangChain LLM Chain node
    - **Configuration:** Defines prompt with instructions for generating a plain-text ASO report with emojis as section headers.
    - Uses input expressions to inject app info, reviews, competitors, market insights into prompt.
    - Outputs the generated report text.
    - **Potential Failures:** 
      - Expression evaluation errors if input data missing or malformed
      - API rate limits or auth errors with OpenRouter
  - **OpenRouter Chat Model:**
    - **Type:** LangChain LLM Chat Model node
    - **Configuration:** Uses Gemini 2.0 Flash experimental free model
    - Connected as the AI language model for the `Basic LLM Chain`
    - **Potential Failures:** Model unavailability or network errors

#### 1.6 Google Docs Integration

- **Overview:** Creates a Google Document named after the app and inserts the AI-generated ASO report text.
- **Nodes Involved:** `Create a document`, `Update a document`
- **Node Details:**
  - **Create a document:**
    - **Type:** Google Docs node
    - **Configuration:** Creates a new doc titled with the app name (`{{ $('Code').item.json.appInfo.name }}`) inside a specified Google Drive folder.
  - **Update a document:**
    - **Type:** Google Docs node
    - **Configuration:** Updates the newly created document by inserting the ASO report text from the LLM node.
    - Uses dynamic document URL from the created doc's ID.
  - **Inputs:** App name and report text
  - **Outputs:** Document metadata including document ID and URL
  - **Potential Failures:**
    - OAuth2 credential issues
    - Rate limits or quota exceeded
    - Invalid folder ID or missing permissions

#### 1.7 Notification

- **Overview:** Sends a Telegram message notifying about the new ASO report with a clickable link to the Google Doc.
- **Nodes Involved:** `Send a text message`
- **Node Details:**
  - **Type:** Telegram node
  - **Configuration:** 
    - Sends formatted text including app name and direct link to the Google Doc
    - Uses specified Telegram chat ID
  - **Inputs:** Document ID and app name
  - **Potential Failures:**
    - Telegram API errors or invalid chat ID
    - Message formatting errors

---

### 3. Summary Table

| Node Name          | Node Type                     | Functional Role                               | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                                               |
|--------------------|-------------------------------|-----------------------------------------------|-----------------------|-------------------------|---------------------------------------------------------------------------------------------------------------------------|
| On form submission  | Form Trigger                  | Entry point; receives Play Store URL input    | -                     | Code1                   | - The workflow starts when a user submits a form with the app's Play Store URL<br>- Extracts the package name (the id= parameter) from the submitted URL.<br>- Uses the package name to call the SensorTower API (or a similar app intelligence API) and fetch details such as the app's name, publisher, category, rating, and more. |
| Code1              | Code                          | Extracts package name from URL                 | On form submission    | HTTP Request            | - The workflow starts when a user submits a form with the app's Play Store URL<br>- Extracts the package name (the id= parameter) from the submitted URL.<br>- Uses the package name to call the SensorTower API (or a similar app intelligence API) and fetch details such as the app's name, publisher, category, rating, and more. |
| HTTP Request       | HTTP Request                  | Fetches app data from SensorTower API         | Code1                 | Code                    | - Parses the raw JSON and formats it into structured text, including app details, featured reviews, competitor data, and market insights.<br>- Sends the structured data to an AI model with a prompt that instructs it to create a neatly formatted ASO report.<br>- Automatically creates a new Google Doc titled with the app's name and stores the generated report inside the specified folder. |
| Code               | Code                          | Parses and formats app data for AI input       | HTTP Request          | Basic LLM Chain         | - Parses the raw JSON and formats it into structured text, including app details, featured reviews, competitor data, and market insights.<br>- Sends the structured data to an AI model with a prompt that instructs it to create a neatly formatted ASO report.<br>- Automatically creates a new Google Doc titled with the app's name and stores the generated report inside the specified folder. |
| OpenRouter Chat Model | LangChain LLM Chat Model     | Provides AI model (Gemini) used by LLM Chain  | -                     | Basic LLM Chain (AI)    |                                                                                                                           |
| Basic LLM Chain    | LangChain Chain LLM            | Generates the ASO report text from formatted data | Code, OpenRouter Chat Model | Create a document       | - Parses the raw JSON and formats it into structured text, including app details, featured reviews, competitor data, and market insights.<br>- Sends the structured data to an AI model with a prompt that instructs it to create a neatly formatted ASO report.<br>- Automatically creates a new Google Doc titled with the app's name and stores the generated report inside the specified folder. |
| Create a document  | Google Docs                   | Creates new Google Doc named after the app    | Basic LLM Chain        | Update a document       | - Inserts the ASO report text into the previously created Google Doc.<br>- Once the document is updated, a Telegram notification is sent to the specified user. |
| Update a document  | Google Docs                   | Inserts ASO report text into the created Doc  | Create a document      | Send a text message     | - Inserts the ASO report text into the previously created Google Doc.<br>- Once the document is updated, a Telegram notification is sent to the specified user. |
| Send a text message | Telegram                     | Sends Telegram notification with report link  | Update a document      | -                       | - Inserts the ASO report text into the previously created Google Doc.<br>- Once the document is updated, a Telegram notification is sent to the specified user. |
| Sticky Note1       | Sticky Note                   | Project overview and credentials summary      | -                     | -                       | # Automated App Analysis & ASO Report Generator ... (full detailed note as above)                                         |
| Sticky Note        | Sticky Note                   | Explains initial workflow block (form to API) | -                     | -                       | - The workflow starts when a user submits a form with the app's Play Store URL<br>- Extracts the package name (the id= parameter) from the submitted URL.<br>- Uses the package name to call the SensorTower API (or a similar app intelligence API) and fetch details such as the app's name, publisher, category, rating, and more. |
| Sticky Note2       | Sticky Note                   | Explains data parsing and AI report generation | -                     | -                       | - Parses the raw JSON and formats it into structured text, including app details, featured reviews, competitor data, and market insights.<br>- Sends the structured data to an AI model with a prompt that instructs it to create a neatly formatted ASO report.<br>- Automatically creates a new Google Doc titled with the app's name and stores the generated report inside the specified folder. |
| Sticky Note3       | Sticky Note                   | Explains Google Docs update and Telegram notification | -                 | -                       | - Inserts the ASO report text into the previously created Google Doc.<br>- Once the document is updated, a Telegram notification is sent to the specified user. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node:**
   - Type: `Form Trigger`
   - Title: "ASO Report"
   - Add one required field: "Play Store URL"
   - This node is the workflow start.

2. **Add a Code Node (`Code1`) to Extract Package Name:**
   - Input: Output from Form Trigger
   - Use JavaScript to parse the submitted URL:
     - Try URL parsing to extract `id` parameter
     - If fails, fallback to regex extraction
   - Output JSON should contain `packageName`

3. **Add an HTTP Request Node:**
   - Input: Output from `Code1`
   - Method: GET
   - URL: `https://app.sensortower.com/api/android/apps/{{ $json.packageName }}?country=US`
   - Credentials: Configure with SensorTower or equivalent API credentials
   - Expect JSON response with app data

4. **Add a Code Node (`Code`) to Parse and Format API Data:**
   - Input: Output from HTTP Request
   - JavaScript logic to extract:
     - App info: name, app_id, publisher, category, installs, rating, rating_count, in-app purchases, last update version
     - Top 3 featured reviews (user, date, rating, content, tags)
     - Top 3 competitors (name, publisher, rating, installs)
     - Market data (downloads and revenue last month)
   - Format reviews and competitors as clean multiline text
   - Output JSON with keys: `appInfo`, `reviewsText`, `competitorsText`, `market`

5. **Add OpenRouter Chat Model Node:**
   - Model: `google/gemini-2.0-flash-exp:free`
   - Connect as AI language model to the next node

6. **Add Basic LLM Chain Node:**
   - Connect input from `Code` node and AI model node
   - Configure prompt to instruct generation of ASO report in plain text with emojis as headers:
     - Sections: App Overview, User Ratings & Reviews, Competitor Analysis, Market Insights, Recommendations
   - Use expressions to inject parsed data (`appInfo`, `reviewsText`, `competitorsText`, `market`)
   - Output: Generated ASO report text

7. **Add Google Docs Node to Create a Document:**
   - Operation: Create
   - Title: Use expression to set title from app name (`{{ $('Code').item.json.appInfo.name }}`)
   - Folder ID: Set to your Google Drive folder ID
   - Credentials: Configure Google Docs OAuth2

8. **Add Google Docs Node to Update the Document:**
   - Operation: Update
   - Document URL: Use document ID from Create node (`={{ $json.id }}`)
   - Insert text action: Insert the ASO report text from Basic LLM Chain node
   - Credentials: Use same Google Docs OAuth2

9. **Add Telegram Node to Send Notification:**
   - Text: Compose message with app name and Google Docs link:
     ```
     📄 New document for app analysis: {{ $('Code').item.json.appInfo.name }}
     🔗 Document link: https://docs.google.com/document/d/{{ $json.documentId }}/edit?tab=t.0
     ```
   - Chat ID: Your Telegram chat ID
   - Credentials: Configure Telegram API credentials

10. **Connect Nodes as Follows:**
    - Form Trigger → Code1 → HTTP Request → Code → Basic LLM Chain → Create a document → Update a document → Send a text message
    - OpenRouter Chat Model → Basic LLM Chain (AI language model input)

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow automates app research and reporting by combining app intelligence APIs, AI text generation, and cloud document storage with instant messaging notifications.                                                                                                                                                                                                                              | General workflow purpose                                                                        |
| Required credentials: SensorTower API (or equivalent), OpenRouter API for Gemini model, Google Docs OAuth2, Telegram API.                                                                                                                                                                                                                                                                            | Credential setup                                                                                |
| The AI prompt instructs the model to generate a business English ASO report with emojis as section headers and bullet points, ensuring consistency and professionalism.                                                                                                                                                                                                                               | AI prompt design                                                                               |
| Google Docs folder ID and Telegram chat ID must be replaced with your actual IDs for the workflow to function correctly.                                                                                                                                                                                                                                                                              | Configuration requirement                                                                      |
| The workflow includes detailed sticky notes inside n8n explaining the purpose of each block and summarizing the workflow benefits and setup.                                                                                                                                                                                                                                                        | Sticky Notes inside workflow                                                                   |
| For best results, ensure the SensorTower API provides consistent and complete data fields as expected by the parsing code, or modify the code accordingly.                                                                                                                                                                                                                                           | Data consistency and customization note                                                       |
| The workflow uses the `Basic LLM Chain` with an OpenRouter Chat Model node for flexible AI prompt management and model selection.                                                                                                                                                                                                                                                                     | AI integration best practice                                                                   |
| Telegram notifications provide real-time alerts for new reports, improving team collaboration and response times.                                                                                                                                                                                                                                                                                    | Notification benefit                                                                           |

---

**Disclaimer:** The provided text is generated exclusively from an automated workflow constructed with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.