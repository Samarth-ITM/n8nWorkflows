Process & Catalog Dress Images with GPT-4o, Cloudinary, and Google Sheets

https://n8nworkflows.xyz/workflows/process---catalog-dress-images-with-gpt-4o--cloudinary--and-google-sheets-9312


# Process & Catalog Dress Images with GPT-4o, Cloudinary, and Google Sheets

### 1. Workflow Overview

This workflow automates the process of managing dress images by integrating Google Drive, Cloudinary, Azure OpenAI GPT-4o, and Google Sheets. Its primary purpose is to:

- Search and retrieve dress image files from Google Drive.
- Download these images locally.
- Upload the images to Cloudinary for optimized hosting and management.
- Use Azure OpenAI GPT-4o to generate structured descriptions or metadata about the dresses.
- Log image URLs and AI-generated descriptions into Google Sheets for cataloging and further use.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and File Discovery:** Trigger the workflow manually and search for relevant dress image files in Google Drive.
- **1.2 File Processing Loop:** Iterate over the found files, downloading and preparing them for upload.
- **1.3 Upload & AI Analysis:** Upload images to Cloudinary, invoke Azure OpenAI GPT-4o for descriptive metadata, parse the AI output.
- **1.4 Output Recording:** Append the processed data (image URLs and descriptions) into Google Sheets.
- **1.5 Flow Control & Utilities:** Includes batching, merging outputs, and setting descriptions for proper data handling.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception and File Discovery

- **Overview:**  
  This block starts the workflow manually and searches for dress image files and folders in Google Drive to process.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Search files and folders (Google Drive)  
  - Loop Over Items (Split in Batches)  
  - Search files and folders1 (Google Drive)

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on manual command.  
    - Input: None  
    - Output: Initiates the search in Google Drive.  
    - Edge cases: User may forget to trigger or trigger multiple times; no error expected.

  - **Search files and folders**  
    - Type: Google Drive node (v3)  
    - Role: Searches Google Drive for initial files/folders (likely dresses image root).  
    - Configuration: Defaults, no filters specified in JSON, possibly searches root or predefined folder.  
    - Input: Trigger from Manual Trigger  
    - Output: List of files/folders to process.  
    - Edge cases: Permissions errors if Drive access is invalid; empty results.

  - **Loop Over Items**  
    - Type: SplitInBatches (v3)  
    - Role: Processes files in manageable batches to avoid overloading downstream nodes.  
    - Configuration: Batch size unspecified, default assumed.  
    - Input: Files from Search files and folders  
    - Output: Processes batches, supports pagination if many files.  
    - Edge cases: Batch size too large may cause timeouts; batch size too small reduces throughput.

  - **Search files and folders1**  
    - Type: Google Drive node (v3)  
    - Role: Secondary search to find nested files or folders per each batch item.  
    - Input: Executed per batch item (loop continuation)  
    - Output: Further nested files/folders for processing.  
    - Edge cases: Same as primary search; permissions and empty results.

---

#### 1.2 File Processing Loop

- **Overview:**  
  For each discovered file, this block downloads the file and prepares it for upload to Cloudinary.

- **Nodes Involved:**  
  - Download file (Google Drive)  
  - upload frames to cloudinary (HTTP Request)  

- **Node Details:**

  - **Download file**  
    - Type: Google Drive node (v3)  
    - Role: Downloads the actual image file locally or as a binary buffer for uploading.  
    - Input: Files from Search files and folders1  
    - Output: Binary data of the image file.  
    - Edge cases: Download failure if file removed or access revoked; large files causing timeout.

  - **upload frames to cloudinary**  
    - Type: HTTP Request (v4.2)  
    - Role: Uploads the downloaded image to Cloudinary via its API.  
    - Configuration: Retries up to 5 times on failure, retryOnFail enabled.  
    - Input: Binary data from Download file  
    - Output: Cloudinary response including URL and metadata.  
    - Edge cases: Network errors, Cloudinary API limits, invalid credentials.

---

#### 1.3 Upload & AI Analysis

- **Overview:**  
  This block sends the Cloudinary image URLs and related data to Azure OpenAI GPT-4o to generate structured metadata or descriptions, parses the output, and prepares it for logging.

- **Nodes Involved:**  
  - azure openai (HTTP Request)  
  - Setting description (Set)  
  - flattening descriptions (Code)  
  - Azure OpenAI Chat Model (LangChain Azure OpenAI)  
  - Structured Output Parser (LangChain output parser)  
  - AI Agent (LangChain agent)  

