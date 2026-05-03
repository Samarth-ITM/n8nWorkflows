Send WooCommerce Discount Coupons to Customers via WhatsApp Using Rapiwa API

https://n8nworkflows.xyz/workflows/send-woocommerce-discount-coupons-to-customers-via-whatsapp-using-rapiwa-api-10140


# Send WooCommerce Discount Coupons to Customers via WhatsApp Using Rapiwa API

### 1. Workflow Overview

This workflow automates sending WooCommerce discount coupons to customers via WhatsApp using the Rapiwa API. It listens for new coupon creations in WooCommerce, formats coupon data, collects all customers, cleans and verifies their WhatsApp numbers, sends personalized promotional messages to verified numbers, and logs every attempt in Google Sheets for audit and tracking.

The workflow is logically grouped into these blocks:

- **1.1 Trigger & Coupon Formatting:** Listens for new WooCommerce coupons and formats coupon data into a structured array.
- **1.2 Customer Retrieval & Looping:** Retrieves all WooCommerce customers and iterates over them individually.
- **1.3 Phone Number Cleaning & Verification:** Cleans customer phone numbers to digits only and verifies WhatsApp registration via Rapiwa API.
- **1.4 Conditional Messaging & Logging:** Sends WhatsApp messages to verified numbers, logs successes and failures in Google Sheets, and adds delays between each processed customer for rate-limiting.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Coupon Formatting

- **Overview:**  
  Initiates the workflow upon WooCommerce coupon creation and transforms raw coupon data into a clean, usable format.

- **Nodes Involved:**  
  - WooCommerce Trigger  
  - Format WooCommerce Trigger Response Data

- **Node Details:**  

  - **WooCommerce Trigger**  
    - Type: Trigger node, listens to WooCommerce events  
    - Config: Listens to `coupon.created` event with WooCommerce API credentials  
    - Inputs: None (triggered externally)  
    - Outputs: Raw coupon data JSON  
    - Edge cases: Webhook misconfiguration, API credential failures, event missing coupon data  

  - **Format WooCommerce Trigger Response Data**  
    - Type: Code node  
    - Config: Processes input coupon(s), maps fields (code, discount, description, dates) into a structured array `data.coupons`  
    - Key expressions: Maps discount amount to include % symbol when discount type is percent  
    - Inputs: Coupon JSON from WooCommerce Trigger  
    - Outputs: Single item containing structured coupon data array  
    - Edge cases: Missing coupon fields, unexpected data types, empty input  

#### 1.2 Customer Retrieval & Looping

- **Overview:**  
  Fetches all WooCommerce customers and iterates over them one by one to prepare for personalized messaging.

- **Nodes Involved:**  
  - Get many customers  
  - Loop Over Items

- **Node Details:**  

  - **Get many customers**  
    - Type: WooCommerce resource node (customer)  
    - Config: Retrieves all customer records, no filters, returns all  
    - Inputs: Output from coupon formatting node  
    - Outputs: Array of customer objects  
    - Edge cases: API limits, empty customer list, credential failures  

  - **Loop Over Items**  
    - Type: SplitInBatches node  
    - Config: Splits customer array into single-item batches (default batch size 1)  
    - Inputs: Customer array  
    - Outputs: Single customer per iteration  
    - Edge cases: Empty input array, batch processing errors  

#### 1.3 Phone Number Cleaning & Verification

- **Overview:**  
  Standardizes customer phone numbers to digit-only format and verifies WhatsApp registration status using Rapiwa API.

- **Nodes Involved:**  
  - Clean WhatsApp Number  
  - Check valid whatsapp number Using Rapiwa  
  - If

- **Node Details:**  

  - **Clean WhatsApp Number**  
    - Type: Code node  
    - Config: Removes all non-digit characters from the customer's phone number field (`number`), overwriting with cleaned string  
    - Key expressions: Uses regex `replace(/\D/g, "")`  
    - Inputs: Single customer JSON item  
    - Outputs: Customer JSON with cleaned `number` field  
    - Edge cases: Missing phone number field, non-string input, empty phone number after cleaning  

  - **Check valid whatsapp number Using Rapiwa**  
    - Type: HTTP Request node  
    - Config: POST request to `https://app.rapiwa.com/api/verify-whatsapp` with `{number: billing.phone}` in body  
    - Authentication: HTTP Bearer token credential (`Rapiwa Bearer YOUR_TOKEN_HERE`)  
    - Inputs: Customer JSON with phone number  
    - Outputs: JSON response indicating if number is a registered WhatsApp user  
    - Edge cases: Network errors, invalid token, API rate limits, unexpected response structure  

  - **If**  
    - Type: Conditional node  
    - Config: Checks if the Rapiwa verification response field `data.exists` is boolean true (exact match)  
    - Inputs: HTTP response from verification node  
    - Outputs: Two branches — true (number verified) and false (number unverified)  
    - Edge cases: Type mismatch (string vs boolean), missing field, expression evaluation errors  

