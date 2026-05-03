Automated QuickBooks Sales Anomaly Detector with Professional Email Alerts

https://n8nworkflows.xyz/workflows/automated-quickbooks-sales-anomaly-detector-with-professional-email-alerts-10486


# Automated QuickBooks Sales Anomaly Detector with Professional Email Alerts

### 1. Workflow Overview

This workflow automates the detection of sales anomalies in QuickBooks data over a recent 30-day period, comparing them against sales from the previous day. Its purpose is to identify unusual sales fluctuations and promptly notify stakeholders via professional email alerts. The workflow is ideal for finance or accounting teams who want automated monitoring of sales performance with minimal manual intervention.

The workflow logic is organized into these main blocks:

- **1.1 Scheduling & Date Setup:** Triggering the workflow daily and defining date ranges for data fetching.
- **1.2 Data Retrieval:** Fetching invoices and sales receipts for the past 30 days and the previous day from QuickBooks and related APIs.
- **1.3 Data Filtering and Preparation:** Filtering paid invoices, extracting sales receipts, and merging invoice and receipt data sets.
- **1.4 Summarization & Calculation:** Summing total sales, calculating averages and deviations.
- **1.5 Anomaly Detection:** Comparing calculated sales deviations against a threshold to decide if an anomaly exists.
- **1.6 Alert Preparation and Sending:** Preparing the email content and sending a professional alert if an anomaly is detected.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling & Date Setup

**Overview:**  
This block triggers the workflow every morning and sets the necessary date ranges for data queries.

**Nodes Involved:**  
- Run_Every_Morning (Schedule Trigger)  
- Create_Date_Range (Set)

**Node Details:**  

- **Run_Every_Morning**  
  - Type: Schedule Trigger  
  - Role: Starts the workflow automatically on a daily schedule (default: every morning).  
  - Configuration: No custom cron expression shown, typically defaults to once per day.  
  - Inputs: None (trigger node)  
  - Outputs: Connects to Create_Date_Range  
  - Failure Modes: Scheduler misconfiguration, workflow disabled  
  - Version: 1.2  

- **Create_Date_Range**  
  - Type: Set  
  - Role: Defines variables for date ranges used to fetch invoices and sales receipts (likely “30 days ago” and “yesterday”).  
  - Configuration: Empty parameters in JSON, but logically expected to set date variables using expressions.  
  - Inputs: From Run_Every_Morning  
  - Outputs: Connects to all initial fetch nodes (Fetch_30Day_Invoices, Fetch_30Day_Sales_Receipts, Fetch_Yesterday_Invoices, Fetch_Yesterday_Sales_Receipts)  
  - Failure Modes: Expression errors if dates are miscalculated or missing  
  - Version: 3.4  

---

#### 2.2 Data Retrieval

**Overview:**  
Fetches invoice and sales receipt data for two periods: the last 30 days and yesterday.

**Nodes Involved:**  
- Fetch_30Day_Invoices (QuickBooks)  
- Fetch_30Day_Sales_Receipts (HTTP Request)  
- Fetch_Yesterday_Invoices (QuickBooks)  
- Fetch_Yesterday_Sales_Receipts (HTTP Request)

**Node Details:**  

- **Fetch_30Day_Invoices**  
  - Type: QuickBooks node  
  - Role: Retrieves all invoices in the last 30 days based on date parameters.  
  - Configuration: Uses date range from Create_Date_Range, no other filters specified.  
  - Inputs: From Create_Date_Range  
  - Outputs: To Filter_Paid_30Day_Invoices  
  - Failure Modes: QuickBooks API authentication, rate limits, network issues  
  - Version: 1  

- **Fetch_30Day_Sales_Receipts**  
  - Type: HTTP Request  
  - Role: Fetches sales receipts for the last 30 days, likely from QuickBooks or related API endpoint not supported natively by QuickBooks node.  
  - Configuration: Uses date range variables, method likely GET.  
  - Inputs: From Create_Date_Range  
  - Outputs: To Total_30Day_Sales_Receipts (Code node)  
  - Failure Modes: API errors, timeout, malformed response  
  - Version: 4.3  

