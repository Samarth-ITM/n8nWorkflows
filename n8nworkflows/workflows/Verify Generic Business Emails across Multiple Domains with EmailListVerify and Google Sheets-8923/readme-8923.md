Verify Generic Business Emails across Multiple Domains with EmailListVerify and Google Sheets

https://n8nworkflows.xyz/workflows/verify-generic-business-emails-across-multiple-domains-with-emaillistverify-and-google-sheets-8923


# Verify Generic Business Emails across Multiple Domains with EmailListVerify and Google Sheets

### 1. Workflow Overview

This workflow is designed to verify generic business email addresses across multiple domains using the EmailListVerify API, integrating Google Sheets as input and output sources. It targets use cases where users want to validate common generic email patterns (e.g., contact@, accounting@) for a list of domains efficiently.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Retrieve email roots (prefixes) and domain lists from Google Sheets.
- **1.2 Email Candidate Generation:** Combine each email root with each domain to generate candidate email addresses.
- **1.3 Email Validation:** Use the EmailListVerify API to check the validity of each generated email address.
- **1.4 Results Aggregation and Storage:** Combine validation results and append them back into a Google Sheet.
- **1.5 Manual Trigger and User Guidance:** Entry point for manual execution and embedded documentation via sticky notes.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block reads two input lists from Google Sheets: a list of email roots (email name prefixes) and a list of domains to check.

- **Nodes Involved:**  
  - Get list of email root  
  - Get list of domain

- **Node Details:**  

  - **Get list of email root**  
    - Type: Google Sheets node  
    - Role: Reads the sheet named "[Input] partern" (likely intended as "pattern") from a Google Sheets document to get email roots such as "contact", "accounting".  
    - Configuration: Reads from a specific sheet (gid=0) in a predefined Google Sheets document (template).  
    - Credentials: Uses Google Sheets OAuth2 credentials.  
    - Input connections: Triggered by manual execution node.  
    - Output connections: Feeds data into "Create email candidates" node as second input.  
    - Edge cases:  
      - Sheet may be empty or contain invalid data (non-string entries).  
      - Missing or invalid Google Sheets credentials will cause authentication errors.  

  - **Get list of domain**  
    - Type: Google Sheets node  
    - Role: Reads the sheet named "[Input] domain" from the same Google Sheets document to get the list of domains (e.g., example.com).  
    - Configuration: Reads from a specific sheet by gid (2121105756).  
    - Credentials: Uses a possibly different Google Sheets OAuth2 credential (named "Google Sheets (Fab)").  
    - Input connections: Triggered by manual execution node.  
    - Output connections: Feeds data into "Create email candidates" node as first input.  
    - Edge cases:  
      - Domains must not include "http://" or "www." prefixes as per instructions; unexpected format may lead to invalid email generation.  
      - Authentication errors if credentials are missing or expired.

---

#### 1.2 Email Candidate Generation

- **Overview:**  
  This block generates all possible email candidates by combining each email root with each domain, creating strings like "contact@example.com".

- **Nodes Involved:**  
  - Create email candidates

- **Node Details:**  

  - **Create email candidates**  
    - Type: Merge node (mode: combine)  
    - Role: Performs a Cartesian product (combine all) of the two input lists: email roots and domains, generating all combinations.  
    - Configuration: No filters or conditions; simply combines all inputs.  
    - Input connections: Receives domains from "Get list of domain" and email roots from "Get list of email root".  
    - Output connections: Sends combined data to "Use EmailListVerify API to check if email is valid" and "Combine results".  
    - Key expressions: Uses `$json.root` and `$json.domain` to build email addresses in downstream nodes.  
    - Edge cases:  
      - Large input lists may cause combinatorial explosion leading to performance issues or API rate limits.  
      - Input data must be sanitized to avoid malformed emails.

---

#### 1.3 Email Validation

- **Overview:**  
  This block checks each generated email candidate's validity using the EmailListVerify API.

- **Nodes Involved:**  
  - Use EmailListVerify API to check if email is valid

- **Node Details:**  

  - **Use EmailListVerify API to check if email is valid**  
    - Type: HTTP Request node  
    - Role: Sends an HTTP GET request to EmailListVerify API endpoint to validate each email address.  
    - Configuration:  
      - URL: `https://api.emaillistverify.com/api/verifyEmail`  
      - Query parameter: `email` composed dynamically as `{{$json.root}}@{{$json.domain}}`.  
      - Authentication: Generic HTTP header authentication with API key (user must add their EmailListVerify API key in credentials).  
    - Input connections: From "Create email candidates".  
    - Output connections: To "Combine results".  
    - Edge cases:  
      - API key missing or invalid leads to 401 Unauthorized errors.  
      - API rate limits may cause throttling or errors.  
      - Network errors or timeouts possible.  
      - Malformed email addresses may return errors or false negatives.

---

