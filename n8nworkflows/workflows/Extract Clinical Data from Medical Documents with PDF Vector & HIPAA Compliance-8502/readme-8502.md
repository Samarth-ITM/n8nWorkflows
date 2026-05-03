Extract Clinical Data from Medical Documents with PDF Vector & HIPAA Compliance

https://n8nworkflows.xyz/workflows/extract-clinical-data-from-medical-documents-with-pdf-vector---hipaa-compliance-8502


# Extract Clinical Data from Medical Documents with PDF Vector & HIPAA Compliance

### 1. Workflow Overview

This workflow automates the extraction and processing of clinical data from medical documents stored on Google Drive, with strict adherence to HIPAA compliance. It is designed for healthcare providers or IT teams handling protected health information (PHI) to securely intake, extract, validate, and store clinical records with de-identification and clinical coding.

The logical structure is divided into these blocks:

- **1.1 Input Reception and Retrieval**: Manual trigger initiates the workflow, retrieving the medical document file securely from Google Drive.
- **1.2 Clinical Data Extraction**: Uses the PDF Vector node to extract structured clinical data from the document, including PHI removal and coding hints.
- **1.3 Data Processing and Validation**: Runs custom JavaScript code to validate the extracted clinical data, flag abnormal labs, and check simple drug interactions.
- **1.4 Conditional Routing and Secure Storage**: Routes only valid records for insertion into a HIPAA-compliant secure PostgreSQL database.
- **1.5 Documentation & Compliance Notes**: Sticky notes provide context on HIPAA compliance, security requirements, and clinical coding standards embedded throughout the workflow.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Retrieval

- **Overview:**  
  Initiates the workflow manually and retrieves the specified medical record file from Google Drive for processing.

- **Nodes Involved:**  
  - Manual Trigger  
  - Google Drive - Get Medical Record

- **Node Details:**

  - **Manual Trigger**  
    - Type: Manual trigger node  
    - Role: Entry point for processing a medical record on demand  
    - Configuration: No parameters; triggered by user interaction  
    - Inputs: None  
    - Outputs: Triggers downstream nodes  
    - Edge Cases: Manual activation required; no automated polling or external event trigger  
    - Notes: Allows controlled and secure processing start

  - **Google Drive - Get Medical Record**  
    - Type: Google Drive node (v3)  
    - Role: Downloads a medical record file from Google Drive based on provided fileId  
    - Configuration: `operation` set to "download"; `fileId` dynamically taken from input JSON (`{{$json.fileId}}`)  
    - Inputs: Trigger from Manual Trigger node, expects input JSON with `fileId`  
    - Outputs: Binary file data of the medical record  
    - Requirements: Google Drive OAuth2 credentials with read access to the relevant Drive location  
    - Edge Cases: Invalid or missing fileId, permission denied, file not found, network errors

#### 2.2 Clinical Data Extraction

- **Overview:**  
  Extracts structured clinical information from the medical document, using OCR if necessary, while excluding PHI such as patient name or SSN.

- **Nodes Involved:**  
  - PDF Vector - Extract Medical Data

- **Node Details:**

  - **PDF Vector - Extract Medical Data**  
    - Type: PDF Vector extraction node (custom or third-party integration)  
    - Role: Performs semantic extraction of medical data from PDF or image files  
    - Configuration:  
      - Prompt instructs extraction of clinical data elements (patient ID, visit date, diagnoses with ICD codes, medications, labs, procedures, follow-up, etc.)  
      - Explicit instructions to exclude identifying information (patient names, SSN) for HIPAA compliance  
      - JSON schema enforces the expected structured output format  
      - Input type: expects binary file from prior Google Drive node  
    - Inputs: Binary medical document file  
    - Outputs: JSON object with extracted structured clinical data  
    - Requirements: Access to PDF Vector service with proper API keys or credentials  
    - Edge Cases: Poor scan quality affecting OCR, unexpected document formats, partial data extraction, schema validation errors

#### 2.3 Data Processing and Validation

- **Overview:**  
  Applies business logic to validate extracted data, create audit logs, flag abnormal labs, and detect simple medication interactions.

- **Nodes Involved:**  
  - Process & Validate Data (Code node)

- **Node Details:**

  - **Process & Validate Data**  
    - Type: Code node (JavaScript)  
    - Role: Processes raw extracted data, validates critical fields, and enriches with metadata and alerts  
    - Configuration:  
      - Validates presence of patient ID, visit date, and diagnoses  
      - Checks ICD-10 code format using regex  
      - Flags abnormal lab results (e.g., high, low, critical)  
      - Simplified drug interaction check (example given: warfarin + aspirin)  
      - Constructs a processed record JSON with clinical data, alerts, validation flags, audit logs, retention info  
    - Inputs: JSON data from PDF Vector node  
    - Outputs: JSON processed and validated record  
    - Edge Cases: Missing critical fields, invalid ICD codes, empty arrays, inconsistencies in lab flags, possible exceptions in code execution

