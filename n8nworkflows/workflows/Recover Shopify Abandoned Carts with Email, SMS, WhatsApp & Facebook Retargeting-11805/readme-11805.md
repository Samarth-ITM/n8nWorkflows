Recover Shopify Abandoned Carts with Email, SMS, WhatsApp & Facebook Retargeting

https://n8nworkflows.xyz/workflows/recover-shopify-abandoned-carts-with-email--sms--whatsapp---facebook-retargeting-11805


# Recover Shopify Abandoned Carts with Email, SMS, WhatsApp & Facebook Retargeting

### 1. Workflow Overview

This workflow automates the recovery of Shopify abandoned carts through a multi-touch, time-delayed sequence leveraging Email, SMS, WhatsApp, and Facebook retargeting channels. It targets eCommerce merchants who want to maximize recovery rates by engaging customers progressively with personalized and segmented messaging informed by cart and customer data analytics.

The workflow is logically divided into the following blocks:

- **1.1 Intake & Configuration:** Receives abandoned cart webhook events and loads global workflow settings.
- **1.2 Normalize & Enrich Cart Data:** Normalizes incoming payloads and enriches data by fetching detailed cart and customer histories.
- **1.3 Reason Prediction & Personalization:** Predicts abandonment reasons, segments customers, assigns discounts, and generates personalized checkout URLs.
- **1.4 Routing Rules:** Applies simple routing based on customer segment and device type to control flow.
- **1.5 Timed Multi-Channel Sequence:**
  - 1h Wait & Email #1: Waits 1 hour, checks order completion, sends initial email if not completed.
  - 4h Wait & SMS: Waits 4 hours, checks order status, sends SMS with discount if not completed.
  - 24h Wait & Email #2: Waits 24 hours, checks order status, sends a second email with social proof and urgency.
  - 48h Wait & WhatsApp + Retargeting: Final check at 48 hours, sends WhatsApp reminder if needed, hashes customer data, adds to Facebook Custom Audience.
- **1.6 Attribution Tracking & High-Value Notification:** Logs all touchpoints to Google Sheets for multi-touch attribution and triggers Slack alerts for recovered high-value carts.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Intake & Configuration

**Overview:**  
Receives the abandoned cart webhook event and initializes global configuration variables used throughout the workflow.

**Nodes Involved:**  
- Abandoned Cart Webhook  
- Workflow Configuration

**Node Details:**

- **Abandoned Cart Webhook**  
  - Type: Webhook  
  - Role: Entry point for abandoned cart POST events from Shopify or middleware.  
  - Config: HTTP POST method on path `abandoned-cart`. Responds with last node output.  
  - Inputs: External webhook call.  
  - Outputs: Outputs raw event JSON.  
  - Edge cases: Invalid payloads, missing fields, network errors.

- **Workflow Configuration**  
  - Type: Set  
  - Role: Defines global constants for API URLs, Shopify domain, Google Sheets ID, and cart value threshold.  
  - Config: Assigns variables `highValueThreshold`, `cartApiUrl`, `customerApiUrl`, `shopifyDomain`, `googleSheetsId`.  
  - Inputs: Output of webhook node.  
  - Outputs: Configuration JSON for downstream nodes.  
  - Edge cases: Placeholder values must be replaced; missing or incorrect config breaks API calls.

---

#### 1.2 Normalize & Enrich Cart Data

**Overview:**  
Processes the raw webhook payload to normalize fields, then fetches detailed cart and customer data via API calls to enrich the information context.

**Nodes Involved:**  
- Normalize Cart Data  
- Fetch Cart Details  
- Fetch Customer History

**Node Details:**

- **Normalize Cart Data**  
  - Type: Set  
  - Role: Extracts and normalizes key fields such as `cart_id`, `customer_id`, `products`, `subtotal`, `device`, `country`, `email` from various potential JSON paths.  
  - Config: Uses expressions to safely access nested properties or provide defaults.  
  - Inputs: Workflow Configuration node output.  
  - Outputs: Standardized cart summary JSON.  
  - Edge cases: Missing fields, empty arrays, malformed payloads.

- **Fetch Cart Details**  
  - Type: HTTP Request  
  - Role: Calls external Cart API endpoint to retrieve full cart details using `cart_id`.  
  - Config: GET request to `${cartApiUrl}/${cart_id}`, JSON response, content-type header set.  
  - Inputs: Output from Normalize Cart Data.  
  - Outputs: Full cart details (line items, prices).  
  - Edge cases: API unavailability, authorization failures, invalid cart ID.

