Facebook/Meta Conversion API for eCommerce Leads/Orders

https://n8nworkflows.xyz/workflows/facebook-meta-conversion-api-for-ecommerce-leads-orders-5136


# Facebook/Meta Conversion API for eCommerce Leads/Orders

### 1. Workflow Overview

This workflow is designed to facilitate integration with Facebook/Meta Conversion API for eCommerce leads and orders. Its main purpose is to receive lead or order data (e.g., customer details and transaction info), normalize and hash sensitive user data for privacy compliance, then send a properly formatted event to Meta’s Conversion API to track conversions such as purchases.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Accept incoming data via webhook or generate test data manually for development and testing purposes.
- **1.2 Data Formatting and Enrichment**: Format raw input data into a structured event payload compatible with Meta’s API requirements.
- **1.3 Data Normalization**: Clean and normalize user data fields, such as phone numbers and timestamps.
- **1.4 Data Encryption**: Hash personal user data fields using SHA256 to meet Meta’s user data hashing requirements.
- **1.5 Event Transmission to Meta**: Compose and send the event data to Facebook/Meta Conversion API, optionally including a test event code during validation.
  
Additional sticky notes provide guidance, instructions, and contact info for support.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block captures incoming data via a webhook or allows manual triggering to generate test data for development and testing.

- **Nodes Involved:**  
  - Webhook  
  - Execute workflow (Manual Trigger)  
  - Test data (Code)  

- **Node Details:**

  - **Webhook**  
    - Type: Webhook  
    - Role: Entry point for live data, listens for POST or GET requests on a unique path.  
    - Configuration: Path set as a UUID, accepts both POST and GET HTTP methods.  
    - Inputs: External calls from clients or systems sending lead/order data.  
    - Outputs: Passes raw request payload downstream.  
    - Edge Cases: Possible failure if webhook path is incorrect or not publicly accessible, or if malformed data is received.

  - **Execute workflow**  
    - Type: Manual Trigger  
    - Role: Allows manual initiation of test data generation.  
    - Configuration: Default manual trigger with no parameters.  
    - Outputs: Triggers the "Test data" node to generate simulated input.  

  - **Test data**  
    - Type: Code (JavaScript)  
    - Role: Generates a sample payload mimicking expected input data structure for testing.  
    - Configuration: Returns a JSON object with fields such as first_name, last_name, phone, IP, user_agent, fbp, fbc, created_at, amount, and currency.  
    - Edge Cases: Static data; does not cover dynamic input variations. Should be updated for testing different scenarios.

---

#### 1.2 Data Formatting and Enrichment

- **Overview:**  
  This block restructures incoming input data into the specific JSON format required by the Facebook Conversion API, including setting event name, pixel ID, user data fields, event time, and custom data such as transaction value and currency.

- **Nodes Involved:**  
  - Format data (Code)  

- **Node Details:**

  - **Format data**  
    - Type: Code (JavaScript)  
    - Role: Maps and formats raw input (from webhook or test data) into a structured event object.  
    - Configuration:  
      - Hardcoded event_name: "Purchase"  
      - Pixel ID: Static string '1714041194725552' (replace with your pixel ID)  
      - Extracts user data fields (first_name, last_name, phone, ip, user_agent, fbp, fbc) from input  
      - Converts event_time to ISO string from created_at  
      - Sets custom_data with amount and currency  
      - Sets action_source to "website" and event_source_url to a placeholder URL  
    - Inputs: Raw data from webhook or test data node  
    - Outputs: Single JSON object formatted for downstream normalization  
    - Edge Cases: Assumes all required fields exist and are valid; missing fields could cause errors downstream.

---

#### 1.3 Data Normalization

- **Overview:**  
  This block cleans and normalizes data fields, such as stripping phone numbers of non-digit characters and leading zeros, converting event_time strings to UNIX timestamps, and ensuring first and last names are correctly assigned.

- **Nodes Involved:**  
  - Normalize data (Set)  

- **Node Details:**

  - **Normalize data**  
    - Type: Set node  
    - Role: Performs data normalization on key user data and event-related fields.  
    - Configuration:  
      - Removes all non-digit characters from phone numbers and strips leading zeros.  
      - Converts event_time string to UNIX timestamp (seconds since epoch).  
      - Passes first_name and last_name unchanged.  
      - Excludes the field user_data.name to avoid unwanted data.  
      - Includes all other fields unchanged.  
    - Inputs: Formatted data from the previous node  
    - Outputs: Normalized data for encryption  
    - Edge Cases: If phone number is missing or improperly formatted, normalization may produce empty or invalid results.