#### 2.4 Conditional Routing and Secure Storage

- **Overview:**  
  Uses an IF node to allow only valid records through and stores them securely in a HIPAA-compliant PostgreSQL database.

- **Nodes Involved:**  
  - Valid Record? (IF node)  
  - Store in Secure Database (Postgres node)

- **Node Details:**

  - **Valid Record?**  
    - Type: IF node  
    - Role: Checks if record validation passed (patient ID and diagnoses present)  
    - Configuration:  
      - Boolean conditions combined with AND operator:  
        - `$json.validation.hasPatientId == true`  
        - `$json.validation.hasDiagnoses == true`  
    - Inputs: Processed data JSON  
    - Outputs:  
      - True branch: valid data  
      - False branch: invalid data (not connected, so discarded or requires further handling if added)  
    - Edge Cases: False negatives if validation logic is too strict; no false branch handling

  - **Store in Secure Database**  
    - Type: PostgreSQL node  
    - Role: Inserts validated clinical data into secure medical_records table  
    - Configuration:  
      - Operation: Insert  
      - Table: `medical_records`  
      - Columns: patient_id, visit_date, diagnoses, medications, lab_results, processed_at, data_classification  
      - Maps columns to corresponding JSON fields  
    - Inputs: True branch from IF node  
    - Requirements: Secure PostgreSQL credentials with encrypted connection, access controls, audit logging enabled on DB level  
    - Edge Cases: DB connection failures, constraint violations, data type mismatches, concurrent inserts

#### 2.5 Documentation & Compliance Notes (Sticky Notes)

- **Overview:**  
  Provides visual guidance and compliance context directly within the workflow editor.

- **Nodes Involved:**  
  - HIPAA Overview (sticky note)  
  - Security Requirements (sticky note)  
  - Clinical Codes (sticky note)

- **Node Details:**

  - **HIPAA Overview**  
    - Role: Summarizes workflow purpose and HIPAA compliance highlights  
    - Emphasizes secure SFTP intake, PHI removal, automatic coding (ICD-10, CPT), HL7 FHIR formatting, and EHR integration (Epic/Cerner)  

  - **Security Requirements**  
    - Role: Lists mandatory security controls such as encryption at rest, TLS 1.3, audit logging, PHI de-identification, and access controls  
    - Advises review with compliance teams  

  - **Clinical Codes**  
    - Role: Describes automatic clinical coding mappings to ICD-10, CPT, and NDC  
    - Highlights no PHI is logged during processing  

---

### 3. Summary Table

| Node Name                      | Node Type                | Functional Role                          | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                 |
|-------------------------------|--------------------------|----------------------------------------|----------------------------|----------------------------|---------------------------------------------------------------------------------------------|
| Manual Trigger                | Manual trigger           | Entry point for manual workflow start  | None                       | Google Drive - Get Medical Record | Process medical record                                                                      |
| Google Drive - Get Medical Record | Google Drive (v3)        | Downloads medical record from Drive     | Manual Trigger             | PDF Vector - Extract Medical Data | Retrieve record from Drive                                                                  |
| PDF Vector - Extract Medical Data | PDF Vector extraction     | Extracts structured medical data       | Google Drive - Get Medical Record | Process & Validate Data      | Extract medical information                                                                |
| Process & Validate Data       | Code (JavaScript)        | Validates and enriches clinical data   | PDF Vector - Extract Medical Data | Valid Record?              | Validate and prepare data                                                                   |
| Valid Record?                 | IF node                  | Checks if extracted record is valid    | Process & Validate Data    | Store in Secure Database    |                                                                                             |
| Store in Secure Database      | PostgreSQL node          | Inserts validated data into secure DB  | Valid Record?              | None                       | HIPAA-compliant storage                                                                     |
| HIPAA Overview               | Sticky Note              | Describes workflow purpose & HIPAA     | None                       | None                       | ## 🏥 Medical Records Processor\n\n⚠️ **HIPAA COMPLIANT WORKFLOW**\n\n• **Secure** SFTP intake only\n• **Extracts** clinical data with PHI removal\n• **Codes** ICD-10 & CPT automatically\n• **Formats** HL7 FHIR standard\n• **Integrates** with Epic/Cerner |
| Security Requirements        | Sticky Note              | Lists security controls required        | None                       | None                       | ## 🔐 Security Setup\n\n**REQUIRED:**\n• Encryption at rest\n• TLS 1.3 minimum\n• Audit logging ON\n• PHI de-identification\n• Access controls\n\n⚠️ Review with compliance! |
| Clinical Codes               | Sticky Note              | Summarizes automatic clinical coding    | None                       | None                       | ## 📋 Clinical Coding\n\n**Automatic mapping:**\n• Diagnoses → ICD-10\n• Procedures → CPT\n• Medications → NDC\n\n💡 No PHI in logs! |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a Manual Trigger node with default settings. This will serve as the workflow entry point.

