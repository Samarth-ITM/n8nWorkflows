Send Automated Discount Vouchers to High-Value Magento 2 Customers with Gmail

https://n8nworkflows.xyz/workflows/send-automated-discount-vouchers-to-high-value-magento-2-customers-with-gmail-6982


# Send Automated Discount Vouchers to High-Value Magento 2 Customers with Gmail

### 1. Workflow Overview

This workflow automates the process of sending personalized discount vouchers via Gmail to high-value customers of a Magento 2 e-commerce store. It targets customers who have completed orders within the last three months and generates sales rules and coupons accordingly. The workflow comprises several logical blocks:

- **1.1 Scheduling Trigger:** Initiates the workflow periodically using a Cron node.
- **1.2 Asset Preparation:** Retrieves and processes media assets like the company logo for branding the emails.
- **1.3 Date Range Calculation:** Computes the date range (last 3 months) to filter orders.
- **1.4 Order Retrieval:** Fetches completed orders from Magento 2 within the specified date range.
- **1.5 Customer Value Calculation:** Calculates total order values per customer to identify high-value customers.
- **1.6 Conditional Filtering:** Decides whether customers meet criteria for voucher generation.
- **1.7 Sales Rule and Coupon Generation:** Creates new Magento 2 sales rules and generates coupons per eligible customer.
- **1.8 Voucher Sending:** Sends personalized discount vouchers via Gmail to selected customers.
- **1.9 Reporting:** Sends a summary report email about the voucher dispatch process.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling Trigger

- **Overview:**  
  Triggers the workflow execution on a scheduled basis using a Cron node.

- **Nodes Involved:**  
  - Cron

- **Node Details:**  
  - **Cron**  
    - Type: Trigger  
    - Configuration: Default scheduling parameters (periodicity not explicitly set in JSON, assumed periodic)  
    - Inputs: None (trigger node)  
    - Outputs: Connected to "Get Media Path" node  
    - Edge Cases: Misconfiguration of schedule may cause missed or excessive runs  
    - Version: n8n 1.x and above  

#### 2.2 Asset Preparation

- **Overview:**  
  Retrieves the media path for branding assets, then processes and caches the company logo for use in emails.

- **Nodes Involved:**  
  - Get Media Path  
  - SetOrGet Logo

- **Node Details:**  
  - **Get Media Path**  
    - Type: HTTP Request  
    - Role: Requests media URL or API endpoint to obtain logo path or media files (likely Magento 2 media endpoint)  
    - Configuration: HTTP GET request (details abstracted), expects media path in response  
    - Inputs: From Cron  
    - Outputs: To SetOrGet Logo  
    - Potential Failures: Network timeouts, authentication errors, invalid response format  
  - **SetOrGet Logo**  
    - Type: Code (JavaScript)  
    - Role: Processes the media path response, caches or sets logo data for reuse  
    - Configuration: Custom JavaScript logic to extract and store logo URL or base64 data  
    - Inputs: From Get Media Path  
    - Outputs: To Get 3 Month Dates  
    - Potential Failures: Parsing errors, undefined data handling  

#### 2.3 Date Range Calculation

- **Overview:**  
  Calculates start and end dates corresponding to the last 3 months for filtering orders.

- **Nodes Involved:**  
  - Get 3 Month Dates

- **Node Details:**  
  - **Get 3 Month Dates**  
    - Type: Code (JavaScript)  
    - Role: Generates date range boundaries used in subsequent order queries  
    - Configuration: Computes current date and subtracts three months for start date  
    - Inputs: From SetOrGet Logo  
    - Outputs: To Get Completed Orders  
    - Edge Cases: Timezone handling, daylight savings time adjustments  

#### 2.4 Order Retrieval

- **Overview:**  
  Fetches the list of completed orders from Magento 2 API within the date range calculated.

- **Nodes Involved:**  
  - Get Completed Orders

- **Node Details:**  
  - **Get Completed Orders**  
    - Type: HTTP Request  
    - Role: Calls Magento 2 API to retrieve completed orders filtered by date range  
    - Configuration: HTTP GET with query parameters for filtering orders by status and date  
    - Inputs: From Get 3 Month Dates  
    - Outputs: To Get Customer Order Value  
    - Potential Failures: API authentication issues, rate limiting, malformed queries  

#### 2.5 Customer Value Calculation

- **Overview:**  
  Processes orders to calculate cumulative order value per customer.

- **Nodes Involved:**  
  - Get Customer Order Value

