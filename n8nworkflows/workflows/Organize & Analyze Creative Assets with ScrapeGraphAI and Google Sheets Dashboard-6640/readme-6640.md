Organize & Analyze Creative Assets with ScrapeGraphAI and Google Sheets Dashboard

https://n8nworkflows.xyz/workflows/organize---analyze-creative-assets-with-scrapegraphai-and-google-sheets-dashboard-6640


# Organize & Analyze Creative Assets with ScrapeGraphAI and Google Sheets Dashboard

---

### 1. Workflow Overview

This workflow automates the organization and analysis of creative digital assets (images, videos, documents, audio) uploaded to a system. It integrates AI-powered asset analysis with structured tagging, brand compliance checks, and systematic organization before updating a collaborative Google Sheets dashboard.

**Target Use Cases:**  
- Marketing teams managing visual content  
- Creative agencies organizing multimedia assets  
- Brand teams enforcing compliance on assets  
- Content strategists tracking asset metadata and usage  

**Logical Blocks:**

- **1.1 Input Reception:** Captures asset upload events via webhook.  
- **1.2 AI Asset Analysis:** Uses ScrapeGraphAI to extract structured data from asset URLs.  
- **1.3 Tag Generation:** Generates comprehensive tags based on AI analysis to facilitate search and classification.  
- **1.4 Brand Compliance Evaluation:** Checks asset adherence to brand guidelines for colors, formats, and elements.  
- **1.5 Asset Organization:** Creates folder paths, standardized filenames, metadata, and asset records for storage and tracking.  
- **1.6 Dashboard Update:** Appends or updates the Google Sheets dashboard with the organized asset data for team collaboration and monitoring.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow when a new creative asset is uploaded, receiving the asset URL as input.

- **Nodes Involved:**  
  - Asset Upload Trigger  
  - Sticky Note 1

- **Node Details:**  

  - **Asset Upload Trigger**  
    - Type: Webhook Trigger  
    - Role: Receives HTTP POST requests at endpoint `/asset-upload` containing asset metadata, especially `asset_url`.  
    - Configuration: Webhook path `/asset-upload`, no special options.  
    - Inputs: External HTTP calls (file upload systems, DAM tools, manual uploads).  
    - Outputs: Passes JSON including `asset_url` to next node.  
    - Edge Cases: Missing or malformed URL, unauthorized calls, webhook downtime.  
    - Sticky Note: Explains setup and expected usage for this trigger.

  - **Sticky Note 1**  
    - Purpose: Documentation of Step 1, describes webhook role and configuration.  
    - Content highlights: Webhook setup, asset_url requirement, trigger sources.

---

#### 2.2 AI Asset Analysis

- **Overview:**  
  Uses the ScrapeGraphAI node to analyze the asset URL’s content, extracting detailed metadata about format, colors, description, style, and usage suggestions in JSON.

- **Nodes Involved:**  
  - ScrapeGraphAI Asset Analyzer  
  - Sticky Note 2

- **Node Details:**  

  - **ScrapeGraphAI Asset Analyzer**  
    - Type: ScrapeGraphAI node (AI Content Analyzer)  
    - Role: Sends asset URL to AI model for in-depth analysis, returning structured JSON data.  
    - Configuration:  
      - `userPrompt`: Detailed JSON schema requesting asset_type, colors, descriptions, tags, dimensions, file_format, brand elements, usage context.  
      - `websiteUrl`: Expression `={{ $json.asset_url }}` to dynamically pass asset URL from webhook.  
    - Inputs: JSON with `asset_url` from webhook.  
    - Outputs: JSON with analyzed data.  
    - Edge Cases: Invalid or inaccessible URL, AI timeout, malformed AI response, API auth errors.  
    - Sticky Note: Describes AI analysis scope and outputs.

  - **Sticky Note 2**  
    - Purpose: Describes AI analysis capabilities and expected data extraction.

---

#### 2.3 Tag Generation

- **Overview:**  
  Processes the AI-analyzed asset data to generate enriched, multi-category tags for better asset classification and retrieval.

