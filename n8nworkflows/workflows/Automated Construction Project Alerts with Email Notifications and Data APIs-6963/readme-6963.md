Automated Construction Project Alerts with Email Notifications and Data APIs

https://n8nworkflows.xyz/workflows/automated-construction-project-alerts-with-email-notifications-and-data-apis-6963


# Automated Construction Project Alerts with Email Notifications and Data APIs

### 1. Workflow Overview

This workflow, titled **"Real-Time Competitive Construction Monitoring"**, automates the process of monitoring construction projects in specified geographic areas and sends email alerts with detailed reports. It is designed for construction professionals, project managers, or competitive analysts who need timely updates on public and private construction permits and projects.

The workflow includes two main entry points:  
- **Email Trigger**: Listens for incoming emails with a specific subject line to initiate a custom search based on user requests.  
- **Schedule Trigger**: Automatically runs at scheduled intervals (e.g., weekdays at 9 AM) for regular monitoring without manual input.

Logical blocks grouped by functional role and dependencies:

- **1.1 Input Reception and Validation**  
  Receives trigger inputs from email or scheduled event, validates the email subject to ensure relevance, and extracts location details from the email body.

- **1.2 Data Acquisition**  
  Queries two distinct external data sources: government public construction permit databases and private construction industry project APIs.

- **1.3 Data Processing and Filtering**  
  Normalizes and combines retrieved data, filters projects based on recency and relevance.

- **1.4 Decision and Notification**  
  Checks if projects were found, generates a detailed HTML and plain-text email report for positive cases or sends a notification email if no projects are found.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception and Validation

**Overview:**  
This block handles workflow triggering via scheduled time or incoming email, checks for valid email subjects, and extracts geographic area information from user email requests.

**Nodes Involved:**  
- Schedule Trigger  
- Email Trigger  
- Check Email Subject  
- Extract Location Info  

**Node Details:**

- **Schedule Trigger**  
  - Type: Trigger  
  - Role: Initiates workflow automatically on a time schedule (default: every day, can be configured for weekdays at 9 AM).  
  - Configuration: Interval trigger with default daily interval.  
  - Inputs: None  
  - Outputs: Connects to Extract Location Info for automatic searches.  
  - Failures: Misconfigured schedule or time zone issues.  

- **Email Trigger**  
  - Type: Trigger (Email IMAP)  
  - Role: Watches an IMAP mailbox for new emails.  
  - Configuration: Uses IMAP credentials (configured externally) to read incoming email.  
  - Inputs: None  
  - Outputs: To Check Email Subject node for filtering.  
  - Failures: Authentication failure, mailbox connectivity issues.  

- **Check Email Subject**  
  - Type: Conditional (If)  
  - Role: Filters incoming emails to only process those whose subject contains "Construction Alert Request".  
  - Configuration: Case-sensitive string contains condition on email subject.  
  - Inputs: From Email Trigger  
  - Outputs: Passes valid emails to Extract Location Info, discards others.  
  - Failures: Expression evaluation errors if email subject missing or malformed.  

- **Extract Location Info**  
  - Type: Code (JavaScript)  
  - Role: Parses the email body to extract structured location info: area, city, state, zipcode; falls back to regex extraction if needed.  
  - Configuration: Custom JS to parse text or HTML email content, extracts multiple location fields and builds a search query string.  
  - Inputs: From Check Email Subject or Schedule Trigger (for scheduled runs, input data might be empty or preset).  
  - Outputs: Provides extracted location and search query for downstream API calls.  
  - Failures: Parsing errors if email body format unexpected, missing fields, or empty email bodies.  
  - Edge cases: Unstructured or poorly formatted emails may result in incomplete data extraction.

---

#### Block 1.2: Data Acquisition

**Overview:**  
Queries external data APIs to gather construction project information from government and industry sources using the extracted location.

**Nodes Involved:**  
- Search Government Data  
- Search Construction Sites  