- **Fetch Customer History**  
  - Type: HTTP Request  
  - Role: Calls external Customer API to fetch purchase history and metrics using `customer_id`.  
  - Config: GET request to `${customerApiUrl}/${customer_id}`, JSON response, content-type header.  
  - Inputs: Normalize Cart Data output.  
  - Outputs: Customer order history and spending data.  
  - Edge cases: API errors, missing customer, rate limits.

---

#### 1.3 Reason Prediction & Personalization

**Overview:**  
Combines cart and customer data to predict abandonment reasons, segment customers, assign discounts, and prepare personalized checkout URLs.

**Nodes Involved:**  
- Predict Abandonment Reason (Code)  
- Personalization Engine (Code)  
- Customer Segment Check (If)  
- Device Type Check (If)

**Node Details:**

- **Predict Abandonment Reason**  
  - Type: Code  
  - Role: Aggregates data from cart and customer APIs; applies rule-based logic to predict why cart was abandoned, with confidence and recommended actions.  
  - Key logic: Analyzes cart value, order count, device type, and item count to classify reasons like "Price Sensitivity" or "Technical Issues".  
  - Inputs: Outputs of Fetch Cart Details and Fetch Customer History merged.  
  - Outputs: Enriched JSON including predicted reason, confidence, explanation, recommended action.  
  - Edge cases: Missing data, inconsistent API responses.

- **Personalization Engine**  
  - Type: Code  
  - Role: Segments customers into `new`, `returning`, or `vip`; assigns dynamic discount percentages and codes; determines A/B test group; builds checkout URL using Shopify domain and cart ID.  
  - Inputs: Output of Predict Abandonment Reason.  
  - Outputs: Personalization fields such as `customerSegment`, `discountCode`, `abGroup`, `checkoutUrl`.  
  - Edge cases: Missing customer names or emails, invalid Shopify domain.

- **Customer Segment Check**  
  - Type: If  
  - Role: Routes flow based on whether customer segment is not "new".  
  - Inputs: Personalization Engine output.  
  - Outputs: Passes only non-new customers to next node; new customers skip.  
  - Edge cases: Null or undefined segment.

- **Device Type Check**  
  - Type: If  
  - Role: Checks if device type is mobile or tablet to decide routing or messaging.  
  - Inputs: Personalization Engine output.  
  - Outputs: Branches workflow accordingly.  
  - Edge cases: Unknown device types default to desktop.

---

#### 1.4 Timed Multi-Channel Sequence

**Overview:**  
Executes a series of timed waits, order status checks, and sends messages across Email, SMS, and WhatsApp, with logging and multi-touch attribution.

**Nodes Involved:**  
- Wait 1 Hour, Wait Until 4 Hours, Wait Until 24 Hours, Wait Until 48 Hours (Wait nodes)  
- Check Order Status (1h, 4h, 24h, 48h) (Shopify nodes)  
- Check Order Completed (1h, 4h, 24h, 48h) (If nodes)  
- Generate Email 1 Content, Generate Email 2 Content, Generate WhatsApp Message (Code)  
- Send Email 1, Send Email 2 (SendGrid)  
- Prepare SMS Content (Set)  
- Send SMS with Discount (Twilio)  
- Send WhatsApp Message (WhatsApp node)  
- Log Email 1 Touchpoint, Log Email 2 Touchpoint, Log SMS Touchpoint (Google Sheets)  
- Attribution - Email 1, Merge Attribution - SMS, Merge Attribution - Email 2, Merge Attribution - WhatsApp (Code)  
- Mark Success - Order Completed (1h, 4h, 24h, 48h) (Set)  

**Node Details:**

- **Wait Nodes**  
  - Pause execution for defined hours (1, 4, 24, 48) to space touchpoints.  
  - Inputs: Flow from previous steps or conditional branches.

- **Check Order Status Nodes**  
  - Shopify API calls to retrieve current order status by order or cart ID.  
  - Used to decide if further recovery steps are needed.

- **Check Order Completed Nodes**  
  - Logical checks if order is `paid` or `completed` using financial or fulfillment status.  
  - Split flow into "order completed" and "not completed" branches.