- **Fetch_Yesterday_Invoices**  
  - Type: QuickBooks node  
  - Role: Fetches invoices from the previous day.  
  - Configuration: Uses “yesterday” date parameter from Create_Date_Range.  
  - Inputs: From Create_Date_Range  
  - Outputs: To Filter_Paid_Yesterday_Invoices  
  - Failure Modes: Same as Fetch_30Day_Invoices  
  - Version: 1  

- **Fetch_Yesterday_Sales_Receipts**  
  - Type: HTTP Request  
  - Role: Fetches sales receipts from the previous day.  
  - Configuration: Uses “yesterday” date parameter.  
  - Inputs: From Create_Date_Range  
  - Outputs: To Extract_Yesterday_Sales_Receipts (Code node)  
  - Failure Modes: Same as Fetch_30Day_Sales_Receipts  
  - Version: 4.3  

---

#### 2.3 Data Filtering and Preparation

**Overview:**  
Filters for paid invoices and extracts relevant sales receipt data. Then merges invoices and sales receipts for each period for comprehensive sales totals.

**Nodes Involved:**  
- Filter_Paid_30Day_Invoices (Filter)  
- Total_30Day_Sales_Receipts (Code)  
- Merging_30Day_Invoices_Sales_Receipts (Merge)  
- Filter_Paid_Yesterday_Invoices (Filter)  
- Extract_Yesterday_Sales_Receipts (Code)  
- Merging_Yesterday_Invoices_Sales_Receipts (Merge)

**Node Details:**  

- **Filter_Paid_30Day_Invoices**  
  - Type: Filter  
  - Role: Filters invoices from the last 30 days, keeping only those marked as paid.  
  - Configuration: Condition probably on invoice status or paid flag.  
  - Inputs: From Fetch_30Day_Invoices  
  - Outputs: To Merging_30Day_Invoices_Sales_Receipts (first input)  
  - Failure Modes: Logical errors if invoice status fields change, no paid invoices returned  
  - Version: 2.2  

- **Total_30Day_Sales_Receipts**  
  - Type: Code (JavaScript)  
  - Role: Processes sales receipt data fetched via HTTP, likely sums or restructures receipts.  
  - Configuration: Custom JavaScript code (not explicitly shown) to total sales receipts.  
  - Inputs: From Fetch_30Day_Sales_Receipts  
  - Outputs: To Merging_30Day_Invoices_Sales_Receipts (second input)  
  - Failure Modes: Code errors, unexpected data shape  
  - Version: 2  

- **Merging_30Day_Invoices_Sales_Receipts**  
  - Type: Merge  
  - Role: Combines paid invoices and sales receipts of the last 30 days into a single dataset for summarization.  
  - Configuration: Merge mode not specified, likely “Append” or “Combine Inputs”  
  - Inputs: From Filter_Paid_30Day_Invoices and Total_30Day_Sales_Receipts  
  - Outputs: To Sum_30Day_Totals  
  - Failure Modes: Mismatched data formats, empty inputs  
  - Version: 3.2  

- **Filter_Paid_Yesterday_Invoices**  
  - Type: Filter  
  - Role: Filters yesterday’s invoices for paid status similarly to 30-day filter.  
  - Inputs: From Fetch_Yesterday_Invoices  
  - Outputs: To Merging_Yesterday_Invoices_Sales_Receipts (first input)  
  - Failure Modes: Same as Filter_Paid_30Day_Invoices  
  - Version: 2.2  

- **Extract_Yesterday_Sales_Receipts**  
  - Type: Code  
  - Role: Processes yesterday’s sales receipts data, likely similar to Total_30Day_Sales_Receipts but for one day.  
  - Inputs: From Fetch_Yesterday_Sales_Receipts  
  - Outputs: To Merging_Yesterday_Invoices_Sales_Receipts (second input)  
  - Failure Modes: Code errors, missing or malformed data  
  - Version: 2  

