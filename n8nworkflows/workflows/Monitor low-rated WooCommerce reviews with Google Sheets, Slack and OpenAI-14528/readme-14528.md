Monitor low-rated WooCommerce reviews with Google Sheets, Slack and OpenAI

https://n8nworkflows.xyz/workflows/monitor-low-rated-woocommerce-reviews-with-google-sheets--slack-and-openai-14528


# Monitor low-rated WooCommerce reviews with Google Sheets, Slack and OpenAI

# 1. Workflow Overview

This workflow monitors WooCommerce product reviews on a recurring schedule, identifies approved low-rated reviews, prevents duplicate alerting through Google Sheets, generates a concise alert with OpenAI for newly detected reviews, sends that alert to Slack, and maintains a review log in Google Sheets.

It is designed for support, operations, e-commerce, or reputation-monitoring teams that want to react quickly to negative customer feedback without sending duplicate notifications.

## 1.1 Scheduled Triggering

The workflow starts automatically every 5 hours using a schedule trigger.

## 1.2 Review Retrieval and Normalization

It fetches WooCommerce reviews via HTTP with Basic Auth, then reshapes the returned payload into a consistent schema used by the rest of the workflow.

## 1.3 Per-Review Validation and Filtering

Reviews are processed one at a time. The workflow keeps only approved reviews and then filters for low ratings, defined here as ratings less than or equal to 2.

## 1.4 Deduplication Against Google Sheets

Each qualifying review is looked up in a Google Sheet by `review_id`. If the review is not found, it is treated as new. If found, the existing row is updated.

## 1.5 Alert Generation and Notification

For new low-rated reviews, OpenAI generates a short Slack-ready alert message based on the review content. That message is posted to a Slack channel.

## 1.6 Logging and Review Record Maintenance

New low-rated reviews are appended to Google Sheets after the Slack alert is sent. Existing reviews that are already present in the sheet are updated instead of re-alerted.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Triggering

### Overview

This block launches the workflow automatically at a fixed interval. It serves as the primary entry point for periodic review monitoring.

### Nodes Involved

- Trigger on schedule time

### Node Details

#### Trigger on schedule time
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; workflow entry node that runs on a repeating time interval.
- **Configuration choices:** Configured to execute every 5 hours.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to `Fetch woocommerce reviews`.
- **Version-specific requirements:** Uses `typeVersion` 1.2; standard schedule trigger behavior in recent n8n versions.
- **Edge cases or potential failure types:**
  - Workflow must be active for scheduled execution.
  - Timezone interpretation depends on instance/workflow settings.
  - Long-running executions can overlap if the schedule is too aggressive relative to execution time.
- **Sub-workflow reference:** None.

---

## 2.2 Review Retrieval and Normalization

### Overview

This block retrieves reviews from WooCommerce and standardizes the output fields so downstream nodes can work with a stable structure. It isolates external API variability from later business logic.

### Nodes Involved

- Fetch woocommerce reviews
- Normalize review data

### Node Details

#### Fetch woocommerce reviews
- **Type and technical role:** `n8n-nodes-base.httpRequest`; performs the WooCommerce REST API call.
- **Configuration choices:** 
  - Uses a manually configured URL, currently empty in the JSON and therefore requiring setup.
  - Uses generic credential authentication with `httpBasicAuth`.
  - Intended to call the WooCommerce reviews endpoint.
- **Key expressions or variables used:** None in the current configuration.
- **Input and output connections:** Input from `Trigger on schedule time`; output to `Normalize review data`.
- **Version-specific requirements:** Uses HTTP Request node version 4.3.
- **Edge cases or potential failure types:**
  - Empty URL will cause immediate failure until configured.
  - Basic Auth credentials may fail if WooCommerce keys are wrong or lack permissions.
  - WooCommerce endpoint may paginate results; this workflow currently does not explicitly handle pagination.
  - API rate limits, SSL issues, or store-side firewall restrictions may block access.
  - Unexpected response shape may break downstream field mapping.
- **Sub-workflow reference:** None.

