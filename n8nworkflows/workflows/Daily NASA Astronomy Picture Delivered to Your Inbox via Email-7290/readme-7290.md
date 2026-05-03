Daily NASA Astronomy Picture Delivered to Your Inbox via Email

https://n8nworkflows.xyz/workflows/daily-nasa-astronomy-picture-delivered-to-your-inbox-via-email-7290


# Daily NASA Astronomy Picture Delivered to Your Inbox via Email

### 1. Workflow Overview

This workflow automates the daily retrieval and email delivery of NASA's Astronomy Picture of the Day (APOD) using their official RSS feed. It is designed for users who want to receive a curated summary of the latest astronomy images and explanations directly in their inbox. The workflow is scheduled to run daily and filters content to only include the most recent two days to avoid redundancy.

The workflow can be logically divided into the following blocks:

- **1.1 Input Reception**: Scheduled trigger initiates the workflow once per day.
- **1.2 Data Retrieval**: Reads the NASA APOD RSS feed to obtain recent astronomy pictures and descriptions.
- **1.3 Data Filtering and Transformation**: Filters to keep only entries from the last two days, extracts relevant fields, and aggregates the data into an HTML message.
- **1.4 Email Sending**: Sends the aggregated content as a styled HTML email to a predefined recipient via SMTP.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Initiates the workflow execution daily at a specified time.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Configuration: Triggers at 06:21 AM daily.  
    - Input/Output: No input; output triggers "Get APOD Data" node.  
    - Edge Cases: Misconfigured time zones could cause unexpected trigger times.

#### 1.2 Data Retrieval

- **Overview:**  
  Fetches the latest NASA APOD data via the official RSS feed.

- **Nodes Involved:**  
  - Get APOD Data  
  - Sticky Note16 (commentary)

- **Node Details:**  
  - **Get APOD Data**  
    - Type: RSS Feed Read  
    - Configuration: Reads from the URL `https://apod.com/feed.rss`.  
    - Input: Triggered by Schedule Trigger.  
    - Output: Feeds data into "Set only important fields".  
    - Edge Cases: RSS feed unavailable, network errors, malformed feed data.  
  - **Sticky Note16**  
    - Provides contextual information: "Get data from the official NASA APOD RSS Feed using the url".

#### 1.3 Data Filtering and Transformation

- **Overview:**  
  Filters entries to include only those from the last two days. Then extracts key fields (title, content, link, date) and formats them into an HTML snippet for the email body. Finally, summarizes the messages if multiple exist.

- **Nodes Involved:**  
  - Set only important fields  
  - Filter only last 2 days  
  - Aggregate data  
  - Summarize  
  - Sticky Note (Filter explanation)

- **Node Details:**  
  - **Set only important fields**  
    - Type: Set  
    - Configuration: Extracts and renames fields from RSS feed JSON:  
      - "Titre" ← title  
      - "Contenu" ← content:encoded  
      - "Lien" ← link  
      - "Date" ← pubDate  
    - Input: Output of "Get APOD Data".  
    - Output: Feeds into "Filter only last 2 days".  
    - Edge Cases: Missing or malformed fields in RSS entries.  
  - **Filter only last 2 days**  
    - Type: Filter  
    - Configuration: Keeps only entries where the "Date" field is after the current date minus 2 days.  
    - Input: Output of "Set only important fields".  
    - Output: Feeds into "Aggregate data".  
    - Edge Cases: Date parsing errors, time zone discrepancies.  
  - **Aggregate data**  
    - Type: Set  
    - Configuration: Creates an HTML article snippet using the filtered data fields to be included in the email body.  
    - Input: Output of "Filter only last 2 days".  
    - Output: Feeds into "Summarize".  
    - Key Expression: HTML template embedding `{{ $json.Titre }}`, `{{ $json.Contenu }}`, `{{ $json.Lien }}`, `{{ $json.Date }}`.  
  - **Summarize**  
    - Type: Summarize  
    - Configuration: Appends all "message" fields into one concatenated string.  
    - Input: Output of "Aggregate data".  
    - Output: Feeds into "Send email".  
  - **Sticky Note (Filter explanation)**  
    - Content: "Filter the pictures to only get the one from the last 2 days (or the last one)".

#### 1.4 Email Sending

- **Overview:**  
  Sends the compiled APOD content as a styled HTML email to a specified recipient via SMTP.

- **Nodes Involved:**  
  - Send email  
  - Sticky Note1 (email configuration instructions)  
  - Sticky Note2 (general workflow description)

- **Node Details:**  
  - **Send email**  
    - Type: Email Send  
    - Configuration:  
      - Subject: "Astronomy Picture of the Day"  
      - To: "destination@email.com" (to be customized)  
      - From: "NASA @N8N <you@gmail.com>" (to be customized)  
      - HTML Body: Custom HTML template embedding `{{ $json.appended_message }}` which contains the aggregated APOD articles.  
      - SMTP Credentials: Preconfigured SMTP account (e.g., Gmail SMTP with OAuth2 or API keys).  
    - Input: Output of "Summarize".  
    - Output: None (end node).  
    - Edge Cases: Authentication failures, network issues, invalid email addresses, SMTP rate limits.  
  - **Sticky Note1**  
    - Guides user to configure SMTP with correct Gmail API keys and customize the email HTML.  
  - **Sticky Note2**  
    - Provides general description of the workflow and customization suggestions.

---

### 3. Summary Table