- **Node Details:**  
  - **Get Customer Order Value**  
    - Type: Code (JavaScript)  
    - Role: Aggregates order totals by customer ID or email  
    - Configuration: Custom script iterating over orders, summing totals per customer  
    - Inputs: From Get Completed Orders  
    - Outputs: To If node (conditional)  
    - Edge Cases: Missing customer identifiers, zero or negative order values, empty order list  

#### 2.6 Conditional Filtering

- **Overview:**  
  Determines if a customer qualifies for receiving a voucher based on order value or other criteria.

- **Nodes Involved:**  
  - If

- **Node Details:**  
  - **If**  
    - Type: Conditional Logic  
    - Role: Checks if customer order value meets threshold (e.g., high-value customer)  
    - Configuration: Expression evaluating order value against predefined limits  
    - Inputs: From Get Customer Order Value  
    - Outputs:  
      - True branch → Create New Sales Rule  
      - False branch → Send Report  
    - Edge Cases: Expression evaluation errors, missing data  

#### 2.7 Sales Rule and Coupon Generation

- **Overview:**  
  Creates a new sales rule in Magento 2 and generates discount coupons for the qualifying customers.

- **Nodes Involved:**  
  - Create New Sales Rule  
  - Generate Coupon for Each Customer

- **Node Details:**  
  - **Create New Sales Rule**  
    - Type: HTTP Request  
    - Role: Calls Magento 2 API to create a sales rule (discount campaign)  
    - Configuration: HTTP POST with payload defining rule parameters (discount amount, conditions, validity)  
    - Inputs: From If (true branch)  
    - Outputs: To Generate Coupon for Each Customer  
    - On Error: Continue regular output (does not block workflow if rule creation fails)  
    - Potential Failures: API errors, validation failures, permission issues  
  - **Generate Coupon for Each Customer**  
    - Type: HTTP Request  
    - Role: Creates individual coupons tied to the sales rule for each customer  
    - Configuration: HTTP POST per customer with coupon details  
    - Inputs: From Create New Sales Rule  
    - Outputs: To Send Voucher  
    - On Error: Continue regular output  
    - Potential Failures: API limits, invalid coupon data  

#### 2.8 Voucher Sending

- **Overview:**  
  Sends personalized voucher emails to each eligible customer using Gmail integration.

- **Nodes Involved:**  
  - Send Voucher

- **Node Details:**  
  - **Send Voucher**  
    - Type: Gmail  
    - Role: Sends email with discount voucher details and branding assets  
    - Configuration: Uses Gmail OAuth2 credentials, email templates likely include logo and coupon code  
    - Inputs: From Generate Coupon for Each Customer  
    - Outputs: None (end node for this branch)  
    - Edge Cases: Email sending failures, invalid email addresses, quota limits  

#### 2.9 Reporting

- **Overview:**  
  Sends an email report summarizing the voucher generation and sending process.

- **Nodes Involved:**  
  - Send Report

- **Node Details:**  
  - **Send Report**  
    - Type: Gmail  
    - Role: Sends a report email, typically used if no customers qualify or as a summary notification  
    - Configuration: Gmail node with predefined recipients and formatted report content  
    - Inputs: From If (false branch)  
    - Outputs: None  
    - Edge Cases: Email failures, empty reports  

---

### 3. Summary Table

| Node Name               | Node Type       | Functional Role                                  | Input Node(s)              | Output Node(s)                        | Sticky Note                 |
|-------------------------|-----------------|-------------------------------------------------|----------------------------|-------------------------------------|-----------------------------|
| Cron                    | Cron Trigger    | Initiates workflow periodically                  | None                       | Get Media Path                      |                             |
| Get Media Path          | HTTP Request    | Retrieves media URL/path for branding assets     | Cron                       | SetOrGet Logo                      |                             |
| SetOrGet Logo           | Code            | Processes and caches logo data                    | Get Media Path              | Get 3 Month Dates                  |                             |
| Get 3 Month Dates       | Code            | Calculates date range for filtering orders       | SetOrGet Logo               | Get Completed Orders               |                             |
| Get Completed Orders    | HTTP Request    | Fetches completed orders from Magento 2          | Get 3 Month Dates           | Get Customer Order Value           |                             |
| Get Customer Order Value| Code            | Aggregates order values per customer              | Get Completed Orders        | If                               |                             |
| If                      | Conditional     | Filters customers based on order value            | Get Customer Order Value    | Create New Sales Rule (true branch) / Send Report (false branch) |                             |
| Create New Sales Rule   | HTTP Request    | Creates sales rule in Magento 2                    | If (true branch)            | Generate Coupon for Each Customer  |                             |
| Generate Coupon for Each Customer | HTTP Request | Generates coupons tied to sales rule for customers | Create New Sales Rule       | Send Voucher                     |                             |
| Send Voucher            | Gmail           | Sends discount voucher emails                      | Generate Coupon for Each Customer | None                          |                             |
| Send Report             | Gmail           | Sends summary report email                         | If (false branch)           | None                              |                             |
| Sticky Note             | Sticky Note     | Blank                                              | None                       | None                              |                             |
| Sticky Note1            | Sticky Note     | Blank                                              | None                       | None                              |                             |
| Sticky Note2            | Sticky Note     | Blank                                              | None                       | None                              |                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron node ("Cron")**  
   - Set scheduling parameters (e.g., daily at a specific time) to trigger workflow execution.

