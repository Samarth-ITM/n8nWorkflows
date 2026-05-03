Send Personalized Product Recommendations via Email After Shopify Purchases

https://n8nworkflows.xyz/workflows/send-personalized-product-recommendations-via-email-after-shopify-purchases-6579


# Send Personalized Product Recommendations via Email After Shopify Purchases

### 1. Workflow Overview

This workflow is designed to **send personalized product recommendation emails to customers after they complete a purchase on a Shopify store**. It targets e-commerce use cases aiming to increase upsell and cross-sell opportunities by leveraging purchase data to generate relevant product suggestions.

The workflow is organized into the following logical blocks:

- **1.1 Order Reception and Delay**: Listens for new Shopify orders and waits 10 minutes before processing, allowing any immediate post-order processes to complete.
- **1.2 Recommendation Generation**: Extracts or generates product recommendation IDs based on the purchased items.
- **1.3 Conditional Flow**: Checks if recommendations exist to decide whether to proceed.
- **1.4 Product Data Enrichment**: Fetches detailed product information for each recommended product.
- **1.5 Recommendation Formatting and Email Sending**: Formats the recommendations into an HTML email and sends it to the customer.

---

### 2. Block-by-Block Analysis

#### 1.1 Order Reception and Delay

- **Overview**: This block initiates the workflow when a new order is created in Shopify and introduces a delay to ensure all order data is finalized before proceeding.
- **Nodes Involved**:  
  - Order Created Trigger  
  - Wait 10 Minutes

##### Node: Order Created Trigger
- **Type and Role**: Shopify Trigger node that listens for new order creation events.
- **Configuration**: Uses the built-in Shopify webhook to detect "order created" events.
- **Expressions/Variables**: Receives the full order payload from Shopify.
- **Connections**: Outputs to "Wait 10 Minutes".
- **Version Requirements**: n8n version supporting Shopify Trigger node (v1).
- **Potential Failures**: Webhook registration failure, Shopify API downtime, authentication errors.
- **Notes**: Webhook ID is "order-upsell-webhook".

##### Node: Wait 10 Minutes
- **Type and Role**: Wait node that pauses the workflow for 10 minutes.
- **Configuration**: Default wait duration set to 10 minutes.
- **Expressions/Variables**: None.
- **Connections**: Outputs to "Generate Recommendation IDs".
- **Version Requirements**: Requires n8n v1.1 or above for this wait node version.
- **Potential Failures**: None typical, but long delays may cause workflow queuing issues.

#### 1.2 Recommendation Generation

- **Overview**: Generates product recommendation IDs based on the customer's order to prepare for fetching detailed product information.
- **Nodes Involved**:  
  - Generate Recommendation IDs  
  - If Recommendations Exist

##### Node: Generate Recommendation IDs
- **Type and Role**: Code node that processes the order data and derives a list of recommended product IDs.
- **Configuration**: Custom JavaScript logic to extract or determine related product IDs.
- **Expressions/Variables**: Likely accesses order items, customer data, or predefined recommendation logic.
- **Connections**: Outputs to "If Recommendations Exist".
- **Version Requirements**: Code node version 2.
- **Potential Failures**: Code errors, empty recommendation list, data format inconsistencies.

##### Node: If Recommendations Exist
- **Type and Role**: Conditional node that checks whether the recommendation list is non-empty.
- **Configuration**: Condition set to verify that the list of recommendation IDs is not empty.
- **Expressions/Variables**: Checks array length or presence of recommendation IDs.
- **Connections**: True branch to "Split Recommended Products"; False branch implicitly ends workflow.
- **Version Requirements**: If node version 2.
- **Potential Failures**: Expression evaluation errors; misconfigured condition.

#### 1.3 Product Data Enrichment

- **Overview**: Splits the recommendation list into batches, retrieves detailed product data for each recommended product, and merges the results.
- **Nodes Involved**:  
  - Split Recommended Products  
  - Get Product Details  
  - Merge Product Details (v3.2)

##### Node: Split Recommended Products
- **Type and Role**: SplitInBatches node that processes recommendations in manageable chunks.
- **Configuration**: Default batch size; splits the array of recommended product IDs.
- **Expressions/Variables**: Receives the array of IDs from previous node.
- **Connections**: Outputs each batch to "Get Product Details".
- **Version Requirements**: SplitInBatches node version 3.
- **Potential Failures**: Batch size misconfiguration, empty input arrays.

##### Node: Get Product Details
- **Type and Role**: Shopify node that fetches detailed product information for each recommended product ID.
- **Configuration**: Uses Shopify API credentials and product IDs from the batch.
- **Expressions/Variables**: Product IDs passed dynamically.
- **Connections**: Outputs to "Merge Product Details (v3.2)".
- **Version Requirements**: Shopify node v1.
- **Potential Failures**: API errors, rate limits, authentication failures, missing products.

##### Node: Merge Product Details (v3.2)
- **Type and Role**: Merge node that consolidates product details from batches into a single dataset.
- **Configuration**: Merge mode to combine multiple inputs into one output.
- **Expressions/Variables**: None.
- **Connections**: Outputs to "Format Recommendations HTML".
- **Version Requirements**: Merge node version 3.2.
- **Potential Failures**: Mismatched input data, empty inputs.

#### 1.4 Recommendation Formatting and Email Sending

- **Overview**: Formats the detailed product recommendations into an HTML email and dispatches the email to the customer.
- **Nodes Involved**:  
  - Format Recommendations HTML  
  - Send Upsell Email

