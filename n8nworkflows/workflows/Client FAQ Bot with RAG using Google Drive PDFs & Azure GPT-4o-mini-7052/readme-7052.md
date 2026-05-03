Client FAQ Bot with RAG using Google Drive PDFs & Azure GPT-4o-mini

https://n8nworkflows.xyz/workflows/client-faq-bot-with-rag-using-google-drive-pdfs---azure-gpt-4o-mini-7052


# Client FAQ Bot with RAG using Google Drive PDFs & Azure GPT-4o-mini

### 1. Workflow Overview

This workflow implements a **Client FAQ Bot** that uses **Retrieval-Augmented Generation (RAG)** to answer client queries by referencing PDF documents stored on Google Drive. It leverages **Azure GPT-4o-mini** as the language model to generate context-aware responses. The workflow is designed to:

- Receive client questions via webhook
- Search Google Drive for relevant PDF files
- Download and extract content from these PDFs
- Use the extracted information as context for the Azure OpenAI GPT-4o-mini model
- Generate and return an informed answer to the client

The workflow is logically divided into these blocks:

- **1.1 Input Reception and Preprocessing:** Receives the client question and prepares search parameters.
- **1.2 Google Drive Search and Download:** Searches Google Drive for relevant files and downloads them.
- **1.3 File Content Extraction and Preparation:** Extracts text from downloaded PDF files and formats the content.
- **1.4 AI Processing with Azure GPT-4o-mini:** Runs the extracted content through the Azure GPT-4o-mini model using a Langchain-based LLM chain.
- **1.5 Response Delivery:** Returns the answer through the webhook response.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Preprocessing

- **Overview:**  
  This block receives the user’s query via webhook, then sets up the necessary fields to perform a Google Drive search.

- **Nodes Involved:**  
  - Webhook  
  - Edit Fields

- **Node Details:**

  - **Webhook**  
    - Type: Webhook (Entry Point)  
    - Role: Receives incoming HTTP requests containing client questions.  
    - Configuration: Uses a fixed webhook ID, listens for any HTTP method by default.  
    - Inputs: External HTTP requests  
    - Outputs: Passes data to Edit Fields node  
    - Failure Modes: Connection issues, invalid requests, missing expected data fields.

  - **Edit Fields**  
    - Type: Set  
    - Role: Modifies and prepares data fields for searching files in Google Drive.  
    - Configuration: Likely sets parameters such as search query or folder ID based on webhook input.  
    - Inputs: Data from Webhook  
    - Outputs: Pass data to Google Drive search node  
    - Failure Modes: Expression errors if expected variables from webhook are missing.

#### 1.2 Google Drive Search and Download

- **Overview:**  
  This block searches Google Drive for matching PDF files and downloads them for further processing.

- **Nodes Involved:**  
  - Search files and folders (Google Drive)  
  - Download File (Google Drive)  
  - Loop Over Items (SplitInBatches)

- **Node Details:**

  - **Search files and folders**  
    - Type: Google Drive  
    - Role: Searches Google Drive for files/folders using parameters set earlier.  
    - Configuration: Uses search criteria likely derived from the client query or fixed folder path.  
    - Inputs: From Edit Fields  
    - Outputs: List of files to Download File node  
    - Failure Modes: Auth errors, API limits, empty search results.

  - **Download File**  
    - Type: Google Drive  
    - Role: Downloads each file found in the search step.  
    - Configuration: Uses file IDs from the search node to fetch file content.  
    - Inputs: List of files from Search files and folders  
    - Outputs: Feeds SplitInBatches node for iterative processing  
    - Failure Modes: File not found, permission denied, download errors.

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Iterates over downloaded files one by one for extraction.  
    - Configuration: Batch size likely set to 1 to process files individually.  
    - Inputs: Downloaded files from Download File  
    - Outputs: Passes individual files to Extract from File node  
    - Failure Modes: Batch processing errors, empty batches.

#### 1.3 File Content Extraction and Preparation

- **Overview:**  
  Extracts text content from each downloaded PDF file and refines fields for AI processing.

- **Nodes Involved:**  
  - Extract from File  
  - Edit Fields1

