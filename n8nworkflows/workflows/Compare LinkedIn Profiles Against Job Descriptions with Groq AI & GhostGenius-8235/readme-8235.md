Compare LinkedIn Profiles Against Job Descriptions with Groq AI & GhostGenius

https://n8nworkflows.xyz/workflows/compare-linkedin-profiles-against-job-descriptions-with-groq-ai---ghostgenius-8235


# Compare LinkedIn Profiles Against Job Descriptions with Groq AI & GhostGenius

### 1. Workflow Overview

This workflow automates the comparison of LinkedIn profiles (Candidate CVs) against Job Descriptions (JDs) using AI-powered analysis. It targets recruiters, hiring managers, and job seekers who need a quick, data-driven evaluation of candidate fit through an ATS-style report. The workflow integrates external APIs for data extraction, processes structured JSON objects, and uses a Groq AI language model for semantic matching and recommendations.

**Logical Blocks:**

- **1.1 Input Reception & Validation**  
  Captures LinkedIn profile and job description URLs via webhook and validates their format.

- **1.2 Data Fetching**  
  Retrieves detailed profile and job description data from GhostGenius API.

- **1.3 Data Structuring**  
  Transforms raw profile and job data into structured JSON objects representing CV and JD.

- **1.4 Data Aggregation & Merging**  
  Aggregates multiple structured data items and merges CV and JD into a combined dataset.

- **1.5 AI-Powered Analysis**  
  Uses Groq AI and LangChain to semantically compare CV vs JD according to a strict schema.

- **1.6 Result Formatting & Response**  
  Parses AI JSON output, formats it into an HTML summary, and responds to the webhook.

- **1.7 Error Handling**  
  Handles invalid input URLs and provides user feedback.


---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation

- **Overview:**  
  This block captures incoming HTTP POST requests containing LinkedIn URLs for CV and Job Description, then validates these URLs to ensure they contain "linkedin.com/". If validation fails, it returns an error response.

- **Nodes Involved:**  
  - Webhook (GET Form details)  
  - If  
  - Error_node

- **Node Details:**

  - **Webhook (GET Form details)**  
    - Type: Webhook Trigger  
    - Configuration: Listens on path `linkedin` with POST method, basic auth enabled for security.  
    - Inputs: External HTTP POST with JSON body containing `LinkedIn_CV` and `LinkedIn_JD` URLs.  
    - Outputs: Forwards payload to "If" node.  
    - Failures: Unauthorized access if basic auth fails; malformed requests may cause errors downstream.

  - **If**  
    - Type: Conditional Logic  
    - Configuration: Checks if both URLs contain substring "linkedin.com/".  
    - Inputs: Data from webhook.  
    - Outputs:  
      - True branch: Proceeds to fetch profile and job details.  
      - False branch: Routes to Error_node.  
    - Edge Cases: URLs may be syntactically valid but not accessible; this node only checks substring presence.

  - **Error_node**  
    - Type: Respond to Webhook  
    - Configuration: Returns a styled HTML error message for invalid URLs.  
    - Inputs: Triggered from false condition in If node.  
    - Outputs: Ends workflow with error response to client.  
    - Edge Cases: None; purely user notification.

---

#### 2.2 Data Fetching

- **Overview:**  
  Fetches detailed profile and job description data from the GhostGenius API using provided LinkedIn URLs.

- **Nodes Involved:**  
  - Get profile details​  
  - Get job details​

- **Node Details:**

  - **Get profile details​**  
    - Type: HTTP Request  
    - Configuration: Sends GET request to `https://api.ghostgenius.fr/v2/profile` with query param `url` set to the LinkedIn CV URL. Uses HTTP header authentication with stored credentials.  
    - Inputs: Validated LinkedIn CV URL from If node.  
    - Outputs: JSON response containing structured profile data (headline, summary, skills, experiences, etc.).  
    - Failures: API auth errors, network timeouts, invalid or inaccessible URL inputs.

  - **Get job details​**  
    - Type: HTTP Request  
    - Configuration: Sends GET request to `https://api.ghostgenius.fr/v2/job` with query param `url` set to the LinkedIn JD URL. Uses HTTP header authentication.  
    - Inputs: Validated LinkedIn JD URL from If node.  
    - Outputs: JSON response containing job description details (title, description, contract type, company info, etc.).  
    - Failures: Same risks as profile details node.

---

#### 2.3 Data Structuring

- **Overview:**  
  Converts raw API responses into structured, explicit JSON objects for CV and JD by selecting and labeling relevant fields.

- **Nodes Involved:**  
  - Build CV  
  - Build JD

