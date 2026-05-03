Intelligent Legal Document Review and Compliance Automation

https://n8nworkflows.xyz/workflows/intelligent-legal-document-review-and-compliance-automation-11861


# Intelligent Legal Document Review and Compliance Automation

### 1. Workflow Overview

This workflow automates the intelligent review and compliance checking of legal documents, specifically contracts. It is designed for legal teams, contract managers, and compliance officers who want to streamline the extraction, analysis, and risk assessment of contractual clauses, as well as generate alternative compliant wordings and comprehensive summary reports. The workflow encompasses the following logical blocks:

- **1.1 Input Reception and Configuration:** Receives uploaded legal documents via a webhook and sets up workflow parameters including compliance rules, risk thresholds, and document type.
- **1.2 Document Text Extraction:** Extracts raw text from PDFs to prepare for AI-driven analysis.
- **1.3 Clause Analysis:** Uses OpenAI-powered agents to identify, categorize, and extract key contractual clauses with their locations and characteristics.
- **1.4 Compliance Checking:** Cross-examines extracted clauses against predefined compliance rules and risk thresholds to identify problematic areas and missing clauses.
- **1.5 Alternative Wording Generation:** Provides suggested rewordings for risky clauses and drafts missing clauses to reduce legal risk while preserving intent.
- **1.6 Summary Report Creation:** Compiles all analysis data into a professional HTML report summarizing findings, risks, suggestions, and recommendations.
- **1.7 Database Logging and Notification:** Stores detailed review results in a database and sends notifications to stakeholders.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Configuration

**Overview:**  
This block initiates the workflow by receiving legal documents through a webhook and setting important configuration parameters for downstream processing.

**Nodes Involved:**  
- Document Upload Webhook  
- Workflow Configuration

**Node Details:**

- **Document Upload Webhook**  
  - *Type:* Webhook node  
  - *Role:* Receives POST requests with legal document files uploaded by users.  
  - *Configuration:* Path set to `legal-document-upload`, HTTP method POST, response returned from the last node connected downstream.  
  - *Input/Output:* Input is external HTTP POST with document payload; outputs raw request data to next node.  
  - *Edge Cases:* Upload failures, unsupported file types, malformed requests, large files causing timeout.  
  - *Version:* 2.1

- **Workflow Configuration**  
  - *Type:* Set node  
  - *Role:* Defines runtime parameters such as compliance rules, risk threshold, and document type for AI agents.  
  - *Configuration:* Sets three string variables: `complianceRules` (placeholder for rules), `riskThreshold` (default "medium"), and `documentType` (default "contract").  
  - *Input/Output:* Receives webhook data; outputs enriched JSON with configuration for downstream nodes.  
  - *Edge Cases:* Missing or invalid configuration values may affect AI logic.  
  - *Version:* 3.4

---

#### 1.2 Document Text Extraction

**Overview:**  
Extracts readable text from the uploaded PDF document to prepare for AI-based clause analysis.

**Nodes Involved:**  
- Extract Document Text

**Node Details:**

- **Extract Document Text**  
  - *Type:* Extract from File node  
  - *Role:* Parses PDF to extract textual content.  
  - *Configuration:* Operation set to "pdf" extraction.  
  - *Input/Output:* Input is raw uploaded file from previous node; output is extracted document text available as `$json.text`.  
  - *Edge Cases:* Corrupted PDFs, scanned images without OCR, extraction failures.  
  - *Version:* 1.1

---

#### 1.3 Clause Analysis

**Overview:**  
Uses an AI agent powered by OpenAI to identify and extract key clauses from the document text, categorizing them and noting locations and any non-standard clauses.

**Nodes Involved:**  
- Clause Analysis Agent  
- OpenAI Model - Clause Analysis  
- Clause Analysis Output Parser

**Node Details:**

- **Clause Analysis Agent**  
  - *Type:* Langchain Agent node  
  - *Role:* Main logic node running the clause extraction prompt using AI.  
  - *Configuration:* Receives extracted text; system message defines detailed instructions to identify and categorize clauses, locate them, and flag unusual clauses. Returns structured JSON.  
  - *Input/Output:* Input: extracted text; Output: JSON with clauses array.  
  - *Edge Cases:* AI misinterpretation, incomplete extraction, prompt failures.  
  - *Version:* 3

