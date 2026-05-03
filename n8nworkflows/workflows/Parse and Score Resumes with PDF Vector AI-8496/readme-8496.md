Parse and Score Resumes with PDF Vector AI

https://n8nworkflows.xyz/workflows/parse-and-score-resumes-with-pdf-vector-ai-8496


# Parse and Score Resumes with PDF Vector AI

### 1. Workflow Overview

This workflow, titled **"Parse and Score Resumes with PDF Vector AI"**, is an automated recruitment processing system designed to streamline candidate evaluation by:

- Collecting resumes from external sources (Google Drive)
- Extracting structured candidate information using AI-powered PDF Vector parsing
- Computing experience and skill metrics based on extracted data
- Generating an AI-based assessment summarizing candidate strengths and seniority
- Combining all processed data into a candidate profile ready for Applicant Tracking System (ATS) integration

The workflow is logically grouped into these blocks:

- **1.1 Input Reception**: Receiving resumes from Google Drive upon manual triggering.
- **1.2 AI Data Extraction**: Using PDF Vector AI to parse resumes and extract structured candidate data.
- **1.3 Metrics Calculation**: Calculating experience years and skill proficiency scores via a custom code node.
- **1.4 AI Candidate Assessment**: Generating a qualitative AI assessment of the candidate profile.
- **1.5 Profile Aggregation**: Consolidating extracted data, metrics, and AI assessment into a final structured candidate profile.

Supporting notes provide setup instructions, input format details, and scoring methodology.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block initiates the workflow manually and fetches the candidate resume file from Google Drive for further processing.

**Nodes Involved:**  
- Manual Trigger  
- Get Resume from Google Drive

**Node Details:**

- **Manual Trigger**  
  - *Type:* Manual trigger node  
  - *Role:* Starts the workflow on user command; no parameters needed.  
  - *Connections:* Output connects to "Get Resume from Google Drive".  
  - *Edge Cases:* None expected; user must manually start.

- **Get Resume from Google Drive**  
  - *Type:* Google Drive node  
  - *Role:* Downloads the resume file specified by `fileId` from Google Drive.  
  - *Configuration:*  
    - Operation: Download  
    - File ID: Dynamic expression `={{ $json.fileId }}` (expects input data containing `fileId`)  
  - *Connections:* Output connects to "PDF Vector - Parse Resume".  
  - *Edge Cases:*  
    - Invalid or missing `fileId` causes download failure.  
    - Authentication errors if Google Drive credentials are not configured or expired.  
    - Network timeouts or API limits.  
  - *Credential:* Requires Google Drive OAuth2 credentials.

---

#### 1.2 AI Data Extraction

**Overview:**  
Parses the downloaded resume document or image using PDF Vector AI to extract structured candidate information including personal details, work history, education, skills, and certifications.

**Nodes Involved:**  
- PDF Vector - Parse Resume

**Node Details:**

- **PDF Vector - Parse Resume**  
  - *Type:* PDF Vector node (document extraction)  
  - *Role:* Uses AI to extract comprehensive resume data into a structured JSON schema.  
  - *Configuration:*  
    - Operation: Extract  
    - Input Type: File (binary data from Google Drive download)  
    - Prompt: Detailed instructions to extract personal info, experience (with dates), education, skills, certifications, etc.  
    - Schema: JSON schema specifying expected structured output (nested objects and arrays for personal info, work experience, education, skills, certifications)  
    - Binary Property Name: `data`  
  - *Connections:* Output connects to "Calculate Experience Metrics".  
  - *Edge Cases:*  
    - Parsing errors due to unrecognized resume formats or low scan quality.  
    - Missing or incomplete fields in output JSON.  
    - API key authentication or usage limits with PDF Vector service.  
  - *Credential:* Requires PDF Vector API key configured separately.

---

#### 1.3 Metrics Calculation

**Overview:**  
Processes the structured resume data to calculate total years of work experience, skill-specific proficiency scores based on experience length, education level, certification count, and language count.

**Nodes Involved:**  
- Calculate Experience Metrics

**Node Details:**

- **Calculate Experience Metrics**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Computes numeric metrics from parsed resume JSON.  
  - *Configuration:*  
    - Custom JavaScript code calculates:  
      - Total work experience in years by summing durations of all jobs (handling "Present" end dates)  
      - Skill experience by technology, assigning proficiency scores from 1 to 5 based on years  
      - Extracts highest education degree, certification count, and language count  
    - Outputs a JSON object containing the original candidate profile, computed metrics, and a processing timestamp.  
  - *Connections:* Output connects to "PDF Vector - AI Assessment".  
  - *Edge Cases:*  
    - Missing or malformed dates can cause calculation errors.  
    - Empty or missing work experience or skills arrays handled gracefully, results default to zero or "Not specified".  
    - Date parsing errors or inconsistent date formats.  
  - *Version Requirements:* Uses JavaScript runtime with ES6+ support (n8n version 0.146+ recommended).