2. **Add Google Drive Node**  
   - Add a Google Drive node (version 3).  
   - Set `Operation` to "Download".  
   - For `File ID`, use the expression `={{ $json.fileId }}` to dynamically pass the file ID from input.  
   - Connect the output of Manual Trigger to this node.  
   - Configure Google Drive OAuth2 credentials with read permissions to the appropriate Drive folder.

3. **Add PDF Vector Extraction Node**  
   - Add a PDF Vector node (or equivalent third-party node capable of semantic extraction).  
   - Set `Resource` to "Document".  
   - Set `Operation` to "Extract".  
   - Use the binary property from the Google Drive node as input (typically named "data").  
   - Enter the prompt for data extraction:  
     > Extract medical information from this document or image including patient ID (not name), visit date, chief complaint, diagnoses with ICD codes, medications with dosages, vital signs, lab results with values and reference ranges, procedures performed, and follow-up instructions. Do not extract patient names, SSN, or other identifying information. Use OCR if this is a scanned document or medical image.  
   - Provide the JSON schema defining expected output structure to enforce data validation.  
   - Connect Google Drive node output to this node.  
   - Ensure API credentials or keys for PDF Vector service are configured.

4. **Add Code Node for Processing & Validation**  
   - Add a Code node (JavaScript).  
   - Paste the provided validation and processing script which:  
     - Extracts patient and clinical data  
     - Creates an audit log entry  
     - Validates presence and format of critical fields (patient ID, visit date, ICD codes)  
     - Flags abnormal labs  
     - Performs simplified medication interaction checks  
     - Assembles processed record JSON with metadata  
   - Connect PDF Vector node output to this node.

5. **Add IF Node to Validate Record**  
   - Add an IF node.  
   - Configure two boolean conditions combined with AND:  
     - Expression `{{$json.validation.hasPatientId}}` equals `true`  
     - Expression `{{$json.validation.hasDiagnoses}}` equals `true`  
   - Connect the Code node output to the IF node.

6. **Add PostgreSQL Node for Secure Storage**  
   - Add a PostgreSQL node.  
   - Set `Operation` to "Insert".  
   - Set `Table` to `medical_records`.  
   - Map columns:  
     - `patient_id` → `{{$json.patientRecord.patientId}}`  
     - `visit_date` → `{{$json.patientRecord.visitDate}}`  
     - `diagnoses` → `{{$json.diagnoses}}` (serialized JSON or appropriate DB type)  
     - `medications` → `{{$json.medications}}`  
     - `lab_results` → `{{$json.labResults}}`  
     - `processed_at` → `{{$json.processedAt}}`  
     - `data_classification` → `{{$json.dataClassification}}`  
   - Connect the IF node "true" output to this node.  
   - Configure secure PostgreSQL credentials with encryption, audit, and access control compliance.

7. **Add Sticky Notes for Documentation**  
   - Add three Sticky Note nodes with the following contents:  
     - **HIPAA Overview:** Describe workflow purpose, HIPAA compliance highlights, and integration targets.  
     - **Security Requirements:** List encryption, TLS version, audit logging, PHI de-identification, and access controls.  
     - **Clinical Codes:** Summarize automatic clinical coding mappings and PHI logging restrictions.  
   - Position these notes clearly near relevant nodes.

8. **Test Workflow Execution**  
   - Trigger manually with a valid `fileId` in input JSON.  
   - Verify each step completes without errors.  
   - Confirm data is stored securely in PostgreSQL.  
   - Review audit logs and validation alerts.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                 |
|-------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------|
| This workflow strictly enforces HIPAA compliance through PHI de-identification and secure storage. | HIPAA Overview sticky note & workflow purpose                                                  |
| Ensure TLS 1.3 minimum and encryption at rest on all data in transit and at rest.                | Security Requirements sticky note                                                              |
| Clinical coding includes ICD-10 for diagnoses, CPT for procedures, and NDC for medications.      | Clinical Codes sticky note                                                                     |
| No PHI is ever logged in the workflow logs to maintain privacy.                                  | Clinical Codes sticky note                                                                     |
| Consider integrating this workflow with HL7 FHIR standards and EHR systems such as Epic or Cerner for downstream processing. | HIPAA Overview sticky note                                                                     |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.