#### 1.4 Conditional Messaging & Logging

- **Overview:**  
  Sends a personalized WhatsApp promotional message to verified numbers, logs the outcome to Google Sheets, and waits before proceeding to the next customer.

- **Nodes Involved:**  
  - Send Message Using Rapiwa  
  - Save State of Rows in Verified & Sent  
  - Save State of Rows in Unerified & Not sent  
  - Wait

- **Node Details:**  

  - **Send Message Using Rapiwa**  
    - Type: HTTP Request node  
    - Config: POST to `https://app.rapiwa.com/api/send-message` with body including:  
      - `number`: verified WhatsApp number  
      - `message_type`: text  
      - `message`: templated with customer first/last names and coupon information from formatted coupon data  
    - Authentication: HTTP Bearer token (`Rapiwa Bearer YOUR_TOKEN_HERE`)  
    - Inputs: Verified WhatsApp customer info and coupon data  
    - Outputs: Response from Rapiwa confirming message sent  
    - Edge cases: Network issues, invalid token, message formatting errors, API throttling  

  - **Save State of Rows in Verified & Sent**  
    - Type: Google Sheets node  
    - Config: Appends a row to a Google Sheet recording: customer name, email, number, coupon details, status "sent", validity "verified"  
    - Credentials: Google Sheets OAuth2 credentials  
    - Inputs: Data from message sending node and customer info  
    - Outputs: Confirmation of row append  
    - Edge cases: API quota limits, credential expiration, sheet not found, column mismatch, typo in field name `aouponAmount` (should be `couponAmount`)  

  - **Save State of Rows in Unerified & Not sent**  
    - Type: Google Sheets node  
    - Config: Appends a row to a separate Google Sheet recording similar fields but with status "not sent" and validity "unverified"  
    - Credentials: Google Sheets OAuth2 credentials  
    - Inputs: Customer data from unverified branch  
    - Outputs: Confirmation of row append  
    - Edge cases: Same as above  

  - **Wait**  
    - Type: Wait node  
    - Config: Default delay (no parameters specified) to throttle requests, preventing flooding the API or customers  
    - Inputs: After logging nodes  
    - Outputs: Triggers next iteration of Loop Over Items  
    - Edge cases: Workflow pauses unexpectedly if delay too long or misconfigured  

---

### 3. Summary Table

