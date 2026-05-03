Automated Monthly Energy Reports with PostgreSQL, PDF.co & Email Delivery

https://n8nworkflows.xyz/workflows/automated-monthly-energy-reports-with-postgresql--pdf-co---email-delivery-8113


# Automated Monthly Energy Reports with PostgreSQL, PDF.co & Email Delivery

---

### 1. Workflow Overview

This workflow automates the generation and delivery of a monthly energy generation report. It is designed to fetch energy production data from a PostgreSQL database, transform and format this data into a PDF report using PDF.co API, and then email the generated report link to a specified recipient using Gmail. The workflow is organized into five logical blocks:

- **1.1 Scheduling Trigger:** Automatically initiates the workflow on a monthly basis.
- **1.2 Data Collection:** Queries the PostgreSQL database to retrieve energy production records.
- **1.3 Data Transformation:** Converts raw database records into a structured JSON format suitable for PDF generation.
- **1.4 PDF Generation:** Sends transformed data to an external API (PDF.co) to convert it into a downloadable PDF report.
- **1.5 Email Delivery:** Emails the PDF report link to the designated recipient.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduling Trigger

- **Overview:**  
  This block initiates the workflow once every month automatically, ensuring the report is generated and sent without manual intervention.

- **Nodes Involved:**  
  - Monthly Trigger

- **Node Details:**  
  - **Monthly Trigger**  
    - *Type & Role:* Schedule Trigger node, triggering workflow execution on a time-based schedule.  
    - *Configuration:* Configured to run monthly, triggering at 1 minute past the hour.  
    - *Expressions/Variables:* None specific; purely time-based trigger.  
    - *Input/Output:* No input; output triggers next node "Get energy data".  
    - *Version Requirements:* n8n v0.142.0+ supports Schedule Trigger with monthly intervals.  
    - *Potential Failures:* Workflow will not run if n8n instance is offline or scheduler is disabled.  
    - *Sub-workflow:* N/A

#### 1.2 Data Collection

- **Overview:**  
  Queries the PostgreSQL database for energy generation data from the `energy_data` table to supply raw data for the report.

- **Nodes Involved:**  
  - Get energy data

- **Node Details:**  
  - **Get energy data**  
    - *Type & Role:* PostgreSQL node, performs a SELECT operation.  
    - *Configuration:* Selects all records from the `public.energy_data` table. No filters or ordering applied.  
    - *Expressions/Variables:* None; static table and schema names.  
    - *Input/Output:* Input from "Monthly Trigger"; outputs raw database rows to "Transform data".  
    - *Version Requirements:* Requires PostgreSQL credentials configured in n8n.  
    - *Potential Failures:* Database connectivity issues, authentication errors, empty or malformed data.  
    - *Sub-workflow:* N/A

#### 1.3 Data Transformation

- **Overview:**  
  Transforms raw database records into a consolidated JSON string containing metadata (date range, note) and a list of records formatted for report generation.

- **Nodes Involved:**  
  - Transform data

- **Node Details:**  
  - **Transform data**  
    - *Type & Role:* Code node executing JavaScript to restructure data.  
    - *Configuration:*  
      - Generates a JSON string with:  
        - `date_range` hardcoded as "2025-07-01 to 2025-07-03" (likely placeholder or test value).  
        - `records` array mapped from each input record extracting fields: `id`, `site_name`, `generation_date`, `energy_generated_kwh`, `peak_power_kw`, `remarks`.  
        - `note` set to "TEST".  
    - *Expressions/Variables:* Uses `$input.all()` to process all incoming data items.  
    - *Input/Output:* Input from "Get energy data"; output JSON with serialized string to "Convert data to pdf".  
    - *Version Requirements:* JavaScript environment available in n8n; no specific version constraints.  
    - *Potential Failures:* JavaScript errors in mapping, empty input data, malformed fields.  
    - *Sub-workflow:* N/A

#### 1.4 PDF Generation

- **Overview:**  
  Sends the transformed JSON data to the PDF.co API to convert the data into a PDF report and returns a URL for the downloadable report.

- **Nodes Involved:**  
  - Convert data to pdf

- **Node Details:**  
  - **Convert data to pdf**  
    - *Type & Role:* HTTP Request node making an API call to PDF.co's `pdf/convert/from/html` endpoint.  
    - *Configuration:*  
      - POST request to `https://api.pdf.co/v1/pdf/convert/from/html`.  
      - Sends transformed JSON data in the request body.  
      - No additional query or header parameters configured in the provided snapshot (likely needs API key in headers, but not explicitly shown).  
      - `executeOnce` set to true, meaning the request is made once per invocation.  
    - *Expressions/Variables:* Uses output from "Transform data" node as request body content.  
    - *Input/Output:* Input from "Transform data"; outputs an object containing report URL and metadata to "Send Report".  
    - *Version Requirements:* Requires valid PDF.co API key configured in credentials or headers.  
    - *Potential Failures:* Authentication failure (invalid/missing API key), API rate limiting, network timeouts, invalid data format errors, empty API response.  
    - *Sub-workflow:* N/A

#### 1.5 Email Delivery

- **Overview:**  
  Sends an email with the generated PDF report link to a specified recipient using Gmail's SMTP service with OAuth2 authentication.

- **Nodes Involved:**  
  - Send Report

