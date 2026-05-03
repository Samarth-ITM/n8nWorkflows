Monitor iOS App Store Reviews & Send Email Notifications Automatically

https://n8nworkflows.xyz/workflows/monitor-ios-app-store-reviews---send-email-notifications-automatically-9343


# Monitor iOS App Store Reviews & Send Email Notifications Automatically

### 1. Workflow Overview

This workflow automates the monitoring of iOS App Store reviews for a specific app and sends email notifications when new reviews appear. It is designed for app developers, product managers, or customer success teams who want timely updates on user feedback without manual checking.

The workflow is structured into logical blocks:

- **1.1 Scheduled Trigger**: Periodically initiates the workflow every 30 minutes.
- **1.2 Configuration Setup**: Defines app-specific parameters such as App Store ID, country, and language.
- **1.3 Fetching Reviews**: Calls Apple’s iTunes RSS API to retrieve the latest customer reviews.
- **1.4 Data Extraction and Filtering**: Parses the API response, extracts relevant review fields, and filters for reviews posted within the last ~78 hours.
- **1.5 New Review Detection**: Checks if there are any new reviews after filtering.
- **1.6 Notification Dispatch**: Sends an email notification via Gmail for each new review or sets a status message if no new reviews are found.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
Starts the workflow automatically every 30 minutes to perform review checks without manual intervention.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Sticky Note (runs frequency explanation)

- **Node Details:**  

  - **Schedule Trigger**  
    - *Type & Role:* Built-in n8n schedule trigger node; initiates the workflow on a timed basis.  
    - *Configuration:* Cron expression `*/30 * * * *` for every 30 minutes.  
    - *Input/Output:* No input; output triggers the next node (App Config).  
    - *Edge Cases:* Potential issues if the cron expression is misconfigured or n8n is down during scheduled time slots.  
    - *Sticky Note:* Explains the runtime frequency can be adjusted as needed for monitoring sensitivity.

---

#### 2.2 Configuration Setup

- **Overview:**  
Sets static parameters for the monitored iOS app, including the App Store ID, country code, and language, which are used in API requests.

- **Nodes Involved:**  
  - App Config  
  - Sticky Note (configuration instructions)

- **Node Details:**  

  - **App Config**  
    - *Type & Role:* Set node used to store app configuration parameters as key-value pairs.  
    - *Configuration:*  
      - `appId`: `"1234567890"` (placeholder App Store ID)  
      - `country`: `"US"` (country code for App Store region)  
      - `lang`: `"en"` (language code for review localization)  
    - *Input/Output:* Receives trigger from Schedule Trigger; outputs to Fetch Reviews.  
    - *Edge Cases:* If incorrect App Store ID or country/language codes are input, API calls may fail or return no data.  
    - *Sticky Note:* Provides instructions on locating the App ID and setting country/language correctly.

---

#### 2.3 Fetching Reviews

- **Overview:**  
Constructs and sends an HTTP GET request to Apple’s iTunes RSS API to retrieve recent customer reviews for the specified app.

- **Nodes Involved:**  
  - Fetch Reviews  
  - Sticky Note (API call explanation)

- **Node Details:**  

  - **Fetch Reviews**  
    - *Type & Role:* HTTP Request node to call external API.  
    - *Configuration:*  
      - URL dynamically constructed using expressions to insert `appId`, `lang`, and `country`:  
        `https://itunes.apple.com/rss/customerreviews/page=1/id={{$json["appId"]}}/sortby=mostrecent/json?l={{$json["lang"]}}&cc={{$json["country"]}}`  
      - Uses GET method by default.  
    - *Input/Output:* Input from App Config; outputs raw API JSON to Extract Reviews.  
    - *Edge Cases:* API rate limits, network errors, or changes in API format could cause failures. If no reviews exist, the JSON might have missing fields.  
    - *Sticky Note:* Notes that reviews are sorted by most recent first.

---

#### 2.4 Data Extraction and Filtering

- **Overview:**  
Parses the API response JSON, extracts relevant review fields into a simplified format, then filters reviews to only those newer than approximately 78 hours ago.

- **Nodes Involved:**  
  - Extract Reviews (Code)  
  - Filter latest Review (Code)  
  - Sticky Notes (extraction and filtering explanations)