- **Merging_Yesterday_Invoices_Sales_Receipts**  
  - Type: Merge  
  - Role: Combines yesterday’s paid invoices and sales receipts into a single dataset.  
  - Inputs: From Filter_Paid_Yesterday_Invoices and Extract_Yesterday_Sales_Receipts  
  - Outputs: To Sum_Yesterday_Totals  
  - Failure Modes: Same as Merging_30Day_Invoices_Sales_Receipts  
  - Version: 3.2  

---

#### 2.4 Summarization & Calculation

**Overview:**  
Calculates total sales sums, averages over 30 days, and deviations comparing yesterday’s sales to the average.

**Nodes Involved:**  
- Sum_30Day_Totals (Summarize)  
- Calculate_30Day_Average (Set)  
- Sum_Yesterday_Totals (Summarize)  
- Combine_Final_Totals (Merge)  
- Combining_Overall_Invoices_Sales_Receipts (Code)  
- Calculate_Deviation (Set)

**Node Details:**  

- **Sum_30Day_Totals**  
  - Type: Summarize  
  - Role: Totals sales amounts from merged 30-day invoices and receipts.  
  - Inputs: From Merging_30Day_Invoices_Sales_Receipts  
  - Outputs: To Calculate_30Day_Average  
  - Failure Modes: Missing or malformed sales fields  
  - Version: 1.1  

- **Calculate_30Day_Average**  
  - Type: Set  
  - Role: Calculates the average daily sales over the 30-day period from the total.  
  - Inputs: From Sum_30Day_Totals  
  - Outputs: To Combine_Final_Totals (first input)  
  - Failure Modes: Division by zero if no sales data  
  - Version: 3.4  

- **Sum_Yesterday_Totals**  
  - Type: Summarize  
  - Role: Totals sales amounts from merged yesterday’s invoices and receipts.  
  - Inputs: From Merging_Yesterday_Invoices_Sales_Receipts  
  - Outputs: To Combine_Final_Totals (second input)  
  - Failure Modes: Same as Sum_30Day_Totals  
  - Version: 1.1  

- **Combine_Final_Totals**  
  - Type: Merge  
  - Role: Brings together the 30-day average and yesterday’s total to a single node for comparison.  
  - Inputs: From Calculate_30Day_Average and Sum_Yesterday_Totals  
  - Outputs: To Combining_Overall_Invoices_Sales_Receipts  
  - Failure Modes: Empty inputs, mismatched data  
  - Version: 3.2  

- **Combining_Overall_Invoices_Sales_Receipts**  
  - Type: Code  
  - Role: Processes combined data to prepare for deviation calculation, likely formatting or extracting relevant fields.  
  - Inputs: From Combine_Final_Totals  
  - Outputs: To Calculate_Deviation  
  - Failure Modes: Code errors, missing fields  
  - Version: 2  

- **Calculate_Deviation**  
  - Type: Set  
  - Role: Calculates the deviation percentage or absolute difference between yesterday’s sales and the 30-day average to detect anomalies.  
  - Inputs: From Combining_Overall_Invoices_Sales_Receipts  
  - Outputs: To Check_If_Anomaly  
  - Failure Modes: Division by zero, incorrect data types  
  - Version: 3.4  

---

#### 2.5 Anomaly Detection

**Overview:**  
Determines if the sales deviation exceeds a threshold, indicating an anomaly.

**Nodes Involved:**  
- Check_If_Anomaly (If)

**Node Details:**  

- **Check_If_Anomaly**  
  - Type: If  
  - Role: Evaluates whether the calculated deviation surpasses a predefined threshold to detect anomaly.  
  - Configuration: Condition comparing deviation variable to threshold (not specified but typically percentage or value).  
  - Inputs: From Calculate_Deviation  
  - Outputs: True branch to Prepare_Email_Content, False branch ends workflow or no-op.  
  - Failure Modes: Expression evaluation errors, missing deviation data  
  - Version: 2.2  

