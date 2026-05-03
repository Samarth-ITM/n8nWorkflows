Generate SEO anchor texts from Google Sheets with Claude 4 Sonnet

https://n8nworkflows.xyz/workflows/generate-seo-anchor-texts-from-google-sheets-with-claude-4-sonnet-8928


# Generate SEO anchor texts from Google Sheets with Claude 4 Sonnet

### 1. Workflow Overview

This n8n workflow automates the generation of SEO-optimized internal link anchor texts using an AI language model (Claude 4 Sonnet) based on page data stored in a Google Sheets document. It targets SEO specialists, content managers, and digital marketers who want to streamline and scale the creation of diverse, semantically relevant anchor texts for internal linking. The workflow includes the following logical blocks:

- **1.1 Input Reception and Initialization:** Receives a chat message containing a Google Sheets URL and extracts input parameters.
- **1.2 Data Retrieval and Filtering:** Connects to Google Sheets via OAuth2, retrieves page data, and filters for pages needing anchor generation.
- **1.3 Batch Processing Setup:** Splits filtered data into manageable batches for sequential processing.
- **1.4 AI-Powered Anchor Generation:** For each page, sends a detailed prompt with page context to the Claude 4 Sonnet AI model to generate 10 SEO anchor texts with linguistic variations.
- **1.5 Data Integration and Sheet Update:** Merges AI-generated anchors into the original data structure and updates the Google Sheets document with the new anchor texts.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

- **Overview:**  
  This block triggers the workflow upon receiving a chat message. It captures the Google Sheets URL from the incoming message and initiates the data retrieval process.

- **Nodes Involved:**  
  - When chat message received  
  - Ge sheets (Google Sheets read node)  
  - Sticky Note (Phase 1: Workflow Initialization)  
  - Sticky Note3 (User instructions and system overview)  

- **Node Details:**  

  - **When chat message received**  
    - *Type & Role:* LangChain Chat Trigger node, webhook mode.  
    - *Configuration:* Public webhook to receive chat inputs; response mode set to output through response node.  
    - *Expressions:* Extracts the chat input text containing the Google Sheets URL.  
    - *Connections:* Output to "Ge sheets" node.  
    - *Potential Failures:* Webhook timeout, invalid or missing chat input, malformed URL.  
    - *Version:* 1.3  

  - **Ge sheets**  
    - *Type & Role:* Google Sheets node, reads data from the "Anchor" sheet.  
    - *Configuration:* Reads entire sheet using URL from chat input, OAuth2 credentials required.  
    - *Expressions:* Document ID dynamically set from chat message input.  
    - *Connections:* Output to "Filter" node.  
    - *Potential Failures:* OAuth2 authentication errors, invalid sheet URL, API quota exceeded, empty or corrupt sheet data.  
    - *Version:* 4.7  

  - **Sticky Notes (Phase 1 and Instructions)**  
    - Provide user-facing comments and detailed instructions about workflow initialization and expected input format.

---

#### 2.2 Data Retrieval and Filtering

- **Overview:**  
  Filters the retrieved data to select only pages that have a URL but lack previously generated anchor texts. This ensures the workflow processes only necessary entries.

- **Nodes Involved:**  
  - Filter  
  - Sticky Note (Phase 2: Data Filtering and Validation)  
  - Sticky Note5 (Detailed filtering explanation)  

- **Node Details:**  

  - **Filter**  
    - *Type & Role:* Filters data based on conditions.  
    - *Configuration:*  
      - Condition 1: URL field must not be empty.  
      - Condition 2: Anchors field must be empty.  
    - *Connections:* Output to "Loop Over Items".  
    - *Potential Failures:* Expression errors if expected fields are missing, no matching items resulting in empty batches.  
    - *Version:* 2.2  

  - **Sticky Notes**  
    - Explain filtering logic and validation goals to ensure clean dataset preparation.

---

#### 2.3 Batch Processing Setup

- **Overview:**  
  Splits the filtered dataset into batches (default size 1) to process pages sequentially, avoiding memory overload and enabling reliable iterative processing.

- **Nodes Involved:**  
  - Loop Over Items  
  - Sticky Note (Phase 3: Batch Processing Setup)  
  - Sticky Note7 (Batch processing explanation)  

- **Node Details:**  

  - **Loop Over Items**  
    - *Type & Role:* SplitInBatches node to iterate over items.  
    - *Configuration:* Default batch size (implicitly 1 item per batch).  
    - *Connections:*  
      - Main output 1 (empty) presumably for batch completion or parallel processing.  
      - Main output 2 to "Générateur d'ancres" for AI processing.  
    - *Potential Failures:* Empty input causes no loops; batch size misconfiguration may cause delays or memory issues.  
    - *Version:* 3  

  - **Sticky Notes**  
    - Describe the batch processing logic and iterative handling benefits.

---

#### 2.4 AI-Powered Anchor Generation

- **Overview:**  
  Uses Claude 4 Sonnet (Anthropic’s AI model) to generate 10 SEO anchor texts with multiple linguistic variations per page, based on rich contextual information from the Google Sheet.

