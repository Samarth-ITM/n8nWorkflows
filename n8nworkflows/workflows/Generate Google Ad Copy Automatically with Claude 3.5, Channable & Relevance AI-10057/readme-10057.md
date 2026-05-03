Generate Google Ad Copy Automatically with Claude 3.5, Channable & Relevance AI

https://n8nworkflows.xyz/workflows/generate-google-ad-copy-automatically-with-claude-3-5--channable---relevance-ai-10057


# Generate Google Ad Copy Automatically with Claude 3.5, Channable & Relevance AI

---

### 1. Workflow Overview

This workflow automates the generation of Google Ads copy using AI (Claude 3.5 via Relevance AI), integrating product data from a Channable feed, validating ad text, checking compliance, and exporting results for further use. It is designed for e-commerce or marketing teams who want daily refreshed and policy-compliant Google Ads text, minimizing manual effort and improving ad quality.

**Logical Blocks:**

- **1.1 Schedule Trigger**: Automatically initiates the workflow daily at midnight to refresh ad copy.
- **1.2 Product Data Retrieval**: Fetches live product feed data from Channable or a similar API.
- **1.3 Batch Processing**: Splits product data into manageable batches to avoid API rate limits.
- **1.4 AI Ad Copy Generation**: Sends product details to Relevance AI’s Claude 3.5-powered tool to generate headlines and descriptions.
- **1.5 Character Limit Validation**: Ensures generated ad text meets Google Ads character limits (headline ≤30 chars, description ≤90 chars), truncating gracefully if needed.
- **1.6 Compliance Checking**: Uses a Relevance AI compliance agent to detect policy violations in ad text.
- **1.7 Routing Based on Compliance**: Routes compliant ads for formatting and aggregation; non-compliant ads trigger Slack alerts.
- **1.8 CSV Formatting and Aggregation**: Aggregates all compliant ads into a CSV-formatted dataset.
- **1.9 Export and Notifications**: Saves the CSV data to Google Sheets and sends a Slack notification summarizing the generation result.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Schedule Trigger

- **Overview:** Initiates the workflow daily at midnight (cron: `0 0 * * *`). Allows manual triggering for testing.
- **Nodes Involved:** `Schedule Trigger - Daily1`
- **Node Details:**
  - Type: Schedule Trigger
  - Configuration: Cron schedule set to midnight daily.
  - Inputs: None (start node).
  - Outputs: Triggers `Get Product Feed`.
  - Edge cases: Cron misconfiguration could prevent execution; manual trigger recommended for testing.

---

#### 2.2 Product Data Retrieval

- **Overview:** Fetches product data from Channable API or alternative product feed endpoint.
- **Nodes Involved:** `Get Product Feed`
- **Node Details:**
  - Type: HTTP Request
  - Configuration:
    - GET request to URL constructed from environment variables `${CHANNABLE_API_URL}/v1/projects/${PROJECT_ID}/items`.
    - Auth: HTTP header authentication with credentials.
  - Inputs: Trigger from schedule node.
  - Outputs: Sends JSON product feed to batch splitter.
  - Edge cases:
    - API endpoint down or unauthorized access.
    - Empty feed returns no products.
    - Rate limits or network timeouts.

---

#### 2.3 Batch Processing

- **Overview:** Splits large product sets into batches of 50 to avoid rate limits on downstream APIs.
- **Nodes Involved:** `Split Into Batches1`
- **Node Details:**
  - Type: SplitInBatches
  - Configuration: Batch size fixed at 50 items.
  - Inputs: Product feed JSON array.
  - Outputs: Processes batches sequentially to `Generate Ad Copy - Relevance AI1`.
  - Edge cases:
    - Incorrect batch size can cause API throttling.
    - Empty batches if input data is empty.

---

#### 2.4 AI Ad Copy Generation

- **Overview:** Calls Relevance AI’s Google Ads copy generator tool to create ad headlines and descriptions using Claude 3.5.
- **Nodes Involved:** `Generate Ad Copy - Relevance AI1`
- **Node Details:**
  - Type: HTTP Request (POST)
  - Configuration:
    - URL: `${RELEVANCE_AI_API_URL}/tools/google_text_ad_copy_generator/run`.
    - Body JSON includes product title, description, price, category, brand.
    - Auth: HTTP header authentication credentials.
    - Timeout: 60 seconds.
  - Inputs: Batch of products.
  - Outputs: Generated ad text JSON to validation node.
  - Edge cases:
    - API timeout or rate limits.
    - Model may return empty or malformed text.
    - Missing environment variables (API URL, credentials).
  
---

