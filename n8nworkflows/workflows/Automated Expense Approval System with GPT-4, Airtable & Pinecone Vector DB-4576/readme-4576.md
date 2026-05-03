Automated Expense Approval System with GPT-4, Airtable & Pinecone Vector DB

https://n8nworkflows.xyz/workflows/automated-expense-approval-system-with-gpt-4--airtable---pinecone-vector-db-4576


# Automated Expense Approval System with GPT-4, Airtable & Pinecone Vector DB

---

### 1. Workflow Overview

This workflow, titled **"Automated Expense Approval System with GPT-4, Airtable & Pinecone Vector DB"**, automates the monitoring, intelligent review, auditing, and updating of expense requests submitted via Airtable. It is designed for finance teams, particularly CFOs, to streamline expense approvals through AI-driven reasoning, providing explainable decisions while maintaining an auditable history of reviews.

The workflow is logically divided into four main blocks:

- **1.1 Intake – Monitoring Expense Requests**  
  Continuously watches Airtable for new or updated expense entries requiring review.

- **1.2 AI Analysis – CFO Reasoning Engine**  
  Utilizes an AI agent powered by OpenAI GPT-4 to evaluate each expense, flagging suspicious ones with detailed reasoning.

- **1.3 Audit Trail – Embedding & Storage**  
  Processes the AI-generated reasoning text by splitting, embedding it, and storing it in Pinecone Vector DB for searchable, long-term compliance and insights.

- **1.4 Output – Updating Records**  
  Updates the original Airtable expense record with the AI’s decision and reasoning, ensuring the source of truth reflects the latest status.

---

### 2. Block-by-Block Analysis

#### 2.1 Intake – Monitoring Expense Requests

**Overview:**  
This block detects new or modified expense requests in Airtable that have a `Status` field indicating they are pending review. It triggers the entire workflow for further processing.

**Nodes Involved:**  
- Watch New Expense Requests

**Node Details:**

- **Watch New Expense Requests**  
  - **Type:** Airtable Trigger  
  - **Role:** Monitors the `Expenses` table in Airtable for new or updated records based on the `Amount` field and `Status = Pending`.  
  - **Configuration:**  
    - Base and Table referenced via URLs to the Airtable app and table views.  
    - Polling set to trigger every hour.  
    - Authentication via Airtable Personal Access Token credential.  
  - **Inputs:** None (trigger node)  
  - **Outputs:** Emits new or updated expense records for downstream nodes.  
  - **Edge Cases:**  
    - Airtable API rate limits or token expiration may cause missed triggers.  
    - Network timeouts during polling.  
  - **Sticky Notes:** Section 1 notes highlight its role ensuring real-time detection.

---

#### 2.2 AI Analysis – CFO Reasoning Engine

**Overview:**  
This block uses a GPT-4-powered AI agent to analyze expense details, decide on approval or flagging, and provide an explainable rationale. The output is parsed into structured JSON for further use.

**Nodes Involved:**  
- CFO Expense Review Agent  
- OpenAI GPT-4 Model  
- Parse CFO Agent Response

**Node Details:**

- **CFO Expense Review Agent**  
  - **Type:** Langchain Agent (AI Agent with tool integration)  
  - **Role:** Acts as a virtual CFO, receiving expense data and generating a textual reasoning output.  
  - **Configuration:**  
    - Input prompt dynamically composed including Amount, Submitted By, Category, Description, Date Submitted, and Status from the Airtable record.  
    - System message sets the agent’s persona as "CFO expense analysis agent" with instructions to flag suspicious expenses with reasons.  
    - Output parsing enabled for structured response.  
  - **Inputs:** Receives records from the Airtable trigger.  
  - **Outputs:** Textual reasoning passed to the OpenAI GPT-4 Model node and then parsed.  
  - **Edge Cases:**  
    - Expression failures if input fields are missing or malformed.  
    - AI model timeouts or API quota limits.  
  - **Sub-workflow:** Uses OpenAI GPT-4 and output parser nodes as tools.