- **Generate Email 1 Content**  
  - Code node generates a personalized HTML email for first reminder, including dynamic subject and body based on customer segment and cart info.

- **Send Email 1**  
  - SendGrid node sends the generated email using configured sender and recipient addresses.

- **Log Email 1 Touchpoint**  
  - Logs the sent email event details to Google Sheets for attribution tracking.

- **Attribution - Email 1**  
  - Code node initializes the customer journey with the first touchpoint recorded.

- **Prepare SMS Content**  
  - Constructs SMS message text with personalization and discount info.

- **Send SMS with Discount**  
  - Twilio node sends SMS message to customer phone number.

- **Log SMS Touchpoint**  
  - Logs SMS send event to Google Sheets.

- **Merge Attribution - SMS**  
  - Adds SMS touchpoint to existing attribution journey.

- **Generate Email 2 Content**  
  - Creates a second email with social proof, urgency messaging, and discount details.

- **Send Email 2**  
  - Sends the second email via SendGrid.

- **Log Email 2 Touchpoint**  
  - Logs second email touchpoint.

- **Merge Attribution - Email 2**  
  - Appends second email touchpoint to the journey.

- **Generate WhatsApp Message**  
  - Creates a final reminder message for WhatsApp including discount and urgency.

- **Send WhatsApp Message**  
  - Sends message via WhatsApp node with templated content.

- **Hash Customer Data for Facebook**  
  - Code node hashes customer email and phone using SHA256 for privacy-compliant Facebook Custom Audience upload.

- **Add to Retargeting Audience**  
  - Facebook Graph API node adds hashed identifiers to a specified Facebook Custom Audience for retargeting.

- **Merge Attribution - WhatsApp**  
  - Adds WhatsApp touchpoint to journey attribution.

- **Mark Success - Order Completed Nodes**  
  - Set nodes mark the recovery status and timestamp when an order is detected as completed at different checkpoints.

**Edge Cases:**  
- API rate limits or failures in Shopify, Twilio, SendGrid, Facebook, or Google Sheets can interrupt flow.  
- Incorrect or missing contact info can cause sending failures.  
- Timing delays depend on workflow execution environment; delays may drift.  
- WhatsApp template usage requires prior template approval and valid credentials.

---

#### 1.5 Attribution Tracking & High-Value Notification

**Overview:**  
Consolidates all touchpoint data, evaluates if the recovered cart exceeds a configured high-value threshold, and sends Slack notifications for such cases.

**Nodes Involved:**  
- Merge Attribution Data (Merge node)  
- Check Cart Value Threshold (If node)  
- Notify High-Value Recovery (Slack node)

**Node Details:**

- **Merge Attribution Data**  
  - Combines multiple attribution streams on `cart_id` to create a unified journey record.

- **Check Cart Value Threshold**  
  - Checks if cart subtotal exceeds the threshold defined in workflow configuration.

- **Notify High-Value Recovery**  
  - Sends Slack message to configured channel with cart details for recovered high-value orders.

---

### 3. Summary Table