#### Normalize review data
- **Type and technical role:** `n8n-nodes-base.set`; maps incoming WooCommerce fields to a consistent internal schema.
- **Configuration choices:** Creates the following fields:
  - `review_id` from `{{$json.id}}`
  - `product_name` from `{{$json.product_name}}`
  - `rating` from `{{$json.rating}}`
  - `reviewer` from `{{$json.reviewer}}`
  - `review_text` from `{{$json.review}}`
  - `created_at` from `{{$json.date_created}}`
  - `product_permalink` from `{{$json.product_permalink}}`
  - `status` from `{{$json.status}}`
- **Key expressions or variables used:** Direct field references from the WooCommerce response.
- **Input and output connections:** Input from `Fetch woocommerce reviews`; output to `Process reviews one by one`.
- **Version-specific requirements:** Uses Set node version 3.4.
- **Edge cases or potential failure types:**
  - Missing fields in WooCommerce response may produce null/empty values.
  - If the HTTP response is not an array of reviews, processing may not behave as expected.
  - Type coercion issues can occur if rating or id arrives as an unexpected type.
- **Sub-workflow reference:** None.

---

## 2.3 Per-Review Validation and Filtering

### Overview

This block serializes review processing and filters down to only the reviews that matter operationally: approved reviews with low ratings. It also controls loop behavior so each review is handled individually.

### Nodes Involved

- Process reviews one by one
- Check review approval
- Check low rating

### Node Details

#### Process reviews one by one
- **Type and technical role:** `n8n-nodes-base.splitInBatches`; iterates through review items sequentially.
- **Configuration choices:** Default batch behavior, effectively one review at a time in this design.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input from `Normalize review data`
  - Loop-back input from `Log new review`
  - Loop-back input from `Update existing review`
  - Secondary loop-back path from `Check low rating`
  - Batch output to `Check review approval`
- **Version-specific requirements:** Uses Split In Batches version 3.
- **Edge cases or potential failure types:**
  - Incorrect loop wiring can lead to premature completion or skipped items.
  - If downstream nodes fail and do not return control, batch processing stops.
  - The existing connection pattern includes a direct false-path return from `Check low rating`, which is valid but should be understood carefully to avoid confusion.
- **Sub-workflow reference:** None.

#### Check review approval
- **Type and technical role:** `n8n-nodes-base.if`; filters reviews based on moderation status.
- **Configuration choices:** Passes only items where `{{$json.status}}` equals `approved`.
- **Key expressions or variables used:** `{{$json.status}}`
- **Input and output connections:**
  - Input from `Process reviews one by one`
  - True output to `Check low rating`
  - False output unused
- **Version-specific requirements:** Uses If node version 2.2 with conditions format version 2.
- **Edge cases or potential failure types:**
  - Case-sensitive comparison may reject values like `Approved` or alternative statuses.
  - If `status` is missing, the condition will fail.
- **Sub-workflow reference:** None.

#### Check low rating
- **Type and technical role:** `n8n-nodes-base.if`; identifies low-rated reviews requiring action.
- **Configuration choices:** Passes items where `{{$json.rating}} <= 2`.
- **Key expressions or variables used:** `{{$json.rating}}`
- **Input and output connections:**
  - Input from `Check review approval`
  - True output to `Find review in sheet`
  - False output loops back to `Process reviews one by one`
- **Version-specific requirements:** Uses If node version 2.2.
- **Edge cases or potential failure types:**
  - If `rating` is missing or non-numeric, strict numeric comparison may fail.
  - Stores using different rating scales would require adjustment.
- **Sub-workflow reference:** None.

---

## 2.4 Deduplication Against Google Sheets

### Overview

This block checks whether a low-rated approved review has already been recorded. It uses the Google Sheet as a lightweight state store to prevent duplicate Slack alerts and decide between append and update behavior.

### Nodes Involved

- Find review in sheet
- Is new review?
- Update existing review

### Node Details

#### Find review in sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`; searches the sheet for a row matching the current `review_id`.
- **Configuration choices:**
  - Document: `New Order Track`
  - Sheet/tab: `product_rating`
  - Filter uses `review_id` as lookup value against the column `review_id`
- **Key expressions or variables used:** `{{$json.review_id}}`
- **Input and output connections:** Input from `Check low rating`; output to `Is new review?`
- **Version-specific requirements:** Uses Google Sheets node version 4.7 with OAuth2 credentials.
- **Edge cases or potential failure types:**
  - OAuth token expiration or missing spreadsheet permissions.
  - Lookup column name must exactly match the sheet header.
  - If the sheet has duplicate `review_id` rows, downstream logic may become ambiguous.
  - Empty or malformed `review_id` values reduce reliability.