- **Node Details:**

  - **Build CV**  
    - Type: Set  
    - Configuration: Assigns named fields such as `headline`, `summary`, `languages`, `experiences`, `skills`, and boolean flags (`is_premium`, `is_creator`, etc.) extracted from the profile JSON.  
    - Inputs: JSON from "Get profile details​".  
    - Outputs: Structured CV JSON object.  
    - Edge Cases: Missing fields in source JSON result in null or empty values.

  - **Build JD**  
    - Type: Set  
    - Configuration: Assigns named fields such as `title`, `description`, `contract_type`, `company`, `location`, etc. extracted from job JSON.  
    - Inputs: JSON from "Get job details​".  
    - Outputs: Structured JD JSON object.  
    - Edge Cases: Same as Build CV.

---

#### 2.4 Data Aggregation & Merging

- **Overview:**  
  Aggregates all data items into single JSON arrays and merges the CV and JD datasets for joint analysis.

- **Nodes Involved:**  
  - Combine_CV  
  - Combine_JD  
  - Merge2  
  - ATS compare

- **Node Details:**

  - **Combine_CV**  
    - Type: Aggregate  
    - Configuration: Aggregates all CV JSON items into an array named `CV`.  
    - Inputs: Output from "Build CV".  
    - Outputs: Single item with `CV` array.  
    - Edge Cases: Empty input results in empty array.

  - **Combine_JD**  
    - Type: Aggregate  
    - Configuration: Aggregates all JD JSON items into an array named `JD`.  
    - Inputs: Output from "Build JD".  
    - Outputs: Single item with `JD` array.  
    - Edge Cases: Same as Combine_CV.

  - **Merge2**  
    - Type: Merge  
    - Configuration: Combines the `CV` and `JD` arrays into one data structure using "combineAll" mode.  
    - Inputs: Outputs from Combine_CV and Combine_JD.  
    - Outputs: Single combined item containing both `CV` and `JD` arrays.  
    - Edge Cases: Missing either input breaks merge.

  - **ATS compare**  
    - Type: Set  
    - Configuration: Prepares a JSON object containing the merged `CV` and `JD` arrays to pass to the AI analysis node.  
    - Inputs: Output from Merge2.  
    - Outputs: Structured input for AI evaluation.

---

#### 2.5 AI-Powered Analysis

- **Overview:**  
  Uses a Groq AI language model and LangChain agent node to semantically analyze and compare CV and JD JSON data, producing a structured match report conforming to a strict schema.

- **Nodes Involved:**  
  - Groq Chat Model4  
  - Recruiter Check

- **Node Details:**

  - **Groq Chat Model4**  
    - Type: Groq AI Language Model (LangChain integration)  
    - Configuration: Uses model `moonshotai/kimi-k2-instruct` with temperature 0.5 for balanced creativity and determinism.  
    - Credentials: Groq API key stored securely.  
    - Inputs: Text prompt generated by Recruiter Check node's prompt.  
    - Outputs: AI-generated JSON analysis string.  
    - Failures: Authentication errors, rate limits, or malformed prompts.

  - **Recruiter Check**  
    - Type: LangChain Agent Node  
    - Configuration:  
      - Input prompt embeds the full JSON arrays for JD and CV.  
      - System message instructs the AI to produce ONLY a valid JSON object conforming exactly to the `Match_Result` schema with fields: status, reasoning, recommendation, matched_keywords, missing_keywords_required, missing_keywords_nice_to_have, optimization_tips, and location_match.  
      - Strict output parsing enabled to enforce JSON-only response.  
    - Inputs: Merged CV and JD arrays from ATS compare node.  
    - Outputs: Parsed JSON object with match analysis.  
    - Edge Cases: AI may output invalid JSON or fail to parse; fallback handling is needed.

---

#### 2.6 Result Formatting & Response

- **Overview:**  
  Parses the AI JSON output, creates a user-friendly HTML summary, and responds to the original HTTP request with the analysis results.

- **Nodes Involved:**  
  - ThankYOU message  
  - Respond to Webhook

- **Node Details:**

  - **ThankYOU message**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Parses AI JSON output safely; catches JSON parse errors.  
      - Generates an HTML snippet summarizing status, reasoning, recommendation, matched keywords, missing required/nice-to-have keywords, and optimization tips as bullet lists.  
      - Returns this HTML as a field `thankYouMessage`.  
    - Inputs: JSON from Recruiter Check node.  
    - Outputs: JSON with rendered HTML string.  
    - Edge Cases: Invalid JSON input results in error field with details.

  - **Respond to Webhook**  
    - Type: Respond to Webhook  
    - Configuration: Responds to the original webhook call with the JSON containing the HTML summary.  
    - Inputs: Output from ThankYOU message node.  
    - Outputs: HTTP response to client.  
    - Edge Cases: Response failures may occur if webhook times out or client disconnects.

---

