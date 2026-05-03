Classify & Auto-Sort Invoices in Google Drive with GPT-4o

https://n8nworkflows.xyz/workflows/classify---auto-sort-invoices-in-google-drive-with-gpt-4o-6682


# Classify & Auto-Sort Invoices in Google Drive with GPT-4o

### 1. Workflow Overview

This workflow automates the classification and organization of invoice PDF files stored in a specific Google Drive folder by leveraging AI-powered text analysis. Its core purpose is to analyze the textual content of each invoice, categorize it into one of three predefined industries (Retail, Manufacturing, EdTech), and then move the file into the corresponding industry-specific Google Drive folder. This process reduces manual sorting effort and improves document management efficiency.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & File Discovery:** Initiates the workflow manually and searches for invoice files within a designated Google Drive folder.
- **1.2 File Download & Text Extraction:** Downloads each invoice file and extracts its textual content, preparing it for AI analysis.
- **1.3 AI Classification:** Uses a LangChain AI Agent powered by an Azure OpenAI GPT-4o-mini model to classify the extracted text into one of three industry categories.
- **1.4 Conditional Routing & File Organization:** Based on the AI classification result, routes the invoice file to the appropriate Google Drive folder (Retail, Manufacturing, or EdTech).
- **1.5 Batch Processing Control:** Manages the processing of multiple files individually in a loop, ensuring each invoice is handled one at a time.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & File Discovery

**Overview:**  
This block initiates the workflow manually and searches for all invoice files in a specific Google Drive folder to be processed.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Search files and folders (Google Drive)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow on user command  
  - Configuration: No parameters; manual start only  
  - Inputs: None  
  - Outputs: Connects to "Search files and folders"  
  - Edge Cases: User must manually trigger to start process

- **Search files and folders**  
  - Type: Google Drive node (fileFolder resource)  
  - Role: Searches for all files in a specific folder (ID: `1CcHp9tPoC-f7-umMjLdUU-Y7GedKLQkx` named "Indore")  
  - Configuration: Returns all files found; uses OAuth2 credentials for Google Drive  
  - Inputs: Triggered by manual node  
  - Outputs: List of files to process, passed to "Download File"  
  - Edge Cases:  
    - Folder ID must be accessible with provided credentials  
    - Returns all files regardless of type (assumes all are invoices)  
    - Potential API rate limits or permission errors

---

#### 2.2 File Download & Text Extraction

**Overview:**  
Downloads each invoice file from Google Drive and extracts its textual content from the PDF for AI processing.

**Nodes Involved:**  
- Download File (Google Drive)  
- Extract from File (PDF Extraction)  
- Loop Over Items (Split in Batches)

**Node Details:**

- **Download File**  
  - Type: Google Drive node  
  - Role: Downloads the invoice PDF file content using file ID from search results  
  - Configuration: Operation set to "download", file ID taken from each item's JSON field `id`  
  - Credentials: Google Drive OAuth2 account  
  - Inputs: Files from "Search files and folders"  
  - Outputs: Passes file content to "Extract from File"  
  - Edge Cases:  
    - File ID must be valid and accessible  
    - Download failures due to network or permission issues

- **Extract from File**  
  - Type: Extract from File node  
  - Role: Extracts text from downloaded PDF to prepare for AI classification  
  - Configuration: Operation set to "pdf" extraction  
  - Inputs: PDF file content from "Download File"  
  - Outputs: Extracted text passed to "Loop Over Items"  
  - Edge Cases:  
    - PDF with complex formatting or scanned images may yield poor text extraction  
    - Extraction failures due to corrupted or encrypted PDFs  

- **Loop Over Items**  
  - Type: Split in Batches  
  - Role: Processes each invoice file individually in a loop to sequentially handle extraction and classification  
  - Configuration: Defaults; no batch size specified, processes one item at a time  
  - Inputs: Extracted text items from "Extract from File"  
  - Outputs: Forwards individual items to "AI Agent" for classification after each loop  
  - Edge Cases:  
    - Large file sets may cause longer processing time  
    - Node split must be properly connected to avoid data loss or skips