- **OpenAI GPT-4 Model**  
  - **Type:** Langchain OpenAI Chat Model  
  - **Role:** Provides the advanced natural language reasoning capabilities.  
  - **Configuration:**  
    - Model set to `gpt-4o-mini` for cost-effective GPT-4 performance.  
    - Uses the prompt from CFO Expense Review Agent.  
    - Authenticated via OpenAI API credentials.  
  - **Inputs:** Prompt text from CFO Expense Review Agent.  
  - **Outputs:** Raw AI completion text to be parsed.  
  - **Edge Cases:**  
    - OpenAI API key quota exhaustion.  
    - Model response latency or failures.

- **Parse CFO Agent Response**  
  - **Type:** Langchain Structured Output Parser  
  - **Role:** Converts unstructured AI text into JSON with well-defined fields such as `amount`, `submitted_by`, `category`, `description`, `date_submitted`, `status`, `decision`, and `reason`.  
  - **Configuration:**  
    - JSON schema example provided to guide parsing accuracy.  
  - **Inputs:** Text from OpenAI GPT-4 Model.  
  - **Outputs:** Structured JSON for downstream processing.  
  - **Edge Cases:**  
    - Parsing errors if AI output does not conform to schema.  
    - Missing fields or inconsistent data.

- **Sticky Notes:** Section 2 details the AI reasoning engine and its explainability.

---

#### 2.3 Audit Trail – Embedding & Storage

**Overview:**  
This block creates a persistent audit trail by converting the textual reasoning into embeddings, chunking large text if necessary, and storing vectors with metadata in Pinecone for future search and compliance checks.

**Nodes Involved:**  
- Split Reasoning Text  
- Prepare Data for Pinecone  
- Generate Embeddings  
- Store Decision in Pinecone

**Node Details:**

- **Split Reasoning Text**  
  - **Type:** Recursive Character Text Splitter  
  - **Role:** Splits long reasoning strings into chunks (100 characters with 20 overlap) to comply with embedding model limits.  
  - **Configuration:** Default options with specified chunk and overlap sizes.  
  - **Inputs:** Reasoning text from the Parse CFO Agent Response or CFO Expense Review Agent output.  
  - **Outputs:** Text chunks for embedding.  
  - **Edge Cases:**  
    - Incorrect chunk size may cause data loss or excessive splitting.  
    - Empty or null input text.

- **Prepare Data for Pinecone**  
  - **Type:** Document Default Data Loader  
  - **Role:** Loads the split text chunks into a format suitable for vector embedding.  
  - **Configuration:** Defaults used; expects text input.  
  - **Inputs:** Chunks from Split Reasoning Text.  
  - **Outputs:** Documents ready for embedding.  
  - **Edge Cases:**  
    - Unexpected document format may cause failures.

- **Generate Embeddings**  
  - **Type:** OpenAI Embeddings  
  - **Role:** Transforms text documents into vector embeddings using OpenAI embedding models.  
  - **Configuration:** Default embedding model and options.  
  - **Inputs:** Documents from Prepare Data for Pinecone.  
  - **Outputs:** Vector embeddings.  
  - **Credentials:** OpenAI API credentials.  
  - **Edge Cases:**  
    - API limits or rate throttling.

- **Store Decision in Pinecone**  
  - **Type:** Pinecone Vector Store Node  
  - **Role:** Inserts vector embeddings into the Pinecone index with metadata such as decision status, amount, and reason stored in the namespace.  
  - **Configuration:**  
    - Mode set to "insert".  
    - Namespace dynamically set to the decision value (e.g., "Flagged" or "Approved").  
    - Pinecone index selected from credentials.  
  - **Inputs:** Embeddings and metadata from previous nodes, also directly connected from CFO Expense Review Agent for metadata.  
  - **Outputs:** Triggers next node for Airtable update.  
  - **Credentials:** Pinecone API credentials.  
  - **Edge Cases:**  
    - Pinecone index limits or network errors.  
    - Namespace misconfiguration causing data siloing.

- **Sticky Notes:** Section 3 explains the creation of a scalable memory system for auditability and compliance.

---

#### 2.4 Output – Updating Records