#### 2.5 Character Limit Validation

- **Overview:** Validates and truncates headlines and descriptions to meet Google Ads character limits (headline ≤30, description ≤90).
- **Nodes Involved:** `Validate Character Limits1`
- **Node Details:**
  - Type: Code (JavaScript)
  - Configuration:
    - Counts characters precisely, trims trailing punctuation.
    - Truncates with ellipsis if over limit.
    - Returns validated headline and description plus metadata (truncated flag, counts).
  - Inputs: AI-generated ad copy.
  - Outputs: Validated text to compliance check.
  - Edge cases:
    - Unexpected or missing input fields.
    - Unicode or multi-byte characters affecting length.
    - Logic errors in truncation.

---

#### 2.6 Compliance Checking

- **Overview:** Uses Relevance AI compliance agent to check ad copy against Google Ads policies.
- **Nodes Involved:** `Compliance Check Agent1`
- **Node Details:**
  - Type: HTTP Request (POST)
  - Configuration:
    - URL: `${RELEVANCE_AI_API_URL}/agents/google_ads_compliance_checker/run`.
    - Body: Contains ad text and agent ID.
    - Auth: HTTP header authentication.
    - Timeout: 60 seconds.
  - Inputs: Validated ad copy.
  - Outputs: Compliance status (`APPROVED` or other) to IF node.
  - Edge cases:
    - Agent API errors.
    - Incorrect or expired agent ID.
    - Delays causing slow workflow.

---

#### 2.7 Routing Based on Compliance

- **Overview:** Routes ads based on compliance status; compliant ads proceed to formatting, others trigger alerts.
- **Nodes Involved:** `IF Compliant1`, `Alert - Non-Compliant`
- **Node Details:**
  - Type: IF node
  - Configuration: Checks if compliance status contains `"APPROVED"`.
  - Inputs: Compliance check output.
  - Outputs:
    - True branch: `Format for CSV`.
    - False branch: `Alert - Non-Compliant`.
  - Alert node:
    - Type: Slack
    - Sends detailed message about non-compliant ad including product data and timestamp.
    - Can be replaced with email or other alerting methods.
  - Edge cases:
    - False negatives or positives in compliance.
    - Slack API failures or missing webhook.
  
---

#### 2.8 CSV Formatting and Aggregation

- **Overview:** Formats compliant ads into CSV lines and combines batches into a single dataset.
- **Nodes Involved:** `Format for CSV`, `Aggregate Batches`, `Generate CSV File1`
- **Node Details:**
  - `Format for CSV`:
    - Type: Set node (basic formatting for CSV compliance).
    - Inputs: Compliant ad JSON.
    - Outputs: Passes formatted data to aggregation.
  - `Aggregate Batches`:
    - Type: Aggregate node.
    - Aggregates all batch outputs into one dataset.
  - `Generate CSV File1`:
    - Type: Code node.
    - Converts aggregated JSON to escaped CSV string with headers.
    - Outputs CSV data, total ads count, generation timestamp, and filename.
  - Edge cases:
    - Misformatted CSV if fields contain special characters.
    - Empty datasets from no compliant ads.

---

#### 2.9 Export and Notifications

- **Overview:** Saves generated CSV data to Google Sheets and sends a Slack notification summarizing the process.
- **Nodes Involved:** `Save to Google Sheets`, `Success Notification1`
- **Node Details:**
  - `Save to Google Sheets`:
    - Type: Google Sheets node.
    - Writes CSV data into sheet named "Generated Ads" in document with ID from environment variable.
    - OAuth2 credentials required.
  - `Success Notification1`:
    - Type: Slack.
    - Sends confirmation message with total ads generated, sheet ID, timestamp, next steps, and CSV filename.
  - Edge cases:
    - Google Sheets API quota or auth errors.
    - Slack API failures.
    - Missing or invalid environment variables.

---

### 3. Summary Table