- **Nodes Involved:**  
  - Générateur d'ancres (LangChain Agent)  
  - Anthropic Chat Model (Language Model node)  
  - Sticky Note1 (Phase 4: AI-Powered Anchor Generation)  
  - Sticky Note8 (Detailed AI generation explanation)  

- **Node Details:**  

  - **Générateur d'ancres**  
    - *Type & Role:* LangChain Agent node forming the prompt for AI.  
    - *Configuration:*  
      - Detailed prompt with instructions for generating SEO anchors, including semantic relevance, keyword optimization, naturalness, anchor types, linguistic variations, and output format.  
      - Dynamic insertion of page data fields: URL, hierarchical titles (Niv 0-3), page description.  
    - *Connections:* Input from "Loop Over Items" batch output; output to "Import Sheets".  
    - *Potential Failures:* Prompt generation errors, missing input fields, AI model response timeout or failure, API quota limits.  
    - *Version:* 2.2  

  - **Anthropic Chat Model**  
    - *Type & Role:* Connects to Claude 4 Sonnet AI model.  
    - *Configuration:* Model set to "claude-sonnet-4-20250514".  
    - *Credentials:* Requires valid Anthropic API credentials.  
    - *Connections:* Connected as AI language model for the "Générateur d'ancres" node.  
    - *Potential Failures:* Authentication failure, API quota exceeded, network issues, model unavailability.  
    - *Version:* 1.3  

  - **Sticky Notes**  
    - Explain the AI model’s role in generating diverse, SEO-compliant anchor texts with linguistic variations to improve internal linking strategies.

---

#### 2.5 Data Integration and Google Sheets Update

- **Overview:**  
  Integrates AI-generated anchors back into the original dataset, preserving all existing columns, and updates the Google Sheets document to add the new anchor texts in the “Ancre” column.

- **Nodes Involved:**  
  - Import Sheets (Code node)  
  - Update sheets (Google Sheets update node)  
  - Sticky Note2 (Phase 5: Data Integration and Google Sheets Update)  
  - Sticky Note9 (Detailed data integration explanation)  

- **Node Details:**  

  - **Import Sheets**  
    - *Type & Role:* JavaScript Code node that merges AI-generated anchors with existing sheet data.  
    - *Configuration:*  
      - Copies all existing properties from batch item.  
      - Replaces or adds the "Ancre" field with AI output.  
      - Returns transformed item for update.  
    - *Connections:* Output to "Update sheets".  
    - *Potential Failures:* Code logic errors, missing fields, data type mismatches.  
    - *Version:* 2  

  - **Update sheets**  
    - *Type & Role:* Google Sheets node, updates rows in the "Anchor" sheet.  
    - *Configuration:*  
      - Operation set to "update".  
      - Auto-mapping input data to columns including the new "Ancre" field.  
      - Document ID dynamically set from chat input URL.  
      - Requires OAuth2 credentials.  
    - *Connections:* Output not connected further (end of processing for each batch).  
    - *Potential Failures:* Authentication errors, concurrency conflicts, data overwrite risks, API rate limits.  
    - *Version:* 4.5  

  - **Sticky Notes**  
    - Describe the smooth integration of generated anchors into Google Sheets, maintaining data integrity and enabling immediate use for SEO efforts.

---

### 3. Summary Table

| Node Name                 | Node Type                          | Functional Role                                  | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                              |
|---------------------------|----------------------------------|-------------------------------------------------|------------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------|
| When chat message received| LangChain Chat Trigger            | Receives chat message with Google Sheets URL    |                              | Ge sheets                     | # Phase 1: Workflow Initialization                                                                      |
| Ge sheets                 | Google Sheets Read                | Reads page data from Google Sheets "Anchor"     | When chat message received    | Filter                        | # Phase 1: Workflow Initialization<br>### What you do: Instructions for input data                      |
| Filter                    | Filter                           | Filters pages with URL and empty anchor field   | Ge sheets                    | Loop Over Items               | # Phase 2: Data Filtering and Validation<br>### What the system does: Filtering logic and validation    |
| Loop Over Items           | SplitInBatches                   | Splits filtered data into batches for iteration | Filter                       | Générateur d'ancres (main 2)  | # Phase 3: Batch Processing Setup<br>### What the system does: Batch handling and loop setup             |
| Générateur d'ancres       | LangChain Agent                  | Constructs prompt for SEO anchor generation AI  | Loop Over Items              | Import Sheets                 | # Phase 4: AI-Powered Anchor Generation<br>## What the system does: AI generates SEO anchors            |
| Anthropic Chat Model      | AI Language Model (Anthropic)    | Runs Claude 4 Sonnet model with prompt           |                             | Générateur d'ancres (ai_languageModel) | # Phase 4: AI-Powered Anchor Generation                                                               |
| Import Sheets             | Code                            | Merges AI output with existing row data         | Générateur d'ancres          | Update sheets                | # Phase 5: Data Integration and Google Sheets Update<br>## What the system does: Data merging          |
| Update sheets             | Google Sheets Update             | Updates Google Sheets with generated anchors     | Import Sheets                |                               | # Phase 5: Data Integration and Google Sheets Update                                                   |
| Sticky Note               | Sticky Note                     | Instructional/organizational                     |                              |                               | Various sticky notes provide phase explanations and usage instructions (refer to detailed notes above) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Trigger Node:**  
   - Add a LangChain Chat Trigger node named **"When chat message received"**.  
   - Set mode to “webhook”, make it public, and set response mode to “responseNode”.  
   - This node receives chat messages containing the Google Sheets URL.