- **Nodes Involved:**  
  - Tag Generator (Code)  
  - Sticky Note 3

- **Node Details:**  

  - **Tag Generator**  
    - Type: Code node (JavaScript)  
    - Role: Generates tags based on asset type, colors, style elements, suggested tags, usage context, brand elements, and dimensions.  
    - Configuration:  
      - Custom JS code implementing:  
        - Basic type tags (e.g., image, video)  
        - Color name mapping from hex codes  
        - Style and suggested tags inclusion  
        - Usage context tags with suffixes  
        - Brand element prefixed tags  
        - Orientation tags (landscape, portrait, square) based on dimensions  
        - Resolution-based tags (high-resolution, web-optimized)  
    - Key Expressions: Reads AI result JSON from input, outputs enhanced tags array, tag count, and processed timestamp.  
    - Inputs: AI analysis JSON.  
    - Outputs: Extended JSON with `generated_tags`, `tag_count`, and `processed_at`.  
    - Edge Cases: Missing fields in AI data, unexpected color codes, empty arrays.  
    - Sticky Note: Explains tag categories and logic.

  - **Sticky Note 3**  
    - Purpose: Details the tag generation strategy and categories.

---

#### 2.4 Brand Compliance Evaluation

- **Overview:**  
  Checks the asset against predefined brand guidelines, scoring compliance and identifying issues or warnings.

- **Nodes Involved:**  
  - Brand Compliance Checker (Code)  
  - Sticky Note 4

- **Node Details:**  

  - **Brand Compliance Checker**  
    - Type: Code node (JavaScript)  
    - Role: Validates asset colors, presence of brand elements, file format, and resolution against brand rules.  
    - Configuration:  
      - Hardcoded brand guidelines including approved colors, forbidden colors, required elements, approved fonts, max file size, approved formats, and minimum resolution.  
      - Compliance logic calculates score starting at 100, deducts points for issues or warnings, sets status as 'approved', 'approved-with-warnings', or 'rejected'.  
      - Returns detailed issues, warnings, boolean pass, and compliance status.  
    - Inputs: Extended asset data with tags and analysis.  
    - Outputs: Asset data augmented with `brand_compliance` object and `compliance_checked_at` timestamp.  
    - Edge Cases: Missing brand elements, unapproved colors, unsupported file formats, low resolution.  
    - Sticky Note: Describes compliance criteria and scoring.

  - **Sticky Note 4**  
    - Purpose: Explains brand compliance logic and checks.

---

#### 2.5 Asset Organization

- **Overview:**  
  Generates a structured folder path and standardized filename for the asset, compiles comprehensive metadata, and prepares an asset record for storage and tracking.

- **Nodes Involved:**  
  - Asset Organizer (Code)  
  - Sticky Note 5

- **Node Details:**  

  - **Asset Organizer**  
    - Type: Code node (JavaScript)  
    - Role:  
      - Creates a date-based folder hierarchy: `/creative-assets/YYYY/MM/asset_type/usage_context`  
      - Constructs standardized filename with asset type, usage context, and timestamp plus file extension.  
      - Compiles extensive metadata including original filename, upload date, dimensions, tags, brand compliance, style, description, searchable keywords.  
      - Generates a unique asset record ID and sets status from compliance results.  
    - Inputs: Asset data with compliance and tags.  
    - Outputs: JSON containing organization details (`folder_structure`, `file_naming`, `metadata`, `storage_location`) plus final `asset_record`.  
    - Edge Cases: Missing asset type, file format, or usage context; date function errors.  
    - Sticky Note: Summarizes folder structure, naming conventions, and metadata creation.

  - **Sticky Note 5**  
    - Purpose: Explains folder hierarchy, naming, and metadata strategies.

---

#### 2.6 Dashboard Update

- **Overview:**  
  Updates a Google Sheets document with the organized asset data to provide a live dashboard for the creative team.

- **Nodes Involved:**  
  - Creative Team Dashboard (Google Sheets)  
  - Sticky Note 6