---

#### 2.3 AI Classification

**Overview:**  
Analyzes the extracted invoice text to classify the invoice into one of three industries using a LangChain AI Agent powered by Azure OpenAI GPT-4o-mini.

**Nodes Involved:**  
- AI Agent (LangChain Agent)  
- Azure OpenAI Chat Model (GPT-4o-mini)

**Node Details:**

- **AI Agent**  
  - Type: LangChain AI Agent  
  - Role: Receives invoice text and classifies it into Retail, Manufacturing, or EdTech  
  - Configuration:  
    - Uses a system prompt instructing classification into one of three industries  
    - Input text is the extracted invoice text (`{{$json.text}}`)  
  - Inputs: Text from "Loop Over Items"  
  - Outputs: Classification result JSON with `output` field indicating industry  
  - Sub-node: Uses "Azure OpenAI Chat Model" as underlying language model  
  - Edge Cases:  
    - Text too short or ambiguous for confident classification  
    - API timeouts or quota limits from Azure OpenAI  
    - Misclassification if prompt misunderstood or text noisy  

- **Azure OpenAI Chat Model**  
  - Type: LangChain Azure OpenAI node  
  - Role: Provides GPT-4o-mini model for language understanding and classification  
  - Configuration: Model set to `gpt-4o-mini`, no extra options  
  - Credentials: Azure OpenAI API  
  - Inputs: Connected as AI language model for "AI Agent"  
  - Outputs: Responses passed back to "AI Agent"  
  - Edge Cases:  
    - API authentication errors  
    - Model availability or performance issues  

---

#### 2.4 Conditional Routing & File Organization

**Overview:**  
Routes the invoice file to its respective Google Drive folder based on AI classification.

**Nodes Involved:**  
- Switch (Conditional Logic)  
- Retail (Google Drive move operation)  
- Manufacturing (Google Drive move operation)  
- EdTech (Google Drive move operation)

**Node Details:**

- **Switch**  
  - Type: Switch (Conditional) node  
  - Role: Routes workflow according to AI classification output (`$json.output`)  
  - Configuration:  
    - Three branches for values: "Retail", "Manufacturing", "EdTech"  
  - Inputs: Classification results from "AI Agent"  
  - Outputs: Routes to corresponding Google Drive move nodes  
  - Edge Cases:  
    - Unexpected classification outputs (not matching any branch) will cause no routing  
    - Case sensitivity enforced; output must exactly match one of the three strings

- **Retail**  
  - Type: Google Drive node (move operation)  
  - Role: Moves the invoice file into the Retail folder (`folderId` = `13YZe1eYMQGXqR62n8aQIlYeSHE2HqpAv`)  
  - Configuration: Uses `fileId` from "Download File" node to identify file  
  - Credentials: Google Drive OAuth2  
  - Inputs: Routed from "Switch" if output is "Retail"  
  - Edge Cases:  
    - File must exist and be accessible  
    - Move failures due to permission or lock issues  

- **Manufacturing**  
  - Type: Google Drive node (move operation)  
  - Role: Moves invoice to Manufacturing folder (`folderId` = `1-fmURc7hW1JcQCcP_lk4VxhkOk4GA00H`)  
  - Configuration and credentials same as Retail node  
  - Inputs: Routed from "Switch" if output is "Manufacturing"  
  - Edge Cases: Same as Retail

- **EdTech**  
  - Type: Google Drive node (move operation)  
  - Role: Moves invoice to EdTech folder (`folderId` = `1H-JpO6eNDYGHT4G92-mB846NvaNu8P-1`)  
  - Configuration and credentials same as Retail node  
  - Inputs: Routed from "Switch" if output is "EdTech"  
  - Edge Cases: Same as Retail

---

### 3. Summary Table

