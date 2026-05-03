Fix & Resend Guest Order Emails in Magento 2 via REST API

https://n8nworkflows.xyz/workflows/fix---resend-guest-order-emails-in-magento-2-via-rest-api-6707


# Fix & Resend Guest Order Emails in Magento 2 via REST API

### 1. Workflow Overview

This workflow is designed to fix and resend guest order confirmation emails in Magento 2 using the Magento REST API and direct database updates. It is triggered by a form submission which provides an order increment ID. The workflow then verifies if the order exists, updates the guest email in the database if necessary, and triggers a resend of the order confirmation email through Magento’s REST API.

**Target Use Cases:**  
- Merchants needing to resend order confirmation emails for guest checkout orders in Magento 2.  
- Correcting email addresses for guest orders after initial submission errors.  
- Automating the resend process to avoid manual intervention.

**Logical Blocks:**

- **1.1 Input Reception:** Captures the order increment ID from a form submission.
- **1.2 Order Retrieval and Verification:** Queries Magento via REST API to find the order.
- **1.3 Order Existence Check:** Conditional logic to verify if the order exists.
- **1.4 Guest Email Update:** Updates the guest email in the Magento database using MySQL.
- **1.5 Resend Confirmation Email:** Calls Magento REST API to resend the order confirmation email.
- **1.6 Flow Control:** Conditional nodes managing the branches and progression of the workflow.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception  
**Overview:**  
Receives the order increment ID submitted from an external form to trigger the workflow.

**Nodes Involved:**  
- On form submission

**Node Details:**  
- **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point of the workflow; listens for incoming form submissions.  
  - Configuration: Uses webhook to receive form data, expecting at least the order increment ID.  
  - Inputs: External HTTP POST form submission.  
  - Outputs: Passes form data to the next node.  
  - Edge Cases: Failure if form data is missing or malformed; webhook misconfiguration.  
  - Version: 2.2  

---

#### 2.2 Order Retrieval and Verification  
**Overview:**  
Fetches order details from Magento 2 REST API using the provided increment ID and verifies if the order exists.

**Nodes Involved:**  
- Get Order By Increment ID  
- Checks If Order Exist

**Node Details:**  
- **Get Order By Increment ID**  
  - Type: HTTP Request  
  - Role: Queries Magento 2 REST API endpoint to retrieve order data by increment ID.  
  - Configuration: Uses GET method; authentication credentials configured for Magento REST API access.  
  - Key Expressions: Uses order increment ID from the form submission.  
  - Inputs: Data from "On form submission" node.  
  - Outputs: Returns order JSON data or error if not found.  
  - Edge Cases: API authentication failure, network issues, order not found.  
  - Version: 4.2

- **Checks If Order Exist**  
  - Type: If  
  - Role: Checks if the response from the previous node contains valid order data.  
  - Configuration: Condition checks presence of order data or specific field existence.  
  - Inputs: Output from "Get Order By Increment ID".  
  - Outputs: Two branches — true (order exists), false (order does not exist).  
  - Edge Cases: False negatives if API returns unexpected data format.  
  - Version: 2.2

---

#### 2.3 Guest Email Update  
**Overview:**  
Updates the guest order email address in the Magento database directly using a MySQL node after verifying the order exists.

**Nodes Involved:**  
- Code  
- Update Guest Order Email

**Node Details:**  
- **Code**  
  - Type: Code  
  - Role: Prepares or transforms data for the database update, possibly formatting SQL or parameters.  
  - Configuration: JavaScript code processing the order data and form input to create a SQL update statement or parameters.  
  - Inputs: From "Checks If Order Exist" node's true branch.  
  - Outputs: SQL query or parameters for the MySQL node.  
  - Edge Cases: Runtime errors in code, unexpected input data structure.  
  - Version: 2

- **Update Guest Order Email**  
  - Type: MySQL  
  - Role: Executes the SQL update statement to modify the guest email in the database.  
  - Configuration: Uses preconfigured MySQL credentials with write permissions on Magento database.  
  - Inputs: SQL parameters from "Code" node.  
  - Outputs: Confirmation of SQL execution success or failure.  
  - Edge Cases: Database connection failure, SQL syntax errors, permission issues.  
  - Version: 2.4

---

#### 2.4 Resend Confirmation Email  
**Overview:**  
Triggers Magento REST API to resend the guest order confirmation email after the guest email update.

**Nodes Involved:**  
- Checks for Resend  
- resend order confirmation

**Node Details:**  
- **Checks for Resend**  
  - Type: If  
  - Role: Validates conditions to proceed with resending the confirmation email (e.g., ensuring update succeeded).  
  - Configuration: Condition based on previous node outputs to confirm readiness for resend.  
  - Inputs: From "Update Guest Order Email".  
  - Outputs: True branch triggers email resend, false branch halts workflow.  
  - Edge Cases: False negatives if update result is misinterpreted.  
  - Version: 2.2