- **Node Details:**  

  - **Extract Reviews**  
    - *Type & Role:* Code node using JavaScript to parse JSON and map review data.  
    - *Configuration:*  
      - Parses the JSON response from the HTTP node.  
      - Extracts: rating, title, content, author, version, date, and reviewId for each review entry.  
    - *Key Expressions:* Uses optional chaining and mapping to safely extract fields.  
    - *Input/Output:* Input from Fetch Reviews; outputs array of review objects to Filter latest Review.  
    - *Edge Cases:* If the API returns unexpected JSON structure, errors may occur parsing or mapping. Empty review lists handled gracefully with default empty array.  

  - **Filter latest Review**  
    - *Type & Role:* Code node to filter review array by date.  
    - *Configuration:*  
      - Defines a cutoff date as current time minus 78 hours (approx. 3.25 days).  
      - Returns only reviews whose `date` is more recent than this cutoff.  
    - *Input/Output:* Input from Extract Reviews; outputs filtered recent reviews to Any new reviews?  
    - *Edge Cases:* If date parsing fails or review date is missing, those reviews might be excluded or cause errors. Timezone differences might cause slight inaccuracies.

---

#### 2.5 New Review Detection

- **Overview:**  
Determines if there are any new reviews after filtering and routes workflow accordingly.

- **Nodes Involved:**  
  - Any new reviews? (IF node)

- **Node Details:**  

  - **Any new reviews?**  
    - *Type & Role:* IF node that checks if the array of filtered reviews has length > 0.  
    - *Configuration:*  
      - Condition: number of items `> 0`.  
    - *Input/Output:* Input from Filter latest Review; two outputs:  
      - True branch → Send notification  
      - False branch → False Case (no new reviews)  
    - *Edge Cases:* If input is undefined or not an array, the condition may fail or cause errors.

---

#### 2.6 Notification Dispatch

- **Overview:**  
Sends email notifications for each new review or sets a status message if none found.

- **Nodes Involved:**  
  - Send notification (Gmail node)  
  - False Case (Set node for status)  

- **Node Details:**  

  - **Send notification**  
    - *Type & Role:* Gmail node to send email alerts about new reviews.  
    - *Configuration:*  
      - Recipient: `infor@exmaple.com` (likely a typo, intended as example)  
      - Subject: includes review rating and title, e.g., `New App Store Review (⭐ 5/5) — Great app`  
      - Message body: formatted with rating, title, author, version, and review content.  
      - Credentials: Gmail OAuth2 credentials selected.  
    - *Input/Output:* True branch from IF; outputs not connected further.  
    - *Edge Cases:* Potential Gmail API errors, credential expiration, sending limits, or invalid email addresses.  
    - *Sticky Note:* None.  

  - **False Case**  
    - *Type & Role:* Set node to assign a status string when no new reviews are found.  
    - *Configuration:* Sets `status` field to `"no new reviews"`.  
    - *Input/Output:* False branch from IF; no further output connections.  
    - *Edge Cases:* None significant; serves as fallback.

---

### 3. Summary Table

| Node Name         | Node Type                 | Functional Role                         | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                  |
|-------------------|---------------------------|---------------------------------------|------------------------|-------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger  | Schedule Trigger           | Initiates workflow every 30 minutes   | —                      | App Config              | **Runs every 6 hours to check for new reviews - adjust timing based on your monitoring needs** |
| App Config        | Set                       | Stores appId, country, language       | Schedule Trigger       | Fetch Reviews           | **Set your App Store ID, country (us/gb/de), and language (en/es/fr) - find App ID in your App Store URL** |
| Fetch Reviews     | HTTP Request              | Calls iTunes API to get latest reviews| App Config             | Extract Reviews         | **Calls Apple's iTunes API to get latest customer reviews - sorted by most recent first**    |
| Extract Reviews   | Code                      | Parses and extracts review data       | Fetch Reviews          | Filter latest Review    |                                                                                              |
| Filter latest Review | Code                    | Filters reviews posted in last ~78 hrs| Extract Reviews        | Any new reviews?        | **Compares with last check time to find only NEW reviews since previous run - prevents duplicate notifications** |
| Any new reviews?  | IF                        | Checks if any new reviews exist       | Filter latest Review   | Send notification, False Case |                                                                                              |
| Send notification | Gmail                     | Sends email notifications for new reviews | Any new reviews? (true) | —                       |                                                                                              |
| False Case        | Set                       | Sets status when no new reviews found | Any new reviews? (false) | —                       |                                                                                              |
| Sticky Note       | Sticky Note               | Notes on schedule trigger              | —                      | —                       | **Runs every 6 hours to check for new reviews - adjust timing based on your monitoring needs** |
| Sticky Note1      | Sticky Note               | Notes on app configuration             | —                      | —                       | **Set your App Store ID, country (us/gb/de), and language (en/es/fr) - find App ID in your App Store URL** |
| Sticky Note2      | Sticky Note               | Notes on API call                      | —                      | —                       | **Calls Apple's iTunes API to get latest customer reviews - sorted by most recent first**    |
| Sticky Note3      | Sticky Note               | Notes on filtering new reviews        | —                      | —                       | **Compares with last check time to find only NEW reviews since previous run - prevents duplicate notifications** |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set Cron Expression: `*/30 * * * *` (every 30 minutes)  
   - Connect output to next node.

