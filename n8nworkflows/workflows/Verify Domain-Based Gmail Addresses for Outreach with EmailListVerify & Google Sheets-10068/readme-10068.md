Verify Domain-Based Gmail Addresses for Outreach with EmailListVerify & Google Sheets

https://n8nworkflows.xyz/workflows/verify-domain-based-gmail-addresses-for-outreach-with-emaillistverify---google-sheets-10068


# Verify Domain-Based Gmail Addresses for Outreach with EmailListVerify & Google Sheets

### 1. Workflow Overview

This workflow is designed to support link-building outreach by discovering potential Gmail-based (and other email provider) addresses that correspond to given domain names. It generates candidate emails by combining domain roots with email extensions, verifies these emails via the EmailListVerify API, and stores the validated results into a Google Sheet for further use.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Preparation**: Retrieve domain names and email extensions from Google Sheets, then clean domain URLs to extract domain roots.
- **1.2 Email Candidate Generation**: Combine domain roots with email extensions to create a list of candidate email addresses.
- **1.3 Email Validation**: Use the EmailListVerify API to check if each candidate email is valid.
- **1.4 Results Aggregation and Storage**: Merge verification results and append them to an output Google Sheet.

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception and Preparation

**Overview:**  
This block fetches the source data required for email generation: domain names and email extensions. It also processes domain URLs to extract clean domain roots suitable for email address construction.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Get list of domain (Google Sheets)  
- Get list of email extension (Google Sheets)  
- Transform website into domain name (Code)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point to start the workflow manually.  
  - Configuration: No parameters; triggers downstream nodes on manual execution.  
  - Connections: Outputs to "Get list of domain" and "Get list of email extension" nodes.  
  - Edge cases: None typical, but workflow will not run unless triggered.

- **Get list of domain**  
  - Type: Google Sheets  
  - Role: Reads domain names from the "[Input] domain" sheet of a specific Google Sheet document.  
  - Configuration: Reads from sheet with GID 2121105756 in the referenced spreadsheet. Uses OAuth2 credentials for access.  
  - Input: Trigger node.  
  - Output: JSON array of rows containing domain URLs.  
  - Edge cases: Auth errors (OAuth token issues), sheet access/permission issues, empty or malformed domain cells.

- **Get list of email extension**  
  - Type: Google Sheets  
  - Role: Reads email extensions (like gmail.com, laposte.net) from the "[Input] pattern" sheet of the same Google Sheet document.  
  - Configuration: Reads from sheet at GID 0 in the same spreadsheet. Uses OAuth2 credentials.  
  - Input: Trigger node.  
  - Output: JSON array of email extensions.  
  - Edge cases: Similar to "Get list of domain", including empty extension list or malformed entries.

- **Transform website into domain name**  
  - Type: Code (JavaScript)  
  - Role: Cleans domain URLs by removing protocols (http/https) and “www.” prefixes, then extracts the root domain part (substring before the first dot).  
  - Configuration: Custom JS code iterating over input items, parsing domain strings, and adding a new property `root` with the domain root.  
  - Expressions/Variables: Uses `$input.all()` to access all input data; processes each domain string.  
  - Input: Output from "Get list of domain".  
  - Output: JSON array with domain roots under `root`.  
  - Edge cases: Domains without dots, malformed URLs, empty strings, or missing `domain` field handled with try/catch but may log errors without stopping workflow.

---

#### 2.2 Email Candidate Generation

**Overview:**  
This block cross-combines each domain root with each email extension to form potential email addresses for validation.

**Nodes Involved:**  
- Create email candidates (Merge)

**Node Details:**

- **Create email candidates**  
  - Type: Merge  
  - Role: Combines two input streams (domains roots and email extensions) in “combine” mode to produce all possible pairs.  
  - Configuration: Mode set to "combineAll" to cross-product all items from both inputs.  
  - Input:  
    - First input: Output from "Transform website into domain name" (domain roots).  
    - Second input: Output from "Get list of email extension" (email extensions).  
  - Output: JSON items combining root and extension for downstream processing.  
  - Edge cases: If either input is empty, output will be empty; large inputs may cause performance issues due to combinatorial explosion.