---

#### 2.6 Alert Preparation and Sending

**Overview:**  
Prepares the content of the alert email and sends it to designated recipients.

**Nodes Involved:**  
- Prepare_Email_Content (Set)  
- Send_Alert_Email (Email Send)

**Node Details:**  

- **Prepare_Email_Content**  
  - Type: Set  
  - Role: Constructs the email body and subject with details about detected anomaly and sales figures.  
  - Inputs: From Check_If_Anomaly (true branch)  
  - Outputs: To Send_Alert_Email  
  - Failure Modes: Missing data for email, expression errors  
  - Version: 3.4  

- **Send_Alert_Email**  
  - Type: Email Send  
  - Role: Sends the alert email to recipients using configured SMTP or email credentials.  
  - Configuration: Email parameters such as To, From, Subject, and Body set via expressions or static values.  
  - Inputs: From Prepare_Email_Content  
  - Outputs: None (terminal node)  
  - Failure Modes: SMTP errors, authentication failure, invalid email addresses  
  - Version: 2.1  

---

### 3. Summary Table

| Node Name                        | Node Type           | Functional Role                                   | Input Node(s)                             | Output Node(s)                            | Sticky Note                          |
|---------------------------------|---------------------|-------------------------------------------------|------------------------------------------|------------------------------------------|------------------------------------|
| Run_Every_Morning               | Schedule Trigger    | Triggers workflow daily                          | None                                     | Create_Date_Range                        |                                    |
| Create_Date_Range               | Set                 | Defines date ranges for data fetching            | Run_Every_Morning                        | Fetch_30Day_Invoices, Fetch_30Day_Sales_Receipts, Fetch_Yesterday_Invoices, Fetch_Yesterday_Sales_Receipts |                                    |
| Fetch_30Day_Invoices            | QuickBooks          | Fetches 30-day invoices                           | Create_Date_Range                        | Filter_Paid_30Day_Invoices               |                                    |
| Filter_Paid_30Day_Invoices      | Filter              | Filters for paid 30-day invoices                  | Fetch_30Day_Invoices                     | Merging_30Day_Invoices_Sales_Receipts   |                                    |
| Fetch_30Day_Sales_Receipts      | HTTP Request        | Fetches 30-day sales receipts                      | Create_Date_Range                        | Total_30Day_Sales_Receipts               |                                    |
| Total_30Day_Sales_Receipts      | Code                | Totals 30-day sales receipts                       | Fetch_30Day_Sales_Receipts               | Merging_30Day_Invoices_Sales_Receipts   |                                    |
| Merging_30Day_Invoices_Sales_Receipts | Merge          | Merges 30-day invoices and sales receipts         | Filter_Paid_30Day_Invoices, Total_30Day_Sales_Receipts | Sum_30Day_Totals                        |                                    |
| Sum_30Day_Totals                | Summarize           | Sums total sales for 30-day period                 | Merging_30Day_Invoices_Sales_Receipts   | Calculate_30Day_Average                  |                                    |
| Calculate_30Day_Average         | Set                 | Calculates average daily sales over 30 days       | Sum_30Day_Totals                        | Combine_Final_Totals (first input)       |                                    |
| Fetch_Yesterday_Invoices        | QuickBooks          | Fetches yesterday’s invoices                       | Create_Date_Range                        | Filter_Paid_Yesterday_Invoices           |                                    |
| Filter_Paid_Yesterday_Invoices  | Filter              | Filters for paid invoices from yesterday           | Fetch_Yesterday_Invoices                 | Merging_Yesterday_Invoices_Sales_Receipts |                                    |
| Fetch_Yesterday_Sales_Receipts  | HTTP Request        | Fetches yesterday’s sales receipts                  | Create_Date_Range                        | Extract_Yesterday_Sales_Receipts         |                                    |
| Extract_Yesterday_Sales_Receipts| Code                | Processes yesterday’s sales receipts                | Fetch_Yesterday_Sales_Receipts           | Merging_Yesterday_Invoices_Sales_Receipts |                                    |
| Merging_Yesterday_Invoices_Sales_Receipts | Merge          | Merges yesterday invoices and sales receipts       | Filter_Paid_Yesterday_Invoices, Extract_Yesterday_Sales_Receipts | Sum_Yesterday_Totals                    |                                    |
| Sum_Yesterday_Totals            | Summarize           | Sums total sales for yesterday                       | Merging_Yesterday_Invoices_Sales_Receipts | Combine_Final_Totals (second input)      |                                    |
| Combine_Final_Totals            | Merge               | Merges calculated 30-day average and yesterday totals | Calculate_30Day_Average, Sum_Yesterday_Totals | Combining_Overall_Invoices_Sales_Receipts |                                    |
| Combining_Overall_Invoices_Sales_Receipts | Code          | Prepares overall data for deviation calculation    | Combine_Final_Totals                    | Calculate_Deviation                      |                                    |
| Calculate_Deviation             | Set                 | Calculates deviation between yesterday and average | Combining_Overall_Invoices_Sales_Receipts | Check_If_Anomaly                         |                                    |
| Check_If_Anomaly               | If                  | Checks if deviation indicates anomaly               | Calculate_Deviation                     | Prepare_Email_Content (true), none (false) |                                    |
| Prepare_Email_Content           | Set                 | Prepares email content for anomaly alert            | Check_If_Anomaly (true)                 | Send_Alert_Email                         |                                    |
| Send_Alert_Email                | Email Send          | Sends alert email to recipients                      | Prepare_Email_Content                   | None                                     |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Name: `Run_Every_Morning`  
   - Configure to trigger daily at a chosen morning hour (default is fine).  
   - No credentials needed.