2. **Create Set node named "App Config"**  
   - Add three string fields:  
     - `appId`: `1234567890` (replace with your App Store ID)  
     - `country`: `US` (replace with your target country code)  
     - `lang`: `en` (replace with your language code)  
   - Connect input from Schedule Trigger node.

3. **Create HTTP Request node named "Fetch Reviews"**  
   - Method: GET  
   - URL (Set as expression):  
     ```
     https://itunes.apple.com/rss/customerreviews/page=1/id={{$json["appId"]}}/sortby=mostrecent/json?l={{$json["lang"]}}&cc={{$json["country"]}}
     ```  
   - Connect input from App Config node.

4. **Create Code node named "Extract Reviews"**  
   - Paste JavaScript code:  
     ```js
     const jsonData = JSON.parse($input.all()[0].json.data);
     const reviews = jsonData.feed.entry || [];
     return reviews.map(review => ({
       rating: review['im:rating']?.label,
       title: review.title?.label,
       content: review.content?.label,
       author: review.author?.name?.label,
       version: review['im:version']?.label,
       date: review.updated?.label,
       reviewId: review.id?.label
     }));
     ```  
   - Connect input from Fetch Reviews node.

5. **Create Code node named "Filter latest Review"**  
   - Paste JavaScript code:  
     ```js
     const items = $input.all();
     const sixHoursAgo = new Date(Date.now() - 78 * 60 * 60 * 1000);
     const newOnes = [];
     for (const item of items) {
       const reviewDate = new Date(item.json.date);
       if (reviewDate > sixHoursAgo) {
         newOnes.push(item);
       }
     }
     return newOnes;
     ```  
   - Connect input from Extract Reviews node.

6. **Create IF node named "Any new reviews?"**  
   - Condition:  
     - Type: Number  
     - Operation: Greater than (`>`)  
     - Left value: `={{ $items().length }}`  
     - Right value: `0`  
   - Connect input from Filter latest Review node.

7. **Create Gmail node named "Send notification"**  
   - Set credentials: Gmail OAuth2 (configure credentials with your Gmail account)  
   - Recipient email: `infor@exmaple.com` (correct to your recipient)  
   - Subject (expression):  
     ```
     New App Store Review (⭐ {{$json.rating}}/5) — {{$json.title}}
     ```  
   - Message (expression):  
     ```
     🆕 New iOS App Store Review (⭐ {{$json.rating}}/5) “{{$json.title}}” by {{$json.author}} — v{{$json.version}} — {{$json.content}}
     ```  
   - Connect input from "Any new reviews?" node’s True output.

8. **Create Set node named "False Case"**  
   - Add string field: `status` = `"no new reviews"`  
   - Connect input from "Any new reviews?" node’s False output.

9. **Add Sticky Notes** (optional for clarity)  
   - Near Schedule Trigger: "Runs every 6 hours to check for new reviews - adjust timing based on your monitoring needs"  
   - Near App Config: "Set your App Store ID, country (us/gb/de), and language (en/es/fr) - find App ID in your App Store URL"  
   - Near Fetch Reviews: "Calls Apple's iTunes API to get latest customer reviews - sorted by most recent first"  
   - Near Filter latest Review: "Compares with last check time to find only NEW reviews since previous run - prevents duplicate notifications"

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                       |
|------------------------------------------------------------------------------|------------------------------------------------------|
| App Store ID can be found in your App Store URL, e.g., `id1234567890`.       | Official Apple App Store URL                          |
| Adjust cron expression to your monitoring frequency preference.               | n8n Cron Trigger Documentation                        |
| Gmail OAuth2 credentials must be configured in n8n Credentials for email sending. | n8n Gmail OAuth2 Credential Setup                     |
| Apple iTunes RSS API documentation: https://developer.apple.com/library/archive/documentation/AudioVideo/Conceptual/iTuneSearchAPI/index.html | Apple iTunes API Documentation                         |
| Review date filtering uses a 78-hour window to avoid missing reviews due to schedule delays; adjust as needed. | Workflow design consideration                          |

---

**Disclaimer:** The text provided is exclusively from an automated workflow built with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.