- **Sub-workflow reference:** None.

#### Is new review?
- **Type and technical role:** `n8n-nodes-base.if`; determines whether the lookup returned any matching rows.
- **Configuration choices:** Checks whether `{{$items().length}}` equals `0`.
- **Key expressions or variables used:** `{{$items().length}}`
- **Input and output connections:**
  - Input from `Find review in sheet`
  - True output to `Genrate Alert message`
  - False output to `Update existing review`
- **Version-specific requirements:** Uses If node version 2.2.
- **Edge cases or potential failure types:**
  - This logic assumes the node output item count accurately reflects lookup results.
  - Depending on Google Sheets node behavior and n8n version, empty lookups may need validation in testing.
- **Sub-workflow reference:** None.

#### Update existing review
- **Type and technical role:** `n8n-nodes-base.googleSheets`; updates an existing sheet row using `row_number`.
- **Configuration choices:**
  - Operation: `update`
  - Matching column: `row_number`
  - Sheet: `product_rating`
  - Document: `New Order Track`
  - Updates fields:
    - `rating`
    - `review`
    - `reviewer`
    - `review_id`
    - `created_at`
    - `row_number`
    - `Product_name`
    - `product_link`
- **Key expressions or variables used:**
  - Uses current item fields like `{{$json.rating}}`, `{{$json.review}}`, `{{$json.row_number}}`, etc.
- **Input and output connections:**
  - Input from `Is new review` false path
  - Output back to `Process reviews one by one`
- **Version-specific requirements:** Uses Google Sheets node version 4.7.
- **Edge cases or potential failure types:**
  - There is a field mismatch risk: earlier normalization uses `review_text`, `product_name`, and `product_permalink`, while this update node expects `review`, `Product_name`, and `product_link`.
  - If `Find review in sheet` does not return these exact field names from the sheet, updates may write blanks.
  - `row_number` must be present; without it, update will fail or not match any row.
  - Sheet headers are case-sensitive in practice for mapping consistency.
- **Sub-workflow reference:** None.

**Important implementation note:**  
The update path appears to rely on the Google Sheets lookup result schema, not the normalized WooCommerce schema. This is workable only if the returned sheet row contains the required fields and `row_number`. If the intent is to update the sheet with the latest WooCommerce review content, the node may need expressions referencing upstream review data explicitly, similar to the append node.

---

## 2.5 Alert Generation and Slack Notification

### Overview

This block generates a concise, professional alert message for new low-rated reviews and posts it to Slack. The OpenAI node transforms raw review text into an operationally useful message.

### Nodes Involved

- Genrate Alert message
- Send slack alert

### Node Details