| Node Name                   | Node Type           | Functional Role                                         | Input Node(s)               | Output Node(s)                | Sticky Note                                                                                                              |
|-----------------------------|---------------------|---------------------------------------------------------|-----------------------------|------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger - Daily1    | Schedule Trigger    | Triggers workflow daily at midnight                      | None                        | Get Product Feed              | 🟦 Schedule Trigger - Daily: Automatically runs every night at midnight. Tip: manual execution possible.                 |
| Get Product Feed             | HTTP Request        | Fetches product feed from Channable API                  | Schedule Trigger - Daily1   | Split Into Batches1           | 🟦 Get Product Feed (Channable): Fetches live product data; replace with your store API if needed.                      |
| Split Into Batches1          | SplitInBatches      | Splits products into batches of 50                        | Get Product Feed            | Generate Ad Copy - Relevance AI1 | 🟦 Split Into Batches: Processes 50 products per batch to avoid API limits.                                              |
| Generate Ad Copy - Relevance AI1 | HTTP Request    | Calls Relevance AI Claude 3.5 tool to generate ad copy   | Split Into Batches1         | Validate Character Limits1    | 🟦 Generate Ad Copy - Relevance AI: Calls your Relevance AI tool using Claude 3.5 / GPT-4.                               |
| Validate Character Limits1   | Code                | Validates and truncates ad text to meet char limits      | Generate Ad Copy - Relevance AI1 | Compliance Check Agent1    | 🟦 Validate Character Limits: Ensures headlines ≤30, descriptions ≤90 chars with graceful truncation.                   |
| Compliance Check Agent1      | HTTP Request        | Checks ad copy compliance with Google Ads policies       | Validate Character Limits1  | IF Compliant1                | 🟦 Compliance Check Agent: Flags policy issues with your Relevance AI agent.                                            |
| IF Compliant1               | IF                  | Routes compliant/non-compliant ads                        | Compliance Check Agent1      | Format for CSV / Alert - Non-Compliant |                                                                                                                          |
| Alert - Non-Compliant        | Slack               | Sends alert for non-compliant ads                         | IF Compliant1 (false branch) | None                       |                                                                                                                          |
| Format for CSV              | Set                 | Formats compliant ads for CSV                              | IF Compliant1 (true branch) | Aggregate Batches             |                                                                                                                          |
| Aggregate Batches           | Aggregate            | Aggregates all batches into single dataset                | Format for CSV              | Generate CSV File1            |                                                                                                                          |
| Generate CSV File1          | Code                 | Converts aggregated JSON to escaped CSV                   | Aggregate Batches           | Save to Google Sheets         |                                                                                                                          |
| Save to Google Sheets       | Google Sheets        | Saves CSV data to Google Sheets document                  | Generate CSV File1          | Success Notification1         | 🟦 Saves CSV to Google Sheets for manual import or automation.                                                           |
| Success Notification1       | Slack                | Sends success message with summary stats                  | Save to Google Sheets       | None                         |                                                                                                                          |
| Sticky Note                 | Sticky Note          | Various explanatory notes covering multiple nodes         | None                       | None                         | See notes in “Sticky Note” contents in node details section.                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**
   - Type: Schedule Trigger
   - Set Cron expression: `0 0 * * *` (daily at midnight)
   - No credentials needed
   - Connect output to `Get Product Feed`.

2. **Create HTTP Request Node for Product Feed**
   - Name: Get Product Feed
   - Method: GET
   - URL: Use environment variables `${CHANNABLE_API_URL}/v1/projects/${PROJECT_ID}/items`
   - Authentication: HTTP Header Auth (set header key and token as per your Channable or API)
   - Connect input from Schedule Trigger
   - Output to `Split Into Batches1`.

3. **Create Split In Batches Node**
   - Name: Split Into Batches1
   - Batch Size: 50
   - Input: Product array from HTTP Request
   - Output to `Generate Ad Copy - Relevance AI1`.

4. **Create HTTP Request Node for AI Ad Copy Generation**
   - Name: Generate Ad Copy - Relevance AI1
   - Method: POST
   - URL: `${RELEVANCE_AI_API_URL}/tools/google_text_ad_copy_generator/run`
   - Body Type: JSON
   - Body Content:
     ```json
     {
       "params": {
         "product_title": "{{$json.title}}",
         "product_description": "{{$json.description}}",
         "price": "{{$json.price}}",
         "category": "{{$json.category}}",
         "brand": "{{$json.brand}}"
       }
     }
     ```
   - Authentication: HTTP Header Auth (use your Relevance AI API key)
   - Timeout: 60000ms
   - Connect output to `Validate Character Limits1`.

5. **Create Code Node for Character Limit Validation**
   - Name: Validate Character Limits1
   - Paste JavaScript code that:
     - Reads headline and description from input
     - Counts characters precisely
     - Truncates if headline >30 or description >90 chars, adding ellipsis
     - Returns validated fields with metadata
   - Input from AI generation node
   - Output to `Compliance Check Agent1`.