---

#### 2.3 Email Validation

**Overview:**  
Checks each candidate email's validity using the EmailListVerify API.

**Nodes Involved:**  
- Use EmailListVerify API to check if email is valid (HTTP Request)

**Node Details:**

- **Use EmailListVerify API to check if email is valid**  
  - Type: HTTP Request  
  - Role: Sends GET request to EmailListVerify API with candidate email as a query parameter for validation.  
  - Configuration:  
    - URL: `https://api.emaillistverify.com/api/verifyEmail`  
    - Query parameter: `email` constructed as `${root}@${extension}` using expressions.  
    - Authentication: Uses generic HTTP Header Authentication with credentials storing the API key.  
  - Input: Output from "Create email candidates".  
  - Output: API response containing email validation status.  
  - Edge cases:  
    - API rate limits or quota exceeded.  
    - HTTP errors (timeouts, connectivity issues).  
    - Invalid or missing API key credential.  
    - Unexpected API response format.  
  - Requirements: User must add their EmailListVerify API key in credentials.

---

#### 2.4 Results Aggregation and Storage

**Overview:**  
Aggregates the validated email data and stores the results by appending them to a specified Google Sheet.

**Nodes Involved:**  
- Combine results (Merge)  
- Save results (Google Sheets)

**Node Details:**

- **Combine results**  
  - Type: Merge  
  - Role: Synchronizes the outputs from the email validation node and the candidate emails node by position to combine their data for final output.  
  - Configuration: Mode set to "combineByPosition".  
  - Input:  
    - First input: Output from "Use EmailListVerify API to check if email is valid" (validation results).  
    - Second input: Output from "Create email candidates" (candidate emails).  
  - Output: Combined JSON containing email and validation status.  
  - Edge cases: Mismatched array lengths or missing data could cause errors or incomplete merges.

- **Save results**  
  - Type: Google Sheets  
  - Role: Appends combined email and verification status data to the “[OutPut] emails” sheet of the Google Sheet document.  
  - Configuration:  
    - Columns mapped:  
      - `Email` formatted as `${root}@${extension}`  
      - `Status` from API validation response field `data`  
    - Append operation to sheet with GID 1262572795.  
    - Uses Google OAuth2 credentials.  
  - Input: Output from "Combine results".  
  - Output: None (side effect of appending to Google Sheet).  
  - Edge cases:  
    - Google Sheets API errors (auth, rate limits, sheet access).  
    - Data type mismatches or empty data causing malformed rows.

---

### 3. Summary Table