- **Sticky Note1:** Marks this block as "Data normalization".

---

#### 1.4 Data Encryption

- **Overview:**  
  This block hashes sensitive user data fields (first name, last name, phone) using SHA256 hashing as per Meta’s Conversion API requirements for user data.

- **Nodes Involved:**  
  - Encrypt data (Set)  

- **Node Details:**

  - **Encrypt data**  
    - Type: Set node  
    - Role: Hashes user data fields using SHA256 to preserve user privacy.  
    - Configuration:  
      - Applies SHA256 hash to user_data.first_name, user_data.last_name, and user_data.phone.  
      - Leaves other fields unchanged.  
    - Inputs: Normalized data  
    - Outputs: Hashed/encrypted data for sending to Meta  
    - Edge Cases: Hashing failure unlikely but malformed input could produce unexpected hash values.

- **Sticky Note2:** Marks this block as "Data encryption".

---

#### 1.5 Event Transmission to Meta

- **Overview:**  
  This block sends the finalized event payload to Facebook/Meta Conversion API’s events edge with proper authentication and parameters. It includes a test event code during initial testing to verify integration.

- **Nodes Involved:**  
  - Send event to Meta (Facebook Graph API)  

- **Node Details:**

  - **Send event to Meta**  
    - Type: Facebook Graph API node  
    - Role: Sends the conversion event to Meta's Conversion API endpoint with authentication.  
    - Configuration:  
      - API Node: Uses pixel_id from "Format data" node as the Facebook node parameter.  
      - Edge: "events"  
      - HTTP Method: POST  
      - API Version: v20.0  
      - Query Parameters JSON: Constructs JSON payload with event_name, event_time, event_source_url, action_source, user_data (hashed), custom_data (value and currency), and test_event_code "TEST1337".  
      - Credentials: Uses OAuth2 credentials configured under "Meta conversion api (WIJDANPROMO)".  
    - Inputs: Encrypted data from "Encrypt data" node  
    - Outputs: Response from Facebook API (success or error)  
    - Edge Cases:  
      - Authentication failures (expired or invalid OAuth token)  
      - API rate limits or downtime  
      - Incorrect pixel ID or malformed payload causing errors  
    - Notes: Test event code should be removed in production to log official events.