#### Genrate Alert message
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`; sends a prompt to OpenAI and returns generated content.
- **Configuration choices:**
  - Model: `gpt-4-turbo`
  - Prompt instructs the model to:
    - act as a customer support assistant
    - create a short Slack alert message
    - keep it under 4 lines
    - use a professional and urgent tone
    - remove HTML
    - mention product name and issue clearly
    - avoid inventing facts
  - Prompt injects:
    - `product_name`
    - `rating`
    - `reviewer`
    - `review_text`
- **Key expressions or variables used:**
  - `{{$json.product_name}}`
  - `{{$json.rating}}`
  - `{{$json.reviewer}}`
  - `{{ $json.review_text }}`
- **Input and output connections:** Input from `Is new review` true path; output to `Send slack alert`.
- **Version-specific requirements:** Uses LangChain OpenAI node version 2; requires compatible OpenAI credentials and model access.
- **Edge cases or potential failure types:**
  - Invalid OpenAI credentials or unavailable model.
  - API quota exhaustion or rate limits.
  - Unexpected output schema if node version/model behavior changes.
  - Prompt may include malformed HTML or very long text; the model usually handles it but output may vary.
- **Sub-workflow reference:** None.

#### Send slack alert
- **Type and technical role:** `n8n-nodes-base.slack`; posts the generated alert to a Slack channel.
- **Configuration choices:**
  - Sends text from OpenAI output: `{{$json.output[0].content[0].text}}`
  - Target channel selected by ID: `C09S57E2JQ2`
  - Workflow link inclusion disabled
- **Key expressions or variables used:** `{{$json.output[0].content[0].text}}`
- **Input and output connections:** Input from `Genrate Alert message`; output to `Log new review`.
- **Version-specific requirements:** Uses Slack node version 2.3.
- **Edge cases or potential failure types:**
  - Slack credential issues or missing channel access.
  - If OpenAI output schema differs, the expression may fail.
  - Bot may not be a member of the channel.
  - Slack formatting or message length constraints are usually minor here due to short prompts.
- **Sub-workflow reference:** None.

---

## 2.6 Logging and Sheet Maintenance

### Overview

This block persists newly alerted reviews in Google Sheets so the workflow can detect them in future runs and avoid duplicate alerts. It closes the processing loop and returns control for the next batched review.

### Nodes Involved

- Log new review

### Node Details

#### Log new review
- **Type and technical role:** `n8n-nodes-base.googleSheets`; appends a new row for a newly detected low-rated review.
- **Configuration choices:**
  - Operation: `append`
  - Document: `New Order Track`
  - Sheet: `product_rating`
  - Maps values from the `Check low rating` node item rather than the immediate Slack/OpenAI output
  - Appended fields:
    - `rating`
    - `review`
    - `reviewer`
    - `review_id`
    - `created_at`
    - `Product_name`
    - `product_link`
- **Key expressions or variables used:**
  - `{{$('Check low rating').item.json.rating}}`
  - `{{$('Check low rating').item.json.review_text}}`
  - `{{$('Check low rating').item.json.reviewer}}`
  - `{{$('Check low rating').item.json.review_id}}`
  - `{{$('Check low rating').item.json.created_at}}`
  - `{{$('Check low rating').item.json.product_name}}`
  - `{{$('Check low rating').item.json.product_permalink}}`
- **Input and output connections:**
  - Input from `Send slack alert`
  - Output back to `Process reviews one by one`
- **Version-specific requirements:** Uses Google Sheets node version 4.7.
- **Edge cases or potential failure types:**
  - Google Sheets credential or permission issues.
  - Column names in the spreadsheet must match the configured schema.
  - If Slack fails, this node will not run, meaning the review is not logged and may alert again next cycle.
  - This node intentionally references a previous node to preserve original review data after the AI transformation.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Trigger on schedule time | n8n-nodes-base.scheduleTrigger | Starts the workflow every 5 hours |  | Fetch woocommerce reviews | Triggers the workflow automatically every 5 hours to check for new product reviews. |
| Fetch woocommerce reviews | n8n-nodes-base.httpRequest | Retrieves WooCommerce reviews via API | Trigger on schedule time | Normalize review data | ## Fetch Reviews & Normalize reviews<br>**Fetch woocommerce reviews :** Fetches all product reviews from WooCommerce using REST API authentication.<br>**Normalize Review Data :** Cleans and maps required review fields into a consistent format for processing. |
| Normalize review data | n8n-nodes-base.set | Maps review fields into a normalized structure | Fetch woocommerce reviews | Process reviews one by one | ## Fetch Reviews & Normalize reviews<br>**Fetch woocommerce reviews :** Fetches all product reviews from WooCommerce using REST API authentication.<br>**Normalize Review Data :** Cleans and maps required review fields into a consistent format for processing. |
| Process reviews one by one | n8n-nodes-base.splitInBatches | Iterates reviews sequentially | Normalize review data; Log new review; Update existing review; Check low rating | Check review approval | ## Review Validation & Deduplication Process<br>**Process Reviews One by One :** Processes each review individually to avoid duplicates and control execution flow.<br>**Check Review Approval :** Allows only approved reviews to continue further in the workflow.<br>**Check Low Rating :** Filters reviews to identify low-rated products that need attention.<br>**Find Review in Sheet :** Checks Google Sheet to see if the review already exists using review ID.<br>**Is New Review? :** Determines whether the review is new or already present in the sheet. |
| Check review approval | n8n-nodes-base.if | Filters only approved reviews | Process reviews one by one | Check low rating | ## Review Validation & Deduplication Process<br>**Process Reviews One by One :** Processes each review individually to avoid duplicates and control execution flow.<br>**Check Review Approval :** Allows only approved reviews to continue further in the workflow.<br>**Check Low Rating :** Filters reviews to identify low-rated products that need attention.<br>**Find Review in Sheet :** Checks Google Sheet to see if the review already exists using review ID.<br>**Is New Review? :** Determines whether the review is new or already present in the sheet. |
| Check low rating | n8n-nodes-base.if | Filters reviews with rating less than or equal to 2 | Check review approval | Find review in sheet; Process reviews one by one | ## Review Validation & Deduplication Process<br>**Process Reviews One by One :** Processes each review individually to avoid duplicates and control execution flow.<br>**Check Review Approval :** Allows only approved reviews to continue further in the workflow.<br>**Check Low Rating :** Filters reviews to identify low-rated products that need attention.<br>**Find Review in Sheet :** Checks Google Sheet to see if the review already exists using review ID.<br>**Is New Review? :** Determines whether the review is new or already present in the sheet. |
| Find review in sheet | n8n-nodes-base.googleSheets | Looks up existing review rows by review ID | Check low rating | Is new review? | ## Review Validation & Deduplication Process<br>**Process Reviews One by One :** Processes each review individually to avoid duplicates and control execution flow.<br>**Check Review Approval :** Allows only approved reviews to continue further in the workflow.<br>**Check Low Rating :** Filters reviews to identify low-rated products that need attention.<br>**Find Review in Sheet :** Checks Google Sheet to see if the review already exists using review ID.<br>**Is New Review? :** Determines whether the review is new or already present in the sheet. |
| Is new review? | n8n-nodes-base.if | Branches between new review handling and update flow | Find review in sheet | Genrate Alert message; Update existing review | ## Review Validation & Deduplication Process<br>**Process Reviews One by One :** Processes each review individually to avoid duplicates and control execution flow.<br>**Check Review Approval :** Allows only approved reviews to continue further in the workflow.<br>**Check Low Rating :** Filters reviews to identify low-rated products that need attention.<br>**Find Review in Sheet :** Checks Google Sheet to see if the review already exists using review ID.<br>**Is New Review? :** Determines whether the review is new or already present in the sheet. |
| Genrate Alert message | @n8n/n8n-nodes-langchain.openAi | Generates a Slack-ready alert from review content | Is new review? | Send slack alert | ## Generate Message & Log Review<br>**Generate Alert Message :** Creates a short, professional Slack alert message for a new low-rated review.<br>**Send slack Alert :** Sends the generated alert message to the support Slack channel.<br>**Log New Review :** Stores new low-rated review details in Google Sheets for tracking. |
| Send slack alert | n8n-nodes-base.slack | Posts the alert to Slack | Genrate Alert message | Log new review | ## Generate Message & Log Review<br>**Generate Alert Message :** Creates a short, professional Slack alert message for a new low-rated review.<br>**Send slack Alert :** Sends the generated alert message to the support Slack channel.<br>**Log New Review :** Stores new low-rated review details in Google Sheets for tracking. |
| Log new review | n8n-nodes-base.googleSheets | Appends new low-rated review records to the sheet | Send slack alert | Process reviews one by one | ## Generate Message & Log Review<br>**Generate Alert Message :** Creates a short, professional Slack alert message for a new low-rated review.<br>**Send slack Alert :** Sends the generated alert message to the support Slack channel.<br>**Log New Review :** Stores new low-rated review details in Google Sheets for tracking. |
| Update existing review | n8n-nodes-base.googleSheets | Updates an existing logged review row | Is new review? | Process reviews one by one | Updates the existing review record in Google Sheets if it already exists. |
| Sticky Note | n8n-nodes-base.stickyNote | Canvas documentation / design note |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Canvas note for trigger block |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Canvas note for fetch/normalize block |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Canvas note for AI/Slack/logging block |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Canvas note for validation/deduplication block |  |  |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Canvas note for update block |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Monitor low-rated WooCommerce reviews with Google Sheets, Slack and OpenAI`.