- **OpenAI Model - Clause Analysis**  
  - *Type:* Langchain OpenAI Chat model  
  - *Role:* Executes the GPT-4.1-mini model for clause analysis.  
  - *Configuration:* Model set to GPT-4.1-mini; linked to Clause Analysis Agent.  
  - *Credentials:* Requires OpenAI API key.  
  - *Edge Cases:* API rate limits, authentication errors, timeouts.  
  - *Version:* 1.3

- **Clause Analysis Output Parser**  
  - *Type:* Langchain Structured Output Parser  
  - *Role:* Parses raw AI text output into structured JSON with clauses and their attributes (type, text, location, isStandard boolean).  
  - *Configuration:* Custom JSON schema to enforce output format.  
  - *Edge Cases:* Parsing errors if AI output deviates from expected format.  
  - *Version:* 1.3

---

#### 1.4 Compliance Checking

**Overview:**  
Checks extracted clauses against compliance rules and risk thresholds using AI to identify risky or missing clauses, assessing their risk levels and legal implications.

**Nodes Involved:**  
- Compliance Check Agent  
- OpenAI Model - Compliance  
- Compliance Output Parser

**Node Details:**

- **Compliance Check Agent**  
  - *Type:* Langchain Agent node  
  - *Role:* Analyzes clauses in context of compliance rules and risk threshold, identifies violations and missing clauses.  
  - *Configuration:* Input includes JSON clauses, compliance rules, and risk threshold variables. Returns structured JSON with risky areas and missing clauses.  
  - *Edge Cases:* Incomplete compliance rules, ambiguous risk assessment, AI misclassification.  
  - *Version:* 3

- **OpenAI Model - Compliance**  
  - *Type:* Langchain OpenAI Chat model  
  - *Role:* Executes GPT-4.1-mini for compliance review.  
  - *Credentials:* OpenAI API key required.  
  - *Edge Cases:* API errors, latency, prompt degradation.  
  - *Version:* 1.3

- **Compliance Output Parser**  
  - *Type:* Langchain Structured Output Parser  
  - *Role:* Parses compliance analysis output to JSON arrays of risky areas and missing clauses with risk levels and explanations.  
  - *Configuration:* Custom JSON schema defines structure.  
  - *Edge Cases:* Deviations in AI output format causing parse failure.  
  - *Version:* 1.3

---

#### 1.5 Alternative Wording Generation

**Overview:**  
Generates alternative, legally sound clause wordings for risky clauses and drafts missing clauses to reduce risk while preserving original intent.

**Nodes Involved:**  
- Alternative Wording Agent  
- OpenAI Model - Alternative Wording  
- Alternative Wording Output Parser

**Node Details:**

- **Alternative Wording Agent**  
  - *Type:* Langchain Agent node  
  - *Role:* Provides rewording suggestions and explanations for reducing legal risk.  
  - *Configuration:* Input is JSON of risky areas; system message instructs to maintain intent, clarity, and legal soundness. Returns structured JSON with suggestions.  
  - *Edge Cases:* Potential misunderstanding of intent, incomplete suggestions.  
  - *Version:* 3

- **OpenAI Model - Alternative Wording**  
  - *Type:* Langchain OpenAI Chat model  
  - *Role:* Runs GPT-4.1-mini for wording generation.  
  - *Credentials:* OpenAI API key required.  
  - *Edge Cases:* API access errors, prompt failures.  
  - *Version:* 1.3

- **Alternative Wording Output Parser**  
  - *Type:* Langchain Structured Output Parser  
  - *Role:* Parses alternative wording suggestions into structured JSON.  
  - *Configuration:* Schema includes suggestions and missing clause wording.  
  - *Edge Cases:* Parsing errors if AI output format is unexpected.  
  - *Version:* 1.3

---

#### 1.6 Summary Report Creation

**Overview:**  
Aggregates all previous analysis results into a professional HTML report, including executive summary, risky areas, alternative wordings, and recommendations.

**Nodes Involved:**  
- Summary Report Generator  
- OpenAI Model - Summary

**Node Details:**

- **Summary Report Generator**  
  - *Type:* Langchain Agent node  
  - *Role:* Combines clauses, risks, missing clauses, suggestions into a formatted HTML report.  
  - *Configuration:* Input includes complete analysis JSON data from multiple upstream nodes; system message requests professional HTML format with sections and styling.  
  - *Edge Cases:* Overly long reports causing token limits, formatting errors.  
  - *Version:* 3

- **OpenAI Model - Summary**  
  - *Type:* Langchain OpenAI Chat model  
  - *Role:* Runs GPT-4.1-mini to generate report content.  
  - *Credentials:* OpenAI API key required.  
  - *Edge Cases:* API limits or timeout, incomplete report generation.  
  - *Version:* 1.3