- **Node Details:**

  - **upload frames to cloudinary** (connects to azure openai)  
    - Sends image info to the AI model for captioning or metadata generation.

  - **azure openai**  
    - Type: HTTP Request (v4.2)  
    - Role: Makes direct API call to Azure OpenAI endpoint (GPT-4o).  
    - Input: Cloudinary image data.  
    - Output: Raw AI response.  
    - Edge cases: API rate limits, timeouts, malformed requests.

  - **Setting description**  
    - Type: Set (v3.4)  
    - Role: Prepares or sets key parameters or variables for AI processing (e.g., prompt structure).  
    - Input: AI raw response.  
    - Output: Structured data for next step.

  - **flattening descriptions**  
    - Type: Code (JavaScript)  
    - Role: Processes AI output to flatten nested structures, making it easier to handle downstream.  
    - Input: Data from Setting description  
    - Output: Flattened descriptions.  
    - Edge cases: Code errors if data format unexpected.

  - **Azure OpenAI Chat Model**  
    - Type: LangChain Azure OpenAI (v1)  
    - Role: Sends prompts and receives chat-based AI responses using GPT-4o.  
    - Input: Flattened descriptions or prompts.  
    - Output: Parsed AI chat response.  
    - Edge cases: Model errors, token limits.

  - **Structured Output Parser**  
    - Type: LangChain Structured Output Parser (v1.3)  
    - Role: Parses AI response into consistent structured format (e.g., JSON).  
    - Input: AI chat response.  
    - Output: Structured metadata.  
    - Edge cases: Parsing errors if AI output deviates from expected format.

  - **AI Agent**  
    - Type: LangChain Agent (v2)  
    - Role: Central AI orchestrator node that manages prompt, model invocation, and output parsing.  
    - Input: Flattened descriptions and AI model outputs.  
    - Output: Final structured metadata for logging.  
    - Output connections: Sends data to Google Sheets append nodes.  
    - Edge cases: Expression or integration failures, AI timeouts.

---

#### 1.4 Output Recording

- **Overview:**  
  Logs the Cloudinary image URLs and AI-generated metadata into Google Sheets, preserving catalog records.

- **Nodes Involved:**  
  - Append row in sheet (Google Sheets)  
  - Append row in sheet2 (Google Sheets)  
  - Merge outputs for Logic (Merge)

- **Node Details:**

  - **Append row in sheet**  
    - Type: Google Sheets (v4.6)  
    - Role: Appends one part of data (likely image URLs) into a sheet.  
    - Input: AI Agent output  
    - Output: Confirmation of sheet update.  
    - Edge cases: Sheet access errors, quota limits.

  - **Append row in sheet2**  
    - Type: Google Sheets (v4.6)  
    - Role: Appends other part of data (likely AI descriptions or metadata) into another sheet or tab.  
    - Input: AI Agent output (parallel to first Append row)  
    - Output: Confirmation of update.  
    - Edge cases: Same as above.

  - **Merge outputs for Logic**  
    - Type: Merge (v3.2)  
    - Role: Combines outputs from both Append row nodes and links back to Loop Over Items to continue processing.  
    - Input: From Append row nodes  
    - Output: Feeds Loop Over Items to process next batch.  
    - Edge cases: Merge conflicts, data synchronization issues.

---

#### 1.5 Flow Control & Utilities

- **Overview:**  
  Contains nodes that manage the flow order, retry logic, and temporary data setting for the workflow.

- **Nodes Involved:**  
  - Loop Over Items (Split in Batches)  
  - Merge outputs for Logic (Merge)  
  - Setting description (Set)  
  - flattening descriptions (Code)  

- **Role:**  
  These nodes orchestrate the main data flow, batch processing, retry, and data preparation for AI interaction.

---

### 3. Summary Table

| Node Name                     | Node Type                                  | Functional Role                                | Input Node(s)                 | Output Node(s)                      | Sticky Note                 |
|-------------------------------|--------------------------------------------|-----------------------------------------------|------------------------------|-----------------------------------|-----------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                            | Workflow start trigger                         | None                         | Search files and folders           |                             |
| Search files and folders       | Google Drive (v3)                          | Search root Google Drive files/folders        | When clicking ‘Execute workflow’ | Loop Over Items                   |                             |
| Loop Over Items                | SplitInBatches (v3)                        | Batch iteration over files                      | Search files and folders       | Search files and folders1, Loop continuation|                             |
| Search files and folders1      | Google Drive (v3)                          | Nested file/folder search per batch item      | Loop Over Items               | Download file                     |                             |
| Download file                 | Google Drive (v3)                          | Download file binary data                       | Search files and folders1     | upload frames to cloudinary        |                             |
| upload frames to cloudinary   | HTTP Request (v4.2)                        | Upload images to Cloudinary                     | Download file                | azure openai                     |                             |
| azure openai                 | HTTP Request (v4.2)                        | Call Azure OpenAI GPT-4o API                    | upload frames to cloudinary  | Setting description               |                             |
| Setting description          | Set (v3.4)                                | Prepare AI prompt/parameters                     | azure openai                 | flattening descriptions            |                             |
| flattening descriptions      | Code (v2)                                 | Flatten AI response data                         | Setting description          | AI Agent                        |                             |
| Azure OpenAI Chat Model      | LangChain Azure OpenAI (v1)                | Chat-based AI processing                        | flattening descriptions      | AI Agent (ai_languageModel input) |                             |
| Structured Output Parser     | LangChain Structured Output Parser (v1.3) | Parse AI output into structured format          | AI Agent (ai_outputParser)   | AI Agent                        |                             |
| AI Agent                    | LangChain Agent (v2)                        | Orchestrate AI prompt, model, and parse output | flattening descriptions, Azure OpenAI Chat Model, Structured Output Parser | Append row in sheet, Append row in sheet2 |                             |
| Append row in sheet          | Google Sheets (v4.6)                        | Log image URLs or related data in Google Sheets | AI Agent                    | Merge outputs for Logic           |                             |
| Append row in sheet2         | Google Sheets (v4.6)                        | Log AI-generated descriptions/metadata          | AI Agent                    | Merge outputs for Logic           |                             |
| Merge outputs for Logic      | Merge (v3.2)                               | Combine sheet append results and loop continuation | Append row in sheet, Append row in sheet2 | Loop Over Items               |                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node:**  
   - Type: Manual Trigger  
   - Purpose: Start workflow on demand.