- **Sticky Note3:** Provides usage instructions and link to Facebook’s event testing documentation:  
  [Read more](https://business.facebook.com/business/help/2040882565969969)

---

### 3. Summary Table

| Node Name          | Node Type             | Functional Role                    | Input Node(s)          | Output Node(s)         | Sticky Note                                                                                      |
|--------------------|-----------------------|----------------------------------|-----------------------|------------------------|------------------------------------------------------------------------------------------------|
| Webhook            | Webhook               | Receives live input data          | (external)            | Format data            | ## Production                                                                                   |
| Execute workflow   | Manual Trigger         | Starts test data generation       | -                     | Test data              | ## Test Workflow Click this button to test the workflow with fake data                         |
| Test data          | Code                  | Generates test input data          | Execute workflow      | Format data            | ## Test data You can chenge test data here                                                     |
| Format data        | Code                  | Formats input to Meta event JSON  | Webhook, Test data     | Normalize data         | ## Collect data In this node you link between your webhook data with this node parameters, also you can add more data. |
| Normalize data     | Set                   | Cleans and normalizes data fields | Format data            | Encrypt data           | ## Data normalization                                                                          |
| Encrypt data       | Set                   | Hashes sensitive user data        | Normalize data         | Send event to Meta     | ## Data encryption                                                                             |
| Send event to Meta | Facebook Graph API     | Posts event to Meta Conversion API| Encrypt data           | -                      | ## Send event You can use a test code from Facebook Events Manager to verify your integration. Add your test code to the test_code parameter inside the JSON payload. Once everything is confirmed to be working, remove the test_event_code so that events are officially logged. [**Read more**](https://business.facebook.com/business/help/2040882565969969) |
| Sticky Note1       | Sticky Note            | Marks Data normalization block    | -                     | -                      | ## Data normalization                                                                          |
| Sticky Note2       | Sticky Note            | Marks Data encryption block       | -                     | -                      | ## Data encryption                                                                            |
| Sticky Note3       | Sticky Note            | Instructions for sending events   | -                     | -                      | ## Send event You can use a test code from Facebook Events Manager to verify your integration. Add your test code to the test_code parameter inside the JSON payload. Once everything is confirmed to be working, remove the test_event_code so that events are officially logged. [**Read more**](https://business.facebook.com/business/help/2040882565969969) |
| Sticky Note4       | Sticky Note            | Explains data collection node     | -                     | -                      | ## Collect data In this node you link between your webhook data with this node parameters, also you can add more data. |
| Sticky Note5       | Sticky Note            | Marks Test workflow initiation    | -                     | -                      | ## Test Workflow Click this button to test the workflow with fake data                         |
| Sticky Note6       | Sticky Note            | Indicates where to change config  | -                     | -                      | ## Change here                                                                               |
| Sticky Note7       | Sticky Note            | Support contact info               | -                     | -                      | ## Do you need more help or have any suggestions? Contact me at mediaplus.ma@gmail.com        |
| Sticky Note8       | Sticky Note            | Marks test area                   | -                     | -                      | ## Just to test                                                                             |
| Sticky Note9       | Sticky Note            | Marks production webhook          | -                     | -                      | ## Production                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node**  
   - Set HTTP Method to accept POST and GET.  
   - Set Path to a unique identifier (e.g., UUID or custom string).  
   - This node serves as the live entry point for incoming lead/order data.

2. **Create a Manual Trigger Node ("Execute workflow")**  
   - This node will allow manual initiation of test data for development.

3. **Create a Code Node ("Test data")**  
   - JavaScript code to output a JSON object simulating typical input data fields: first_name, last_name, phone, IP, user_agent, fbp, fbc, created_at, amount, currency.  
   - Connect Manual Trigger output to this node.

4. **Create a Code Node ("Format data")**  
   - Write JavaScript code to map input data (from webhook or test data) into the Facebook event payload format:  
     - event_name: "Purchase"  
     - pixel_id: Your actual Facebook Pixel ID (replace '1714041194725552')  
     - user_data: first_name, last_name, phone, ip, user_agent, fbp, fbc  
     - event_time: use created_at timestamp  
     - custom_data: value (amount), currency  
     - action_source: "website"  
     - event_source_url: a relevant URL (e.g., confirmation page)  
   - Connect Webhook and Test data nodes to this node’s input.

5. **Create a Set Node ("Normalize data")**  
   - Configure to modify user_data.phone by removing all non-digit characters and leading zeros.  
   - Convert event_time string into a UNIX timestamp (seconds since epoch).  
   - Pass first_name and last_name unchanged.  
   - Exclude any unnecessary fields such as user_data.name.  
   - Include all other fields unchanged.  
   - Connect "Format data" output to this node.

6. **Create a Set Node ("Encrypt data")**  
   - Configure to hash user_data.first_name, user_data.last_name, and user_data.phone using SHA256.  
   - Leave all other fields unchanged.  
   - Connect "Normalize data" output to this node.

7. **Create a Facebook Graph API Node ("Send event to Meta")**  
   - Set API Version to v20.0.  
   - Set Node parameter dynamically to pixel_id from "Format data".  
   - Set Edge to "events".  
   - HTTP Method: POST.  
   - In query parameters, build JSON payload referencing "Encrypt data" fields:  
     - event_name, event_time, event_source_url, action_source  
     - user_data fields (hashed) including email if available, phone, first_name, last_name, fbc, fbp, ip, user_agent  
     - custom_data with value and currency  
     - Include "test_event_code" during testing; remove before production.  
   - Credentials: Configure with Facebook OAuth2 credentials authorized for Conversion API access.  
   - Connect "Encrypt data" output to this node.

8. **Add Sticky Notes** as per your preference to document:  
   - Data normalization  
   - Data encryption  
   - Sending event instructions with link to Facebook docs  
   - Test data usage and workflow testing instructions  
   - Production webhook note  
   - Contact information for further help  

---

### 5. General Notes & Resources

| Note Content                                                                                                            | Context or Link                                                                                              |
|-------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| You can use a test event code from Facebook Events Manager to verify your integration. Add your test code to the test_code parameter inside the JSON payload. Once confirmed working, remove it for production. | https://business.facebook.com/business/help/2040882565969969                                                |
| Contact me at mediaplus.ma@gmail.com for help or suggestions.                                                           | Support contact                                                                                            |
| The pixel_id in the "Format data" node must be replaced with your actual Facebook Pixel ID.                             | Configuration instruction                                                                                   |
| Ensure Facebook OAuth2 credentials are set up correctly for Meta Conversion API access in n8n credentials.              | Credential setup                                                                                            |

---

**Disclaimer:** The provided content originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.