- **Node Details:**

  - **Extract from File**  
    - Type: ExtractFromFile  
    - Role: Extracts text and metadata from the PDF files.  
    - Configuration: Uses default or configured extraction method suitable for PDF format.  
    - Inputs: Single file from Loop Over Items  
    - Outputs: Text content to Edit Fields1  
    - Failure Modes: Extraction failures due to file corruption, unsupported formats.

  - **Edit Fields1**  
    - Type: Set  
    - Role: Adjusts extracted content fields to match AI input requirements.  
    - Configuration: Sets or renames fields such as prompt or context for LLM chain.  
    - Inputs: Extracted content  
    - Outputs: Passes data to Basic LLM Chain node  
    - Failure Modes: Expression errors, missing content fields.

#### 1.4 AI Processing with Azure GPT-4o-mini

- **Overview:**  
  Processes the prepared text using Azure GPT-4o-mini via a Langchain LLM chain to generate a contextual answer.

- **Nodes Involved:**  
  - Azure OpenAI Chat Model  
  - Basic LLM Chain

- **Node Details:**

  - **Azure OpenAI Chat Model**  
    - Type: Langchain LM Chat (Azure OpenAI)  
    - Role: Interacts with Azure OpenAI GPT-4o-mini model for chat completions.  
    - Configuration: Uses configured Azure OpenAI credentials, model name GPT-4o-mini, temperature, max tokens.  
    - Inputs: Receives prompt/context from Basic LLM Chain as ai_languageModel input  
    - Outputs: Returns generated text to Basic LLM Chain  
    - Failure Modes: Auth errors, rate limits, API timeouts.

  - **Basic LLM Chain**  
    - Type: Langchain Chain LLM  
    - Role: Chains prompt formatting and calls Azure OpenAI Chat Model, structures AI interaction.  
    - Configuration: Defines prompt template, input variables, and output format.  
    - Inputs: Edited fields from Edit Fields1; ai_languageModel input from Azure OpenAI Chat Model  
    - Outputs: Final answer to Return Answer node  
    - Failure Modes: Expression/template errors, API failures.

#### 1.5 Response Delivery

- **Overview:**  
  Sends the final generated answer back to the client via webhook response.

- **Nodes Involved:**  
  - Return Answer

- **Node Details:**

  - **Return Answer**  
    - Type: Respond To Webhook  
    - Role: Sends HTTP response with the generated answer to client.  
    - Configuration: Sends back JSON or text response containing AI answer.  
    - Inputs: From Basic LLM Chain  
    - Outputs: HTTP response to original client request  
    - Failure Modes: Network errors, response timeouts.

---

### 3. Summary Table

| Node Name             | Node Type                          | Functional Role                      | Input Node(s)            | Output Node(s)          | Sticky Note |
|-----------------------|----------------------------------|------------------------------------|--------------------------|-------------------------|-------------|
| Webhook               | Webhook                          | Receives client question            | —                        | Edit Fields             |             |
| Edit Fields           | Set                              | Prepares search parameters          | Webhook                  | Search files and folders|             |
| Search files and folders | Google Drive                    | Searches for relevant files         | Edit Fields              | Download File           |             |
| Download File         | Google Drive                     | Downloads files from Drive          | Search files and folders | Loop Over Items         |             |
| Loop Over Items       | SplitInBatches                   | Iterates over downloaded files      | Download File            | Extract from File       |             |
| Extract from File     | ExtractFromFile                  | Extracts text from PDFs              | Loop Over Items          | Edit Fields1            |             |
| Edit Fields1          | Set                              | Prepares extracted content for AI   | Extract from File        | Basic LLM Chain         |             |
| Azure OpenAI Chat Model | Langchain LM Chat (Azure OpenAI)| Calls Azure GPT-4o-mini model        | Basic LLM Chain (ai_languageModel input) | Basic LLM Chain |             |
| Basic LLM Chain       | Langchain Chain LLM              | Formats prompt and processes AI     | Edit Fields1, Azure OpenAI Chat Model | Return Answer |         |
| Return Answer         | Respond To Webhook               | Returns AI-generated answer         | Basic LLM Chain          | —                       |             |
| Sticky Note           | Sticky Note                     | Visual notes                       | —                        | —                       |             |
| Sticky Note1          | Sticky Note                     | Visual notes                       | —                        | —                       |             |
| Sticky Note2          | Sticky Note                     | Visual notes                       | —                        | —                       |             |
| Sticky Note3          | Sticky Note                     | Visual notes                       | —                        | —                       |             |
| Sticky Note4          | Sticky Note                     | Visual notes                       | —                        | —                       |             |
| Sticky Note5          | Sticky Note                     | Visual notes                       | —                        | —                       |             |
| Sticky Note6          | Sticky Note                     | Visual notes                       | —                        | —                       |             |
| Sticky Note7          | Sticky Note                     | Visual notes                       | —                        | —                       |             |
| Sticky Note8          | Sticky Note                     | Visual notes                       | —                        | —                       |             |
| Sticky Note9          | Sticky Note                     | Visual notes                       | —                        | —                       |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Configure: Use default settings or assign a unique webhook path.  
   - Purpose: Receives incoming client questions (e.g., JSON payload with question text).