| Node Name                     | Node Type           | Functional Role                                   | Input Node(s)                      | Output Node(s)                           | Sticky Note                                                                                              |
|-------------------------------|---------------------|--------------------------------------------------|----------------------------------|-----------------------------------------|--------------------------------------------------------------------------------------------------------|
| Abandoned Cart Webhook         | Webhook             | Receives abandoned cart POST events              | -                                | Workflow Configuration                   | ## Intake & Configuration: Receives cart event and loads global settings                              |
| Workflow Configuration         | Set                 | Sets global parameters like API URLs, thresholds| Abandoned Cart Webhook            | Normalize Cart Data                      | ## Intake & Configuration                                                                             |
| Normalize Cart Data            | Set                 | Normalizes incoming cart payload                  | Workflow Configuration            | Fetch Cart Details, Fetch Customer History | ## Normalize & Enrich Cart Data                                                                       |
| Fetch Cart Details             | HTTP Request        | Fetches full cart details from API                | Normalize Cart Data               | Predict Abandonment Reason               | ## Normalize & Enrich Cart Data                                                                       |
| Fetch Customer History         | HTTP Request        | Fetches customer purchase history                  | Normalize Cart Data               | Predict Abandonment Reason               | ## Normalize & Enrich Cart Data                                                                       |
| Predict Abandonment Reason     | Code                | Predicts reason for abandonment                    | Fetch Cart Details, Fetch Customer History | Personalization Engine               | ## Reason Prediction & Personalization                                                               |
| Personalization Engine         | Code                | Segments customer, assigns discount, builds URL  | Predict Abandonment Reason        | Customer Segment Check                   | ## Reason Prediction & Personalization                                                               |
| Customer Segment Check         | If                  | Routes based on customer segment                   | Personalization Engine            | Device Type Check                        | ## Routing Rules                                                                                      |
| Device Type Check              | If                  | Routes based on device type                         | Customer Segment Check            | Wait 1 Hour                             | ## Routing Rules                                                                                      |
| Wait 1 Hour                   | Wait                | Waits 1 hour before next action                     | Device Type Check                | Check Order Status (1h)                  | ## 1h Checkpoint & Email 1                                                                            |
| Check Order Status (1h)        | Shopify             | Fetch order status at 1 hour                        | Wait 1 Hour                     | Check Order Completed (1h)               | ## 1h Checkpoint & Email 1                                                                            |
| Check Order Completed (1h)     | If                  | Checks if order is completed at 1 hour             | Check Order Status (1h)          | Mark Success - Order Completed (1h), Generate Email 1 Content | ## 1h Checkpoint & Email 1                                                                            |
| Mark Success - Order Completed (1h) | Set           | Marks order completed at 1 hour                     | Check Order Completed (1h)       | -                                       |                                                                                                        |
| Generate Email 1 Content       | Code                | Generates first personalized email content         | Check Order Completed (1h)       | Send Email 1                            | ## 1h Checkpoint & Email 1                                                                            |
| Send Email 1                  | SendGrid            | Sends first email                                   | Generate Email 1 Content         | Log Email 1 Touchpoint                   | ## 1h Checkpoint & Email 1                                                                            |
| Log Email 1 Touchpoint         | Google Sheets       | Logs first email touchpoint                          | Send Email 1                    | Attribution - Email 1                    | ## 1h Checkpoint & Email 1                                                                            |
| Attribution - Email 1          | Code                | Initiates attribution journey with email_1         | Log Email 1 Touchpoint           | Wait Until 4 Hours                      |                                                                                                        |
| Wait Until 4 Hours             | Wait                | Waits 4 hours before next action                     | Attribution - Email 1            | Check Order Status (4h)                  | ## 4h Checkpoint & SMS                                                                               |
| Check Order Status (4h)        | Shopify             | Fetch order status at 4 hours                        | Wait Until 4 Hours              | Check Order Completed (4h)               | ## 4h Checkpoint & SMS                                                                               |
| Check Order Completed (4h)     | If                  | Checks if order is completed at 4 hours             | Check Order Status (4h)          | Mark Success - Order Completed (4h), Prepare SMS Content | ## 4h Checkpoint & SMS                                                                               |
| Mark Success - Order Completed (4h) | Set           | Marks order completed at 4 hours                     | Check Order Completed (4h)       | -                                       |                                                                                                        |
| Prepare SMS Content            | Set                 | Builds personalized SMS message                      | Check Order Completed (4h)       | Send SMS with Discount                  | ## 4h Checkpoint & SMS                                                                               |
| Send SMS with Discount         | Twilio              | Sends SMS with discount code                         | Prepare SMS Content              | Log SMS Touchpoint                      | ## 4h Checkpoint & SMS                                                                               |
| Log SMS Touchpoint             | Google Sheets       | Logs SMS touchpoint                                  | Send SMS with Discount          | Wait Until 24 Hours, Merge Attribution - SMS | ## 4h Checkpoint & SMS                                                                               |
| Merge Attribution - SMS        | Code                | Adds SMS touchpoint to attribution journey           | Log SMS Touchpoint              | Merge Attribution Data                   |                                                                                                        |
| Wait Until 24 Hours            | Wait                | Waits 24 hours before next action                     | Log SMS Touchpoint, Merge Attribution - Email 2 | Check Order Status (24h)         | ## 24h Checkpoint & Email 2                                                                         |
| Check Order Status (24h)       | Shopify             | Fetch order status at 24 hours                        | Wait Until 24 Hours             | Check Order Completed (24h)              | ## 24h Checkpoint & Email 2                                                                         |
| Check Order Completed (24h)    | If                  | Checks if order is completed at 24 hours             | Check Order Status (24h)         | Mark Success - Order Completed (24h), Generate Email 2 Content | ## 24h Checkpoint & Email 2                                                                         |
| Mark Success - Order Completed (24h) | Set           | Marks order completed at 24 hours                     | Check Order Completed (24h)      | -                                       |                                                                                                        |
| Generate Email 2 Content       | Code                | Generates second email with social proof & urgency   | Check Order Completed (24h)      | Send Email 2                            | ## 24h Checkpoint & Email 2                                                                         |
| Send Email 2                  | SendGrid            | Sends second email                                   | Generate Email 2 Content         | Log Email 2 Touchpoint                   | ## 24h Checkpoint & Email 2                                                                         |
| Log Email 2 Touchpoint         | Google Sheets       | Logs second email touchpoint                          | Send Email 2                   | Wait Until 48 Hours, Merge Attribution - Email 2 | ## 24h Checkpoint & Email 2                                                                         |
| Merge Attribution - Email 2    | Code                | Adds second email touchpoint to journey               | Log Email 2 Touchpoint          | Wait Until 24 Hours                      |                                                                                                        |
| Wait Until 48 Hours            | Wait                | Waits 48 hours before next action                     | Log Email 2 Touchpoint, Merge Attribution - Email 2 | Check Order Completed (48h)     | ## 48h Final Touchpoint & Retargeting                                                             |
| Check Order Completed (48h)    | If                  | Checks if order is completed at 48 hours             | Wait Until 48 Hours             | Mark Success - Order Completed (48h), Generate WhatsApp Message | ## 48h Final Touchpoint & Retargeting                                                             |
| Mark Success - Order Completed (48h) | Set           | Marks order completed at 48 hours                     | Check Order Completed (48h)      | -                                       |                                                                                                        |
| Generate WhatsApp Message      | Code                | Generates final WhatsApp reminder message              | Check Order Completed (48h)      | Send WhatsApp Message                   | ## 48h Final Touchpoint & Retargeting                                                             |
| Send WhatsApp Message          | WhatsApp            | Sends WhatsApp message using template                 | Generate WhatsApp Message        | Hash Customer Data for Facebook          | ## 48h Final Touchpoint & Retargeting                                                             |
| Hash Customer Data for Facebook| Code                | Hashes email and phone for Facebook Custom Audience   | Send WhatsApp Message            | Add to Retargeting Audience              | ## 48h Final Touchpoint & Retargeting                                                             |
| Add to Retargeting Audience    | Facebook Graph API  | Adds hashed customer data to Facebook Custom Audience | Hash Customer Data for Facebook  | Merge Attribution - WhatsApp             | ## 48h Final Touchpoint & Retargeting                                                             |
| Merge Attribution - WhatsApp   | Code                | Adds WhatsApp touchpoint to journey                    | Add to Retargeting Audience      | Merge Attribution Data                   |                                                                                                        |
| Merge Attribution Data         | Merge               | Combines all touchpoints into a single journey record | Attribution - Email 1, Merge Attribution - SMS, Merge Attribution - WhatsApp | Check Cart Value Threshold      | ## Attribution Merge & High-Value Alert                                                            |
| Check Cart Value Threshold     | If                  | Checks if cart value exceeds high-value threshold     | Merge Attribution Data           | Notify High-Value Recovery                | ## Attribution Merge & High-Value Alert                                                            |
| Notify High-Value Recovery     | Slack               | Sends Slack notification for high-value recovered cart | Check Cart Value Threshold       | -                                       | ## Attribution Merge & High-Value Alert                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `abandoned-cart`  
   - Response Mode: Last Node