| Node Name               | Node Type         | Functional Role                            | Input Node(s)        | Output Node(s)       | Sticky Note                                                                                                    |
|-------------------------|-------------------|-------------------------------------------|----------------------|----------------------|---------------------------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger  | Initiates workflow daily at 06:21 AM      | None                 | Get APOD Data        |                                                                                                               |
| Get APOD Data           | RSS Feed Read     | Fetches NASA APOD RSS feed data            | Schedule Trigger     | Set only important fields | Get data from the official NASA APOD RSS Feed using the url                                                  |
| Set only important fields| Set              | Extracts and renames key RSS fields        | Get APOD Data        | Filter only last 2 days|                                                                                                               |
| Filter only last 2 days | Filter            | Filters entries from last 2 days            | Set only important fields | Aggregate data     | Filter the pictures to only get the one from the last 2 days (or the last one)                                |
| Aggregate data          | Set               | Creates HTML article snippets from data    | Filter only last 2 days | Summarize           |                                                                                                               |
| Summarize               | Summarize         | Concatenates multiple HTML snippets         | Aggregate data       | Send email            |                                                                                                               |
| Send email              | Email Send        | Sends the compiled APOD HTML via SMTP       | Summarize            | None                  | Configure an email sender with SMTP (e.g., Gmail). Customize the HTML content as desired.                      |
| Sticky Note16           | Sticky Note       | Comment on RSS feed data retrieval          | None                 | None                  | Get data from the official NASA APOD RSS Feed using the url                                                  |
| Sticky Note             | Sticky Note       | Comment on filtering logic                   | None                 | None                  | Filter the pictures to only get the one from the last 2 days (or the last one)                                |
| Sticky Note1            | Sticky Note       | Comment on email sender configuration        | None                 | None                  | Configure an email sender. Use Gmail SMTP with API keys. Customize HTML email body.                            |
| Sticky Note2            | Sticky Note       | General workflow description                  | None                 | None                  | NASA Astronomy Picture of the Day. Customize schedule and email HTML body as you want!                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n and name it "NASA APOD".**

2. **Add a Schedule Trigger node:**
   - Set to trigger daily at 06:21 AM (adjust time zone as needed).
   - This node has no input and triggers the next node.

3. **Add an RSS Feed Read node named "Get APOD Data":**
   - Set the RSS feed URL to `https://apod.com/feed.rss`.
   - Connect the Schedule Trigger output to this node's input.

4. **Add a Set node named "Set only important fields":**
   - Extract and rename fields from the RSS feed JSON. Assign:
     - `Titre` = `{{$json["title"]}}`
     - `Contenu` = `{{$json["content:encoded"]}}`
     - `Lien` = `{{$json["link"]}}`
     - `Date` = `{{$json["pubDate"]}}`
   - Connect output of "Get APOD Data" to this node.

5. **Add a Filter node named "Filter only last 2 days":**
   - Condition: Keep items where `Date` is after the current date minus 2 days.
   - Use expression:  
     - Left value: `{{$json.Date}}`  
     - Operator: dateTime `after`  
     - Right value: `{{$today.minus({ days: 2 })}}`
   - Connect "Set only important fields" output to this node.

6. **Add a Set node named "Aggregate data":**
   - Assign a new field `message` with this HTML content (use n8n expression syntax):  

```html
<article style="max-width: 600px; margin: 20px; padding: 20px; border: 1px solid #ddd; border-radius: 8px; font-family: Arial, sans-serif; background-color: #f9f9f9;">
  <h2 style="color: #333; margin: 0 0 15px 0; font-size: 1.4em; line-height: 1.3;">
    <a href="{{ $json.Lien }}" 
       style="color: #0066cc; text-decoration: none;" 
       target="_blank">
      {{ $json.Titre }}
    </a>
  </h2>
  
  <p style="color: #666; margin: 0 0 15px 0; line-height: 1.5; font-size: 1em;">
    {{ $json.Contenu }}
  </p>
  
  <time style="color: #999; font-size: 0.9em; font-style: italic;">
    {{ $json.Date }}
  </time>
</article>
```
   - Connect output of "Filter only last 2 days" to this node.

7. **Add a Summarize node named "Summarize":**
   - Summarize by appending all values in the `message` field.
   - Connect output of "Aggregate data" to this node.

8. **Add an Email Send node named "Send email":**
   - Configure SMTP credentials (e.g., Gmail SMTP with OAuth2 or API keys). Create a credential in n8n beforehand.
   - Set email parameters:  
     - To: destination email address (replace `destination@email.com`)  
     - From: sender email (e.g., "NASA @N8N <you@gmail.com>")  
     - Subject: "Astronomy Picture of the Day"  
     - HTML Body: Use expression to embed the summarized message: `{{ $json.appended_message }}`
   - Connect output of "Summarize" to this node.

9. **Activate the workflow to enable daily execution.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                               | Context or Link                                          |
|------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------|
| NASA Astronomy Picture of the Day delivered by email. Customize the Schedule Trigger and HTML email body as desired.                                       | Workflow description sticky note                         |
| SMTP configuration requires valid credentials; Gmail users must create an SMTP account with appropriate API keys or OAuth2 credentials in n8n.             | Email configuration sticky note                          |
| RSS Feed URL used: https://apod.com/feed.rss                                                                                                              | RSS feed source note                                     |
| The email HTML template uses inline CSS styling for better compatibility across email clients.                                                           | Email HTML styling note                                  |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.