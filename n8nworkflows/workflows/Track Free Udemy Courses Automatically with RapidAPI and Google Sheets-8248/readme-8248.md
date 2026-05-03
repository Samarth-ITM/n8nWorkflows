Track Free Udemy Courses Automatically with RapidAPI and Google Sheets

https://n8nworkflows.xyz/workflows/track-free-udemy-courses-automatically-with-rapidapi-and-google-sheets-8248


# Track Free Udemy Courses Automatically with RapidAPI and Google Sheets

---

### 1. Workflow Overview

This workflow automates the process of fetching free Udemy courses (coupons) hourly from a RapidAPI endpoint and synchronizes the filtered free courses to a Google Sheet. It includes error handling through email notifications if the API request fails.

The workflow consists of the following logical blocks:

- **1.1 Schedule Trigger:** Initiates the workflow execution every hour.
- **1.2 API Request to Fetch Udemy Coupons:** Sends a POST request to retrieve featured Udemy courses via RapidAPI.
- **1.3 API Response Validation:** Checks whether the API call was successful.
- **1.4 Course Filtering:** Filters the list of courses to retain only free courses (where sale price is zero).
- **1.5 Error Handling and Notification:** Sends an alert email if the API call fails.
- **1.6 Data Synchronization:** Updates or appends the filtered free courses into a Google Sheet, ensuring data is current.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger

- **Overview:**  
  Triggers the workflow on an hourly basis to automate the Udemy coupon fetching process.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Type:** Schedule Trigger  
  - **Role:** Initiates workflow execution periodically  
  - **Configuration:** Set to trigger every 1 hour (interval set on hours field)  
  - **Input:** None (start node)  
  - **Output:** Connects to "Fetch Udemy Coupons" node  
  - **Edge Cases:** If n8n server is down or paused, triggers will be missed leading to data staleness  
  - **Version Requirements:** Compatible with n8n versions supporting scheduleTrigger v1.2  
  - **Sticky Note Content:** Explains hourly triggering purpose  

#### 2.2 API Request to Fetch Udemy Coupons

- **Overview:**  
  Sends a POST HTTP request to the RapidAPI endpoint to fetch featured Udemy courses including necessary headers and form-data parameters.

- **Nodes Involved:**  
  - Fetch Udemy Coupons

- **Node Details:**  
  - **Type:** HTTP Request  
  - **Role:** Calls Udemy coupons API via RapidAPI to request featured courses  
  - **Configuration:**  
    - Method: POST  
    - URL: `https://udemy-coupons-and-courses.p.rapidapi.com/featured.php`  
    - Headers: Includes `x-rapidapi-host` and `x-rapidapi-key` (API key placeholder must be replaced with a valid key)  
    - Content-Type: multipart/form-data  
    - Body Parameters: `page=1`  
    - Response: Full HTTP response requested  
  - **Input:** From Schedule Trigger  
  - **Output:** To Check API Success node  
  - **Edge Cases:**  
    - Invalid or missing API key causes authentication failure  
    - Network timeouts or RapidAPI rate limiting may result in errors  
    - API schema changes might affect response handling  
  - **Version Requirements:** HTTP Request node version 4.2 or higher recommended  
  - **Sticky Note Content:** Describes API call setup for Udemy coupon fetching  

#### 2.3 API Response Validation

- **Overview:**  
  Evaluates the API response to determine if the fetch was successful by checking the `success` field in the response JSON.

- **Nodes Involved:**  
  - Check API Success (If node)

- **Node Details:**  
  - **Type:** If  
  - **Role:** Branches workflow based on API success status  
  - **Configuration:**  
    - Condition: Boolean check where `{{$json.body.success}} === true`  
    - On success: proceeds to Filter Free Courses  
    - On failure: routes to Send Error Notification  
  - **Input:** From Fetch Udemy Coupons  
  - **Output:**  
    - True branch → Filter Free Courses  
    - False branch → Send Error Notification  
  - **Edge Cases:**  
    - Missing or malformed `success` field can cause condition failure  
    - False negatives if API returns success=false but actual data present  
  - **Version Requirements:** If node version 2.2 or higher  
  - **Sticky Note Content:** Explains logic for API success check and routing  

#### 2.4 Course Filtering