3. **Add a Set node:**  
   - Name: `Create_Date_Range`  
   - Connect from `Run_Every_Morning`.  
   - Set date variables:  
     - `startDate` = Expression: `{{$moment().subtract(30, 'days').format('YYYY-MM-DD')}}`  
     - `endDate` = Expression: `{{$moment().format('YYYY-MM-DD')}}` (or today)  
     - `yesterdayDate` = Expression: `{{$moment().subtract(1, 'days').format('YYYY-MM-DD')}}`  
   - No credentials needed.

4. **Add QuickBooks node:**  
   - Name: `Fetch_30Day_Invoices`  
   - Operation: Query invoices.  
   - Filter: Invoice date between `startDate` and `endDate` from `Create_Date_Range`.  
   - Connect from `Create_Date_Range`.  
   - Configure QuickBooks credentials.

5. **Add HTTP Request node:**  
   - Name: `Fetch_30Day_Sales_Receipts`  
   - Method: GET  
   - URL: Your sales receipts API endpoint with query parameters for 30-day range.  
   - Use date variables from `Create_Date_Range`.  
   - Connect from `Create_Date_Range`.  
   - Configure authentication as needed.

6. **Add QuickBooks node:**  
   - Name: `Fetch_Yesterday_Invoices`  
   - Query invoices where invoice date = `yesterdayDate`.  
   - Connect from `Create_Date_Range`.  
   - Use QuickBooks credentials.

7. **Add HTTP Request node:**  
   - Name: `Fetch_Yesterday_Sales_Receipts`  
   - Method: GET  
   - URL: Your sales receipts API endpoint with query parameter for `yesterdayDate`.  
   - Connect from `Create_Date_Range`.  
   - Configure authentication.

8. **Add Filter node:**  
   - Name: `Filter_Paid_30Day_Invoices`  
   - Condition: Invoice status equals “Paid”.  
   - Connect from `Fetch_30Day_Invoices`.

9. **Add Code node:**  
   - Name: `Total_30Day_Sales_Receipts`  
   - JavaScript code to sum or process sales receipts data.  
   - Connect from `Fetch_30Day_Sales_Receipts`.