---

#### 1.4 AI Candidate Assessment

**Overview:**  
Requests an AI-generated qualitative assessment of the candidate, highlighting strengths, suitable roles, notable achievements, and estimated seniority level.

**Nodes Involved:**  
- PDF Vector - AI Assessment

**Node Details:**

- **PDF Vector - AI Assessment**  
  - *Type:* PDF Vector node (AI question answering)  
  - *Role:* Uses AI to analyze the resume document and generate a textual assessment.  
  - *Configuration:*  
    - Operation: Ask  
    - Input Type: File (resume binary data)  
    - Prompt: Requests strengths, role fit, achievements, and seniority estimation (Junior/Mid/Senior/Lead)  
    - Binary Property Name: `data`  
  - *Connections:* Output connects to "Create Candidate Profile".  
  - *Edge Cases:*  
    - AI response may vary in quality depending on resume clarity.  
    - API key authentication or rate limiting failures.  
    - Missing or corrupted input file causes failure.  
  - *Credential:* Requires PDF Vector API key.

---

#### 1.5 Profile Aggregation

**Overview:**  
Combines the candidate’s parsed profile, calculated metrics, and AI assessment text into a final candidate profile object flagged as ready for ATS integration.

**Nodes Involved:**  
- Create Candidate Profile

**Node Details:**

- **Create Candidate Profile**  
  - *Type:* Set node  
  - *Role:* Aggregates multiple data points into a single JSON object.  
  - *Configuration:*  
    - Assigns:  
      - `profile`: The original parsed candidate profile from "Calculate Experience Metrics"  
      - `metrics`: Calculated metrics (experience, skills, education, certifications)  
      - `aiAssessment`: AI-generated assessment string from "PDF Vector - AI Assessment"  
      - `atsReady`: Boolean flag set to true indicating readiness for ATS ingestion  
  - *Connections:* Final node, no outputs.  
  - *Edge Cases:* Should handle missing data gracefully, but depends on upstream nodes’ success.

---

### 3. Summary Table

| Node Name                     | Node Type               | Functional Role                     | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                              |
|-------------------------------|-------------------------|-----------------------------------|------------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------|
| System Overview               | Sticky Note             | Overview and workflow summary     |                              |                             | ## 👥 AI Resume Processing System Automated recruitment workflow: • **Collects** resumes from multiple sources • **Extracts** skills, experience, education • **Matches** against job requirements • **Scores** candidates objectively • **Integrates** with your ATS |
| Setup Required               | Sticky Note             | Setup instructions                |                              |                             | ## ⚙️ Configure First 1. Set job requirements JSON 2. Adjust scoring weights 3. Add PDF Vector API key 4. Connect ATS database 5. Set up notifications                   |
| Input Formats                | Sticky Note             | Accepted input resume formats     |                              |                             | ## 📄 Resume Input Accepts multiple formats: • PDF resumes • Word documents • LinkedIn exports • Text files 💡 Bulk processing capable                                    |
| Data Extraction             | Sticky Note             | AI data extraction details        |                              |                             | ## 🤖 AI Extraction PDF Vector extracts: • Personal info • Work experience • Education details • Skills & technologies • Certifications ✨ Structured JSON output           |
| Candidate Scoring           | Sticky Note             | Scoring weights and methodology   |                              |                             | ## 📊 Scoring Engine **Weighted scoring:** • Skills match: 40% • Experience: 30% • Education: 20% • Certifications: 10% 🎯 Customizable weights                             |
| Manual Trigger              | Manual Trigger          | Start resume parsing manually     |                              | Get Resume from Google Drive | Start resume parsing                                                                                   |
| Get Resume from Google Drive | Google Drive            | Download resume file              | Manual Trigger               | PDF Vector - Parse Resume    | Download resume from Google Drive                                                                        |
| PDF Vector - Parse Resume    | PDF Vector              | Extract candidate info            | Get Resume from Google Drive | Calculate Experience Metrics | Extract candidate information                                                                            |
| Calculate Experience Metrics | Code                    | Calculate experience and skill scores | PDF Vector - Parse Resume    | PDF Vector - AI Assessment   | Calculate experience and skill scores                                                                   |
| PDF Vector - AI Assessment   | PDF Vector              | Generate AI candidate assessment | Calculate Experience Metrics | Create Candidate Profile     | Get AI assessment                                                                                        |
| Create Candidate Profile     | Set                     | Aggregate profile and scores      | PDF Vector - AI Assessment   |                             | Combine all data                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: `Manual Trigger`  
   - Purpose: To start the workflow manually. No parameters needed.