- **Overview:**  
  Filters the list of courses received from the API response to retain only free courses where the sale price equals zero.

- **Nodes Involved:**  
  - Filter Free Courses (Code node)

- **Node Details:**  
  - **Type:** Code (JavaScript)  
  - **Role:** Filters input JSON array to output only free courses  
  - **Configuration:**  
    - Input JSON accessed via `$input.first().json.courses`  
    - Filtering condition: `course.sale_price === 0`  
    - Returns filtered array of free courses for further processing  
  - **Input:** From Check API Success (true branch)  
  - **Output:** To Sync Courses to Google Sheet  
  - **Edge Cases:**  
    - If `courses` array is empty or missing, output will be empty  
    - If `sale_price` field is missing or non-numeric, filter may behave unexpectedly  
  - **Version Requirements:** Code node version 2 or higher  
  - **Sticky Note Content:** Describes filtering logic for free courses  

#### 2.5 Error Handling and Notification

- **Overview:**  
  Sends an email notification alerting the admin when the API fetch fails, prompting investigation.

- **Nodes Involved:**  
  - Send Error Notification (Email Send node)

- **Node Details:**  
  - **Type:** Email Send  
  - **Role:** Disseminates error alerts on API failure  
  - **Configuration:**  
    - Subject: "Udemy Coupons Fetch Error - Immediate Attention Required"  
    - Body: Informative HTML message about API fetch failure  
    - Recipient: dev@test.com (admin email)  
    - Sender: itadmin@test.com  
    - SMTP credentials configured for sending email  
  - **Input:** From Check API Success (false branch)  
  - **Output:** None (terminal node)  
  - **Edge Cases:**  
    - SMTP service downtime or credential issues may cause email failures  
    - Email address misconfiguration leads to undelivered alerts  
  - **Version Requirements:** Email Send node version 2.1 or higher  
  - **Sticky Note Content:** Explains purpose of sending error notification email  

#### 2.6 Data Synchronization

- **Overview:**  
  Inserts or updates the filtered free Udemy courses into a specified Google Sheet, ensuring the data remains current.

- **Nodes Involved:**  
  - Sync Courses to Google Sheet (Google Sheets node)

- **Node Details:**  
  - **Type:** Google Sheets  
  - **Role:** Synchronizes course data into Google Sheets document  
  - **Configuration:**  
    - Operation: Append or Update (merges data based on matching column)  
    - Sheet Name: Uses sheet identified by `gid=0` named "Courses"  
    - Document ID: Must be set to target Google Sheets document URL or ID  
    - Columns: Mapped fields include `id, name, slug, image, price, store, views, rating, category, language, lectures, sale_price, sale_start, subcategory`  
    - Matching Columns: "id" to identify existing rows for update  
    - Authentication: Uses Google Service Account credentials  
  - **Input:** From Filter Free Courses  
  - **Output:** None (terminal node)  
  - **Edge Cases:**  
    - Missing or incorrect Google Sheets document ID causes failures  
    - Credential expiration or permission issues may block updates  
    - Data type mismatches or empty fields might cause row update inconsistencies  
  - **Version Requirements:** Google Sheets node version 4.7 or higher  
  - **Sticky Note Content:** Explains syncing logic and data update behavior  

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                          | Input Node(s)           | Output Node(s)              | Sticky Note                                                                                      |
|---------------------------|---------------------|----------------------------------------|-------------------------|----------------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger           | Schedule Trigger    | Triggers workflow hourly                | None                    | Fetch Udemy Coupons         | Triggers the workflow on an hourly schedule. Starts the process of fetching Udemy coupons automatically. |
| Fetch Udemy Coupons        | HTTP Request        | Fetches featured Udemy coupons via API | Schedule Trigger        | Check API Success           | Sends a POST request to fetch featured Udemy courses via API. Includes necessary headers and form-data for the API call. |
| Check API Success          | If                  | Validates API response success          | Fetch Udemy Coupons      | Filter Free Courses, Send Error Notification | Checks if the API response indicates a successful fetch. Routes workflow either to filter courses or send an error notification. |
| Filter Free Courses        | Code                | Filters for free courses (sale_price=0)| Check API Success (true) | Sync Courses to Google Sheet| Filters out only free courses where sale price is zero. Prepares data for syncing with Google Sheets. |
| Send Error Notification    | Email Send          | Sends alert email on API failure        | Check API Success (false)| None                       | Sends an email alert if the API fetch fails. Notifies the admin to check the workflow or API.     |
| Sync Courses to Google Sheet| Google Sheets       | Updates Google Sheet with filtered data | Filter Free Courses      | None                       | Adds or updates filtered courses into the Google Sheet. Ensures the sheet is always up-to-date with latest free courses. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set interval to run every 1 hour (field: hours, value: 1)  
   - No authentication needed  
   - Position: Start of workflow