| Node Name                              | Node Type                | Functional Role                                    | Input Node(s)                      | Output Node(s)                            | Sticky Note                                                                                                                                                       |
|--------------------------------------|--------------------------|---------------------------------------------------|----------------------------------|------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| WooCommerce Trigger                   | WooCommerce Trigger      | Starts workflow on coupon creation event          | None                             | Format WooCommerce Trigger Response Data | ## WooCommerce Trigger - Purpose: Starts the workflow when a new coupon is created in WooCommerce.                                                              |
| Format WooCommerce Trigger Response Data | Code                    | Formats raw coupon data into structured array     | WooCommerce Trigger              | Get many customers                        | ## Format Coupon Data - Purpose: Cleans and structures coupon data into a consistent format.                                                                     |
| Get many customers                   | WooCommerce              | Retrieves all customers from WooCommerce          | Format WooCommerce Trigger Response Data | Loop Over Items                      | ## Get All Customers - Purpose: Fetches all WooCommerce customer records.                                                                                        |
| Loop Over Items                     | SplitInBatches           | Iterates over each customer individually           | Get many customers               | Clean WhatsApp Number                     |                                                                                                                                                                  |
| Clean WhatsApp Number                | Code                     | Cleans phone numbers to digits only                | Loop Over Items                  | Check valid whatsapp number Using Rapiwa | ## Clean WhatsApp Numbers - Purpose: Standardizes phone numbers for WhatsApp validation. Removes non-digit characters.                                           |
| Check valid whatsapp number Using Rapiwa | HTTP Request           | Verifies WhatsApp registration of phone number    | Clean WhatsApp Number            | If                                       | ## Verify WhatsApp Number with Rapiwa - Purpose: Checks if a number is registered on WhatsApp.                                                                   |
| If                                 | If                       | Branches workflow based on WhatsApp verification  | Check valid whatsapp number Using Rapiwa | Send Message Using Rapiwa (true branch), Save State of Rows in Unerified & Not sent (false branch) | ## If (Conditional) - Purpose: Branch the flow depending on Rapiwa verification.                                                                                 |
| Send Message Using Rapiwa            | HTTP Request             | Sends personalized WhatsApp promotional message   | If (true branch)                 | Save State of Rows in Verified & Sent    | ## Send WhatsApp Message (via Rapiwa) - Purpose: Sends a personalized WhatsApp coupon message using Rapiwa.                                                      |
| Save State of Rows in Verified & Sent | Google Sheets            | Logs successful message sends                       | Send Message Using Rapiwa        | Wait                                     | ## Log Verified & Sent (Google Sheets) - Purpose: Saves message delivery info for verified users.                                                                |
| Save State of Rows in Unerified & Not sent | Google Sheets            | Logs customers whose numbers are unverified        | If (false branch)                | Wait                                     | ## Log Unverified & Not Sent (Google Sheets) - Purpose: Saves info for users not registered on WhatsApp.                                                        |
| Wait                               | Wait                     | Adds delay before next customer processing         | Save State of Rows in Verified & Sent, Save State of Rows in Unerified & Not sent | Loop Over Items          | ## Wait Before Next - Purpose: Adds delay between customer processing.                                                                                           |
| Sticky Note                         | Sticky Note              | Documentation and overview content                  | None                           | None                                     | # Send WooCommerce Discount Coupons to Customers via WhatsApp Using Rapiwa API ... (Full detailed overview and instructions)                                     |
| Sticky Note1                        | Sticky Note              | Explains WooCommerce trigger and coupon formatting | None                           | None                                     | ## WooCommerce Trigger and Format Coupon Data explanations                                                                                                       |
| Sticky Note2                        | Sticky Note              | Explains Get All Customers node                     | None                           | None                                     | ## Get All Customers - Purpose: Fetches all WooCommerce customer records.                                                                                        |
| Sticky Note3                        | Sticky Note              | Explains cleaning and WhatsApp verification nodes  | None                           | None                                     | ## Clean WhatsApp Numbers and Verify WhatsApp Number with Rapiwa explanations                                                                                     |
| Sticky Note5                        | Sticky Note              | Explains If node, sending message, logging, and wait | None                           | None                                     | ## If (Conditional), Send WhatsApp Message, Log Verified & Sent, Log Unverified & Not Sent, Wait node explanations                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WooCommerce Trigger Node:**  
   - Type: WooCommerce Trigger  
   - Event: `coupon.created`  
   - Credential: WooCommerce API with read access to coupons  
   - Purpose: Trigger workflow when a new coupon is created  

2. **Add Code Node to Format Coupon Data:**  
   - Name: `Format WooCommerce Trigger Response Data`  
   - Code:
     ```javascript
     const coupons = items[0].json;
     const couponsArray = Array.isArray(coupons) ? coupons : [coupons];
     const formattedCoupons = couponsArray.map(coupon => {
       const amountWithSymbol = coupon.discount_type === "percent" ? coupon.amount + "%" : coupon.amount;
       return {
         couponCode: coupon.code,
         discountAmount: amountWithSymbol,
         discountType: coupon.discount_type,
         description: coupon.description,
         dateCreate: coupon.date_modified,
         dateExpires: coupon.date_expires,
         freeShipping: coupon.free_shipping,
       };
     });
     return [{ json: { data: { coupons: formattedCoupons } } }];
     ```
   - Connect WooCommerce Trigger output to this node input  

3. **Add WooCommerce Node to Get All Customers:**  
   - Resource: Customer  
   - Operation: Get All (return all customers)  
   - Credential: WooCommerce API  
   - Connect output of coupon formatting node to this node  

4. **Add SplitInBatches Node (`Loop Over Items`):**  
   - Default batch size (1)  
   - Connect output of Get many customers node to this node  

5. **Add Code Node to Clean WhatsApp Numbers:**  
   - Name: `Clean WhatsApp Number`  
   - Code:
     ```javascript
     const items = $input.all();
     const updatedItems = items.map((item) => {
       let rawNumber = item?.json["number"];
       rawNumber = rawNumber ? String(rawNumber) : "";
       const cleanedNumber = rawNumber.replace(/\D/g, "");
       item.json["number"] = cleanedNumber;
       return item;
     });
     return updatedItems;
     ```
   - Connect Loop Over Items output to this node  