2. **Add a Schedule Trigger node**
   - Node type: `Schedule Trigger`
   - Configure interval mode.
   - Set it to run every `5 hours`.
   - This is the workflow entry point.

3. **Add an HTTP Request node**
   - Name it: `Fetch woocommerce reviews`
   - Connect it after the Schedule Trigger.
   - Node type: `HTTP Request`
   - Set authentication to `Generic Credential Type`.
   - Choose `HTTP Basic Auth`.
   - Create WooCommerce credentials using the store API key/secret or supported Basic Auth method.
   - Set the URL to the WooCommerce product reviews endpoint, for example your store’s reviews API endpoint.
   - Ensure the endpoint returns review objects including:
     - `id`
     - `product_name`
     - `rating`
     - `reviewer`
     - `review`
     - `date_created`
     - `product_permalink`
     - `status`

4. **Add a Set node**
   - Name it: `Normalize review data`
   - Connect it after the HTTP Request node.
   - Create these fields:
     - `review_id` → `{{$json.id}}`
     - `product_name` → `{{$json.product_name}}`
     - `rating` → `{{$json.rating}}`
     - `reviewer` → `{{$json.reviewer}}`
     - `review_text` → `{{$json.review}}`
     - `created_at` → `{{$json.date_created}}`
     - `product_permalink` → `{{$json.product_permalink}}`
     - `status` → `{{$json.status}}`
   - This creates a stable internal schema.