**Node Details:**

- **Search Government Data**  
  - Type: HTTP Request  
  - Role: Queries a government API endpoint for jobs/public projects related to construction permits in the specified location.  
  - Configuration: HTTP GET with query parameters including "construction permit" keyword and location from extracted data. Uses generic HTTP Query authentication with stored credentials.  
  - Inputs: From Extract Location Info  
  - Outputs: JSON response with public project listings.  
  - Failures: Network errors, API throttling, authentication errors, invalid query parameters.  
  - Version: Uses HTTP Request node version 4.1 with query auth.  

- **Search Construction Sites**  
  - Type: HTTP Request  
  - Role: Queries a private construction industry API for projects matching location and construction keywords.  
  - Configuration: HTTP GET with query and header parameters simulating a browser User-Agent, includes search term and result limit.  
  - Inputs: From Extract Location Info  
  - Outputs: JSON response with private construction project data.  
  - Failures: Timeout (configured 10s), network issues, API response errors, rate limits.  

---

#### Block 1.3: Data Processing and Filtering

**Overview:**  
Combines and normalizes API responses, applies filtering logic to identify recent or upcoming projects, and prepares summarized data for reporting.

**Nodes Involved:**  
- Process Construction Data  
- Wait For Data  

**Node Details:**

- **Process Construction Data**  
  - Type: Code (JavaScript)  
  - Role:  
    - Extracts projects from both API responses.  
    - Normalizes project attributes (title, location, description, source, date, type).  
    - Applies a filter for projects starting now or within approximately 3 months.  
    - Provides a combined list limited to the top 10 relevant projects.  
  - Configuration: Custom JS merges arrays, applies date filtering, supports fallback mock data if private API returns no projects.  
  - Inputs: From Search Government Data and Search Construction Sites (two inputs merged).  
  - Outputs: Filtered project data including metadata for email generation.  
  - Failures: JS exceptions if input data missing or with unexpected structure.  
  - Edge cases: Empty or incomplete API responses handled with fallback data to avoid empty reports.  

- **Wait For Data**  
  - Type: Wait  
  - Role: Ensures synchronization point/wait for data processing completion before decision making.  
  - Configuration: Default wait (no delay), acts as a buffer node.  
  - Inputs: From Process Construction Data  
  - Outputs: To Check if Projects Found node.  
  - Failures: Minimal, unless workflow execution issues.  

---

#### Block 1.4: Decision and Notification

**Overview:**  
Determines if any projects were found and sends the appropriate email notification to the requester or an admin.

**Nodes Involved:**  
- Check if Projects Found  
- Generate Email Report  
- Send Alert Email  
- Send No Results Email  

**Node Details:**

- **Check if Projects Found**  
  - Type: Conditional (If)  
  - Role: Checks if the total number of projects found is greater than zero.  
  - Configuration: Numeric comparison on the totalProjects field.  
  - Inputs: From Wait For Data  
  - Outputs:  
    - True: To Generate Email Report  
    - False: To Send No Results Email  
  - Failures: Expression failures if data missing.  

- **Generate Email Report**  
  - Type: Code (JavaScript)  
  - Role: Builds a detailed HTML and plain text email body summarizing the construction projects found, including styling and structure for readability.  
  - Configuration: Custom JS constructs email content with project details, summary, and next steps.  
  - Inputs: From Check if Projects Found (true branch)  
  - Outputs: JSON object with email subject, HTML body, text body, recipient email, and metadata.  
  - Failures: JS exceptions if input data incomplete or malformed.  

- **Send Alert Email**  
  - Type: Email Send  
  - Role: Sends the constructed alert email to the original requester email address.  
  - Configuration: Uses SMTP credentials (configured externally), sends HTML and text email with subject and from address set.  
  - Inputs: From Generate Email Report  
  - Outputs: None (terminal node)  
  - Failures: SMTP authentication errors, network issues, invalid recipient email.  