2. **Add Set Node "Workflow Configuration":**  
   - Add variables:  
     - `highValueThreshold` (number, e.g., 500)  
     - `cartApiUrl` (string, set your Cart API endpoint)  
     - `customerApiUrl` (string, set your Customer API endpoint)  
     - `shopifyDomain` (string, your Shopify store domain)  
     - `googleSheetsId` (string, your Google Sheets spreadsheet ID)  
   - Connect from Webhook node.

3. **Add Set Node "Normalize Cart Data":**  
   - Extract and normalize fields:  
     - `cart_id`, `customer_id`, `products`, `subtotal`, `device`, `country`, `email`  
   - Use expressions to safely access nested payloads.  
   - Connect from Workflow Configuration.

4. **Add HTTP Request Nodes:**  
   - "Fetch Cart Details": GET `${cartApiUrl}/${cart_id}` with JSON headers.  
   - "Fetch Customer History": GET `${customerApiUrl}/${customer_id}` with JSON headers.  
   - Connect both from Normalize Cart Data.

5. **Add Code Node "Predict Abandonment Reason":**  
   - Merge data from cart details, customer history, and normalized data.  
   - Implement rule-based logic for abandonment reasons.  
   - Connect from both Fetch Cart Details and Fetch Customer History.

6. **Add Code Node "Personalization Engine":**  
   - Segment customers (new, returning, vip).  
   - Assign discount codes and percentages.  
   - Assign A/B test group randomly.  
   - Build checkout URL using Shopify domain and cart ID.  
   - Connect from Predict Abandonment Reason.