| Node Name                | Node Type                          | Functional Role                              | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                  |
|--------------------------|----------------------------------|----------------------------------------------|-------------------------------|--------------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                   | Starts workflow manually                      | None                          | Search files and folders        |                                                                                              |
| Search files and folders  | Google Drive (fileFolder)         | Searches invoice files in a specific folder  | When clicking ‘Execute workflow’ | Download File                  | Searches for files in Google Drive folder "Indore" using OAuth2                              |
| Download File            | Google Drive                      | Downloads each invoice file                   | Search files and folders       | Extract from File               | Downloads files using file IDs from search results                                           |
| Extract from File         | Extract from File                 | Extracts text content from PDF invoices      | Download File                 | Loop Over Items                | Extracts text from PDF files to prepare for AI analysis                                     |
| Loop Over Items          | Split in Batches                  | Processes invoices individually in a loop    | Extract from File             | AI Agent (after second output) | Processes files one by one to manage batch processing                                       |
| AI Agent                 | LangChain Agent                  | Classifies invoice text into an industry     | Loop Over Items               | Switch                        | AI Agent uses system prompt to classify invoices into Retail, Manufacturing, or EdTech       |
| Azure OpenAI Chat Model  | LangChain Azure OpenAI            | Language model powering AI Agent             | None (as AI language model)   | AI Agent                      | Uses GPT-4o-mini model from Azure OpenAI API                                                |
| Switch                   | Switch                          | Routes file based on AI classification       | AI Agent                     | Retail, Manufacturing, EdTech  | Routes workflow to move files based on classification result                                |
| Retail                   | Google Drive                    | Moves invoice file to Retail folder           | Switch (if Retail)            | Loop Over Items                | Moves file to Retail folder in Google Drive                                                |
| Manufacturing            | Google Drive                    | Moves invoice file to Manufacturing folder    | Switch (if Manufacturing)     | Loop Over Items                | Moves file to Manufacturing folder in Google Drive                                         |
| EdTech                   | Google Drive                    | Moves invoice file to EdTech folder           | Switch (if EdTech)            | Loop Over Items                | Moves file to EdTech folder in Google Drive                                                |
| Sticky Note              | Sticky Note                     | Notes on Search files and folders node        | None                         | None                         | Search files and folders (Google Drive): Searches for files in a specific folder            |
| Sticky Note1             | Sticky Note                     | Notes on Download File node                     | None                         | None                         | Download File (Google Drive): Downloads each file found                                     |
| Sticky Note2             | Sticky Note                     | Notes on Extract from File node                 | None                         | None                         | Extract from File: Extracts text from PDF invoices                                          |
| Sticky Note3             | Sticky Note                     | Notes on Loop Over Items node                   | None                         | None                         | Loop Over Items: Processes multiple files individually                                     |
| Sticky Note4             | Sticky Note                     | Notes on AI Agent node                           | None                         | None                         | AI Agent (LangChain): Classifies invoices into Retail, Manufacturing, or EdTech             |
| Sticky Note5             | Sticky Note                     | Notes on Azure OpenAI Chat Model node           | None                         | None                         | Azure OpenAI Chat Model: GPT-4o-mini powers the AI Agent                                   |
| Sticky Note6             | Sticky Note                     | Notes on Switch node                             | None                         | None                         | Switch node: Routes workflow based on AI classification results                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a “Manual Trigger” node named "When clicking ‘Execute workflow’".  
   - No special configuration needed.

2. **Add Google Drive “Search files and folders” Node:**  
   - Set resource to `fileFolder`.  
   - Operation: Search for files in folder.  
   - Configure Folder ID to `1CcHp9tPoC-f7-umMjLdUU-Y7GedKLQkx` ("Indore" folder).  
   - Enable "Return All" to fetch all files.  
   - Use Google Drive OAuth2 credentials.

3. **Connect Manual Trigger → Search files and folders.**