2. **Add Google Drive node “Search files and folders”:**  
   - Type: Google Drive (v3)  
   - Configure to search the root or specific folder containing dress images.  
   - Connect input from Manual Trigger.

3. **Add SplitInBatches node “Loop Over Items”:**  
   - Type: SplitInBatches (v3)  
   - Connect input from Google Drive search node.  
   - Set batch size as desired (e.g., 5-10).

4. **Add Google Drive node “Search files and folders1”:**  
   - Type: Google Drive (v3)  
   - Configure to search nested folders or specific file types as required.  
   - Connect input from second output of SplitInBatches node.

5. **Add Google Drive node “Download file”:**  
   - Type: Google Drive (v3)  
   - Configure to download file content (binary).  
   - Input from “Search files and folders1”.

6. **Add HTTP Request node “upload frames to cloudinary”:**  
   - Type: HTTP Request (v4.2)  
   - Configure to POST binary file data to Cloudinary upload API endpoint.  
   - Set retry attempts to 5, enable retry on fail.  
   - Input from “Download file”.

7. **Add HTTP Request node “azure openai”:**  
   - Type: HTTP Request (v4.2)  
   - Configure with Azure OpenAI GPT-4o endpoint URL, authentication headers (Azure Key).  
   - Input: Cloudinary upload response containing image URL.  
   - Purpose: Send image URL or related prompt to GPT-4o for description generation.

8. **Add Set node “Setting description”:**  
   - Type: Set (v3.4)  
   - Prepare prompt or data structure for AI input based on Azure OpenAI response.  
   - Input from “azure openai”.

9. **Add Code node “flattening descriptions”:**  
   - Type: Code (JavaScript)  
   - Flatten any nested AI response objects to flat key-value pairs for easier handling.  
   - Input from “Setting description”.

10. **Add LangChain Azure OpenAI node “Azure OpenAI Chat Model”:**  
    - Type: LangChain Azure OpenAI (v1)  
    - Configure with Azure OpenAI credentials.  
    - Input: Flattened prompt or data from “flattening descriptions”.

11. **Add LangChain Structured Output Parser node “Structured Output Parser”:**  
    - Type: Structured Output Parser (v1.3)  
    - Configure to parse AI chat outputs into structured JSON format.

12. **Add LangChain Agent node “AI Agent”:**  
    - Type: LangChain Agent (v2)  
    - Connect ai_languageModel input from “Azure OpenAI Chat Model”.  
    - Connect ai_outputParser input from “Structured Output Parser”.  
    - Connect main input from “flattening descriptions”.  
    - Output: AI-processed data for logging.

13. **Add Google Sheets node “Append row in sheet”:**  
    - Type: Google Sheets (v4.6)  
    - Configure with OAuth2 credentials for Google Sheets access.  
    - Target sheet to append Cloudinary URLs or related data.  
    - Input from “AI Agent”.

14. **Add Google Sheets node “Append row in sheet2”:**  
    - Type: Google Sheets (v4.6)  
    - Configure another sheet/tab to append AI-generated metadata/descriptions.  
    - Input from “AI Agent”.

15. **Add Merge node “Merge outputs for Logic”:**  
    - Type: Merge (v3.2)  
    - Combine outputs from both Append row nodes.  
    - Set to wait for both inputs before continuing.  
    - Output connected back to “Loop Over Items” to continue batch processing.

---

### 5. General Notes & Resources

| Note Content                                                                                                                               | Context or Link                               |
|--------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------|
| The workflow uses Azure OpenAI GPT-4o model; ensure your Azure subscription supports this model and credentials are correctly configured. | Azure OpenAI documentation                    |
| Cloudinary upload node uses HTTP POST requests with retry logic to handle transient network failures.                                      | Cloudinary API docs: https://cloudinary.com/documentation/image_upload_api_reference |
| Google Drive and Google Sheets nodes require OAuth2 credentials with appropriate scope permissions for file and sheet access.             | Google API OAuth2 setup instructions          |
| LangChain nodes provide advanced AI workflow orchestration, including chat models and structured output parsing for consistent data.       | LangChain n8n integration docs                |
| Recommended batch size depends on execution environment limits and API rate limits; adjust SplitInBatches accordingly.                      | n8n batching best practices                    |

---

**Disclaimer:** The text provided comes exclusively from an automation workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.