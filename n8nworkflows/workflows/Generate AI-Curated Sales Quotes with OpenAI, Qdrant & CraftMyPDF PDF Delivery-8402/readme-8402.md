Generate AI-Curated Sales Quotes with OpenAI, Qdrant & CraftMyPDF PDF Delivery

https://n8nworkflows.xyz/workflows/generate-ai-curated-sales-quotes-with-openai--qdrant---craftmypdf-pdf-delivery-8402


# Generate AI-Curated Sales Quotes with OpenAI, Qdrant & CraftMyPDF PDF Delivery

### 1. Workflow Overview

This workflow automates the generation and delivery of AI-curated sales quotes based on user requests. It integrates form input reception, contextual AI processing using OpenAI and Google Gemini language models, vector similarity search with Qdrant for product data retrieval, and PDF generation with CraftMyPDF, culminating in email delivery of the personalized sales quote.

Logical blocks:

- **1.1 Input Reception:** Captures user input through a form trigger.
- **1.2 Context Preparation:** Prepares and sets the context for the AI agent based on form data.
- **1.3 AI Processing:** Uses OpenAI and Google Gemini chat models to generate a sales quote agent and retrieve relevant product information by querying a Qdrant vector store with OpenAI embeddings.
- **1.4 PDF Generation:** Maps AI-generated data to CraftMyPDF format and creates a PDF document.
- **1.5 Delivery:** Downloads the PDF and sends it via email to the requester.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block captures the initial user request for a quote via a web form.
- **Nodes Involved:**  
  - Form: Request a Quote
- **Node Details:**

  - **Form: Request a Quote**  
    - Type: Form Trigger  
    - Role: Entry point receiving quote requests via a webhook form submission.  
    - Configuration: Activated on webhook ID "FORM-QUOTE-ID". No additional parameters configured.  
    - Inputs: External HTTP form submission.  
    - Outputs: Passes form data downstream.  
    - Edge Cases: Missing or malformed form data, webhook timeout or misconfiguration.  
    - Version: 2.2  

#### 2.2 Context Preparation

- **Overview:** Prepares the AI agent's context based on the incoming form data, setting necessary variables or metadata for downstream AI nodes.
- **Nodes Involved:**  
  - Set: Prepare Agent Context
- **Node Details:**

  - **Set: Prepare Agent Context**  
    - Type: Set  
    - Role: Defines or manipulates data fields required by the sales quote agent.  
    - Configuration: No explicit parameters shown; likely sets variables from form data for AI context.  
    - Inputs: From Form: Request a Quote node.  
    - Outputs: Passes enriched data to AI agent node.  
    - Edge Cases: Missing expected form fields causing context set failure.  
    - Version: 3.4  

#### 2.3 AI Processing

- **Overview:** Handles AI-driven quote generation, product data retrieval, and context-aware responses using multiple AI components and vector search.
- **Nodes Involved:**  
  - Sales Quote Agent  
  - OpenAI Chat Model (Agent)  
  - Google Gemini Chat Model  
  - PRODUCTS_QDRANT  
  - Qdrant Vector Store (Products)  
  - Embeddings OpenAI (Products)  

- **Node Details:**

  - **Sales Quote Agent**  
    - Type: LangChain Agent  
    - Role: Core AI agent orchestrating the sales quote generation using multiple AI tools and vector store.  
    - Configuration: Uses AI language models and vector store tools connected via ai_languageModel and ai_tool inputs.  
    - Inputs: Receives context from Set: Prepare Agent Context, AI language models, and vector store tools.  
    - Outputs: Produces structured data for PDF mapping.  
    - Edge Cases: AI model downtime, timeout errors, malformed AI responses.  
    - Version: 1.8  

  - **OpenAI Chat Model (Agent)**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Provides conversational AI capabilities to the Sales Quote Agent.  
    - Configuration: Uses OpenAI API with configured credentials (not shown).  
    - Inputs: Connected as ai_languageModel to Sales Quote Agent.  
    - Outputs: AI-generated text responses.  
    - Edge Cases: API rate limits, authentication failures, network issues.  
    - Version: 1.2  

  - **Google Gemini Chat Model**  
    - Type: LangChain Google Gemini Chat Model  
    - Role: Alternative conversational AI model feeding into the vector store query node.  
    - Configuration: Connected to PRODUCTS_QDRANT as ai_languageModel.  
    - Inputs: Receives data from upstream nodes (not explicitly shown).  
    - Outputs: Enhances vector store queries.  
    - Edge Cases: API access restrictions, credential expiry.  
    - Version: 1  

  - **PRODUCTS_QDRANT**  
    - Type: LangChain Vector Store Tool  
    - Role: Tool interfacing with the Qdrant vector store for product-related queries.  
    - Configuration: Receives embeddings and vector store inputs.  
    - Inputs: ai_languageModel from Google Gemini Chat Model, ai_vectorStore from Qdrant Vector Store (Products).  
    - Outputs: Connected as ai_tool to Sales Quote Agent.  
    - Edge Cases: Vector store connectivity issues, query failures.  
    - Version: 1  

  - **Qdrant Vector Store (Products)**  
    - Type: LangChain Qdrant Vector Store  
    - Role: Stores and retrieves product embeddings for similarity search.  
    - Configuration: Receives embeddings from Embeddings OpenAI (Products).  
    - Inputs: ai_embedding from Embeddings OpenAI (Products).  
    - Outputs: ai_vectorStore to PRODUCTS_QDRANT.  
    - Edge Cases: Vector store service downtime, malformed embeddings.  
    - Version: 1  

  - **Embeddings OpenAI (Products)**  
    - Type: LangChain OpenAI Embeddings  
    - Role: Generates vector embeddings for product data to enable semantic search.  
    - Configuration: Connects to Qdrant Vector Store (Products).  
    - Inputs: Product data or queries.  
    - Outputs: ai_embedding to Qdrant Vector Store (Products).  
    - Edge Cases: API limits, embedding generation errors.  
    - Version: 1.1  