2. **Add HTTP Request Node - Fetch Udemy Coupons**  
   - Connect Schedule Trigger output to this node input  
   - Method: POST  
   - URL: `https://udemy-coupons-and-courses.p.rapidapi.com/featured.php`  
   - Content-Type: multipart/form-data  
   - Body Parameter: Add form-data parameter `page` with value `1`  
   - Header Parameters:  
     - `x-rapidapi-host` = `udemy-coupons-and-courses.p.rapidapi.com`  
     - `x-rapidapi-key` = YOUR_RAPIDAPI_KEY (replace with actual API key)  
   - Enable sending full HTTP response  
   - No authentication required beyond headers  
   - Position after Schedule Trigger

3. **Add If Node - Check API Success**  
   - Connect output of Fetch Udemy Coupons to this node input  
   - Condition:  
     - Type: Boolean  
     - Expression: `{{$json.body.success}}`  
     - Operator: equals true  
   - True output connects to Filter Free Courses node  
   - False output connects to Send Error Notification node  
   - Position after Fetch Udemy Coupons

4. **Add Code Node - Filter Free Courses**  
   - Connect If node true output to this node input  
   - Code (JavaScript):  
     ```javascript
     const courses = $input.first().json.courses;
     const freeCourses = courses.filter(course => course.sale_price === 0);
     return freeCourses;
     ```  
   - Position after Check API Success true branch

5. **Add Google Sheets Node - Sync Courses to Google Sheet**  
   - Connect Filter Free Courses output to this node input  
   - Operation: Append or Update  
   - Document ID: Set your target Google Sheet document ID or URL  
   - Sheet Name: Use `gid=0` or actual sheet name "Courses"  
   - Columns Mapping: Map fields `id, name, slug, image, price, store, views, rating, category, language, lectures, sale_price, sale_start, subcategory` with expressions referencing corresponding JSON properties  
   - Matching Columns: Set to `id` for update matching  
   - Authentication: Use Google Service Account credentials with permission to edit the sheet  
   - Position after Filter Free Courses

6. **Add Email Send Node - Send Error Notification**  
   - Connect Check API Success false output to this node input  
   - Configure SMTP credentials for sending email  
   - From Email: itadmin@test.com (or your sender address)  
   - To Email: dev@test.com (or admin recipient)  
   - Subject: "Udemy Coupons Fetch Error - Immediate Attention Required"  
   - Body (HTML):  
     ```
     Hello,<br><br>
     An error occurred while trying to fetch Udemy coupons from the API.<br><br>
     Please check the API and the workflow to resolve the issue.
     ```  
   - Position after Check API Success false branch

---

### 5. General Notes & Resources

| Note Content                                                                                                                                              | Context or Link                                   |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| This workflow automates fetching free Udemy courses hourly, filtering them, and syncing to Google Sheets with error alerts.                              | Workflow description and purpose                  |
| Replace `"your key"` in HTTP Request node headers with a valid RapidAPI key for Udemy coupons API access.                                                  | RapidAPI key setup                                 |
| Ensure Google Service Account credentials have editor permissions on the target Google Sheet for successful data sync.                                    | Google Sheets API credentials                      |
| Email notifications depend on valid SMTP credentials and correct email addresses to alert administrators on API failures.                                | Email node configuration                           |
| Sticky notes inside the workflow provide inline documentation for ease of maintenance and understanding.                                                  | Workflow documentation inside n8n                  |
| Consider monitoring workflow executions and error logs to promptly detect and resolve issues such as API downtime or credential expirations.            | Operational best practices                          |

---

**Disclaimer:** The provided text derives exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.