**Overview:**  
Finalizes the workflow by updating the original Airtable record with the AI agent’s decision and reasoning, marking the expense as reviewed.

**Nodes Involved:**  
- Update Airtable Record

**Node Details:**

- **Update Airtable Record**  
  - **Type:** Airtable Update Node  
  - **Role:** Updates fields in the Airtable `Expenses` table with the AI’s decision (`Approved` or `Flagged`), detailed reasoning, status set to `completed`, and other expense details.  
  - **Configuration:**  
    - Base and Table IDs set to match the input Airtable.  
    - Updates columns: Amount, Reason, Status (to "completed"), Category, decision, Description, Submitted By.  
    - Record ID matched dynamically from the trigger node.  
    - Mapping configured in "defineBelow" mode for explicit field assignments.  
    - Authentication via Airtable Personal Access Token.  
  - **Inputs:** Structured JSON from CFO Expense Review Agent output / Parse CFO Agent Response.  
  - **Outputs:** End of workflow.  
  - **Edge Cases:**  
    - Airtable API errors or permission issues.  
    - Record ID mismatches or missing fields.

- **Sticky Notes:** Section 4 highlights maintaining Airtable as the source of truth.

---

### 3. Summary Table

| Node Name                 | Node Type                          | Functional Role                       | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                           |
|---------------------------|----------------------------------|------------------------------------|-------------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------|
| Watch New Expense Requests | Airtable Trigger                 | Detect new/updated expenses         | None                          | CFO Expense Review Agent       | Ensures real-time detection of incoming requests needing analysis (Section 1)                       |
| CFO Expense Review Agent   | Langchain Agent                  | AI reasoning on expense data        | Watch New Expense Requests     | Store Decision in Pinecone, OpenAI GPT-4 Model | Acts as virtual CFO, flags suspicious expenses, provides detailed reasoning (Section 2)             |
| OpenAI GPT-4 Model        | Langchain OpenAI Chat Model      | Provides GPT-4 powered reasoning    | CFO Expense Review Agent       | Parse CFO Agent Response       | Powers CFO decision-making with structured prompt (Section 2)                                      |
| Parse CFO Agent Response   | Langchain Output Parser           | Parses AI response to JSON          | OpenAI GPT-4 Model            | CFO Expense Review Agent       | Converts unstructured GPT output to structured JSON fields (Section 2)                             |
| Store Decision in Pinecone | Pinecone Vector Store Node        | Store embedding & metadata          | CFO Expense Review Agent, Prepare Data for Pinecone | Update Airtable Record         | Saves vector and metadata, creating searchable audit trail (Section 3)                             |
| Generate Embeddings        | OpenAI Embeddings                 | Create vector embeddings             | Prepare Data for Pinecone      | Store Decision in Pinecone     | Converts text to vectors for Pinecone storage (Section 3)                                          |
| Prepare Data for Pinecone  | Document Default Data Loader      | Prepare text data for embedding     | Split Reasoning Text           | Generate Embeddings            | Loads reasoning text for embedding (Section 3)                                                     |
| Split Reasoning Text       | Recursive Character Text Splitter| Split long text into chunks         | Parse CFO Agent Response       | Prepare Data for Pinecone      | Ensures text fits embedding model input size (Section 3)                                          |
| Update Airtable Record     | Airtable Update                  | Update expense record with decision | Store Decision in Pinecone     | None                          | Writes final decision & reasoning back to Airtable, marking status completed (Section 4)           |
| Sticky Note                | Sticky Note                      | Documentation                      | None                          | None                          | Section 1 notes                                                                                      |
| Sticky Note1               | Sticky Note                      | Documentation                      | None                          | None                          | Section 2 notes                                                                                      |
| Sticky Note2               | Sticky Note                      | Documentation                      | None                          | None                          | Section 3 notes                                                                                      |
| Sticky Note3               | Sticky Note                      | Documentation                      | None                          | None                          | Section 4 notes                                                                                      |
| Sticky Note4               | Sticky Note                      | Documentation                      | None                          | None                          | Full workflow overview                                                                              |
| Sticky Note9               | Sticky Note                      | Support contact and resources      | None                          | None                          | Contact info and tutorial links                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Airtable Trigger Node**  
   - Type: Airtable Trigger  
   - Configure to monitor your Airtable base and table for expense records, filtering on `Status = Pending` or changes to `Amount`.  
   - Set polling frequency to every hour or as preferred.  
   - Authenticate with Airtable Personal Access Token.