#### 2.4 PDF Generation

- **Overview:** Converts the AI agent’s output into a PDF-ready format and generates the PDF document using CraftMyPDF.
- **Nodes Involved:**  
  - Code: Map for CraftMyPDF  
  - Create a PDF  

- **Node Details:**

  - **Code: Map for CraftMyPDF**  
    - Type: Code  
    - Role: Transforms structured AI output into the JSON format required by CraftMyPDF.  
    - Configuration: Custom JavaScript code (not shown specifically) that maps data fields to PDF template variables.  
    - Inputs: Receives output from Sales Quote Agent.  
    - Outputs: Prepared data for PDF creation.  
    - Edge Cases: Code execution errors, unexpected AI output format.  
    - Version: 2  

  - **Create a PDF**  
    - Type: CraftMyPDF node  
    - Role: Generates a PDF document from the mapped data using a predefined CraftMyPDF template.  
    - Configuration: Template ID and variable bindings (not explicitly shown) must be set in parameters.  
    - Inputs: Data from Code: Map for CraftMyPDF.  
    - Outputs: Returns a PDF file URL or ID.  
    - Edge Cases: Template errors, CraftMyPDF service downtime.  
    - Version: 1  

#### 2.5 Delivery

- **Overview:** Downloads the generated PDF and sends it as an email attachment to the requester.
- **Nodes Involved:**  
  - Get PDF file  
  - Email: Send Quote  

- **Node Details:**

  - **Get PDF file**  
    - Type: HTTP Request  
    - Role: Downloads the PDF file from CraftMyPDF’s output URL.  
    - Configuration: HTTP GET request to the PDF file URL returned by Create a PDF node.  
    - Inputs: PDF file URL from Create a PDF node.  
    - Outputs: Binary PDF data.  
    - Edge Cases: HTTP errors, broken links, timeout.  
    - Version: 4.2  

  - **Email: Send Quote**  
    - Type: Email Send  
    - Role: Sends the generated PDF quote via email to the intended recipient.  
    - Configuration: SMTP or email service credentials configured outside JSON; email recipient and content set dynamically, likely from form data.  
    - Inputs: Binary PDF data from Get PDF file node.  
    - Outputs: Confirmation of email sent.  
    - Edge Cases: Email delivery failures, SMTP authentication errors, attachment size limits.  
    - Version: 2.1  
    - Webhook ID: "8934d5d2-fe1f-4b79-9445-c794acf61d3d" (possibly for status callbacks)  

---

### 3. Summary Table

| Node Name                      | Node Type                           | Functional Role                  | Input Node(s)                  | Output Node(s)                | Sticky Note                                      |
|-------------------------------|-----------------------------------|--------------------------------|-------------------------------|------------------------------|-------------------------------------------------|
| Sticky Note                   | Sticky Note                       | Annotation                     |                               |                              |                                                 |
| Sticky Note1                  | Sticky Note                       | Annotation                     |                               |                              |                                                 |
| Sticky Note2                  | Sticky Note                       | Annotation                     |                               |                              |                                                 |
| Form: Request a Quote         | Form Trigger                     | Input Reception                |                               | Set: Prepare Agent Context    |                                                 |
| Set: Prepare Agent Context    | Set                              | Context Preparation            | Form: Request a Quote          | Sales Quote Agent            |                                                 |
| Sales Quote Agent             | LangChain Agent                  | AI Processing Core             | Set: Prepare Agent Context, OpenAI Chat Model (Agent), PRODUCTS_QDRANT | Code: Map for CraftMyPDF    |                                                 |
| OpenAI Chat Model (Agent)     | LangChain OpenAI Chat Model      | AI Language Model              |                               | Sales Quote Agent            |                                                 |
| Google Gemini Chat Model      | LangChain Google Gemini Chat Model | AI Language Model              |                               | PRODUCTS_QDRANT              |                                                 |
| PRODUCTS_QDRANT              | LangChain Vector Store Tool       | Vector Store Tool Integration  | Google Gemini Chat Model, Qdrant Vector Store (Products) | Sales Quote Agent          |                                                 |
| Qdrant Vector Store (Products)| LangChain Qdrant Vector Store    | Vector Store                  | Embeddings OpenAI (Products)  | PRODUCTS_QDRANT              |                                                 |
| Embeddings OpenAI (Products) | LangChain OpenAI Embeddings       | Embeddings Generation          |                               | Qdrant Vector Store (Products)|                                                 |
| Code: Map for CraftMyPDF      | Code                             | Data Mapping for PDF           | Sales Quote Agent             | Create a PDF                 |                                                 |
| Create a PDF                 | CraftMyPDF                       | PDF Generation                 | Code: Map for CraftMyPDF       | Get PDF file                 |                                                 |
| Get PDF file                 | HTTP Request                    | PDF Download                   | Create a PDF                  | Email: Send Quote            |                                                 |
| Email: Send Quote             | Email Send                      | PDF Delivery via Email         | Get PDF file                  |                              |                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node** named "Form: Request a Quote"  
   - Set webhook ID to "FORM-QUOTE-ID".  
   - No additional parameters required.  
   - This node receives user quote requests.