7. **Add If Node "Customer Segment Check":**  
   - Condition: Customer segment not equal to "new".  
   - Connect from Personalization Engine.

8. **Add If Node "Device Type Check":**  
   - Condition: Device equals "mobile" or "tablet".  
   - Connect from Customer Segment Check.

9. **Add Wait Node "Wait 1 Hour":**  
   - Amount: 1 hour.  
   - Connect from Device Type Check.

10. **Add Shopify Node "Check Order Status (1h)":**  
    - Operation: Get order by `order_id` or `cart_id`.  
    - Connect from Wait 1 Hour.

11. **Add If Node "Check Order Completed (1h)":**  
    - Condition: financial_status == "paid" OR fulfillment_status == "completed".  
    - Connect from Check Order Status (1h).

12. **Add Set Node "Mark Success - Order Completed (1h)":**  
    - Assign recovery status and timestamp.  
    - Connect from Check Order Completed (1h) — success branch.

13. **Add Code Node "Generate Email 1 Content":**  
    - Personalized HTML email with subject and body including customer name, cart value, checkout URL.  
    - Connect from Check Order Completed (1h) — failure branch.

14. **Add SendGrid Node "Send Email 1":**  
    - Configure sender email, recipient email from customer data, content type HTML.  
    - Use email content from previous node.  
    - Connect from Generate Email 1 Content.

15. **Add Google Sheets Node "Log Email 1 Touchpoint":**  
    - Append or update row in "Touchpoints" sheet with cart ID, channel=email, touchpoint type=email_1, timestamp, customer ID, AB group.  
    - Use service account credentials.  
    - Connect from Send Email 1.

16. **Add Code Node "Attribution - Email 1":**  
    - Initiates journey attribution data with first touchpoint.  
    - Connect from Log Email 1 Touchpoint.

17. **Add Wait Node "Wait Until 4 Hours":**  
    - Wait 4 hours.  
    - Connect from Attribution - Email 1.

18. **Add Shopify Node "Check Order Status (4h)":**  
    - Same config as 1h check.  
    - Connect from Wait Until 4 Hours.

19. **Add If Node "Check Order Completed (4h)":**  
    - Same condition as 1h check.  
    - Connect from Check Order Status (4h).

20. **Add Set Node "Mark Success - Order Completed (4h)":**  
    - Assign recovery status and timestamp, touchpoint=email_1.  
    - Connect from Check Order Completed (4h) success branch.

21. **Add Set Node "Prepare SMS Content":**  
    - Build SMS text with customer name, item count, discount code, discount percent, checkout URL.  
    - Connect from Check Order Completed (4h) failure branch.

22. **Add Twilio Node "Send SMS with Discount":**  
    - Configure Twilio credentials and phone numbers.  
    - Send SMS with message from previous node.  
    - Connect from Prepare SMS Content.

23. **Add Google Sheets Node "Log SMS Touchpoint":**  
    - Log SMS send event similar to email logging.  
    - Connect from Send SMS with Discount.

24. **Add Code Node "Merge Attribution - SMS":**  
    - Add SMS touchpoint to journey.  
    - Connect from Log SMS Touchpoint.

25. **Add Wait Node "Wait Until 24 Hours":**  
    - Wait 24 hours.  
    - Connect from Log SMS Touchpoint and from Merge Attribution - Email 2 (see below).

26. **Add Shopify Node "Check Order Status (24h)":**  
    - Connect from Wait Until 24 Hours.

27. **Add If Node "Check Order Completed (24h)":**  
    - Connect from Check Order Status (24h).