---

#### 1.7 Database Logging and Notification

**Overview:**  
Prepares data for database logging, inserts review results into a Postgres database, and sends notification via HTTP webhook (e.g., Slack, Teams, or email service).

**Nodes Involved:**  
- Prepare Database Record  
- Log to Contract Database  
- Send Notification

**Node Details:**

- **Prepare Database Record**  
  - *Type:* Set node  
  - *Role:* Formats review metadata and summaries into fields for database insertion.  
  - *Configuration:* Assigns fields like document name, review date (ISO timestamp), risk summary, suggestions summary, full HTML report, total clauses count, and count of high-risk clauses.  
  - *Edge Cases:* Missing upstream data causing null fields.  
  - *Version:* 3.4

- **Log to Contract Database**  
  - *Type:* Postgres node  
  - *Role:* Inserts prepared data into specified Postgres table and schema.  
  - *Configuration:* Table and schema names are placeholders to be customized; columns mapped to fields from previous node.  
  - *Edge Cases:* DB connection failure, constraint violations, schema mismatch.  
  - *Version:* 2.6

- **Send Notification**  
  - *Type:* HTTP Request node  
  - *Role:* Sends POST request to notification webhook with summary of review results.  
  - *Configuration:* URL is a placeholder; JSON body includes document name, review date, total clauses, high-risk count, and a status message.  
  - *Edge Cases:* Invalid webhook URL, network errors, failed message delivery.  
  - *Version:* 4.3

---

### 3. Summary Table

| Node Name                  | Node Type                             | Functional Role                               | Input Node(s)               | Output Node(s)                      | Sticky Note                                                                                         |
|----------------------------|-------------------------------------|-----------------------------------------------|-----------------------------|-----------------------------------|---------------------------------------------------------------------------------------------------|
| Document Upload Webhook     | Webhook                             | Entry point to receive uploaded legal docs    | (External HTTP POST)         | Workflow Configuration             | ## How It Works This template automates legal document review... (covers initial nodes)            |
| Workflow Configuration      | Set                                 | Defines compliance rules, risk threshold       | Document Upload Webhook      | Extract Document Text              | ## How It Works This template automates legal document review...                                   |
| Extract Document Text       | Extract from File                   | Extracts text from uploaded PDF                 | Workflow Configuration       | Clause Analysis Agent              | ## Extract & Analyze Automated extraction eliminates manual reading...                              |
| Clause Analysis Agent       | Langchain Agent                    | Extracts and categorizes contract clauses      | Extract Document Text        | Compliance Check Agent             | ## Extract & Analyze Automated extraction eliminates manual reading...                              |
| OpenAI Model - Clause Analysis | Langchain OpenAI Chat Model       | Executes AI model for clause analysis          | Clause Analysis Agent        | Clause Analysis Agent              | ## Extract & Analyze Automated extraction eliminates manual reading...                              |
| Clause Analysis Output Parser | Langchain Output Parser Structured  | Parses clause analysis output                   | OpenAI Model - Clause Analysis | Clause Analysis Agent           | ## Extract & Analyze Automated extraction eliminates manual reading...                              |
| Compliance Check Agent      | Langchain Agent                    | Checks clauses vs compliance rules              | Clause Analysis Agent        | Alternative Wording Agent          | ## Compliance & Alternatives Compliance agent verifies standards...                                |
| OpenAI Model - Compliance   | Langchain OpenAI Chat Model        | Runs compliance checking AI model               | Compliance Check Agent       | Compliance Check Agent             | ## Compliance & Alternatives Compliance agent verifies standards...                                |
| Compliance Output Parser    | Langchain Output Parser Structured  | Parses compliance check output                   | OpenAI Model - Compliance    | Compliance Check Agent             | ## Compliance & Alternatives Compliance agent verifies standards...                                |
| Alternative Wording Agent   | Langchain Agent                    | Generates alternative clause wordings          | Compliance Check Agent       | Summary Report Generator           | ## Compliance & Alternatives Compliance agent verifies standards...                                |
| OpenAI Model - Alternative Wording | Langchain OpenAI Chat Model        | Runs AI model for alternative wording           | Alternative Wording Agent    | Alternative Wording Agent          | ## Compliance & Alternatives Compliance agent verifies standards...                                |
| Alternative Wording Output Parser | Langchain Output Parser Structured  | Parses alternative wording suggestions          | OpenAI Model - Alternative Wording | Alternative Wording Agent     | ## Compliance & Alternatives Compliance agent verifies standards...                                |
| Summary Report Generator    | Langchain Agent                    | Creates final HTML summary report                | Alternative Wording Agent, Clause Analysis Agent, Compliance Check Agent | Prepare Database Record | ## Report & Archive Summary generator creates unified reports...                                  |
| OpenAI Model - Summary      | Langchain OpenAI Chat Model        | Generates HTML report content                     | Summary Report Generator    | Summary Report Generator           | ## Report & Archive Summary generator creates unified reports...                                  |
| Prepare Database Record     | Set                                 | Formats data for DB insertion                     | Summary Report Generator    | Log to Contract Database           | ## Report & Archive Summary generator creates unified reports...                                  |
| Log to Contract Database    | Postgres                           | Inserts review data into Postgres DB             | Prepare Database Record     | Send Notification                 | ## Report & Archive Summary generator creates unified reports...                                  |
| Send Notification           | HTTP Request                      | Notifies stakeholders via webhook                 | Log to Contract Database    | (End)                            | ## Report & Archive Summary generator creates unified reports...                                  |
| Sticky Note                | Sticky Note                       | Provides overview and workflow explanation       | N/A                         | N/A                              | ## How It Works This template automates legal document review by extracting text...                |
| Sticky Note1               | Sticky Note                       | Customization and benefits notes                  | N/A                         | N/A                              | ## Customization Modify OpenAI prompts for industry-specific clauses...                            |
| Sticky Note2               | Sticky Note                       | Prerequisites and use cases                        | N/A                         | N/A                              | ## Prerequisites OpenAI API key, Gmail account, Google Sheets access...                            |
| Sticky Note3               | Sticky Note                       | Setup steps summary                                | N/A                         | N/A                              | ## Setup Steps 1. Configure OpenAI API with clause analysis and compliance prompts...              |
| Sticky Note4               | Sticky Note                       | Explanation of extraction and analysis block      | N/A                         | N/A                              | ## Extract & Analyze Automated extraction eliminates manual reading...                              |
| Sticky Note5               | Sticky Note                       | Explanation of compliance and alternatives block  | N/A                         | N/A                              | ## Compliance & Alternatives Compliance agent verifies standards...                                |
| Sticky Note6               | Sticky Note                       | Explanation of report and archive block            | N/A                         | N/A                              | ## Report & Archive Summary generator creates unified reports...                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Name: Document Upload Webhook  
   - Path: `legal-document-upload`  
   - HTTP Method: POST  
   - Response Mode: Last Node  
   - Purpose: Receive legal document files uploaded via HTTP POST.