2. **Add Google Drive Node**  
   - Name: `Get Resume from Google Drive`  
   - Operation: `Download`  
   - File ID: Set expression to `={{ $json.fileId }}` — expects incoming JSON with `fileId` property.  
   - Connect output of `Manual Trigger` to this node’s input.  
   - Configure Google Drive OAuth2 credentials with appropriate API scopes for file access.

3. **Add PDF Vector Node for Parsing Resume**  
   - Name: `PDF Vector - Parse Resume`  
   - Resource: `document`  
   - Operation: `extract`  
   - Input Type: `file`  
   - Binary Property Name: `data` (to accept the file downloaded from Google Drive)  
   - Prompt: Use detailed prompt instructing AI to extract personal info, work experience with dates and achievements, education, skills, certifications, and languages. Include instruction to handle scanned and photographed resumes, emphasizing date extraction for experience calculation.  
   - Schema: Use a JSON schema defining expected output structure with properties for personalInfo, workExperience, education, skills, and certifications, matching the example schema from the original node.  
   - Connect output of `Get Resume from Google Drive` to this node’s input.  
   - Add PDF Vector API key credentials.

4. **Add Code Node to Calculate Metrics**  
   - Name: `Calculate Experience Metrics`  
   - Language: JavaScript  
   - Paste the provided JavaScript code that:  
     - Parses work experience dates and sums total experience  
     - Calculates skill proficiency scores based on years of experience per technology  
     - Extracts education level, certification count, and language count  
     - Returns a JSON object with candidate profile and metrics  
   - Connect output of `PDF Vector - Parse Resume` to this node’s input.

5. **Add PDF Vector Node for AI Assessment**  
   - Name: `PDF Vector - AI Assessment`  
   - Resource: `document`  
   - Operation: `ask`  
   - Input Type: `file`  
   - Binary Property Name: `data`  
   - Prompt: Ask for a brief assessment of candidate strengths, suitable roles, notable achievements, and seniority level (Junior/Mid/Senior/Lead).  
   - Connect output of `Calculate Experience Metrics` to this node’s input.  
   - Use same PDF Vector API key credentials.

6. **Add Set Node to Create Candidate Profile**  
   - Name: `Create Candidate Profile`  
   - Assignments:  
     - `profile` (object): Set to `={{ $node['Calculate Experience Metrics'].json.candidateProfile }}`  
     - `metrics` (object): Set to `={{ $node['Calculate Experience Metrics'].json.metrics }}`  
     - `aiAssessment` (string): Set to `={{ $json.answer }}` from AI Assessment node output  
     - `atsReady` (boolean): Set to `true`  
   - Connect output of `PDF Vector - AI Assessment` to this node’s input.

7. **Verify Connections and Test Workflow**  
   - Ensure all nodes are connected in sequence:  
     Manual Trigger → Get Resume from Google Drive → PDF Vector - Parse Resume → Calculate Experience Metrics → PDF Vector - AI Assessment → Create Candidate Profile  
   - Configure all credentials (Google Drive OAuth2, PDF Vector API key).  
   - Test with sample resume file IDs on Google Drive.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                                                                 |
|--------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------|
| Workflow uses PDF Vector AI for both structured data extraction and AI assessment of resumes.    | PDF Vector official docs: https://docs.pdfvector.ai/                                                           |
| Scoring weights are customizable externally as per setup note instructions.                       | Setup note suggests configuring job requirements JSON and scoring weights before running workflows.            |
| Supports multiple resume formats: PDF, Word, LinkedIn exports, and text files.                    | Input Formats sticky note details accepted formats and bulk processing capability.                              |
| Workflow is designed for integration with ATS systems after profile aggregation (flagged `atsReady`). | Integration requires connecting ATS database separately as per setup instructions.                              |
| Uses Google Drive for resume storage and retrieval, requiring Google OAuth2 credentials setup.    | Ensure Google Drive API project is configured properly with file access scopes.                                 |

---

**Disclaimer:** The provided description and workflow analysis are based exclusively on the given n8n workflow JSON. All data handled are legal and public. The workflow respects content policies and does not include any illegal or offensive elements.