| Node Name                                 | Node Type         | Functional Role                            | Input Node(s)                  | Output Node(s)                | Sticky Note                                                                                   |
|-------------------------------------------|-------------------|--------------------------------------------|-------------------------------|------------------------------|-----------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’          | Manual Trigger    | Start the workflow manually                 | —                             | Get list of domain, Get list of email extension |                                                                                               |
| Get list of domain                        | Google Sheets     | Retrieve domain URLs from input sheet       | When clicking ‘Execute workflow’ | Transform website into domain name | ## Get inputs Make a copy of [the template](https://docs.google.com/spreadsheets/d/1r4DZ4GnqKzivmIhRdv1D35fvS_Mg-VTKgbuZS-7H-HY/edit?usp=sharing) Replace the target spreadsheet with your copy of the template |
| Get list of email extension               | Google Sheets     | Retrieve email extensions from input sheet | When clicking ‘Execute workflow’ | Create email candidates        | ## Get inputs Make a copy of [the template](https://docs.google.com/spreadsheets/d/1r4DZ4GnqKzivmIhRdv1D35fvS_Mg-VTKgbuZS-7H-HY/edit?usp=sharing) Replace the target spreadsheet with your copy of the template |
| Transform website into domain name        | Code              | Clean and extract domain roots from URLs    | Get list of domain              | Create email candidates        | ## clean input remove http and www from website urls                                          |
| Create email candidates                   | Merge             | Cross combine domains and email extensions | Transform website into domain name, Get list of email extension | Use EmailListVerify API to check if email is valid | ## Generate email candidates Combine each domain and email root to generate email candidates |
| Use EmailListVerify API to check if email is valid | HTTP Request      | Validate candidate emails via EmailListVerify API | Create email candidates         | Combine results               | ## Check if email is valid Use EmailListVerify API to check if each email is valid. Add your [EmailListVerify API key](https://app.emaillistverify.com/api?utm_source=n8n&utm_medium=referral&utm_campaign=GmailFinder). |
| Combine results                          | Merge             | Combine candidate emails with validation results | Use EmailListVerify API to check if email is valid, Create email candidates | Save results                  |                                                                                               |
| Save results                            | Google Sheets     | Append validated emails to output sheet     | Combine results                 | —                            | ## Save results Replace the target spreadsheet with your copy of the template                 |
| Sticky Note4                           | Sticky Note       | Workflow usage instructions                  | —                             | —                            | ## Read me This workflow is design for link building. When you outreach some small blog it is common for the owner to have an address like DomainName@gmail.com. This workflow will find such emails for you. **1:** Make a copy of the [GoogleSheet template](https://docs.google.com/spreadsheets/d/1r4DZ4GnqKzivmIhRdv1D35fvS_Mg-VTKgbuZS-7H-HY/edit?usp=sharing) **2:** In "[Input] pattern" sheet write the email extension you want to check. Gmail is a no brainer. Depending on location you might want to include local email provider like laposte.net for France. **3:** In "[Input] domain" put the domain for which you want to find email addresses. **4:** Add your [EmailListVerify API key](https://app.emaillistverify.com/api?utm_source=n8n&utm_medium=referral&utm_campaign=GmaimFinder) to setting to the 3rd step **5:** Update google sheet node to point to your copy of the template **6:** Trigger the workflow |
| Sticky Note                            | Sticky Note       | Reminder on inputs and template copy         | —                             | —                            | ## Get inputs Make a copy of [the template](https://docs.google.com/spreadsheets/d/1r4DZ4GnqKzivmIhRdv1D35fvS_Mg-VTKgbuZS-7H-HY/edit?usp=sharing) Replace the target spreadsheet with your copy of the template |
| Sticky Note1                           | Sticky Note       | Email candidate generation explanation       | —                             | —                            | ## Generate email candidates Combine each domain and email root to generate email candidates |
| Sticky Note2                           | Sticky Note       | Email verification explanation                | —                             | —                            | ## Check if email is valid Use EmailListVerify API to check if each email is valid. Add you [EmailListVerify API key](https://app.emaillistverify.com/api?utm_source=n8n&utm_medium=referral&utm_campaign=GmailFinder). |
| Sticky Note3                           | Sticky Note       | Saving results reminder                        | —                             | —                            | ## Save results Replace the target spreadsheet with your copy of the template                 |
| Sticky Note5                           | Sticky Note       | Input cleaning explanation                      | —                             | —                            | ## clean input remove http and www from website urls                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Purpose: To manually start the workflow.

2. **Add a Google Sheets node to get domain list**  
   - Name: `Get list of domain`  
   - Operation: Read rows  
   - Sheet: `[Input] domain` (sheet with GID 2121105756)  
   - Document ID: Your copied Google Sheet’s ID  
   - Credentials: Configure Google Sheets OAuth2 credentials with read access.

3. **Add a Google Sheets node to get email extensions**  
   - Name: `Get list of email extension`  
   - Operation: Read rows  
   - Sheet: `[Input] pattern` (sheet with GID 0)  
   - Document ID: Same as above  
   - Credentials: Use same Google Sheets OAuth2 credentials.

4. **Connect Manual Trigger outputs to both Google Sheets nodes.**