- **resend order confirmation**  
  - Type: HTTP Request  
  - Role: Sends a POST or PUT request to Magento REST API endpoint to resend the order confirmation email.  
  - Configuration: Authenticated with Magento REST credentials; uses order increment ID or order ID.  
  - Inputs: From "Checks for Resend" node's true branch.  
  - Outputs: API response confirming resend success or error details.  
  - Edge Cases: API rate limits, authentication errors, invalid order state for resending email.  
  - Version: 4.2

---

### 3. Summary Table

| Node Name                 | Node Type         | Functional Role                     | Input Node(s)              | Output Node(s)             | Sticky Note |
|---------------------------|-------------------|-----------------------------------|----------------------------|----------------------------|-------------|
| On form submission        | Form Trigger      | Receive order increment ID via form | -                          | Get Order By Increment ID   |             |
| Get Order By Increment ID | HTTP Request      | Retrieve order data from Magento  | On form submission         | Checks If Order Exist       |             |
| Checks If Order Exist     | If                | Verify if order exists             | Get Order By Increment ID  | Code                       |             |
| Code                     | Code              | Prepare SQL update for guest email | Checks If Order Exist      | Update Guest Order Email    |             |
| Update Guest Order Email  | MySQL             | Update guest email in DB           | Code                       | Checks for Resend           |             |
| Checks for Resend         | If                | Confirm conditions to resend email | Update Guest Order Email   | resend order confirmation   |             |
| resend order confirmation | HTTP Request      | Trigger Magento to resend email    | Checks for Resend          | -                          |             |
| Sticky Note              | Sticky Note       | Comments or instructions           | -                          | -                          |             |
| Sticky Note1             | Sticky Note       | Comments or instructions           | -                          | -                          |             |
| Sticky Note2             | Sticky Note       | Comments or instructions           | -                          | -                          |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node**  
   - Name: "On form submission"  
   - Purpose: Receive order increment ID via webhook/form.  
   - No special parameters besides webhook URL setup.

2. **Create an HTTP Request node**  
   - Name: "Get Order By Increment ID"  
   - Method: GET  
   - URL: Magento REST API endpoint to get order by increment ID (e.g., `/rest/V1/orders?searchCriteria[filter_groups][0][filters][0][field]=increment_id&searchCriteria[filter_groups][0][filters][0][value]={{$json["orderIncrementId"]}}&searchCriteria[filter_groups][0][filters][0][condition_type]=eq`)  
   - Authentication: Configure Magento REST API credentials (OAuth or token).  
   - Connect "On form submission" → "Get Order By Increment ID".

3. **Create an If node**  
   - Name: "Checks If Order Exist"  
   - Condition: Check if the response contains order data (`{{$json["items"] && $json["items"].length > 0}}`).  
   - Connect "Get Order By Increment ID" → "Checks If Order Exist".

4. **Create a Code node**  
   - Name: "Code"  
   - Purpose: Extract guest email and prepare SQL update parameters. Example code snippet:  
     ```js
     const order = $json["items"][0];
     return [{ json: { email: order.customer_email, increment_id: order.increment_id } }];
     ```  
   - Connect true output of "Checks If Order Exist" → "Code".

5. **Create a MySQL node**  
   - Name: "Update Guest Order Email"  
   - Operation: Execute SQL query:  
     ```sql
     UPDATE sales_order SET customer_email = :email WHERE increment_id = :increment_id AND customer_id IS NULL;
     ```  
   - Parameters: Use values from "Code" node.  
   - Credentials: Configure MySQL credentials with access to Magento database.  
   - Connect "Code" → "Update Guest Order Email".

6. **Create an If node**  
   - Name: "Checks for Resend"  
   - Condition: Check if MySQL update was successful (e.g., affectedRows > 0).  
   - Connect "Update Guest Order Email" → "Checks for Resend".

7. **Create an HTTP Request node**  
   - Name: "resend order confirmation"  
   - Method: POST (or appropriate HTTP method per Magento API)  
   - URL: Magento REST API endpoint for resending order confirmation email (e.g., `/rest/V1/order/:orderId/resend-email`)  
   - Authentication: Magento REST API credentials.  
   - Use order ID or increment ID as necessary.  
   - Connect true output of "Checks for Resend" → "resend order confirmation".

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                                     |
|------------------------------------------------------------------------------|-------------------------------------------------------------------|
| The workflow depends on proper Magento 2 REST API authentication setup.      | Magento 2 developer documentation on REST API authentication.    |
| Direct MySQL updates require secure credentials and caution to avoid DB corruption. | Magento 2 database schema reference for sales_order table.        |
| Ensure API endpoints used for resending emails are enabled and accessible.   | Magento 2 REST API docs or custom module if endpoint is custom.   |
| Testing with test orders before production use is recommended.               | Magento 2 testing best practices.                                 |

---

**Disclaimer:** The provided description and workflow are derived exclusively from an automated n8n workflow. All data and operations comply with applicable content policies and handle legal, public data only.