6. **Add HTTP Request Node to Verify WhatsApp Number:**  
   - Name: `Check valid whatsapp number Using Rapiwa`  
   - Method: POST  
   - URL: `https://app.rapiwa.com/api/verify-whatsapp`  
   - Body: JSON `{ "number": "={{ $json.billing.phone }}" }`  
   - Authentication: HTTP Bearer with Rapiwa token credential  
   - Connect Clean WhatsApp Number output to this node  

7. **Add If Node for Verification Branching:**  
   - Condition: Check if `{{$json.data.exists}}` is boolean `true`  
   - Connect HTTP Request node output to If node  

8. **Add HTTP Request Node to Send WhatsApp Message:**  
   - Name: `Send Message Using Rapiwa`  
   - Method: POST  
   - URL: `https://app.rapiwa.com/api/send-message`  
   - Body parameters:
     - number: `={{ $json.data.number }}`
     - message_type: `text`
     - message: Template message using customer first name, last name, and coupon details from `Format WooCommerce Trigger Response Data` node  
   - Authentication: HTTP Bearer with Rapiwa token  
   - Connect If node `true` branch to this node  

9. **Add Google Sheets Node to Log Verified & Sent:**  
   - Operation: Append  
   - Document ID: Your Google Sheet ID  
   - Sheet Name: Sheet with columns like name, email, number, status=sent, validity=verified, couponCode, couponType, createDate, expireDate, couponTitle, couponAmount (fix typo)  
   - Map fields from `Clean WhatsApp Number` and coupon formatting nodes  
   - Credential: Google Sheets OAuth2  
   - Connect Send Message node output to this node  

10. **Add Google Sheets Node to Log Unverified & Not Sent:**  
    - Same Google Sheet document and sheet or different as per your setup  
    - Status: not sent, validity: unverified  
    - Map fields similarly  
    - Connect If node `false` branch to this node  

11. **Add Wait Node:**  
    - Default wait time (configure if needed)  
    - Connect both Google Sheets nodes outputs to the Wait node  

12. **Connect Wait Node output back to Loop Over Items input:**  
    - This creates a controlled loop over each customer with delay  

13. **Set up Credentials:**  
    - WooCommerce API (read access to coupons and customers)  
    - Rapiwa API Bearer Token (create HTTP Bearer credential in n8n)  
    - Google Sheets OAuth2 with access to target sheets  

14. **Validate Field Mappings and Expressions:**  
    - Confirm phone number fields match between cleaning and verification nodes  
    - Adjust coupon field names if your WooCommerce differs  
    - Fix typo `aouponAmount` to `couponAmount` in Google Sheets mappings and sheet headers  

15. **Test Workflow:**  
    - Trigger coupon creation event in WooCommerce  
    - Verify messages sent to valid WhatsApp numbers  
    - Confirm logging accuracy  
    - Adjust delays or error handling as needed  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                             | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The Google Sheets mapping contains a field named `aouponAmount` (typo). Rename the field or the header to `couponAmount` for correctness.                                                                                              | Workflow Google Sheets node mapping                                                                |
| Test the `If` node logic carefully; Rapiwa API might return boolean or string for verification status, normalize before comparison to avoid false negatives.                                                                             | Conditional node best practice                                                                      |
| Do not hard-code API tokens or credentials inside nodes; use n8n credentials management for security and maintainability.                                                                                                             | Security best practice                                                                              |
| Ensure phone number fields used for cleaning and verification are consistent, usually `billing.phone`.                                                                                                                                | Data consistency                                                                                   |
| Rate limiting: Use the Wait node to avoid flooding Rapiwa API or WhatsApp recipients, adjust delay as needed.                                                                                                                         | To prevent API or service throttling                                                               |
| Useful links: Rapiwa dashboard [https://app.rapiwa.com](https://app.rapiwa.com/login), documentation [https://docs.rapiwa.com](https://docs.rapiwa.com/), official site [https://rapiwa.com](https://rapiwa.com/)                    | Rapiwa API resources                                                                               |
| Support channels: WhatsApp chat, Discord, Facebook group, SpaGreen website, and developer portfolio links provided in sticky notes for user help.                                                                                     | Contact and community support                                                                      |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.