5. **Add a Split In Batches node**
   - Name it: `Process reviews one by one`
   - Connect it after the Set node.
   - Use default settings, effectively processing items sequentially.
   - This node will also receive loop-back connections later.

6. **Add an IF node for review approval**
   - Name it: `Check review approval`
   - Connect it from the main batch output of `Process reviews one by one`.
   - Condition:
     - `{{$json.status}}`
     - equals
     - `approved`
   - Leave the false branch unconnected.

7. **Add an IF node for low rating**
   - Name it: `Check low rating`
   - Connect the true output of `Check review approval` to it.
   - Condition:
     - `{{$json.rating}}`
     - less than or equal to
     - `2`
   - Connect:
     - true output → next deduplication step
     - false output → back to `Process reviews one by one`

8. **Prepare Google Sheets for tracking**
   - Create a spreadsheet, for example: `New Order Track`
   - Create a sheet/tab called: `product_rating`
   - Add these columns:
     - `Product_name`
     - `rating`
     - `review_id`
     - `reviewer`
     - `review`
     - `created_at`
     - `product_link`
   - Keep row headers exactly aligned with the node mappings.
   - Ensure the Google account used in n8n has edit access.

9. **Add a Google Sheets lookup node**
   - Name it: `Find review in sheet`
   - Connect it from the true branch of `Check low rating`.
   - Node type: `Google Sheets`
   - Credential: Google Sheets OAuth2
   - Select the spreadsheet and the `product_rating` sheet.
   - Configure it to search rows where:
     - lookup column = `review_id`
     - lookup value = `{{$json.review_id}}`
   - The result should return matching rows, including `row_number` if available.

10. **Add an IF node for new-vs-existing decision**
    - Name it: `Is new review?`
    - Connect it after `Find review in sheet`.
    - Configure the condition:
      - `{{$items().length}}`
      - equals
      - `0`
    - True branch = new review
    - False branch = existing review

11. **Add an OpenAI node**
    - Name it: `Genrate Alert message`
    - Connect it from the true branch of `Is new review?`
    - Node type: `OpenAI` via LangChain
    - Credential: OpenAI API
    - Model: `gpt-4-turbo` or equivalent available model
    - Provide a prompt like:
      - Act as a customer support assistant
      - Create a short Slack alert message for a low-rated review
      - Under 4 lines
      - Professional and urgent tone
      - Remove HTML
      - Mention product name and issue clearly
      - Do not invent facts
    - Insert these variables:
      - `{{$json.product_name}}`
      - `{{$json.rating}}`
      - `{{$json.reviewer}}`
      - `{{$json.review_text}}`

12. **Add a Slack node**
    - Name it: `Send slack alert`
    - Connect it after the OpenAI node.
    - Credential: Slack API
    - Configure message text as:
      - `{{$json.output[0].content[0].text}}`
    - Select the destination channel.
    - Disable workflow link inclusion if desired.
    - Make sure the Slack app is authorized and present in the channel.

13. **Add a Google Sheets append node**
    - Name it: `Log new review`
    - Connect it after `Send slack alert`.
    - Operation: `Append`
    - Spreadsheet: same tracking spreadsheet
    - Sheet: `product_rating`
    - Map columns as follows:
      - `rating` → `{{$('Check low rating').item.json.rating}}`
      - `review` → `{{$('Check low rating').item.json.review_text}}`
      - `reviewer` → `{{$('Check low rating').item.json.reviewer}}`
      - `review_id` → `{{$('Check low rating').item.json.review_id}}`
      - `created_at` → `{{$('Check low rating').item.json.created_at}}`
      - `Product_name` → `{{$('Check low rating').item.json.product_name}}`
      - `product_link` → `{{$('Check low rating').item.json.product_permalink}}`
    - This node logs only newly alerted reviews.

