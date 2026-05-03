Real Estate Daily Deals Automation with Zillow API, Google Sheets and Gmail

https://n8nworkflows.xyz/workflows/real-estate-daily-deals-automation-with-zillow-api--google-sheets-and-gmail-3030


# Real Estate Daily Deals Automation with Zillow API, Google Sheets and Gmail

### 1. Workflow Overview

This workflow automates the daily retrieval and processing of real estate deals from Zillow, calculates investment metrics, updates a Google Sheet with the results, and emails a summary each morning at 9 AM. It is designed for real estate investors who want to monitor market opportunities based on strict search criteria.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow every day at 9 AM.
- **1.2 Search Parameter Setup:** Defines user-specific real estate search criteria.
- **1.3 Zillow API Search:** Queries Zillow API for listings matching the criteria.
- **1.4 Data Splitting:** Splits the Zillow search results into individual listings.
- **1.5 Rent Zestimate Retrieval:** Fetches rent estimates for each property.
- **1.6 Investment Metrics Calculation:** Calculates key investment metrics per listing.
- **1.7 Google Sheets Update:** Updates a Google Sheet with the processed data.
- **1.8 Email Notification:** Sends a summary email with the daily deals.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  This block triggers the entire workflow automatically every day at 9 AM.

- **Nodes Involved:**  
  - 9am Trigger

- **Node Details:**  
  - **9am Trigger**  
    - Type: Schedule Trigger  
    - Configuration: Set to trigger daily at 9:00 AM (default time zone depends on n8n instance settings).  
    - Inputs: None (start node)  
    - Outputs: Connects to "Set Parameters" node  
    - Edge Cases: Workflow will not run if n8n instance is down or if the schedule is misconfigured.  
    - Version: 1.2  

#### 2.2 Search Parameter Setup

- **Overview:**  
  Defines the real estate search criteria such as location, minimum bedrooms, price range, and property type.

- **Nodes Involved:**  
  - Set Parameters

- **Node Details:**  
  - **Set Parameters**  
    - Type: Set  
    - Configuration: Contains key-value pairs representing search criteria, e.g., location = "Austin, TX", min_bed = 2, min_bath = 2, min_price = 250000, max_price = 400000, propertyType = "Single Family" (boolean true).  
    - Inputs: From "9am Trigger"  
    - Outputs: To "Zillow Search"  
    - Expressions: May use static values or expressions to dynamically set parameters.  
    - Edge Cases: Missing or invalid parameters may cause API request failures or empty results.  
    - Version: 3.4  

#### 2.3 Zillow API Search

- **Overview:**  
  Sends an HTTP request to the Zillow API via RapidAPI to retrieve property listings matching the search criteria.

- **Nodes Involved:**  
  - Zillow Search

- **Node Details:**  
  - **Zillow Search**  
    - Type: HTTP Request  
    - Configuration:  
      - Method: GET or POST depending on Zillow API spec  
      - URL: Zillow API endpoint via RapidAPI  
      - Authentication: Uses RapidAPI credentials (API key) configured in n8n credentials  
      - Query Parameters: Populated from "Set Parameters" node outputs  
    - Inputs: From "Set Parameters"  
    - Outputs: To "Split Out"  
    - Edge Cases:  
      - API rate limits exceeded  
      - Invalid API credentials  
      - Network timeouts  
      - Unexpected API response format  
    - Version: 4.2  

#### 2.4 Data Splitting

- **Overview:**  
  Splits the array of property listings returned by Zillow into individual items for further processing.

- **Nodes Involved:**  
  - Split Out

- **Node Details:**  
  - **Split Out**  
    - Type: Split Out  
    - Configuration: Splits input array into separate outputs, one per listing  
    - Inputs: From "Zillow Search"  
    - Outputs: To "RentZestimate"  
    - Edge Cases: Empty or malformed input array will result in no output items.  
    - Version: 1  

#### 2.5 Rent Zestimate Retrieval

- **Overview:**  
  For each individual property, fetches the rent Zestimate (estimated rent) from Zillow API.