2. **Add Langchain Agent Node (CFO Expense Review Agent)**  
   - Type: Langchain Agent  
   - Set prompt text to dynamically include expense fields: `Amount`, `Submitted By`, `Category`, `Description`, `Date Submitted`, `Status`.  
   - Include system message instructing agent to behave as CFO and flag suspicious expenses with reasoning.  
   - Enable output parser to receive structured JSON.  
   - Connect output from Airtable Trigger to this node.

3. **Add OpenAI GPT-4 Chat Model Node**  
   - Type: Langchain OpenAI Chat Model  
   - Select GPT-4 variant (`gpt-4o-mini`) for cost-effective processing.  
   - Authenticate with OpenAI API credentials.  
   - Connect CFO Expense Review Agent’s AI language model output to this node.

4. **Add Langchain Output Parser Node**  
   - Type: Structured Output Parser  
   - Provide an example JSON schema defining fields expected from AI output (`amount`, `submitted_by`, `category`, `description`, `date_submitted`, `status`, `decision`, `reason`).  
   - Connect output of OpenAI GPT-4 Model node to this parser.

5. **Add Recursive Character Text Splitter Node**  
   - Type: Text Splitter  
   - Configure chunk size to 100 characters, overlap 20 characters.  
   - Connect parsed AI response’s reasoning text output to this node.

6. **Add Document Default Data Loader Node**  
   - Type: Document Loader  
   - Use defaults.  
   - Connect output of Split Reasoning Text node to this node.

7. **Add OpenAI Embeddings Node**  
   - Type: Embeddings  
   - Authenticate with OpenAI API credentials.  
   - Connect Document Loader output to this node.

8. **Add Pinecone Vector Store Node**  
   - Type: Pinecone Vector Store  
   - Configure to insert mode.  
   - Set Pinecone namespace dynamically based on parsed `decision` field (e.g., "Flagged" or "Approved").  
   - Authenticate with Pinecone API credentials.  
   - Connect:  
     - AI agent output (for metadata)  
     - Embeddings node output (for vectors)  
     - Document loader output (for documents)  
   - Connect this node’s main output to the Airtable update node next.

9. **Add Airtable Update Node**  
   - Type: Airtable Update  
   - Configure to update the original Airtable record identified by record ID from trigger node.  
   - Fields to update:  
     - `Status` → "completed"  
     - `Reason` → parsed AI reasoning  
     - `decision` → AI decision  
     - Additional fields: `Amount`, `Category`, `Description`, `Submitted By` (to keep data consistent)  
   - Authenticate with Airtable Personal Access Token.  
   - Connect Pinecone store node output to this node.

10. **Add Sticky Notes (Optional but Recommended)**  
    - Add notes to document each section and node purpose for clarity and maintenance.

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                     |
|------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------|
| Workflow automates expense approval with AI-powered reasoning, audit trail, and Airtable integration.                              | Workflow purpose                                                  |
| Contact for support: Yaron@nofluff.online                                                                                         | Support contact                                                   |
| YouTube channel with tips and tutorials: https://www.youtube.com/@YaronBeen/videos                                                 | Video resources                                                  |
| LinkedIn profile of author: https://www.linkedin.com/in/yaronbeen/                                                                | Professional profile                                              |
| The workflow builds a scalable memory system using Pinecone for compliance and pattern recognition.                               | Audit trail / Pinecone integration                               |
| Ensure Airtable and API tokens are kept secure and have correct permissions for read/write operations.                            | Security and best practices                                      |
| OpenAI and Pinecone API quotas and rate limits may affect workflow reliability; monitor usage accordingly.                        | Operational considerations                                       |

---

**Disclaimer:** The provided content originates exclusively from an automated workflow created with n8n, fully compliant with content policies and legal data usage. No illegal, offensive, or protected information is included.

---