#### 2.7 Error Handling (Summary)

- The workflow includes explicit URL validation and returns an HTML error message if LinkedIn URLs are invalid or missing "linkedin.com/".  
- API calls to GhostGenius may fail due to auth issues, network errors, or invalid URLs, which should be monitored externally.  
- AI node may produce invalid JSON; the code node attempts to parse safely and reports errors if any.

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                       | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                                                                    |
|-----------------------------|----------------------------------|-------------------------------------|-----------------------------|----------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| Webhook (GET Form details)  | Webhook                          | Input reception and entry point     | —                           | If                         |                                                                                                                                                |
| If                          | If                               | Validate LinkedIn URLs               | Webhook (GET Form details)  | Get profile details​ / Get job details​ / Error_node |                                                                                                                                               |
| Error_node                  | Respond to Webhook               | Return error on invalid URLs        | If                          | —                          |                                                                                                                                               |
| Get profile details​         | HTTP Request                    | Fetch LinkedIn profile data          | If (true branch)            | Build CV                   |                                                                                                                                               |
| Get job details​             | HTTP Request                    | Fetch LinkedIn job description data  | If (true branch)            | Build JD                   |                                                                                                                                               |
| Build CV                    | Set                             | Structure profile data into CV JSON | Get profile details​         | Combine_CV                 |                                                                                                                                               |
| Build JD                    | Set                             | Structure job description data       | Get job details​             | Combine_JD                 |                                                                                                                                               |
| Combine_CV                  | Aggregate                       | Aggregate CV items into array        | Build CV                    | Merge2                     |                                                                                                                                               |
| Combine_JD                  | Aggregate                       | Aggregate JD items into array        | Build JD                    | Merge2                     |                                                                                                                                               |
| Merge2                      | Merge                          | Combine CV and JD arrays             | Combine_CV, Combine_JD       | ATS compare                |                                                                                                                                               |
| ATS compare                 | Set                             | Prepare merged data for AI           | Merge2                      | Recruiter Check            |                                                                                                                                               |
| Groq Chat Model4            | LangChain Groq AI Model         | AI language model for analysis       | Recruiter Check (ai_languageModel input) | Recruiter Check (ai_languageModel output) |                                                                                                                                               |
| Recruiter Check             | LangChain Agent Node            | AI semantic comparison and scoring   | ATS compare, Groq Chat Model4 | ThankYOU message           |                                                                                                                                               |
| ThankYOU message            | Code                            | Parse AI JSON and format HTML        | Recruiter Check             | Respond to Webhook         |                                                                                                                                               |
| Respond to Webhook          | Respond to Webhook              | Final HTTP response to client        | ThankYOU message            | —                          |                                                                                                                                               |
| Workflow Description (Sticky Note)1 | Sticky Note                    | Workflow description and instructions | —                           | —                          | Contains detailed workflow description, requirements, and customization tips.                                                                 |
| Analysis Result (Sticky Note) | Sticky Note                    | Example ATS analysis result summary  | —                           | —                          | Provides example output with reasoning, matched/missing keywords, recommendations, and optimization tips.                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Name: `Webhook (GET Form details)`  
   - Type: Webhook  
   - Path: `linkedin`  
   - HTTP Method: POST  
   - Authentication: Basic Auth (configure credentials for LinkedIn proof-of-concept tool)  
   - Allowed Origins: `*`  
   - Purpose: Receive LinkedIn CV and JD URLs in JSON body fields `LinkedIn_CV` and `LinkedIn_JD`.

2. **Add If Node**  
   - Name: `If`  
   - Type: If (Condition)  
   - Condition: Check if both `$json.body.LinkedIn_CV` and `$json.body.LinkedIn_JD` contain substring `"linkedin.com/"` (case sensitive).  
   - True branch: Proceed to data fetching  
   - False branch: Route to error response.

3. **Create Error Response Node**  
   - Name: `Error_node`  
   - Type: Respond to Webhook  
   - Response: JSON with HTML content indicating "Invalid URL" and instructions to retry.

4. **Create HTTP Request Nodes**  
   - Name: `Get profile details​`  
     - Method: GET  
     - URL: `https://api.ghostgenius.fr/v2/profile`  
     - Query Parameter: `url` set to `={{ $json.body.LinkedIn_CV }}`  
     - Authentication: HTTP Header Auth (use stored LinkedIn API credentials)  
   - Name: `Get job details​`  
     - Method: GET  
     - URL: `https://api.ghostgenius.fr/v2/job`  
     - Query Parameter: `url` set to `={{ $json.body.LinkedIn_JD }}`  
     - Authentication: HTTP Header Auth (same credentials)