#### 1.4 Results Aggregation and Storage

- **Overview:**  
  This block aggregates all email validation results and appends them to a Google Sheet for record-keeping.

- **Nodes Involved:**  
  - Combine results  
  - Save results

- **Node Details:**  

  - **Combine results**  
    - Type: Merge node (mode: combine by position)  
    - Role: Merges validation results with original email candidate data.  
    - Input connections: Two inputs from "Create email candidates" (original data) and from "Use EmailListVerify API to check if email is valid" (validation results).  
    - Output connections: To "Save results".  
    - Edge cases:  
      - Mismatch in array lengths between inputs may cause data misalignment.  

  - **Save results**  
    - Type: Google Sheets node  
    - Role: Appends the combined data into a Google Sheets document, specifically the sheet named "[OutPut] emails".  
    - Configuration:  
      - Columns mapped:  
        - Email: constructed as `{{$json.root}}@{{$json.domain}}`  
        - Status: mapped from API response field `data`.  
      - Operation: Append rows.  
      - Document and sheet IDs point to the same template file.  
    - Credentials: Uses Google Sheets OAuth2 credentials (named "Google Sheets").  
    - Input connections: From "Combine results".  
    - Edge cases:  
      - Authentication issues if credentials are invalid.  
      - API quota limits in Google Sheets.  
      - Data type mismatches if unexpected API responses occur.

---

#### 1.5 Manual Trigger and User Guidance

- **Overview:**  
  This block facilitates manual triggering of the workflow and provides user instructions via sticky notes.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - Sticky Note, Sticky Note1, Sticky Note2, Sticky Note3, Sticky Note4

- **Node Details:**  

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger node  
    - Role: Entry point that starts the workflow when manually executed by the user.  
    - Output connections: Triggers "Get list of domain" and "Get list of email root".  

  - **Sticky Notes**  
    - Type: Sticky Note nodes  
    - Role: Provide embedded documentation and instructions inside the workflow editor.  
    - Content Highlights:  
      - Sticky Note4: Read-me instructions to copy the Google Sheets template, input email roots and domains, add API key, and trigger the workflow.  
      - Sticky Note: Link to Google Sheets template and reminder to replace with own copy.  
      - Sticky Note1: Explanation of generating email candidates.  
      - Sticky Note2: Instructions to add EmailListVerify API key with link.  
      - Sticky Note3: Reminder to replace target spreadsheet with user’s copy.  
    - Edge cases: None (informational only).

---

### 3. Summary Table

| Node Name                              | Node Type           | Functional Role                              | Input Node(s)                  | Output Node(s)                          | Sticky Note                                                                                                                                                                                 |
|--------------------------------------|---------------------|----------------------------------------------|-------------------------------|----------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’     | Manual Trigger      | Workflow entry point                          | -                             | Get list of domain, Get list of email root | Sticky Note4: Read me with detailed step instructions, including copying template and adding API key.                                                                                       |
| Get list of domain                   | Google Sheets       | Retrieve list of domains                       | When clicking ‘Execute workflow’ | Create email candidates                 | Sticky Note: Link to template spreadsheet and instruction to replace with own copy.                                                                                                         |
| Get list of email root               | Google Sheets       | Retrieve list of email roots (prefixes)       | When clicking ‘Execute workflow’ | Create email candidates                 | Sticky Note: Link to template spreadsheet and instruction to replace with own copy.                                                                                                         |
| Create email candidates              | Merge (combine all) | Generate candidate emails by combining roots and domains | Get list of domain, Get list of email root | Use EmailListVerify API to check if email is valid, Combine results | Sticky Note1: Explanation of generating email candidates by combining roots and domains.                                                                                                       |
| Use EmailListVerify API to check if email is valid | HTTP Request       | Validate email candidates via external API   | Create email candidates         | Combine results                        | Sticky Note2: Instructions to add EmailListVerify API key with link: https://app.emaillistverify.com/api?utm_source=n8n&utm_medium=referral&utm_campaign=genericEmailFinder                  |
| Combine results                     | Merge (combineByPosition) | Aggregate original data and validation results | Create email candidates, Use EmailListVerify API to check if email is valid | Save results                          | Sticky Note3: Save results back to Google Sheets, reminder to replace with own copy of spreadsheet.                                                                                           |
| Save results                       | Google Sheets       | Append validation results to output spreadsheet | Combine results                | -                                      | Sticky Note3: Save results back to Google Sheets, reminder to replace with own copy of spreadsheet.                                                                                           |
| Sticky Note                       | Sticky Note         | User guidance and documentation               | -                             | -                                      | See individual sticky note content in block 1.5.                                                                                                                                             |
| Sticky Note1                      | Sticky Note         | User guidance and documentation               | -                             | -                                      | See individual sticky note content in block 1.5.                                                                                                                                             |
| Sticky Note2                      | Sticky Note         | User guidance and documentation               | -                             | -                                      | See individual sticky note content in block 1.5.                                                                                                                                             |
| Sticky Note3                      | Sticky Note         | User guidance and documentation               | -                             | -                                      | See individual sticky note content in block 1.5.                                                                                                                                             |
| Sticky Note4                      | Sticky Note         | User guidance and documentation               | -                             | -                                      | See individual sticky note content in block 1.5.                                                                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a "Manual Trigger" node named "When clicking ‘Execute workflow’".  
   - This node will start the workflow manually.