- **Send No Results Email**  
  - Type: Email Send  
  - Role: Sends notification email to an admin address when no projects are found.  
  - Configuration: Plain text email with fixed subject, recipient admin@yourcompany.com, from alerts@yourcompany.com, includes info about the search area and time.  
  - Inputs: From Check if Projects Found (false branch)  
  - Outputs: None (terminal node)  
  - Failures: SMTP errors, invalid admin email.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                            | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                                                              |
|-------------------------|---------------------|------------------------------------------|-----------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger         | Schedule Trigger    | Automatic scheduled workflow initiation  | None                        | Extract Location Info        | ## **How it works**<br>* **Schedule Trigger** runs automatically on weekdays at 9 AM for regular monitoring.                            |
| Email Trigger           | Email Read IMAP     | Triggers workflow on new emails           | None                        | Check Email Subject          | ## **How it works**<br>* **Email Trigger** detects emails with "Construction Alert Request" in subject line.                             |
| Check Email Subject     | If                  | Filters emails by subject                  | Email Trigger               | Extract Location Info        | ## **How it works**<br>* **Check Email Subject** validates email contains trigger phrase.                                                |
| Extract Location Info   | Code                 | Parses location info from email body      | Schedule Trigger, Check Email Subject | Search Government Data, Search Construction Sites | ## **How it works**<br>* **Extract Location Info** parses area, city, state, zip from email.                                              |
| Search Government Data  | HTTP Request         | Queries government construction data      | Extract Location Info       | Process Construction Data    | ## **How it works**<br>* **Search Government Data** queries public construction projects API.                                            |
| Search Construction Sites | HTTP Request       | Queries private construction projects API | Extract Location Info       | Process Construction Data    | ## **How it works**<br>* **Search Construction Sites** queries private industry construction projects API.                              |
| Process Construction Data | Code               | Normalizes and combines API results       | Search Government Data, Search Construction Sites | Wait For Data               | ## **How it works**<br>* **Process Construction Data** combines and filters results from both sources.                                   |
| Wait For Data           | Wait                 | Synchronizes data processing               | Process Construction Data   | Check if Projects Found      | ## **How it works**<br>* **Wait For Data** waits for data processing completion.                                                         |
| Check if Projects Found | If                   | Decides if projects were found             | Wait For Data               | Generate Email Report, Send No Results Email | ## **How it works**<br>* **Check If Projects Found** sends report or no-results notification.                                             |
| Generate Email Report   | Code                 | Builds HTML and plain text project report | Check if Projects Found (true) | Send Alert Email            | ## **How it works**<br>* **Generate Email Report** creates professional email with project details and summaries.                       |
| Send Alert Email        | Email Send            | Sends project alert email to requester    | Generate Email Report       | None                        | ## **How it works**<br>* **Send Alert Email** delivers report email to requester.                                                        |
| Send No Results Email   | Email Send            | Sends no-results notification to admin    | Check if Projects Found (false) | None                     | ## **How it works**<br>* **Send No Results Email** notifies admin when no projects found.                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set interval to run daily or weekdays at 9 AM (adjust as needed).  
   - Connect output to Extract Location Info node (to initiate automatic search).  

2. **Create the Email Trigger node:**  
   - Type: Email Read IMAP  
   - Configure IMAP credentials (host, port, username, password).  
   - Set options to check new emails.  
   - Connect output to Check Email Subject node.  

3. **Create the Check Email Subject node:**  
   - Type: If  
   - Configure condition: String contains "Construction Alert Request" on email subject.  
   - Connect true output to Extract Location Info node.  

4. **Create the Extract Location Info node:**  
   - Type: Code (JavaScript)  
   - Use provided JS code to parse email body for area, city, state, zipcode.  
   - Prepare a search query string combining area or city and state.  
   - Connect outputs to both Search Government Data and Search Construction Sites nodes.  