- **Node Details:**  

  - **Creative Team Dashboard**  
    - Type: Google Sheets node  
    - Role: Appends or updates rows in a specified sheet named "Creative Assets Dashboard" within a Google Sheets document.  
    - Configuration:  
      - Operation: `appendOrUpdate` (adds new rows or updates existing ones based on key columns)  
      - Sheet Name: "Creative Assets Dashboard"  
      - Document ID: Placeholder `YOUR_GOOGLE_SHEET_ID` (must be replaced with actual ID)  
    - Inputs: Final asset record JSON from Asset Organizer.  
    - Outputs: Status of sheet operation.  
    - Credentials: Requires Google Sheets OAuth2 credentials with edit permissions.  
    - Edge Cases: Invalid document ID, permission errors, API quota limits, malformed input data.  
    - Sticky Note: Describes dashboard features including inventory tracking, compliance monitoring, and collaboration.

  - **Sticky Note 6**  
    - Purpose: Highlights dashboard capabilities and use cases.

---

### 3. Summary Table

| Node Name                   | Node Type                   | Functional Role                      | Input Node(s)              | Output Node(s)            | Sticky Note                                                                                 |
|-----------------------------|-----------------------------|------------------------------------|----------------------------|---------------------------|---------------------------------------------------------------------------------------------|
| Asset Upload Trigger         | Webhook Trigger             | Receives asset upload events       | -                          | ScrapeGraphAI Asset Analyzer | Step 1: Asset Upload Trigger 📤: Webhook setup and asset_url requirement                      |
| ScrapeGraphAI Asset Analyzer| ScrapeGraphAI Node          | AI analyzes asset URL content      | Asset Upload Trigger       | Tag Generator             | Step 2: ScrapeGraphAI Asset Analyzer 🤖: AI extraction of asset metadata                     |
| Tag Generator               | Code Node (JavaScript)      | Generates enriched asset tags      | ScrapeGraphAI Asset Analyzer| Brand Compliance Checker  | Step 3: Tag Generator 🏷️: Multi-category tag creation logic                                |
| Brand Compliance Checker    | Code Node (JavaScript)      | Evaluates brand compliance         | Tag Generator              | Asset Organizer           | Step 4: Brand Compliance Checker ✅: Scoring and validating against brand guidelines         |
| Asset Organizer             | Code Node (JavaScript)      | Creates folder structure, metadata | Brand Compliance Checker   | Creative Team Dashboard    | Step 5: Asset Organizer 📁: Folder structure, naming, metadata, and record creation          |
| Creative Team Dashboard     | Google Sheets Node          | Updates asset dashboard             | Asset Organizer            | -                         | Step 6: Creative Team Dashboard 📊: Dashboard update with asset inventory and compliance    |
| Sticky Note 1               | Sticky Note                 | Documentation                      | -                          | -                         | Step 1: Asset Upload Trigger 📤 (covers Asset Upload Trigger)                               |
| Sticky Note 2               | Sticky Note                 | Documentation                      | -                          | -                         | Step 2: ScrapeGraphAI Asset Analyzer 🤖 (covers ScrapeGraphAI Asset Analyzer)                |
| Sticky Note 3               | Sticky Note                 | Documentation                      | -                          | -                         | Step 3: Tag Generator 🏷️ (covers Tag Generator)                                            |
| Sticky Note 4               | Sticky Note                 | Documentation                      | -                          | -                         | Step 4: Brand Compliance Checker ✅ (covers Brand Compliance Checker)                       |
| Sticky Note 5               | Sticky Note                 | Documentation                      | -                          | -                         | Step 5: Asset Organizer 📁 (covers Asset Organizer)                                        |
| Sticky Note 6               | Sticky Note                 | Documentation                      | -                          | -                         | Step 6: Creative Team Dashboard 📊 (covers Creative Team Dashboard)                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger Node**  
   - Node Type: Webhook  
   - Name: `Asset Upload Trigger`  
   - Parameters:  
     - Path: `/asset-upload`  
     - HTTP Method: POST (default)  
   - Purpose: Receive asset uploads with JSON containing `asset_url`.  
   - Connect output to ScrapeGraphAI node.