2. **Create Set Node for Configuration**  
   - Type: Set  
   - Name: Workflow Configuration  
   - Add variables:  
     - `complianceRules` (string): Placeholder text `<__PLACEHOLDER_VALUE__Compliance rules and regulations to check against__>`  
     - `riskThreshold` (string): Default `"medium"`  
     - `documentType` (string): Default `"contract"`  
   - Connect output from Webhook to this node.

3. **Create Extract from File Node**  
   - Type: Extract from File  
   - Name: Extract Document Text  
   - Operation: PDF text extraction  
   - Connect output from Workflow Configuration to this node.

4. **Create Langchain Agent Node for Clause Analysis**  
   - Type: Langchain Agent  
   - Name: Clause Analysis Agent  
   - Input Text Expression: `={{ $json.text }}` (from Extract Document Text)  
   - System Message:  
     ```
     You are a legal document analysis expert specializing in contract review.

     Your task is to:
     1. Identify and extract all key clauses from the legal document
     2. Categorize each clause by type (e.g., termination, liability, payment, confidentiality, indemnification)
     3. Extract the exact text of each clause
     4. Identify the location (section/paragraph) of each clause
     5. Note any unusual or non-standard clauses

     Return your analysis in the structured JSON format defined by the output parser.
     ```  
   - Enable Output Parser with manual JSON schema specifying clauses array with properties: type (string), text (string), location (string), isStandard (boolean).  
   - Connect output from Extract Document Text node.

5. **Create OpenAI Chat Model Node for Clause Analysis**  
   - Type: Langchain OpenAI Chat model  
   - Name: OpenAI Model - Clause Analysis  
   - Model: GPT-4.1-mini  
   - Credentials: Link your OpenAI API key  
   - Connect this node as AI Language model for Clause Analysis Agent.