5. **Create the Search Government Data node:**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.usa.gov/jobs/search.json`  
   - Query parameters:  
     - keyword = "construction permit"  
     - location_name = expression from extracted search area  
     - size = 20  
   - Authentication: HTTP Query with stored credentials (set up in n8n credentials).  
   - Connect output to Process Construction Data node.  

6. **Create the Search Construction Sites node:**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://www.construction.com/api/search`  
   - Query parameters:  
     - q = expression from extracted search area + " construction projects"  
     - type = "projects"  
     - limit = 15  
   - Headers:  
     - User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36  
     - Accept: application/json  
   - Timeout: 10000 ms  
   - Connect output to Process Construction Data node.  

7. **Create Process Construction Data node:**  
   - Type: Code (JavaScript)  
   - Use provided JS code to:  
     - Extract projects from both API responses.  
     - Normalize fields (title, location, description, source, URL, startDate, type).  
     - Filter projects starting now or within 3 months or marked TBD.  
     - Limit to top 10 projects.  
     - Provide combined result with metadata.  
   - Connect output to Wait For Data node.  

8. **Create Wait For Data node:**  
   - Type: Wait  
   - Leave default parameters (no delay).  
   - Connect output to Check if Projects Found node.  

9. **Create Check if Projects Found node:**  
   - Type: If  
   - Condition: Check if `totalProjects` > 0 from input JSON.  
   - True output connects to Generate Email Report node.  
   - False output connects to Send No Results Email node.  

10. **Create Generate Email Report node:**  
    - Type: Code (JavaScript)  
    - Use provided JS code to generate well-formatted HTML and plain text email content with project details, summaries, and next steps.  
    - Outputs JSON with subject, htmlContent, textContent, recipientEmail, and metadata.  
    - Connect output to Send Alert Email node.  

11. **Create Send Alert Email node:**  
    - Type: Email Send  
    - Configure SMTP credentials (host, port, username, password).  
    - From email: alerts@yourcompany.com  
    - To email: expression from Generate Email Report output `recipientEmail`  
    - Subject: expression from Generate Email Report output `subject`  
    - HTML Body: expression from Generate Email Report output `htmlContent`  
    - Connect no further nodes (terminal).  

12. **Create Send No Results Email node:**  
    - Type: Email Send  
    - Configure SMTP credentials (same as above).  
    - From email: alerts@yourcompany.com  
    - To email: admin@yourcompany.com  
    - Subject: "No Construction Projects Found"  
    - Text Body: Use expression to include search area, timestamp, and original requestor email from extracted location info node.  
    - No further connections (terminal).  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                   | Context or Link                                                                                       |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| The workflow automatically monitors and processes construction project alerts from both scheduled intervals and email requests.                                             | Workflow Purpose                                                                                     |
| Email trigger uses IMAP for inbound email reading; ensure credentials are properly secured and tested.                                                                         | Email Trigger Configuration                                                                          |
| API credentials must be set up in n8n for both government and private construction data sources to enable authenticated requests.                                            | Credentials Setup                                                                                     |
| The email report includes both HTML and plain text versions for compatibility with various email clients.                                                                    | Generate Email Report Node                                                                            |
| Fallback mock data is used if private API returns no data, ensuring reports are never empty and maintaining consistent output format.                                         | Process Construction Data Node                                                                       |
| The workflow includes a sticky note summarizing its operation logic for user reference.                                                                                       | Internal Documentation (Sticky Note in workflow)                                                    |
| For detailed API documentation refer to: [USA.gov Jobs API](https://developer.usa.gov/) and [Construction.com API](https://www.construction.com/api-docs) (if available).      | External API Links                                                                                   |
| SMTP server must support sending from alerts@yourcompany.com and allow relay to intended recipients.                                                                          | Email Sending Configuration                                                                          |

---

_Disclaimer: The provided text is generated exclusively from an automated workflow created with n8n, a workflow automation platform. This process complies strictly with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public._