10. **Add Merge node:**  
    - Name: `Merging_30Day_Invoices_Sales_Receipts`  
    - Mode: Append or Combine Inputs.  
    - Connect inputs from `Filter_Paid_30Day_Invoices` and `Total_30Day_Sales_Receipts`.

11. **Add Summarize node:**  
    - Name: `Sum_30Day_Totals`  
    - Sum sales amount field from merged data.  
    - Connect from `Merging_30Day_Invoices_Sales_Receipts`.

12. **Add Set node:**  
    - Name: `Calculate_30Day_Average`  
    - Calculate average by dividing total by 30.  
    - Input from `Sum_30Day_Totals`.

13. **Add Filter node:**  
    - Name: `Filter_Paid_Yesterday_Invoices`  
    - Condition: Invoice status equals “Paid”.  
    - Connect from `Fetch_Yesterday_Invoices`.

14. **Add Code node:**  
    - Name: `Extract_Yesterday_Sales_Receipts`  
    - Process yesterday’s sales receipts (similar to step 9).  
    - Connect from `Fetch_Yesterday_Sales_Receipts`.

15. **Add Merge node:**  
    - Name: `Merging_Yesterday_Invoices_Sales_Receipts`  
    - Mode: Append or Combine Inputs.  
    - Connect inputs from `Filter_Paid_Yesterday_Invoices` and `Extract_Yesterday_Sales_Receipts`.

16. **Add Summarize node:**  
    - Name: `Sum_Yesterday_Totals`  
    - Sum sales amount field from merged yesterday data.  
    - Connect from `Merging_Yesterday_Invoices_Sales_Receipts`.

17. **Add Merge node:**  
    - Name: `Combine_Final_Totals`  
    - Combine average 30-day sales and yesterday’s total into one dataset (two inputs).  
    - Connect first input from `Calculate_30Day_Average`, second from `Sum_Yesterday_Totals`.

18. **Add Code node:**  
    - Name: `Combining_Overall_Invoices_Sales_Receipts`  
    - Prepare and structure combined data for deviation calculation.  
    - Connect from `Combine_Final_Totals`.

19. **Add Set node:**  
    - Name: `Calculate_Deviation`  
    - Calculate deviation as `(YesterdaySales - AverageSales) / AverageSales * 100` or similar.  
    - Connect from `Combining_Overall_Invoices_Sales_Receipts`.

20. **Add If node:**  
    - Name: `Check_If_Anomaly`  
    - Condition: Deviation > predefined threshold (e.g., 20%).  
    - Connect from `Calculate_Deviation`.

21. **Add Set node:**  
    - Name: `Prepare_Email_Content`  
    - Compose email subject and body including sales data and anomaly details.  
    - Connect from `Check_If_Anomaly` (true output).

22. **Add Email Send node:**  
    - Name: `Send_Alert_Email`  
    - Configure SMTP or email credentials.  
    - Set email recipients, subject, and body from `Prepare_Email_Content`.  
    - Connect from `Prepare_Email_Content`.

23. **Activate the workflow and test with sample data.**

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                      |
|-----------------------------------------------------------------------------------------------------|------------------------------------|
| The workflow relies on both QuickBooks nodes and HTTP Requests to fetch sales receipts, indicating that some sales receipt data may come from an API not fully supported by native QuickBooks integration. | Workflow Design Consideration       |
| Ensure QuickBooks API credentials have sufficient permissions to fetch invoices.                     | QuickBooks API Documentation       |
| SMTP or email credentials must be configured properly in the Send Alert Email node for successful delivery. | n8n Email Node Documentation       |
| Date handling uses moment.js expressions within Set nodes; adjust date formats if your locale or API requires it. | moment.js Documentation            |
| Consider adding error handling or retries on API calls for robustness in production environments.    | n8n Error Handling Best Practices  |

---

**Disclaimer:**  
The text above is generated exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.