- **Nodes Involved:**  
  - RentZestimate

- **Node Details:**  
  - **RentZestimate**  
    - Type: HTTP Request  
    - Configuration:  
      - Method: GET or POST to Zillow API endpoint for rent estimates  
      - Authentication: Uses RapidAPI credentials  
      - Parameters: Uses property-specific identifiers from the split data  
    - Inputs: From "Split Out"  
    - Outputs: To "Investment Calculator"  
    - Edge Cases:  
      - Missing property identifiers  
      - API failures or rate limits  
      - Null or zero rent estimates  
    - Version: 4.2  

#### 2.6 Investment Metrics Calculation

- **Overview:**  
  Calculates investment metrics such as Down Payment, Cash on Cash ROI, Monthly Cash Flow, Monthly Maintenance, and Vacancy Loss based on property price and rent Zestimate.

- **Nodes Involved:**  
  - Investment Calculator

- **Node Details:**  
  - **Investment Calculator**  
    - Type: Code (JavaScript)  
    - Configuration: Contains custom JavaScript code to compute financial metrics using input data fields (price, rent Zestimate, etc.)  
    - Inputs: From "RentZestimate"  
    - Outputs: To "Google Sheets"  
    - Key Variables: Down Payment %, Interest Rate, Vacancy Rate, Maintenance %, etc. (likely hardcoded or set via parameters)  
    - Edge Cases:  
      - Missing or invalid numeric inputs causing calculation errors  
      - Division by zero or NaN results  
    - Version: 2  

#### 2.7 Google Sheets Update

- **Overview:**  
  Updates a Google Sheet with the processed property data and investment metrics for tracking.

- **Nodes Involved:**  
  - Google Sheets

- **Node Details:**  
  - **Google Sheets**  
    - Type: Google Sheets  
    - Configuration:  
      - Operation: Append or Update Rows  
      - Spreadsheet ID and Sheet Name configured to target the user’s tracking sheet  
      - Column mapping: Address, Price, Rent Zestimate, Cash on Cash ROI, Monthly Cash Flow, Down Payment, etc.  
      - Authentication: Uses Google OAuth2 credentials with Sheets API enabled  
    - Inputs: From "Investment Calculator"  
    - Outputs: To "Gmail"  
    - Edge Cases:  
      - Incorrect spreadsheet or sheet name causing failures  
      - Insufficient permissions or expired OAuth tokens  
      - API quota limits  
    - Version: 4.5  

#### 2.8 Email Notification

- **Overview:**  
  Sends a summary email with the daily real estate deals to the user.

- **Nodes Involved:**  
  - Gmail

- **Node Details:**  
  - **Gmail**  
    - Type: Gmail  
    - Configuration:  
      - Operation: Send Email  
      - Recipient: User’s email address (configured in node parameters)  
      - Subject and Body: Summary of deals, possibly formatted HTML or plain text  
      - Authentication: Uses Gmail OAuth2 credentials  
    - Inputs: From "Google Sheets"  
    - Outputs: None (end node)  
    - Edge Cases:  
      - Authentication errors (expired tokens)  
      - Email quota limits  
      - Invalid recipient address  
    - Version: 2.1  

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                | Input Node(s)       | Output Node(s)     | Sticky Note                                                                                  |
|---------------------|---------------------|-------------------------------|---------------------|--------------------|----------------------------------------------------------------------------------------------|
| 9am Trigger         | Schedule Trigger    | Initiates workflow daily at 9AM | None                | Set Parameters     |                                                                                              |
| Set Parameters      | Set                 | Defines real estate search criteria | 9am Trigger         | Zillow Search      |                                                                                              |
| Zillow Search       | HTTP Request        | Queries Zillow API for listings | Set Parameters      | Split Out          |                                                                                              |
| Split Out           | Split Out           | Splits listings array into items | Zillow Search       | RentZestimate      |                                                                                              |
| RentZestimate       | HTTP Request        | Retrieves rent estimates per property | Split Out           | Investment Calculator |                                                                                              |
| Investment Calculator | Code                | Calculates investment metrics   | RentZestimate       | Google Sheets      |                                                                                              |
| Google Sheets       | Google Sheets       | Updates tracking spreadsheet    | Investment Calculator | Gmail              |                                                                                              |
| Gmail               | Gmail               | Sends daily deals summary email | Google Sheets       | None               |                                                                                              |
| Sticky Note         | Sticky Note         | (Empty)                       | None                | None               |                                                                                              |
| Sticky Note1        | Sticky Note         | (Empty)                       | None                | None               |                                                                                              |
| Sticky Note2        | Sticky Note         | (Empty)                       | None                | None               |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it "Real Estate Daily Deals".