2. **Add Google Sheets Read Node:**  
   - Add a **Google Sheets** node named **"Ge sheets"**.  
   - Operation: Read.  
   - Sheet Name: “Anchor”.  
   - Document ID: Set dynamically using expression `={{ $('When chat message received').item.json.chatInput }}` to extract the URL from webhook input.  
   - Connect credentials for Google Sheets OAuth2.  
   - Connect output of "When chat message received" to this node.

3. **Add Filter Node for Data Validation:**  
   - Add a **Filter** node named **"Filter"**.  
   - Conditions:  
     - URL field must be not empty.  
     - Anchors field must be empty.  
   - Connect output of "Ge sheets" to this node.

4. **Add SplitInBatches Node:**  
   - Add a **SplitInBatches** node named **"Loop Over Items"**.  
   - Default batch size (1) is fine for sequential processing.  
   - Connect output of "Filter" node to this node.

5. **Add LangChain Agent Node:**  
   - Add a **LangChain Agent** node named **"Générateur d'ancres"**.  
   - Paste the detailed prompt provided in the original workflow, which instructs the AI to generate 10 SEO anchors with variations based on input page data fields (URL, Niv 0-3, Description).  
   - Use expressions to dynamically insert page data, e.g., `{{ $json.URL }}` etc.  
   - Connect the second output of "Loop Over Items" to this node.

6. **Add Anthropic Chat Model Node:**  
   - Add an **Anthropic Chat Model** node named **"Anthropic Chat Model"**.  
   - Configure model to “claude-sonnet-4-20250514”.  
   - Enter valid Anthropic API credentials.  
   - Connect this node as the `ai_languageModel` input for the "Générateur d'ancres" node.

7. **Add Code Node for Data Merging:**  
   - Add a **Code** node named **"Import Sheets"**.  
   - Paste the provided JavaScript code that merges AI-generated anchors into the existing page data object, updating the "Ancre" field.  
   - Connect output of "Générateur d'ancres" to this node.

8. **Add Google Sheets Update Node:**  
   - Add a **Google Sheets** node named **"Update sheets"**.  
   - Operation: Update.  
   - Sheet Name: “Anchor”.  
   - Document ID: same expression as before `={{ $('When chat message received').item.json.chatInput }}`.  
   - Map all input columns automatically, including the new "Ancre" column.  
   - Connect Google Sheets OAuth2 credentials.  
   - Connect output of "Import Sheets" to this node.

9. **Connect the Nodes in Proper Sequence:**  
   - "When chat message received" → "Ge sheets" → "Filter" → "Loop Over Items" → "Générateur d'ancres" → "Import Sheets" → "Update sheets".  
   - Connect "Anthropic Chat Model" as AI model input to "Générateur d'ancres".

10. **Add Sticky Notes (Optional):**  
    - Add notes to explain each phase for maintainability and user instruction.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                         |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Duplicate this Google Sheets template to get started: https://docs.google.com/spreadsheets/d/1VNl8xLYgRrNcKrmN9hCdfov1dMnwD44tAALJZAlagCo. Fill in the “Anchor” sheet with your page data (URL, title hierarchy, description).                                                                                                                                                                                                                         | User onboarding and data format instructions                                                         |
| The workflow requires OAuth2 credentials for Google Sheets and API credentials for Anthropic’s Claude 4 Sonnet model. Ensure these are configured securely in n8n.                                                                                                                                                                                                                                                                                  | Credentials setup                                                                                      |
| Avoid generic anchors like “click here” or “read more” as per AI prompt instructions to maintain SEO quality.                                                                                                                                                                                                                                                                                                                                     | SEO best practices embedded in AI prompt                                                             |
| For more advanced enterprise automation, contact Growth-AI.fr via LinkedIn: https://www.linkedin.com/in/allanvaccarizi/ and https://www.linkedin.com/in/hugo-marinier-%F0%9F%A7%B2-6537b633/                                                                                                                                                                                                                                                         | Project credits and contact information                                                              |
| The AI prompt enforces linguistic variations including singular/plural, gender, tense, formality, synonyms, and structural changes to ensure diverse anchor texts that avoid SEO penalties for over-optimization.                                                                                                                                                                                                                                     | SEO anchor generation best practices                                                                 |

---

**Disclaimer:** The text provided is generated exclusively from an automated n8n workflow, strictly respecting all applicable content policies and containing no illegal, offensive, or protected elements. All data processed is legal and public.