##### Node: Format Recommendations HTML
- **Type and Role**: Code node that generates HTML markup for the product recommendations email.
- **Configuration**: Uses JavaScript to craft an email body with product images, links, and descriptions.
- **Expressions/Variables**: Uses merged product details as input.
- **Connections**: Outputs formatted HTML content to "Send Upsell Email".
- **Version Requirements**: Code node v2.
- **Potential Failures**: Code errors, malformed HTML, missing product data.

##### Node: Send Upsell Email
- **Type and Role**: EmailSend node that sends the formatted recommendation email to the customer.
- **Configuration**: Email SMTP or service configured; recipient email dynamically set from order data.
- **Expressions/Variables**: Email subject, body (HTML), recipient address derived from order payload.
- **Connections**: Terminal node.
- **Version Requirements**: EmailSend node v2.1.
- **Potential Failures**: SMTP/auth errors, invalid email addresses, rate limits.

---

### 3. Summary Table

| Node Name                   | Node Type             | Functional Role                     | Input Node(s)             | Output Node(s)                  | Sticky Note                          |
|-----------------------------|-----------------------|-----------------------------------|---------------------------|--------------------------------|------------------------------------|
| Order Created Trigger        | Shopify Trigger       | Initiates workflow on new orders  |                           | Wait 10 Minutes                |                                    |
| Wait 10 Minutes             | Wait                  | Delays processing by 10 minutes   | Order Created Trigger      | Generate Recommendation IDs    |                                    |
| Generate Recommendation IDs | Code                  | Creates list of recommended products | Wait 10 Minutes          | If Recommendations Exist       |                                    |
| If Recommendations Exist    | If                    | Checks if recommendations exist   | Generate Recommendation IDs | Split Recommended Products     |                                    |
| Split Recommended Products  | SplitInBatches        | Processes recommendations in batches | If Recommendations Exist | Get Product Details            |                                    |
| Get Product Details         | Shopify               | Fetches detailed product info     | Split Recommended Products | Merge Product Details (v3.2)   |                                    |
| Merge Product Details (v3.2)| Merge                 | Combines product info batches     | Get Product Details        | Format Recommendations HTML    |                                    |
| Format Recommendations HTML | Code                  | Formats HTML email content         | Merge Product Details (v3.2) | Send Upsell Email             |                                    |
| Send Upsell Email           | EmailSend             | Sends email with recommendations  | Format Recommendations HTML |                              |                                    |
| Workflow Info               | Sticky Note           |                                   |                           |                                |                                    |
| Features & Config           | Sticky Note           |                                   |                           |                                |                                    |
| Business Value              | Sticky Note           |                                   |                           |                                |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node**  
   - Add a **Shopify Trigger** node named "Order Created Trigger".  
   - Configure it to listen for new order creation events via Shopify webhook.  
   - Authenticate with Shopify API credentials.  
   - Ensure webhook ID is set or generated as needed.

2. **Add a Wait Node**  
   - Add a **Wait** node named "Wait 10 Minutes".  
   - Set the wait time to 10 minutes.  
   - Connect output of "Order Created Trigger" to this node.

3. **Add Code Node to Generate Recommendations**  
   - Add a **Code** node named "Generate Recommendation IDs".  
   - Write JavaScript to parse incoming order data and produce an array of recommended product IDs.  
   - Connect output of "Wait 10 Minutes" to this node.

4. **Add Conditional Check Node**  
   - Add an **If** node named "If Recommendations Exist".  
   - Configure the condition to check if the array of recommendations is not empty (e.g., `{{ $json["recommendationIds"].length > 0 }}`).  
   - Connect output of "Generate Recommendation IDs" to this node.

5. **Add SplitInBatches Node**  
   - Add a **SplitInBatches** node named "Split Recommended Products".  
   - Set batch size as appropriate (default is acceptable).  
   - Connect the "True" output of "If Recommendations Exist" to this node.

6. **Add Shopify Node to Get Product Details**  
   - Add a **Shopify** node named "Get Product Details".  
   - Configure it to retrieve product details by product ID from current batch.  
   - Use Shopify credentials.  
   - Connect output of "Split Recommended Products" to this node.

7. **Add Merge Node**  
   - Add a **Merge** node named "Merge Product Details (v3.2)".  
   - Set merge mode to "Merge By Index" or "Append" to combine batches.  
   - Connect output of "Get Product Details" to this node.

8. **Add Code Node to Format HTML Email**  
   - Add a **Code** node named "Format Recommendations HTML".  
   - Write JavaScript to generate an HTML email body using the merged product details.  
   - Connect output of "Merge Product Details (v3.2)" to this node.

9. **Add EmailSend Node**  
   - Add an **EmailSend** node named "Send Upsell Email".  
   - Configure SMTP or email service credentials (e.g., Gmail, Outlook).  
   - Set recipient email dynamically from original order data.  
   - Use the HTML content from "Format Recommendations HTML" as the email body.  
   - Connect output of "Format Recommendations HTML" to this node.

10. **Test the Workflow**  
    - Place test orders in Shopify and verify the workflow triggers, processes recommendations, and sends emails successfully.  
    - Monitor for errors in API calls, email sending, and code execution.

---

### 5. General Notes & Resources

| Note Content                                                                                                                 | Context or Link                      |
|------------------------------------------------------------------------------------------------------------------------------|------------------------------------|
| This workflow enhances customer engagement by automating personalized upsell emails based on real purchase data.             | Business value context              |
| Ensure Shopify API credentials have webhook permissions enabled.                                                              | Shopify API documentation          |
| Use proper SMTP credentials for the EmailSend node to avoid email delivery issues.                                            | n8n Email integration docs         |
| Consider adding error handling nodes or notifications for failed email sends or API errors.                                   | n8n error workflow best practices  |
| For advanced recommendations, integrate machine learning or external AI services to improve product suggestion logic.         | External AI integration suggestion |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.