- **Node Details:**  
  - **Send Report**  
    - *Type & Role:* Gmail node for sending email messages.  
    - *Configuration:*  
      - Recipient email address to be configured (`sendTo` field empty in snapshot).  
      - Email subject: "Energy Report".  
      - Email body text: "Your monthly energy report is as follows {{ $json.url }}", dynamically inserting the PDF report URL from the previous node.  
      - Option to append Gmail attribution enabled.  
      - Email type set to plain text.  
    - *Expressions/Variables:* Uses `{{ $json.url }}` to insert PDF URL from "Convert data to pdf" response.  
    - *Input/Output:* Input from "Convert data to pdf"; no outputs (endpoint node).  
    - *Version Requirements:* Gmail OAuth2 credentials configured in n8n.  
    - *Potential Failures:* Authentication errors, invalid recipient email, Gmail API quota limits, network issues.  
    - *Sub-workflow:* N/A

---

### 3. Summary Table

| Node Name         | Node Type           | Functional Role                      | Input Node(s)        | Output Node(s)        | Sticky Note                                                                                                                      |
|-------------------|---------------------|------------------------------------|----------------------|-----------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Monthly Trigger    | Schedule Trigger    | Initiates workflow monthly         |                      | Get energy data        |                                                                                                                                 |
| Get energy data    | PostgreSQL          | Fetches energy data from DB        | Monthly Trigger       | Transform data         |                                                                                                                                 |
| Transform data     | Code                | Transforms DB rows into JSON string| Get energy data       | Convert data to pdf    |                                                                                                                                 |
| Convert data to pdf| HTTP Request        | Converts data JSON to PDF via API  | Transform data        | Send Report            |                                                                                                                                 |
| Send Report       | Gmail               | Sends email with report link       | Convert data to pdf   |                       |                                                                                                                                 |
| Sticky Note       | Sticky Note         | Workflow title display              |                      |                       | ## Monthly Energy Generation Report (Postgres → PDF → Email)                                                                    |
| Sticky Note1      | Sticky Note         | Purpose and core logic explanation  |                      |                       | ##  **Purpose** Automatically generate and send a monthly energy generation report. It collects energy data from a PostgreSQL database, formats it into a PDF, and emails it to a recipient. <br>##  **Core Logic** 1. **Trigger**: The workflow is scheduled to run monthly via a `Schedule Trigger` node. 2. **Data Collection**: Connects to a PostgreSQL database and fetches energy data from the `energy_data` table. 3. **Transformation**: Uses a `Code` node to transform raw database rows into a structured JSON object with metadata like `date_range`, `note`, and `records`. 4. **PDF Generation**: Sends the transformed JSON to the PDF.co API (`Convert data to pdf` node), which returns a downloadable report URL. 5. **Email Delivery**: Uses the `Gmail` node to send the report link via email to a configured recipient. <br>##  **Outcome** * An energy performance PDF report is automatically generated and emailed monthly. * Enables proactive reporting for energy production across multiple plants (solar, wind, hydro). * Removes manual intervention in creating recurring performance summaries. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Monthly Trigger" node**  
   - Type: Schedule Trigger  
   - Configuration: Set interval to monthly, trigger at 1 minute past the hour.  
   - No credentials required.  

2. **Create "Get energy data" node**  
   - Type: PostgreSQL  
   - Operation: Select  
   - Table: `energy_data`  
   - Schema: `public`  
   - Connect "Monthly Trigger" output to this node's input.  
   - Configure PostgreSQL credentials with appropriate access rights to read from the database.  

3. **Create "Transform data" node**  
   - Type: Code (JavaScript)  
   - Connect "Get energy data" output to this node's input.  
   - Code to insert:  
     ```javascript
     return {
       json: {
         json_string: JSON.stringify({
           date_range: "2025-07-01 to 2025-07-03",
           records: $input.all().map(record => ({
             id: record.json.id,
             site_name: record.json.site_name,
             generation_date: record.json.generation_date,
             energy_generated_kwh: record.json.energy_generated_kwh,
             peak_power_kw: record.json.peak_power_kw,
             remarks: record.json.remarks
           })),
           note: "TEST"
         })
       }
     };
     ```
   - No credentials required.

4. **Create "Convert data to pdf" node**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.pdf.co/v1/pdf/convert/from/html`  
   - Body: Use the output of "Transform data" as the request body (likely `json_string` field).  
   - Headers: Add `x-api-key` with your PDF.co API key.  
   - Set "Execute Once" to true.  
   - Connect "Transform data" output to this node's input.  

5. **Create "Send Report" node**  
   - Type: Gmail  
   - Credentials: Configure Gmail OAuth2 credentials.  
   - To: Specify recipient email address.  
   - Subject: "Energy Report"  
   - Message: "Your monthly energy report is as follows {{ $json.url }}"  
   - Email type: plain text  
   - Connect "Convert data to pdf" output to this node's input.  

6. **(Optional) Add Sticky Notes**  
   - To document workflow title and explanation as per original design.

7. **Activate the workflow**  
   - Ensure all credentials are properly set and tested.  
   - Test workflow manually before enabling monthly schedule.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                               | Context or Link                                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| This workflow automates monthly reporting of energy generation data from multiple types of plants (solar, wind, hydro) to enable proactive analysis and reduce manual reporting effort.                                                                                     | Workflow purpose description                                                                                              |
| PDF.co API requires an API key for authentication; ensure it is securely stored and configured in the HTTP Request node headers. Documentation: https://apidocs.pdf.co/                                                                                                   | PDF.co API documentation                                                                                                 |
| Gmail node requires OAuth2 credentials to send emails. Setup instructions: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.gmail/                                                                                                                              | n8n Gmail node documentation                                                                                              |
| The date range in the transform code is currently hardcoded ("2025-07-01 to 2025-07-03") and note set to "TEST". For production, consider dynamically calculating this range based on current date or schedule to reflect actual reporting period.                             | Suggestion for dynamic date range implementation                                                                          |
| Be aware of API limits and quotas for both PDF.co and Gmail services to avoid workflow failures during peak usages.                                                                                                                                                        | Operational consideration                                                                                                |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, a low-code integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.