2. **Add Set Node "Edit Fields"**  
   - Type: Set  
   - Configuration: Prepare search parameters for Google Drive search based on webhook input (e.g., set query string or folder ID).  
   - Connect: Webhook → Edit Fields.

3. **Add Google Drive Node "Search files and folders"**  
   - Type: Google Drive (Search Files and Folders)  
   - Credentials: Configure Google Drive OAuth2 credentials.  
   - Parameters: Set search query or folder path to locate relevant PDFs.  
   - Connect: Edit Fields → Search files and folders.

4. **Add Google Drive Node "Download File"**  
   - Type: Google Drive (Download File)  
   - Credentials: Use same Google Drive credentials.  
   - Parameters: Input file IDs from previous node.  
   - Connect: Search files and folders → Download File.

5. **Add SplitInBatches Node "Loop Over Items"**  
   - Type: SplitInBatches  
   - Parameters: Batch Size = 1 (to process files individually).  
   - Connect: Download File → Loop Over Items.

6. **Add ExtractFromFile Node "Extract from File"**  
   - Type: ExtractFromFile  
   - Parameters: Extract text content from PDFs.  
   - Connect: Loop Over Items (first output) → Extract from File.

7. **Add Set Node "Edit Fields1"**  
   - Type: Set  
   - Configuration: Adjust fields of extracted content to prepare prompt/context for AI.  
   - Connect: Extract from File → Edit Fields1.

8. **Add Langchain LM Chat Node "Azure OpenAI Chat Model"**  
   - Type: Langchain LM Chat (Azure OpenAI)  
   - Credentials: Setup Azure OpenAI credentials with GPT-4o-mini model access.  
   - Parameters: Define model name, temperature, max tokens, and other relevant parameters.  
   - Connect: This node does not have direct input from previous nodes; it will be linked via ai_languageModel input in the next node.

9. **Add Langchain Chain LLM Node "Basic LLM Chain"**  
   - Type: Langchain Chain LLM  
   - Configuration: Set prompt template that uses extracted content as context to answer the client question.  
   - Inputs: Connect Edit Fields1 → Basic LLM Chain (main input).  
   - Connect Azure OpenAI Chat Model → Basic LLM Chain (ai_languageModel input).  
   - Connect: Basic LLM Chain → Return Answer.

10. **Add Respond to Webhook Node "Return Answer"**  
    - Type: Respond To Webhook  
    - Configuration: Return the AI-generated answer as JSON or text to the client.  
    - Connect: Basic LLM Chain → Return Answer.

11. **Credentials Setup**  
    - Google Drive OAuth2 credentials needed for Search and Download nodes.  
    - Azure OpenAI credentials (API key, endpoint, deployment name for GPT-4o-mini) needed for Langchain LM Chat node.

12. **Testing and Validation**  
    - Test webhook reception with sample client questions.  
    - Verify Google Drive search and file download.  
    - Confirm text extraction is accurate.  
    - Validate AI response quality and response delivery.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                          |
|-----------------------------------------------------------------------------------------------|----------------------------------------|
| This workflow uses Langchain nodes to integrate Azure OpenAI GPT-4o-mini model effectively.   | Langchain docs: https://js.langchain.com/docs/ |
| Google Drive OAuth2 credentials must have access to target folders containing PDFs.            | Google Drive API: https://developers.google.com/drive/api/v3/about-auth |
| Ensure Azure OpenAI resource is deployed with GPT-4o-mini model for compatibility.             | Azure OpenAI Service: https://learn.microsoft.com/en-us/azure/cognitive-services/openai/ |
| The modular design allows substitution of AI models or document stores with minimal changes.   |                                        |
| Recommended to handle API rate limits and errors gracefully in production workflows.           |                                        |

---

**Disclaimer:**  
The content provided originates solely from an n8n automated workflow. It strictly adheres to content policies and contains no illegal or offensive material. All data processed is legal and publicly accessible.