2. **Create Google Sheets Node to Get Domains**  
   - Add a "Google Sheets" node named "Get list of domain".  
   - Set operation to "Read Rows".  
   - Set Sheet Name to the domain input sheet (e.g., "[Input] domain" sheet by its GID or name).  
   - Set Document ID to your Google Sheets template copy.  
   - Configure Google Sheets OAuth2 credentials for access.  
   - Connect "When clicking ‘Execute workflow’" output to this node.

3. **Create Google Sheets Node to Get Email Roots**  
   - Add a "Google Sheets" node named "Get list of email root".  
   - Set operation to "Read Rows".  
   - Set Sheet Name to the email root input sheet (e.g., "[Input] partern" sheet by GID or name).  
   - Use the same Google Sheets document ID as above.  
   - Configure OAuth2 credentials.  
   - Connect "When clicking ‘Execute workflow’" output to this node.

4. **Create Merge Node to Generate Email Candidates**  
   - Add a "Merge" node named "Create email candidates".  
   - Set mode to "Combine".  
   - Connect two inputs: first from "Get list of domain" (index 0), second from "Get list of email root" (index 1).  
   - This will perform a Cartesian product of domains and roots.

5. **Create HTTP Request Node for Email Validation**  
   - Add an "HTTP Request" node named "Use EmailListVerify API to check if email is valid".  
   - Set HTTP Method: GET.  
   - Set URL to `https://api.emaillistverify.com/api/verifyEmail`.  
   - Add Query Parameter: `email` with value `={{ $json.root }}@{{ $json.domain }}` to dynamically build email addresses.  
   - Set Authentication to "HTTP Header Auth".  
   - Add your EmailListVerify API key in the header (e.g., `Authorization` or as per API docs).  
   - Connect input from "Create email candidates".

6. **Create Merge Node to Combine Results**  
   - Add a "Merge" node named "Combine results".  
   - Set mode to "Combine by Position".  
   - Connect first input from "Create email candidates".  
   - Connect second input from "Use EmailListVerify API to check if email is valid".

7. **Create Google Sheets Node to Save Results**  
   - Add a "Google Sheets" node named "Save results".  
   - Set operation to "Append".  
   - Set Sheet Name to the output sheet (e.g., "[OutPut] emails").  
   - Set Document ID to your Google Sheets copy.  
   - Map columns:  
     - Email: `={{ $json.root }}@{{ $json.domain }}`  
     - Status: `={{ $json.data }}` (field from API response)  
   - Configure OAuth2 credentials.  
   - Connect input from "Combine results".

8. **Add Sticky Notes for Documentation (Optional but Recommended)**  
   - Add sticky notes with content describing each step, API key instructions, template links, and usage notes to improve maintainability.

9. **Save and Test Workflow**  
   - Execute manually using the trigger node.  
   - Monitor logs and verify that email candidates are generated, validated, and results appended to Google Sheets.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                | Context or Link                                                                                                                 |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Make a copy of the Google Sheets template before running the workflow. Replace references to the template in all Google Sheets nodes with your copy's document ID.                                                                                                                                          | [Google Sheets Template](https://docs.google.com/spreadsheets/d/11JW2e9w00bZO_ORe0FNaK_u5tnchI8ZOwcq8qN1toZw/edit?usp=sharing) |
| Add your EmailListVerify API key into the HTTP Request node credentials before executing. This is required for validation calls to succeed.                                                                                                                                                                 | [EmailListVerify API Signup and Documentation](https://app.emaillistverify.com/api?utm_source=n8n&utm_medium=referral&utm_campaign=genericEmailFinder) |
| Input constraints: domains should not include "http://" or "www." prefixes; email roots should be simple strings like "contact" or "accounting".                                                                                                                                                            | See workflow input sheets "[Input] domain" and "[Input] partern".                                                                |
| Large input lists may lead to many API calls and potential rate limiting; consider batching or throttling if needed.                                                                                                                                                                                        | Best practice for scalability and error handling.                                                                                |
| The workflow is designed for manual execution and not yet optimized for automated or scheduled runs; consider adding error handling and retry logic for production use.                                                                                                                                     | General operational advice.                                                                                                      |

---

**Disclaimer:**  
The text above is exclusively derived from an automated n8n workflow configuration. It respects all applicable content policies and contains no illegal or offensive material. All data handled is public and legal.