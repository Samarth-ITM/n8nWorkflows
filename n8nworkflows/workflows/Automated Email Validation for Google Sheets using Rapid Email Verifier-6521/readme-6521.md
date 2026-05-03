Automated Email Validation for Google Sheets using Rapid Email Verifier

https://n8nworkflows.xyz/workflows/automated-email-validation-for-google-sheets-using-rapid-email-verifier-6521


# Automated Email Validation for Google Sheets using Rapid Email Verifier

### 1. Workflow Overview

This workflow automates the validation of email addresses stored in a Google Sheets document using the Rapid Email Verifier API. It is designed to monitor newly added rows in the sheet, filter out only unverified emails, process them in batches to respect API rate limits, verify each email in real time, and then update the sheet with the verification results.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Detects new rows added to the Google Sheet for processing.
- **1.2 Filtering Unverified Emails**: Filters rows to only process emails that have not been verified yet.
- **1.3 Batch Processing**: Splits the filtered emails into manageable batches for sequential processing.
- **1.4 Email Verification via API**: Sends each email to the Rapid Email Verifier API for validation.
- **1.5 Results Update**: Updates the original Google Sheet with the verification status.
- **1.6 Informative Sticky Notes**: Provides setup instructions, explanations, and performance metrics for users.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
Monitors the Google Sheets document for new rows added to the sheet and triggers the workflow accordingly. It runs a check every hour to detect any new email entries added by the user.

- **Nodes Involved:**  
  - Monitor New Email Entries

- **Node Details:**

  - **Monitor New Email Entries**  
    - *Type & Role:* Google Sheets Trigger — triggers workflow on new row additions.  
    - *Configuration:*  
      - Triggers on the event "rowAdded".  
      - Polls every hour (mode: everyHour).  
      - Targets Sheet1 (gid=0) of a specified Google Sheets document (document ID to be set by user).  
    - *Expressions:* Uses the document ID and sheet name dynamically with list mode and caching.  
    - *Connections:* Outputs to "Filter Unverified Emails".  
    - *Failure Cases:* Possible auth errors if Google Sheets OAuth2 credential is invalid or expired; connectivity issues; document ID missing or incorrect.  
    - *Version:* TypeVersion 1.

---

#### 2.2 Filtering Unverified Emails

- **Overview:**  
Filters the newly added rows to only process emails which have not yet been verified — i.e., where the "Email Verified" column is empty. This avoids redundant API calls and saves resources.

- **Nodes Involved:**  
  - Filter Unverified Emails

- **Node Details:**

  - **Filter Unverified Emails**  
    - *Type & Role:* Filter node — applies condition-based filtering on incoming data.  
    - *Configuration:*  
      - Condition: `$json['Email Verified']` equals an empty string.  
      - Case sensitive and strict type validation enabled.  
    - *Expressions:* Uses an expression to check the "Email Verified" field.  
    - *Connections:* Outputs to "Process Emails in Batches" if condition met; otherwise stops.  
    - *Failure Cases:* Expression errors if "Email Verified" field missing or named differently; empty input data causes no output.  
    - *Version:* TypeVersion 2.2.

---

#### 2.3 Batch Processing

- **Overview:**  
Splits the list of unverified email rows into batches to process them sequentially. This respects API rate limits and ensures stable, error-tolerant verification.

- **Nodes Involved:**  
  - Process Emails in Batches

- **Node Details:**

  - **Process Emails in Batches**  
    - *Type & Role:* SplitInBatches node — controls batch size and sequential processing.  
    - *Configuration:* Defaults (batch size not explicitly set, so default batch size applies).  
    - *Connections:*  
      - First output (empty array) loops back to itself (for next batch).  
      - Second output sends each batch item to "Verify Email Address" for verification.  
    - *Failure Cases:* Batch processing may stall if data is malformed; large batch size could trigger API rate limits; empty batches result in no further processing.  
    - *Version:* TypeVersion 3.

---

#### 2.4 Email Verification via API

- **Overview:**  
For each email in the batch, sends a HTTP GET request to the Rapid Email Verifier API to validate the email address in real time. The API returns the verification status (valid/invalid/unknown).

- **Nodes Involved:**  
  - Verify Email Address

- **Node Details:**

  - **Verify Email Address**  
    - *Type & Role:* HTTP Request node — connects to external API to validate emails.  
    - *Configuration:*  
      - HTTP Method: GET  
      - URL: `https://rapid-email-verifier.fly.dev/api/validate`  
      - Query Parameter: `email` set dynamically as `={{ $json.Email }}` from the current batch item.  
      - No authentication required.  
    - *Expressions:* Uses dynamic expression to pass email from batch data.  
    - *Connections:* Outputs verification response to "Update Verification Results".  
    - *Failure Cases:* Network errors, API downtime, malformed email input, unexpected API response format.  
    - *Version:* TypeVersion 4.2.