6. **Create Langchain Output Parser Node**  
   - Type: Langchain Output Parser Structured  
   - Name: Clause Analysis Output Parser  
   - Schema: Manual JSON schema matching Clause Analysis Agent output structure  
   - Connect AI output parser from OpenAI Model - Clause Analysis to this node, then to Clause Analysis Agent.

7. **Create Langchain Agent Node for Compliance Check**  
   - Type: Langchain Agent  
   - Name: Compliance Check Agent  
   - Input Text Expression:  
     ```
     Clauses: {{ JSON.stringify($json.clauses) }}

     Compliance Rules: {{ $('Workflow Configuration').first().json.complianceRules }}

     Risk Threshold: {{ $('Workflow Configuration').first().json.riskThreshold }}
     ```  
   - System Message:  
     ```
     You are a legal compliance expert specializing in contract risk assessment.

     Your task is to:
     1. Review each clause against the provided compliance rules and regulations
     2. Identify any clauses that violate or conflict with compliance requirements
     3. Assess the risk level of each problematic clause (low, medium, high, critical)
     4. Highlight specific compliance issues and their potential legal implications
     5. Flag any missing clauses that are required for compliance

     Return your analysis in the structured JSON format defined by the output parser.
     ```  
   - Enable Output Parser with manual JSON schema defining riskyAreas array and missingClauses array.  
   - Connect output of Clause Analysis Agent to this node.

8. **Create OpenAI Chat Model Node for Compliance**  
   - Type: Langchain OpenAI Chat model  
   - Name: OpenAI Model - Compliance  
   - Model: GPT-4.1-mini  
   - Credentials: OpenAI API key  
   - Connect as AI Language model for Compliance Check Agent.

9. **Create Langchain Output Parser Node for Compliance**  
   - Type: Langchain Output Parser Structured  
   - Name: Compliance Output Parser  
   - Schema: Manual JSON schema with riskyAreas and missingClauses as per Compliance Check Agent output.  
   - Connect AI output parser from OpenAI Model - Compliance to this node, then to Compliance Check Agent.

10. **Create Langchain Agent Node for Alternative Wording**  
    - Type: Langchain Agent  
    - Name: Alternative Wording Agent  
    - Input Text Expression: `=Risky Areas: {{ JSON.stringify($json.riskyAreas) }}`  
    - System Message:  
      ```
      You are a legal drafting expert specializing in contract language optimization.

      Your task is to:
      1. For each risky clause identified, provide alternative wording that addresses the compliance issue
      2. Ensure the alternative wording maintains the original intent while reducing legal risk
      3. Make the language clear, unambiguous, and legally sound
      4. Provide a brief explanation of why the alternative wording is better
      5. Suggest wording for any missing clauses that were flagged

      Return your suggestions in the structured JSON format defined by the output parser.
      ```  
    - Enable Output Parser with manual JSON schema defining suggestions and missingClauseSuggestions arrays.  
    - Connect output of Compliance Check Agent to this node.

11. **Create OpenAI Chat Model Node for Alternative Wording**  
    - Type: Langchain OpenAI Chat model  
    - Name: OpenAI Model - Alternative Wording  
    - Model: GPT-4.1-mini  
    - Credentials: OpenAI API key  
    - Connect as AI Language model for Alternative Wording Agent.

12. **Create Langchain Output Parser Node for Alternative Wording**  
    - Type: Langchain Output Parser Structured  
    - Name: Alternative Wording Output Parser  
    - Schema: Manual JSON schema matching Alternative Wording Agent output.  
    - Connect AI output parser from OpenAI Model - Alternative Wording to this node, then to Alternative Wording Agent.

13. **Create Langchain Agent Node for Summary Report**  
    - Type: Langchain Agent  
    - Name: Summary Report Generator  
    - Input Text Expression:  
      ```
      Complete Analysis Data:

      Clauses: {{ JSON.stringify($('Clause Analysis Agent').first().json.clauses) }}

      Risky Areas: {{ JSON.stringify($('Compliance Check Agent').first().json.riskyAreas) }}

      Missing Clauses: {{ JSON.stringify($('Compliance Check Agent').first().json.missingClauses) }}

      Suggestions: {{ JSON.stringify($json.suggestions) }}

      Missing Clause Suggestions: {{ JSON.stringify($json.missingClauseSuggestions) }}
      ```  
    - System Message:  
      ```
      You are a legal document review specialist creating executive summaries.

      Your task is to:
      1. Create a comprehensive summary report of the legal document review
      2. Include an executive summary highlighting the most critical findings
      3. List all risky areas with their risk levels and compliance issues
      4. Provide all alternative wording suggestions in a clear, actionable format
      5. Include recommendations for next steps
      6. Format the report in professional HTML with proper sections and styling

      Return ONLY the complete HTML report (including <html>, <head>, and <body> tags).
      ```  
    - Enable Output Parser disabled (raw text output expected).  
    - Connect output of Alternative Wording Agent and other nodes as inputs.