2. **Create a Set node** named "Set: Prepare Agent Context"  
   - Connect it to the Form: Request a Quote node.  
   - Configure to set variables needed by the AI agent from form input fields (e.g., customer info, requirements).  
   - No raw JSON available; ensure relevant fields are mapped.

3. **Create OpenAI Chat Model node** named "OpenAI Chat Model (Agent)"  
   - Configure with OpenAI API credentials (API key, organization if needed).  
   - Set model parameters as required (e.g., GPT-4 or GPT-3.5).  
   - No direct input connection; will link as ai_languageModel to Sales Quote Agent.

4. **Create Google Gemini Chat Model node** named "Google Gemini Chat Model"  
   - Configure with Google Gemini API credentials.  
   - No direct input connection; will link as ai_languageModel to PRODUCTS_QDRANT.

5. **Create Embeddings OpenAI node** named "Embeddings OpenAI (Products)"  
   - Configure OpenAI API credentials for embeddings generation.  
   - Input expected product data or search queries for embedding generation.

6. **Create Qdrant Vector Store node** named "Qdrant Vector Store (Products)"  
   - Configure connection to Qdrant instance (URL, API key).  
   - Connect ai_embedding input from Embeddings OpenAI (Products).

7. **Create Vector Store Tool node** named "PRODUCTS_QDRANT"  
   - Configure to use Qdrant vector store.  
   - Connect ai_languageModel input from Google Gemini Chat Model.  
   - Connect ai_vectorStore input from Qdrant Vector Store (Products).

8. **Create LangChain Agent node** named "Sales Quote Agent"  
   - Connect main input from Set: Prepare Agent Context.  
   - Connect ai_languageModel input from OpenAI Chat Model (Agent).  
   - Connect ai_tool input from PRODUCTS_QDRANT.  
   - This agent orchestrates AI processing and product data querying.

9. **Create Code node** named "Code: Map for CraftMyPDF"  
   - Connect main input from Sales Quote Agent.  
   - Write JavaScript code to transform AI output to CraftMyPDF JSON template format.  
   - Ensure output matches CraftMyPDF template requirements.

10. **Create CraftMyPDF node** named "Create a PDF"  
    - Connect main input from Code: Map for CraftMyPDF.  
    - Configure with CraftMyPDF API credentials.  
    - Select or create a PDF template for sales quotes.  
    - Map incoming JSON variables accordingly.

11. **Create HTTP Request node** named "Get PDF file"  
    - Connect main input from Create a PDF.  
    - Configure as GET request to download the PDF file URL returned by the previous node.  

12. **Create Email Send node** named "Email: Send Quote"  
    - Connect main input from Get PDF file.  
    - Configure SMTP or email service credentials.  
    - Set recipient email dynamically from the form data.  
    - Attach the downloaded PDF binary data.  
    - Configure email subject and body text appropriately.

13. **Connect all nodes** as per the sequence above.

14. **Test the workflow** by submitting a form request and verifying PDF generation and email delivery.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                             |
|-----------------------------------------------------------------------------------------------------|--------------------------------------------|
| This workflow leverages LangChain nodes for AI orchestration integrating multiple language models. | LangChain n8n nodes documentation          |
| CraftMyPDF node requires prior template creation and API credential setup.                           | https://craftmypdf.com                      |
| Ensure API keys for OpenAI and Google Gemini are valid and have sufficient quotas.                   | OpenAI & Google Cloud console               |
| Webhook IDs "FORM-QUOTE-ID" and email webhook "8934d5d2-fe1f-4b79-9445-c794acf61d3d" must be unique. | n8n webhook configuration                    |
| PDF delivery via email must respect attachment size limits of the SMTP provider.                     | SMTP provider documentation                  |

---

**Disclaimer:** The provided text stems exclusively from an automated n8n workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.