2. **Add a Schedule Trigger node** named "9am Trigger":  
   - Set to trigger daily at 9:00 AM.

3. **Add a Set node** named "Set Parameters":  
   - Define search criteria as key-value pairs, e.g.:  
     - location: "Austin, TX"  
     - min_bed: 2  
     - min_bath: 2  
     - min_price: 250000  
     - max_price: 400000  
     - propertyType: "Single Family" (boolean true)  
   - Connect "9am Trigger" output to this node.

4. **Add an HTTP Request node** named "Zillow Search":  
   - Method: GET or POST as per Zillow API spec.  
   - URL: Zillow API endpoint via RapidAPI.  
   - Authentication: Use RapidAPI credentials with Zillow API key.  
   - Query Parameters: Map from "Set Parameters" outputs.  
   - Connect "Set Parameters" output to this node.

5. **Add a Split Out node** named "Split Out":  
   - Configure to split the array of listings from Zillow Search into individual items.  
   - Connect "Zillow Search" output to this node.

6. **Add an HTTP Request node** named "RentZestimate":  
   - Method: GET or POST to Zillow rent estimate endpoint.  
   - Authentication: Use RapidAPI credentials.  
   - Parameters: Use property identifiers from "Split Out" node.  
   - Connect "Split Out" output to this node.

7. **Add a Code node** named "Investment Calculator":  
   - Write JavaScript code to calculate:  
     - Down Payment (e.g., 20% of price)  
     - Cash on Cash ROI  
     - Monthly Cash Flow  
     - Monthly Maintenance  
     - Vacancy Loss  
   - Use inputs from "RentZestimate" node.  
   - Connect "RentZestimate" output to this node.

8. **Add a Google Sheets node** named "Google Sheets":  
   - Operation: Append or Update Rows.  
   - Configure Spreadsheet ID and Sheet Name (must match your Google Sheet).  
   - Map columns: Address, Price, Rent Zestimate, Cash on Cash ROI, Monthly Cash Flow, Down Payment, etc.  
   - Authentication: Use Google OAuth2 credentials with Sheets API enabled.  
   - Connect "Investment Calculator" output to this node.

9. **Add a Gmail node** named "Gmail":  
   - Operation: Send Email.  
   - Configure recipient email address.  
   - Set subject and body to summarize daily deals.  
   - Authentication: Use Gmail OAuth2 credentials.  
   - Connect "Google Sheets" output to this node.

10. **Credential Setup:**  
    - Configure Google OAuth2 credentials for Google Sheets and Gmail nodes.  
    - Configure RapidAPI credentials with Zillow API key for HTTP Request nodes.

11. **Test the workflow** with a small dataset to verify API responses, calculations, sheet updates, and email delivery.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Google OAuth setup video for credentials configuration                                              | https://youtu.be/LTuy83t_Rt4?si=0XdpxM7G48gtFDe6                                                  |
| Full step-by-step video tutorial for this workflow setup                                            | https://www.youtube.com/watch?v=OSeLeKc375Y&t=6s                                                  |
| Troubleshooting tips: check API credentials, request limits, formula correctness, and column mapping | Included in workflow description                                                                   |
| Prerequisites: n8n account, Google account with Sheets API enabled, RapidAPI account with Zillow API | Included in workflow description                                                                   |

---

This documentation provides a detailed understanding of the workflow’s structure, node configurations, and operational flow, enabling advanced users and AI agents to reproduce, modify, and troubleshoot the automation effectively.