5. **Add a Code node to transform domains**  
   - Name: `Transform website into domain name`  
   - Code (JavaScript):  
     ```javascript
     const data = $input.all();
     let merged = [];
     for (let line of data) {
       try {
         let lineObject = line.json;
         let url = lineObject.domain;
         let domainEnd = url.indexOf(".");
         let domain = url.slice(0, domainEnd);
         if (domain.startsWith("www.")) {
           domain = domain.slice(4);
         }
         lineObject.root = domain;
         merged.push(lineObject);
       } catch (e) {
         // Log and skip errors
         console.log(e);
       }
     }
     return merged;
     ```
   - Input: Connect from `Get list of domain`.

6. **Add a Merge node to create email candidates**  
   - Name: `Create email candidates`  
   - Mode: Combine (combineAll)  
   - Inputs:  
     - Input 1: Output from `Transform website into domain name`  
     - Input 2: Output from `Get list of email extension`

7. **Add an HTTP Request node to verify emails**  
   - Name: `Use EmailListVerify API to check if email is valid`  
   - HTTP Method: GET  
   - URL: `https://api.emaillistverify.com/api/verifyEmail`  
   - Query Parameters:  
     - `email` = Expression: `{{$json.root}}@{{$json.extension}}`  
   - Authentication: HTTP Header Authentication with API key credential  
   - Credential Setup: Create credentials with your EmailListVerify API key.  
   - Input: Connect from `Create email candidates`.

8. **Add a Merge node to combine validation results with email candidates**  
   - Name: `Combine results`  
   - Mode: Combine by Position  
   - Inputs:  
     - Input 1: Output from `Use EmailListVerify API to check if email is valid`  
     - Input 2: Output from `Create email candidates`

9. **Add a Google Sheets node to save results**  
   - Name: `Save results`  
   - Operation: Append rows  
   - Sheet: `[OutPut] emails` (sheet with GID 1262572795)  
   - Document ID: Same Google Sheet ID  
   - Columns to map:  
     - `Email`: Expression `={{ $json.root }}@{{ $json.extension }}`  
     - `Status`: Expression `={{ $json.data }}` (API response field)  
   - Credentials: Use Google Sheets OAuth2 credentials.  
   - Input: Connect from `Combine results`.

10. **Connect nodes according to the original flow:**  
    - Manual Trigger → Get list of domain  
    - Manual Trigger → Get list of email extension  
    - Get list of domain → Transform website into domain name  
    - Transform website into domain name → Create email candidates (input 1)  
    - Get list of email extension → Create email candidates (input 2)  
    - Create email candidates → Use EmailListVerify API  
    - Use EmailListVerify API → Combine results (input 1)  
    - Create email candidates → Combine results (input 2)  
    - Combine results → Save results

11. **Test the workflow by running the manual trigger.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                        | Context or Link                                                                                                          |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| This workflow is designed for link building outreach. It finds email addresses in the format DomainName@gmail.com or similar by combining domain roots with common email provider extensions.                                                                                                   | Sticky Note4 content                                                                                                      |
| Make a copy of the Google Sheets template before running the workflow to customize inputs and outputs.                                                                                                                                                                                           | [Google Sheets template link](https://docs.google.com/spreadsheets/d/1r4DZ4GnqKzivmIhRdv1D35fvS_Mg-VTKgbuZS-7H-HY/edit?usp=sharing) |
| Add your EmailListVerify API key to the HTTP Request node credentials to enable email validation.                                                                                                                                                                                                 | [EmailListVerify API signup](https://app.emaillistverify.com/api?utm_source=n8n&utm_medium=referral&utm_campaign=GmailFinder) |
| The domain cleaning step strips “http”, “https”, and “www.” prefixes to isolate the domain root, which is critical for generating candidate emails accurately.                                                                                                                                   | Sticky Note5 content                                                                                                      |
| Update Google Sheets nodes to point to your own copies of the template to ensure proper data flow and permissions.                                                                                                                                                                               | Sticky Note and Sticky Note3 content                                                                                      |

---

**Disclaimer:** The provided text originates solely from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.