14. **Add a Google Sheets update node**
    - Name it: `Update existing review`
    - Connect it from the false branch of `Is new review?`
    - Operation: `Update`
    - Spreadsheet: same tracking spreadsheet
    - Sheet: `product_rating`
    - Matching column: `row_number`
    - Include column mappings for:
      - `rating`
      - `review`
      - `reviewer`
      - `review_id`
      - `created_at`
      - `row_number`
      - `Product_name`
      - `product_link`

15. **Correct the update mappings carefully**
    - The provided workflow uses:
      - `{{$json.review}}`
      - `{{$json.Product_name}}`
      - `{{$json.product_link}}`
    - This only works if those fields come from the Google Sheets lookup output.
    - If you want to update the row using the latest WooCommerce data, use expressions from the original review item instead, for example by referencing upstream node data explicitly.
    - Also ensure `row_number` is returned by your lookup node.

16. **Create the loop-back connections**
    - Connect `Log new review` back to `Process reviews one by one`
    - Connect `Update existing review` back to `Process reviews one by one`
    - Connect the false output of `Check low rating` back to `Process reviews one by one`
    - This allows the batch node to continue with the next review after each branch completes.

17. **Test with sample WooCommerce data**
    - Verify that:
      - Approved 1- or 2-star reviews proceed
      - Approved 3+ star reviews are skipped
      - Non-approved reviews are skipped
      - New low-rated reviews trigger OpenAI + Slack + append
      - Existing low-rated reviews update their row and do not send Slack again

18. **Validate credentials**
    - **WooCommerce / HTTP Basic Auth**
      - Must have permission to read reviews
      - Confirm API endpoint and authentication method supported by your store
    - **Google Sheets OAuth2**
      - Must have read/write access to the spreadsheet
    - **OpenAI**
      - Model access and sufficient quota required
    - **Slack**
      - Bot token must allow message posting to the selected channel

19. **Activate the workflow**
    - Once tested, activate the workflow so the schedule trigger runs automatically every 5 hours.

20. **Recommended hardening improvements**
    - Add pagination handling in the WooCommerce HTTP Request if the store has many reviews.
    - Add an Error Trigger or error branch for API failures.
    - Normalize HTML stripping before or after OpenAI if you need deterministic formatting.
    - Consider logging alert timestamps in Google Sheets.
    - Add a date filter to fetch only recent reviews if the WooCommerce endpoint supports it.

### Sub-workflow setup
This workflow does **not** invoke any sub-workflows and does not contain any `Execute Workflow` nodes.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow runs on a schedule or webhook and processes reviews one by one to keep everything clean and controlled. First, it checks whether a review is approved and valid. Then it looks at the rating to find low-rated reviews that need attention. Before taking any action, the workflow checks a Google Sheet to see if the review already exists using the review ID. If the review already exists, it updates the existing record. If the review does not exist, it treats it as a new review and sends the data to OpenAI to create a clear alert message. Finally, alerts can be sent to Slack or email, and all reviews are logged in Google Sheets. This ensures no duplicate alerts and keeps a complete history of customer feedback. | Canvas note from the workflow |
| Setup steps included in the workflow canvas: use a schedule trigger or webhook, process reviews with Split in Batches, add IF checks for approval status and low rating, search Google Sheets by review ID, branch for new vs existing review, use OpenAI for alert creation, and send alerts via Slack or email before updating Google Sheets. | Canvas note from the workflow |
| The current implemented workflow uses the schedule trigger entry point only. The mention of webhook/email exists in the design note but is not implemented in the actual node graph. | Important implementation clarification |
| The HTTP Request node URL is empty in the provided workflow and must be configured before the workflow can run successfully. | Required setup note |
| There is a naming inconsistency in the node label `Genrate Alert message`; this is cosmetic and does not affect execution. | Naming note |
| There is a schema mismatch risk between normalized review fields (`review_text`, `product_name`, `product_permalink`) and fields expected in `Update existing review` (`review`, `Product_name`, `product_link`). | Important maintenance note |