6. **Create HTTP Request Node for Compliance Check**
   - Name: Compliance Check Agent1
   - Method: POST
   - URL: `${RELEVANCE_AI_API_URL}/agents/google_ads_compliance_checker/run`
   - Body:
     ```json
     {
       "message": {
         "role": "user",
         "content": "Check compliance for this ad: Headline: {{$json.validated_headline}}, Description: {{$json.validated_description}}, Category: {{$json.category}}"
       },
       "agent_id": "{{$env.RELEVANCE_AGENT_COMPLIANCE_ID}}"
     }
     ```
   - Authentication: HTTP Header Auth (Relevance AI API key)
   - Timeout: 60000ms
   - Output to `IF Compliant1`.

7. **Create IF Node to Route Based on Compliance**
   - Name: IF Compliant1
   - Condition: Check if `compliance_status` or `output` string contains "APPROVED"
   - True branch → `Format for CSV`
   - False branch → `Alert - Non-Compliant`.

8. **Create Slack Node for Non-Compliant Alerts**
   - Name: Alert - Non-Compliant
   - Configure Slack webhook URL or credentials
   - Message:
     ```
     ⚠️ Non-Compliant Ad Flagged

     Product ID: {{$json.product_id}}
     Product Title: {{$json.product_title}}
     Category: {{$json.category}}

     Generated Headline: {{$json.validated_headline}}
     Generated Description: {{$json.validated_description}}

     Compliance Issues: Check agent output

     Timestamp: {{$json.validation_timestamp}}
     ```
   - Input from IF node (false branch).

9. **Create Set Node to Format Ads for CSV**
   - Name: Format for CSV
   - Map necessary fields for CSV compliance (e.g., product_id, headline, description, final_url, display_url)
   - Input from IF node (true branch)
   - Output to `Aggregate Batches`.

10. **Create Aggregate Node to Combine Batches**
    - Name: Aggregate Batches
    - Aggregate all item data into single dataset
    - Input from `Format for CSV`
    - Output to `Generate CSV File1`.

11. **Create Code Node to Generate CSV File**
    - Name: Generate CSV File1
    - JavaScript code to:
      - Convert aggregated JSON to CSV with headers: product_id, headline, description, final_url, display_url.
      - Escape quotes and commas properly.
      - Add metadata: total ads, generation timestamp, filename.
    - Input from `Aggregate Batches`
    - Output to `Save to Google Sheets`.

12. **Create Google Sheets Node to Save CSV**
    - Name: Save to Google Sheets
    - Document ID: `${GOOGLE_SHEET_ID}` (environment variable)
    - Sheet Name: "Generated Ads"
    - Credentials: Google Sheets OAuth2
    - Input from `Generate CSV File1`
    - Output to `Success Notification1`.

13. **Create Slack Node for Success Notification**
    - Name: Success Notification1
    - Slack webhook or OAuth2 credentials
    - Message template:
      ```
      ✅ Google Ads Generation Complete

      📊 Summary:
      • Total Ads Generated: {{$node['Generate CSV File1'].json.total_ads}}
      • Saved to Google Sheets: {{$env.GOOGLE_SHEET_ID}}
      • Timestamp: {{$node['Generate CSV File1'].json.generated_at}}

      🎯 Next Steps:
      • Review ads in Google Sheet
      • Import to Google Ads (manual or scheduled)
      • Monitor for disapprovals in 24 hours

      💡 CSV filename: {{$node['Generate CSV File1'].json.filename}}
      ```
    - Input from `Save to Google Sheets`.

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                                 |
|------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------|
| The workflow uses Relevance AI tools powered by Claude 3.5 / GPT-4 models for text generation and compliance checking. | Ensure you have valid API keys and IDs from your Relevance AI account.                                         |
| Slack nodes require webhook URLs or OAuth2 credentials configured in n8n.                                              | Replace Slack nodes with Email or other notification nodes as needed.                                           |
| Google Sheets node requires OAuth2 credentials with access to your target spreadsheet.                                 | The Google Sheet must have a sheet/tab named "Generated Ads" for data insertion.                               |
| Character limit validation fixes common issues where ads get disapproved for exceeding limits due to whitespace/punctuation. | The JavaScript code carefully trims and counts characters accurately, including multi-byte characters.         |
| Batch size of 50 can be adjusted depending on your Relevance AI and API limits for optimal throughput and reliability. |                                                                                                                |
| The product feed URL and project ID must be defined in environment variables for flexibility and security.             | Useful for integrating with other e-commerce platforms aside from Channable.                                   |
| Slack success notifications include next steps to guide users post-generation.                                         | Helps maintain workflow transparency and operational monitoring.                                               |

---

**Disclaimer:**  
The provided text is exclusively generated from an automated n8n workflow. It strictly complies with applicable content policies and contains no illegal, offensive, or protected material. All data processed is legal and public.

---