---

#### 2.5 Results Update

- **Overview:**  
Updates the original Google Sheet with the verification results, matching rows by Serial Number (SrNo). It appends or updates the "Email Verified" column with the API status, preserving all other original data.

- **Nodes Involved:**  
  - Update Verification Results

- **Node Details:**

  - **Update Verification Results**  
    - *Type & Role:* Google Sheets node — appends or updates rows in the spreadsheet.  
    - *Configuration:*  
      - Operation: appendOrUpdate  
      - Matching column: "SrNo" (Serial Number) to locate the correct row.  
      - Columns updated: "SrNo" and "Email Verified" (status from API response).  
      - Targets Sheet1 (gid=0) with specified document ID.  
    - *Expressions:*  
      - SrNo set as `={{ $('Filter Unverified Emails').item.json.SrNo }}` to match original row.  
      - Email Verified set as `={{ $json.status }}` from API response.  
    - *Connections:* Loops output back to "Process Emails in Batches" to process remaining items.  
    - *Failure Cases:* Auth errors with Google Sheets OAuth2; API rate limits on Google Sheets; mismatch in SrNo; data overwrite risks if SrNo duplicates exist.  
    - *Version:* TypeVersion 4.6.

---

#### 2.6 Informative Sticky Notes

- **Overview:**  
Several sticky notes provide important context, setup instructions, and performance details to assist users in understanding and maintaining the workflow.

- **Nodes Involved:**  
  - Workflow Overview  
  - Setup Instructions  
  - Trigger Explanation  
  - Filter Logic  
  - Batch Processing  
  - API Details  
  - Results Update  
  - Performance Metrics

- **Node Details:**  
  These nodes are of type Sticky Note and contain textual information only. They do not affect workflow execution but are critical for documentation and user guidance.

---

### 3. Summary Table

| Node Name                | Node Type               | Functional Role                    | Input Node(s)              | Output Node(s)                 | Sticky Note                                                                                               |
|--------------------------|-------------------------|----------------------------------|----------------------------|-------------------------------|----------------------------------------------------------------------------------------------------------|
| Monitor New Email Entries | Google Sheets Trigger   | Detect new rows in Google Sheets | -                          | Filter Unverified Emails       | 🚀 MONITORING TRIGGER: Checks every hour, processes only new entries, uses efficient polling             |
| Filter Unverified Emails  | Filter                  | Filter unverified emails          | Monitor New Email Entries   | Process Emails in Batches      | 🔍 SMART FILTERING: Only process emails with empty "Email Verified" column, saves API credits            |
| Process Emails in Batches | SplitInBatches          | Batch processing of emails        | Filter Unverified Emails    | Verify Email Address           | ⚙️ BATCH PROCESSING: Processes emails one by one to respect API limits, ensure reliability               |
| Verify Email Address      | HTTP Request            | Verify email via API              | Process Emails in Batches   | Update Verification Results    | 🌐 EMAIL VALIDATION API: rapid-email-verifier.fly.dev, no auth, real-time, free tier 1000/month           |
| Update Verification Results | Google Sheets          | Update sheet with verification   | Verify Email Address        | Process Emails in Batches      | ✅ RESULTS UPDATE: Updates status by SrNo, preserves data, creates audit trail                            |
| Workflow Overview        | Sticky Note             | Documentation overview            | -                          | -                             | 📧 AUTOMATED EMAIL VERIFICATION SYSTEM: Purpose and use cases overview                                  |
| Setup Instructions       | Sticky Note             | Setup guidance                   | -                          | -                             | ⚙️ QUICK SETUP GUIDE: Stepwise instructions to prepare sheet, connect accounts, test                     |
| Trigger Explanation      | Sticky Note             | Explain trigger node              | -                          | -                             | 🚀 MONITORING TRIGGER: Hourly polling details, efficiency notes                                         |
| Filter Logic             | Sticky Note             | Explain filter node               | -                          | -                             | 🔍 SMART FILTERING: Filter criteria and benefits explained                                              |
| Batch Processing         | Sticky Note             | Explain batch processing          | -                          | -                             | ⚙️ BATCH PROCESSING: Benefits of batch size control and error handling                                  |
| API Details              | Sticky Note             | Email validation API info         | -                          | -                             | 🌐 EMAIL VALIDATION API: API details, response, free tier                                               |
| Results Update           | Sticky Note             | Explain results update node       | -                          | -                             | ✅ RESULTS UPDATE: How data is updated and preserved                                                   |
| Performance Metrics      | Sticky Note             | Workflow performance stats        | -                          | -                             | 📈 WORKFLOW PERFORMANCE: Throughput, accuracy, cost, and pro tips                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheet with Columns:**  
   - Columns: SrNo | Name | Email | Email Verified  
   - Prepare sample data with some unverified emails (empty "Email Verified").