28. **Add Set Node "Mark Success - Order Completed (24h)":**  
    - Recovery status, timestamp, touchpoint=sms.  
    - Connect from success branch.

29. **Add Code Node "Generate Email 2 Content":**  
    - Creates social proof and urgency email with discount.  
    - Connect from failure branch.

30. **Add SendGrid Node "Send Email 2":**  
    - Configure SendGrid as in email 1.  
    - Connect from Generate Email 2 Content.

31. **Add Google Sheets Node "Log Email 2 Touchpoint":**  
    - Log email 2 send event.  
    - Connect from Send Email 2.

32. **Add Code Node "Merge Attribution - Email 2":**  
    - Append email 2 to journey.  
    - Connect from Log Email 2 Touchpoint.

33. **Add Wait Node "Wait Until 48 Hours":**  
    - Wait 48 hours.  
    - Connect from Log Email 2 Touchpoint and Merge Attribution - Email 2.

34. **Add If Node "Check Order Completed (48h)":**  
    - Connect from Wait Until 48 Hours.

35. **Add Set Node "Mark Success - Order Completed (48h)":**  
    - Recovery status, timestamp, touchpoint=email_2.  
    - Connect from success branch.

36. **Add Code Node "Generate WhatsApp Message":**  
    - Final reminder message with discount and urgency.  
    - Connect from failure branch.

37. **Add WhatsApp Node "Send WhatsApp Message":**  
    - Configure template, phone number ID, recipient phone.  
    - Connect from Generate WhatsApp Message.

38. **Add Code Node "Hash Customer Data for Facebook":**  
    - Hash email and phone for FB audience privacy.  
    - Connect from Send WhatsApp Message.

39. **Add Facebook Graph API Node "Add to Retargeting Audience":**  
    - Post hashed data to configured Facebook Custom Audience ID.  
    - Connect from Hash Customer Data for Facebook.

40. **Add Code Node "Merge Attribution - WhatsApp":**  
    - Append WhatsApp touchpoint to journey.  
    - Connect from Add to Retargeting Audience.

41. **Add Merge Node "Merge Attribution Data":**  
    - Combine all attribution streams on cart ID.  
    - Connect from Attribution - Email 1, Merge Attribution - SMS, Merge Attribution - WhatsApp.

42. **Add If Node "Check Cart Value Threshold":**  
    - Condition: `subtotal > highValueThreshold`.  
    - Connect from Merge Attribution Data.

43. **Add Slack Node "Notify High-Value Recovery":**  
    - Sends Slack message with cart and customer info.  
    - Connect from Check Cart Value Threshold success branch.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                       | Context or Link                                                                                           |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow recovers abandoned carts using a timed, multi-channel sequence (Email → SMS → Email → WhatsApp) and logs every touchpoint for multi-touch attribution.                                                                                              | Sticky Note summary at workflow start                                                                    |
| To set up, configure API URLs, Shopify domain, Sheets ID, and thresholds in **Workflow Configuration**; connect required credentials: Shopify, SendGrid, Twilio, WhatsApp, Facebook Graph API, Google Sheets, Slack.                                                | Setup instructions in Sticky Note                                                                        |
| Create a Google Sheets "Touchpoints" sheet with columns: `cart_id`, `customer_id`, `touchpoint_type`, `timestamp`, `channel`, `ab_group` to track engagement across channels.                                                                                      | Google Sheets integration note                                                                            |
| WhatsApp messages use a predefined template `abandoned_cart_final`; ensure WhatsApp Business API access and template approval before use.                                                                                                                         | WhatsApp node configuration note                                                                          |
| Facebook Custom Audience addition uses hashed identifiers for privacy; ensure Facebook App permissions and correct audience ID environment variable setup.                                                                                                         | Facebook Graph API integration note                                                                       |
| Slack notifications require channel name and webhook set in environment variables; useful to monitor high-value recoveries in real-time.                                                                                                                         | Slack integration note                                                                                    |
| The workflow applies basic but effective logic for abandonment reason prediction and personalization; it can be extended with AI or machine learning nodes for enhanced prediction accuracy.                                                                        | Personalization and prediction logic overview                                                            |
| Timing relies on n8n wait nodes; for best results, ensure stable n8n execution environment and consider time zone impacts on customer engagement.                                                                                                                | Workflow timing considerations                                                                             |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. This processing adheres strictly to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.