2. **Create ScrapeGraphAI Node**  
   - Node Type: ScrapeGraphAI (AI analysis)  
   - Name: `ScrapeGraphAI Asset Analyzer`  
   - Parameters:  
     - `userPrompt`: Include JSON schema requesting fields like `asset_type`, `primary_colors`, `content_description`, `text_content`, `style_elements`, `suggested_tags`, `dimensions`, `file_format`, `brand_elements`, `usage_context`.  
     - `websiteUrl`: Expression `={{ $json.asset_url }}` to use webhook payload.  
   - Connect output to Tag Generator node.

3. **Create Code Node for Tag Generation**  
   - Node Type: Code (JavaScript)  
   - Name: `Tag Generator`  
   - Paste JS code that:  
     - Reads AI analysis result.  
     - Maps colors to names.  
     - Generates tags from asset type, colors, styles, suggested tags, usage context, brand elements, dimensions.  
     - Outputs JSON with `generated_tags`, `tag_count`, `processed_at`.  
   - Connect output to Brand Compliance Checker node.

4. **Create Code Node for Brand Compliance**  
   - Node Type: Code (JavaScript)  
   - Name: `Brand Compliance Checker`  
   - Paste JS code that:  
     - Defines brand guidelines (approved colors, forbidden colors, required elements, approved formats, resolution).  
     - Checks asset data against guidelines.  
     - Calculates compliance score and status (`approved`, `approved-with-warnings`, `rejected`).  
     - Outputs asset data enriched with `brand_compliance` and timestamp.  
   - Connect output to Asset Organizer node.

5. **Create Code Node for Asset Organization**  
   - Node Type: Code (JavaScript)  
   - Name: `Asset Organizer`  
   - Paste JS code that:  
     - Builds folder paths based on current year/month, asset type, usage context.  
     - Generates standardized file name with timestamp and extension.  
     - Compiles metadata including tags, compliance, descriptions, searchable keywords.  
     - Creates unique asset ID and status.  
     - Outputs full asset record.  
   - Connect output to Google Sheets node.

6. **Create Google Sheets Node**  
   - Node Type: Google Sheets  
   - Name: `Creative Team Dashboard`  
   - Parameters:  
     - Operation: `appendOrUpdate`  
     - Sheet Name: `Creative Assets Dashboard`  
     - Document ID: Replace `YOUR_GOOGLE_SHEET_ID` with actual Google Sheet ID.  
   - Credentials: Configure Google OAuth2 with edit permissions.  
   - Connect input from Asset Organizer node.

7. **Add Sticky Notes for Documentation** (optional)  
   - Add sticky notes near each main node summarizing their purpose and configuration as per the notes in the workflow.

8. **Final Setup**  
   - Activate webhook endpoint.  
   - Test with valid asset upload JSON containing `asset_url`.  
   - Monitor workflow executions and Google Sheets updates.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                      |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------|
| The workflow relies on ScrapeGraphAI for AI-driven asset metadata extraction.                    | ScrapeGraphAI node, requires valid credentials and internet access.                 |
| Brand guidelines in compliance check are customizable; adapt colors, elements, and formats as needed. | Brand Compliance Checker node JavaScript code.                                      |
| Google Sheets node requires OAuth2 setup with proper scopes for editing sheets.                  | Google Sheets API documentation and OAuth2 setup instructions.                      |
| Folder and file naming conventions follow ISO timestamps and asset properties for consistency.  | Asset Organizer node logic ensures unique, searchable asset records.                |
| Sticky notes provide step-by-step explanations for each workflow block to facilitate maintenance. | n8n Sticky Note nodes attached near corresponding nodes.                            |
| Replace placeholder Google Sheet ID with your actual document ID before activating the workflow. | Google Sheets node configuration.                                                  |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.

---