2. **Add Google Sheets Trigger Node:**  
   - Name: Monitor New Email Entries  
   - Trigger event: rowAdded  
   - Sheet Name: Sheet1 (gid=0)  
   - Document ID: Paste your Google Sheets document ID  
   - Poll interval: every hour  
   - Connect your Google Sheets OAuth2 credentials.

3. **Add Filter Node:**  
   - Name: Filter Unverified Emails  
   - Condition: Check if "Email Verified" field equals empty string (`""`)  
   - Connect input from "Monitor New Email Entries".

4. **Add SplitInBatches Node:**  
   - Name: Process Emails in Batches  
   - Leave batch size as default, or set 1 for sequential processing  
   - Connect input from "Filter Unverified Emails".

5. **Add HTTP Request Node:**  
   - Name: Verify Email Address  
   - Method: GET  
   - URL: `https://rapid-email-verifier.fly.dev/api/validate`  
   - Add query parameter: name = `email`, value = expression `{{$json.Email}}`  
   - No authentication needed.  
   - Connect input from second output of "Process Emails in Batches".

6. **Add Google Sheets Node:**  
   - Name: Update Verification Results  
   - Operation: appendOrUpdate  
   - Document ID and Sheet Name: same as trigger node  
   - Matching Columns: SrNo  
   - Columns to update:  
     - SrNo: expression `{{$('Filter Unverified Emails').item.json.SrNo}}`  
     - Email Verified: expression `{{$json.status}}` from API response  
   - Connect input from "Verify Email Address".

7. **Connect output of "Update Verification Results" back to first output of "Process Emails in Batches"**  
   - This loops to process next batch if any.

8. **Add Sticky Notes:**  
   - Add descriptive sticky notes with titles and content as per documentation for Workflow Overview, Setup Instructions, Trigger Explanation, Filter Logic, Batch Processing, API Details, Results Update, and Performance Metrics.  
   - Position and color-code them for clarity.

9. **Configure Credentials:**  
   - Connect OAuth2 credentials for Google Sheets in both trigger and update nodes.  
   - No API key or authentication needed for Rapid Email Verifier API.

10. **Test Workflow:**  
    - Run with a sample email entry in Google Sheets.  
    - Check the output "Email Verified" column updates accordingly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                             | Context or Link                                                                                             |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| 📧 AUTOMATED EMAIL VERIFICATION SYSTEM: Automatically validates email addresses from Google Sheets and updates results in real time.                                   | Workflow Overview sticky note                                                                               |
| ⚙️ QUICK SETUP GUIDE: Create sheet with required columns, connect credentials, and test with sample email.                                                               | Setup Instructions sticky note                                                                              |
| 🚀 MONITORING TRIGGER: Hourly checks for new rows, efficient polling to avoid rate limits.                                                                               | Trigger Explanation sticky note                                                                             |
| 🔍 SMART FILTERING: Only unverified emails are processed, saving API credits and processing time.                                                                        | Filter Logic sticky note                                                                                     |
| ⚙️ BATCH PROCESSING: Processes emails one at a time to respect API limits and ensure error handling.                                                                     | Batch Processing sticky note                                                                                 |
| 🌐 EMAIL VALIDATION API: Rapid Email Verifier API offers fast, real-time validation with no authentication and a free tier of 1000 verifications/month.                 | API Details sticky note                                                                                      |
| ✅ RESULTS UPDATE: Updates Google Sheet with verification status, matching rows by Serial Number to preserve data integrity and create an audit trail.                   | Results Update sticky note                                                                                   |
| 📈 WORKFLOW PERFORMANCE: Processes ~60 emails/hour with average API response < 500ms and 95%+ verification accuracy; free tier supports 1000/month.                      | Performance Metrics sticky note                                                                              |
| 💡 PRO TIP: For real-time validation, consider using a webhook trigger instead of polling.                                                                               | Performance Metrics sticky note                                                                              |

---

**Disclaimer:**  
The provided workflow is created exclusively with n8n automation tool, respecting all content policies. It handles only legal, public data without any illegal or offensive content.