2. **Add HTTP Request node ("Get Media Path")**  
   - Method: GET  
   - URL: Magento 2 media endpoint or API to retrieve logo/media path  
   - Connect Cron → Get Media Path

3. **Add Code node ("SetOrGet Logo")**  
   - Paste JavaScript code to parse the media path response and cache/set the logo data for later use.  
   - Connect Get Media Path → SetOrGet Logo

4. **Add Code node ("Get 3 Month Dates")**  
   - Implement JavaScript to calculate date range: current date and date minus 3 months.  
   - Connect SetOrGet Logo → Get 3 Month Dates

5. **Add HTTP Request node ("Get Completed Orders")**  
   - Method: GET  
   - URL: Magento 2 API endpoint to fetch orders  
   - Query parameters: filter by status='complete', date range from "Get 3 Month Dates"  
   - Authentication: Magento 2 API credentials configured  
   - Connect Get 3 Month Dates → Get Completed Orders

6. **Add Code node ("Get Customer Order Value")**  
   - JavaScript to iterate over orders, aggregate order totals by customer ID or email.  
   - Connect Get Completed Orders → Get Customer Order Value

7. **Add If node ("If")**  
   - Condition: e.g., order value greater than a threshold (e.g., $500)  
   - Connect Get Customer Order Value → If

8. **Add HTTP Request node ("Create New Sales Rule")**  
   - Method: POST  
   - URL: Magento 2 API endpoint to create sales rules  
   - Payload: Define discount parameters, validity, etc.  
   - On Error: Continue workflow (do not stop if failure)  
   - Connect If (true branch) → Create New Sales Rule

9. **Add HTTP Request node ("Generate Coupon for Each Customer")**  
   - Method: POST  
   - URL: Magento 2 API endpoint to create coupons linked to the sales rule  
   - Payload: Coupon details per customer  
   - On Error: Continue workflow  
   - Connect Create New Sales Rule → Generate Coupon for Each Customer

10. **Add Gmail node ("Send Voucher")**  
    - Credential: Configure Gmail OAuth2 credentials  
    - Setup email template including customer details, coupon code, and logo asset  
    - Connect Generate Coupon for Each Customer → Send Voucher

11. **Add Gmail node ("Send Report")**  
    - Credential: Gmail OAuth2 credentials  
    - Compose summary report email for non-qualifying customers or overall status  
    - Connect If (false branch) → Send Report

12. **Verify all connections and test workflow execution**  
    - Ensure credentials for Magento 2 API and Gmail are properly configured.  
    - Validate error handling paths (continue on error where specified).  
    - Confirm email templates include dynamic data and branding assets.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow automates marketing outreach to high-value Magento 2 customers with personalized discount coupons. | Core use case: e-commerce customer retention and sales promotion                                   |
| Gmail nodes require OAuth2 credentials configured within n8n for sending emails securely.                 | See n8n documentation on Gmail OAuth2 credential setup                                             |
| Magento 2 API endpoints require authentication and correct permission scopes for managing sales rules and coupons. | Consult Magento 2 developer documentation for REST API usage                                       |
| Proper error handling (continue on error) is used in sales rule and coupon creation to avoid workflow halts. | Ensures partial failures do not block entire voucher dispatch                                      |
| Date calculations consider last 3 months to target recent high-value customers.                           | Adjust in "Get 3 Month Dates" code node as needed                                                  |
| Email templates should include branding assets fetched in initial nodes for consistent company identity. | Logo caching in "SetOrGet Logo" node supports this                                                  |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.