4. **Add Google Drive “Download File” Node:**  
   - Operation: Download.  
   - Set `fileId` parameter to `={{ $json.id }}` (dynamic from search results).  
   - Use Google Drive OAuth2 credentials.

5. **Connect Search files and folders → Download File.**

6. **Add “Extract from File” Node:**  
   - Operation: PDF extraction.  
   - Extract text from downloaded PDFs.

7. **Connect Download File → Extract from File.**

8. **Add “Split in Batches” Node named “Loop Over Items”:**  
   - Default settings (process one item at a time).

9. **Connect Extract from File → Loop Over Items.**

10. **Add LangChain AI Agent Node named “AI Agent”:**  
    - Set parameter `text` to `={{ $json.text }}` (extracted text).  
    - Define system prompt:  
      ```
      You are a smart document classifier.

      Your task is to analyze the text content of an invoice and classify it into one of the following industries:
      - Retail
      - Manufacturing
      - EdTech

      Respond with the industry name at top: "Retail", "Manufacturing", or "EdTech".
      ```  
    - Set prompt type to "define".  
    - Connect AI language model to next node.

11. **Add LangChain Azure OpenAI Chat Model Node:**  
    - Model: `gpt-4o-mini`.  
    - No extra options required.  
    - Use Azure OpenAI API credentials.

12. **Connect Azure OpenAI Chat Model → AI Agent (as AI language model).**

13. **Connect Loop Over Items (second output) → AI Agent.**

14. **Add a “Switch” Node:**  
    - Set rules based on `$json.output` field:  
      - Equals "Retail"  
      - Equals "Manufacturing"  
      - Equals "EdTech"  
    - Case sensitive, strict type validation.

15. **Connect AI Agent → Switch.**

16. **Add three Google Drive nodes (one for each industry):**  
    - Operation: Move file.  
    - Set `fileId` to `={{ $('Download File').item.json.id }}` (dynamic from download node).  
    - Drive ID: “My Drive”.  
    - Folder IDs:  
      - Retail: `13YZe1eYMQGXqR62n8aQIlYeSHE2HqpAv`  
      - Manufacturing: `1-fmURc7hW1JcQCcP_lk4VxhkOk4GA00H`  
      - EdTech: `1H-JpO6eNDYGHT4G92-mB846NvaNu8P-1`  
    - Use Google Drive OAuth2 credentials.

17. **Connect Switch outputs to corresponding Google Drive move nodes:**  
    - Retail branch → Retail node  
    - Manufacturing branch → Manufacturing node  
    - EdTech branch → EdTech node

18. **Connect each Google Drive move node back to “Loop Over Items” node:**  
    - This allows processing of next file in batch.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                          | Context or Link                                                                                             |
|-------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| The workflow uses OAuth2 credentials for Google Drive and Azure OpenAI API, which must be configured in n8n beforehand.                              | Credential setup in n8n for Google Drive and Azure OpenAI APIs                                             |
| The AI Agent prompt is critical to accurate classification; it instructs the model to respond exactly with one of three industry names.               | Prompt text inside "AI Agent" node                                                                          |
| Folder IDs for Google Drive are hardcoded; ensure these folders exist and credentials have permissions to move files into them.                      | Folder IDs: Retail, Manufacturing, EdTech                                                                  |
| Batch processing via “Split in Batches” ensures sequential processing, avoiding concurrency issues with API calls and file operations.                | Loop Over Items node                                                                                        |
| For more information on LangChain integration in n8n, visit https://docs.n8n.io/integrations/nodes/ai/agent/                                         | Official LangChain AI Agent documentation                                                                  |
| GPT-4o-mini is an Azure OpenAI model optimized for smaller tasks; ensure your Azure OpenAI subscription supports this model and has sufficient quota. | Azure OpenAI GPT-4o-mini model                                                                              |

---

**Disclaimer:**  
The text provided is solely from an automated workflow created with n8n, an integration and automation tool. This treatment strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.