5. **Add Set Nodes to Build JSON**  
   - Name: `Build CV`  
   - Type: Set  
   - Assign fields explicitly from profile JSON (e.g., `headline`, `summary`, `skills`, etc.)  
   - Name: `Build JD`  
   - Type: Set  
   - Assign fields explicitly from job JSON (e.g., `title`, `description`, `contract_type`, `company`, etc.)

6. **Add Aggregate Nodes**  
   - Name: `Combine_CV`  
   - Type: Aggregate  
   - Aggregate all items from `Build CV` into a single array under field `CV`.  
   - Name: `Combine_JD`  
   - Type: Aggregate  
   - Aggregate all items from `Build JD` into a single array under field `JD`.

7. **Add Merge Node**  
   - Name: `Merge2`  
   - Type: Merge  
   - Mode: Combine  
   - Combine all data from `Combine_CV` and `Combine_JD` into one item containing both arrays.

8. **Add Set Node for ATS Compare Input**  
   - Name: `ATS compare`  
   - Type: Set  
   - Create a new JSON with fields `CV` and `JD` populated from merged data.

9. **Add Groq AI Language Model Node**  
   - Name: `Groq Chat Model4`  
   - Type: `@n8n/n8n-nodes-langchain.lmChatGroq`  
   - Model: `moonshotai/kimi-k2-instruct`  
   - Temperature: 0.5  
   - Credentials: Configure Groq API credentials securely.

10. **Add LangChain Agent Node**  
    - Name: `Recruiter Check`  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Set prompt embedding the JSON arrays from `ATS compare` node.  
    - System Message: Provide detailed instructions to output ONLY valid JSON matching the `Match_Result` schema with explanations, recommendations, and keyword matching logic as per the original prompt.  
    - Enable output parser to enforce JSON-only response.  
    - Connect Groq Chat Model as AI language model input.

11. **Add Code Node to Format Output**  
    - Name: `ThankYOU message`  
    - Type: Code (JavaScript)  
    - Parse `output` JSON string from `Recruiter Check` node safely.  
    - Generate HTML snippet presenting status, reasoning, recommendations, matched/missing keywords, and optimization tips as bullet lists.  
    - Return JSON with `thankYouMessage` field.

12. **Add Respond to Webhook Node**  
    - Name: `Respond to Webhook`  
    - Type: Respond to Webhook  
    - Respond with JSON containing the HTML summary from `ThankYOU message`.  
    - Final node to complete workflow request.

13. **Connect Nodes in Order**  
    - Webhook -> If  
    - If (true) -> Get profile details​ and Get job details​ (parallel)  
    - Get profile details​ -> Build CV -> Combine_CV  
    - Get job details​ -> Build JD -> Combine_JD  
    - Combine_CV and Combine_JD -> Merge2 -> ATS compare  
    - ATS compare -> Recruiter Check (with Groq Chat Model4 as AI language model input)  
    - Recruiter Check -> ThankYOU message -> Respond to Webhook  
    - If (false) -> Error_node

14. **Credentials Setup**  
    - Configure HTTP Header Auth credentials for GhostGenius API access (LinkedIn scraper).  
    - Configure Groq API credentials for AI language model access.  
    - Configure Basic Auth credentials for webhook security.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                       | Context or Link                                                                                          |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| This workflow automates **CV vs JD matching** using LinkedIn profile data, job descriptions, and an AI recruiter check. It provides an ATS-style structured analysis with recommendations and optimization tips.                                                                       | Sticky Note "Workflow Description (Sticky Note)1"                                                     |
| Requirements include a LinkedIn CV/JD scraper API such as GhostGenius, a Groq AI account, and an n8n instance. Credentials must be securely stored; do not hardcode keys in nodes.                                                                                                | Sticky Note "Workflow Description (Sticky Note)1"                                                     |
| The AI prompt enforces strict JSON output conforming to the `Match_Result` schema with detailed reasoning, matched/missing keywords, and recommendations.                                                                                                                        | Recruiter Check node system message                                                                     |
| Example analysis output includes status categories (mismatch, core_match, good_match), reasons, matched/missing keywords, and practical CV optimization tips.                                                                                                                      | Sticky Note "Analysis Result (Sticky Note)"                                                            |
| To customize, adjust the AI prompt for different ATS scoring rules or change the final code node output to other formats (plain text, Slack message blocks, PDF, etc.). Integrate further with Slack, Gmail, Notion for automated report distribution.                              | Sticky Note "Workflow Description (Sticky Note)1"                                                     |
| Ensure n8n version compatibility for LangChain nodes and Groq AI integration (version 1 or above recommended). HTTP Request nodes use version 4.2 to support advanced authentication.                                                                                              | Inferred from node configurations                                                                       |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow and complies with all applicable content policies. It contains no illegal, offensive, or protected elements, and all data handled is legal and publicly accessible.