14. **Create OpenAI Chat Model Node for Summary**  
    - Type: Langchain OpenAI Chat model  
    - Name: OpenAI Model - Summary  
    - Model: GPT-4.1-mini  
    - Credentials: OpenAI API key  
    - Connect as AI Language model for Summary Report Generator.

15. **Create Set Node to Prepare Database Record**  
    - Type: Set  
    - Name: Prepare Database Record  
    - Assign fields:  
      - `document_name`: `={{ $('Document Upload Webhook').first().json.body.filename || "Unknown Document" }}`  
      - `review_date`: `={{ $now.toISO() }}`  
      - `risk_summary`: `={{ JSON.stringify($('Compliance Check Agent').first().json.riskyAreas) }}`  
      - `suggestions_summary`: `={{ JSON.stringify($('Alternative Wording Agent').first().json.suggestions) }}`  
      - `full_report_html`: `={{ $json.output }}` (from Summary Report Generator)  
      - `total_clauses`: `={{ $('Clause Analysis Agent').first().json.clauses.length }}`  
      - `high_risk_count`: `={{ $('Compliance Check Agent').first().json.riskyAreas.filter(r => r.riskLevel === "high" || r.riskLevel === "critical").length }}`  
    - Connect output from Summary Report Generator.

16. **Create Postgres Node to Log to Database**  
    - Type: Postgres  
    - Name: Log to Contract Database  
    - Connection: Configure with your Postgres credentials  
    - Table Name: Placeholder `<__PLACEHOLDER_VALUE__Table name for contract reviews__>`  
    - Schema Name: Placeholder `<__PLACEHOLDER_VALUE__Database schema name__>`  
    - Columns mapped to Set node fields accordingly.  
    - Connect output from Prepare Database Record.

17. **Create HTTP Request Node to Send Notification**  
    - Type: HTTP Request  
    - Name: Send Notification  
    - Method: POST  
    - URL: Placeholder `<__PLACEHOLDER_VALUE__Notification webhook URL (e.g., Slack, Teams, email service)__>`  
    - Body Content-Type: JSON  
    - JSON Body:  
      ```json
      {
        "document": "={{ $('Prepare Database Record').first().json.document_name }}",
        "review_date": "={{ $('Prepare Database Record').first().json.review_date }}",
        "total_clauses": "={{ $('Prepare Database Record').first().json.total_clauses }}",
        "high_risk_count": "={{ $('Prepare Database Record').first().json.high_risk_count }}",
        "message": "Legal document review completed. {{ $('Prepare Database Record').first().json.high_risk_count }} high-risk issues found."
      }
      ```  
    - Connect output from Log to Contract Database.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                   |
|---------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| This template automates legal document review by extracting text, analyzing clauses with OpenAI, checking compliance standards, and generating alternative wordings. | Overview sticky note at workflow start           |
| Modify OpenAI prompts for industry-specific clauses. Swap Google Sheets for your database. Benefits include reducing manual review time by 80% and ensuring consistent clause standardization. | Customization sticky note near end of workflow   |
| Prerequisites: OpenAI API key, Gmail account, Google Sheets access, document upload capability. Use cases include legal teams reviewing SaaS contracts and compliance departments auditing vendor agreements. | Prerequisites sticky note mid-workflow            |
| Setup Steps: Configure OpenAI API with clause analysis and compliance prompts, connect Google Sheets for staging, set Gmail for notifications, input document source, test with sample contracts. | Setup sticky note near configuration nodes        |
| Extraction and Analysis block automates text extraction and parallel clause analysis to accelerate processing and reduce manual labor. | Extraction sticky note near document text extraction block |
| Compliance and Alternatives block runs compliance verification and alternative wording generation in parallel to ensure thorough review and improvement suggestions. | Compliance sticky note near compliance and wording blocks |
| Report and Archive block creates unified HTML reports and archives data for audit trails and compliance tracking. | Reporting sticky note near database logging and notification nodes |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.