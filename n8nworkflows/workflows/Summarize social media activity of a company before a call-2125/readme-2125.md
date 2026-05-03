Summarize social media activity of a company before a call

https://n8nworkflows.xyz/workflows/summarize-social-media-activity-of-a-company-before-a-call-2125


# Summarize social media activity of a company before a call

### 1. Workflow Overview

This n8n workflow automates the process of preparing sales representatives for their daily meetings by summarizing recent social media activity of companies involved in those meetings. It targets sales professionals who need quick, AI-generated overviews of companies’ latest LinkedIn and X (Twitter) posts to gain insights without manual research.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Trigger & Setup**: Initiates the workflow every morning and loads required API keys and recipient emails.
- **1.2 Calendar Events Retrieval & Attendee Processing**: Fetches meetings from Google Calendar, extracts attendee email domains, and splits them for individual processing.
- **1.3 Company Enrichment & Social Media Data Retrieval**: Uses Clearbit to enrich company data, then fetches recent LinkedIn and Twitter posts via RapidAPI.
- **1.4 Data Extraction & Combination**: Extracts relevant post data from responses and merges LinkedIn and Twitter data per company.
- **1.5 AI Summarization & Email Preparation**: Sends combined social media data to OpenAI’s GPT-4 for summarization, formats personalized email content, and sends the emails via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Setup

**Overview:**  
Starts the workflow every day at 7 AM and loads static configuration data like API keys and email recipients.

**Nodes Involved:**  
- Every morning @ 7  
- Setup  
- Sticky Note (instructions)

**Node Details:**

- **Every morning @ 7**  
  - Type: Schedule Trigger  
  - Role: Triggers workflow daily at 7:00 AM  
  - Configuration: Interval trigger with triggerAtHour set to 7  
  - Inputs: None  
  - Outputs: Connects to Setup  
  - Failures: None expected unless scheduler fails  

- **Setup**  
  - Type: Set  
  - Role: Stores API keys (`linkedInAPIKey`, `twitterAPIKey`) and recipient emails in workflow data  
  - Configuration: Static fields set for required keys and emails  
  - Inputs: From Schedule Trigger  
  - Outputs: To Google Calendar node  
  - Failures: Incorrect or missing API keys/emails will cause failures downstream  
  - Sticky Note attached explaining setup steps and links to RapidAPI services  

- **Sticky Note**  
  - Role: Provides setup instructions and links for API registrations and key insertion  
  - Content includes instructions to register on RapidAPI for LinkedIn and Twitter APIs, and where to enter keys and emails  

---

#### 1.2 Calendar Events Retrieval & Attendee Processing

**Overview:**  
Retrieves all calendar events for the user’s Google Calendar within a range (3 days past to 1 day ahead) and extracts attendee email domains and emails for further processing.

**Nodes Involved:**  
- Get meetings for today  
- Get attendees email domains  
- Split Out  
- Keep only ones with the domain  

**Node Details:**

- **Get meetings for today**  
  - Type: Google Calendar  
  - Role: Retrieves all events from Google Calendar for the specified date range  
  - Configuration:  
    - timeMin: 3 days ago  
    - timeMax: 1 day ahead  
    - calendar: user’s calendar email  
    - operation: getAll events, singleEvents enabled  
  - Input: Setup node output  
  - Output: Raw event data with attendees  
  - Failures: Authentication errors (OAuth2), API rate limits, empty calendar  

- **Get attendees email domains**  
  - Type: Set  
  - Role: Extracts domains and attendee emails (excluding organizer) from event data  
  - Configuration:  
    - `domain`: array of email domains extracted by splitting attendee emails  
    - `attendeeEmails`: array of full attendee emails  
  - Input: From Get meetings for today  
  - Output: To Split Out node  

- **Split Out**  
  - Type: Split Out  
  - Role: Splits the array of domains so each domain is processed individually, while including `attendeeEmails` and `start` fields  
  - Input: From Get attendees email domains  
  - Output: To Keep only ones with the domain node  

- **Keep only ones with the domain**  
  - Type: Filter  
  - Role: Filters out items where domain does not exist or is empty (sanity check)  
  - Input: From Split Out  
  - Output: To Enrich attendee company node  
  - Failures: Items missing domain will be dropped  

---

#### 1.3 Company Enrichment & Social Media Data Retrieval

**Overview:**  
Enriches companies using Clearbit based on domain, then branches to fetch recent LinkedIn and Twitter posts via RapidAPI based on availability of handles or IDs.

**Nodes Involved:**  
- Enrich attendee company  
- Switch  
- Get recent LinkedIn posts  
- Get latest tweets  
- Sticky Note1 (instructions)

**Node Details:**

- **Enrich attendee company**  
  - Type: Clearbit  
  - Role: Retrieves company info including social media handles using domain  
  - Configuration: Uses domain from previous filter  
  - Input: From Keep only ones with the domain  
  - Output: To Switch node  
  - Failures: API key issues, rate limits, missing data for domain  

- **Switch**  
  - Type: Switch  
  - Role: Branches workflow depending on availability of social media info from Clearbit: LinkedIn handle or Twitter ID  
  - Configuration:  
    - Output “linkedin” if `linkedin.handle` exists and is not null  
    - Output “twitter” if `twitter.id` exists and is not null  
    - Allows multiple outputs (both branches if both exist)  
  - Input: From Enrich attendee company  
  - Output: To Get recent LinkedIn posts and Get latest tweets nodes  

- **Get recent LinkedIn posts**  
  - Type: HTTP Request  
  - Role: Calls Fresh LinkedIn Profile Data API to fetch recent company posts  
  - Configuration:  
    - URL: `https://fresh-linkedin-profile-data.p.rapidapi.com/get-company-posts`  
    - Query: linkedin_url built from LinkedIn handle  
    - Headers: RapidAPI key and host from Setup node  
  - Input: From Switch (linkedin output)  
  - Output: To Extract important data node  
  - Failures: API key errors, rate limits, invalid handles, HTTP timeout  

- **Get latest tweets**  
  - Type: HTTP Request  
  - Role: Calls Twitter API via RapidAPI to get recent tweets of company’s Twitter ID  
  - Configuration:  
    - URL: `https://twitter154.p.rapidapi.com/user/tweets`  
    - Query: user_id, limit 10, exclude replies and pinned tweets  
    - Headers: RapidAPI key and host from Setup node  
    - Batching: 1 request per 2 seconds to avoid rate limits  
  - Input: From Switch (twitter output)  
  - Output: To Extract important data again node  
  - Failures: API key errors, rate limits, invalid user IDs, connectivity issues  

- **Sticky Note1**  
  - Role: Suggests possibility of adding more social media accounts found by Clearbit with appropriate branching  
  - Content: Reminder to handle them properly if added  

---

#### 1.4 Data Extraction & Combination

**Overview:**  
Extracts the relevant social media post data from API responses, combines LinkedIn and Twitter data per company, preparing for AI summarization.

**Nodes Involved:**  
- Extract important data  
- Extract important data again  
- Combine all activity for a company  

**Node Details:**

- **Extract important data**  
  - Type: Set  
  - Role: Extracts top 10 LinkedIn posts with selected fields (text, likes, comments, postedAt), and adds company name and meeting info  
  - Configuration: Uses JavaScript expressions to slice and map API JSON response  
  - Input: From Get recent LinkedIn posts  
  - Output: To Combine all activity node (input1)  

- **Extract important data again**  
  - Type: Set  
  - Role: Extracts Twitter posts with selected fields (text, favorites, retweets, replies, postedAt), company name, and meeting info  
  - Input: From Get latest tweets  
  - Output: To Combine all activity node (input2)  

- **Combine all activity for a company**  
  - Type: Merge  
  - Role: Merges LinkedIn and Twitter data on company name, combining both sets of posts into one item  
  - Configuration:  
    - Mode: combine (keeps all data)  
    - Clash handling: prefer input 2 on conflicts  
    - Join mode: keep everything  
    - Merge by field: name  
  - Input: From both Extract important data nodes  
  - Output: To Ask AI to summarize and Extract data for email nodes  

---

#### 1.5 AI Summarization & Email Preparation

**Overview:**  
Uses OpenAI GPT-4 to create a textual summary of combined social media posts for each company, prepares personalized emails, and sends them via Gmail to configured recipients.

**Nodes Involved:**  
- Ask AI to summerize  
- Extract data for email  
- Wrap everything together  
- Prepare email template  
- Gmail  
- Sticky Note2 (email example)

**Node Details:**

- **Ask AI to summerize**  
  - Type: OpenAI (Chat)  
  - Role: Sends combined social media data (company name, LinkedIn posts, Twitter posts) to GPT-4 to generate a sales-rep-focused summary  
  - Configuration:  
    - Chat model: gpt-4  
    - Prompt: Detailed instructions to create an impersonal, email-suitable summary  
    - Input: Merged social media data  
  - Output: To Wrap everything together (input1)  
  - Failures: API key/auth errors, request timeouts, prompt errors  

- **Extract data for email**  
  - Type: Set  
  - Role: Extracts attendee email matching company domain, meeting start hour and minute, and includes company name and social media summaries for email template  
  - Input: From Combine all activity for a company  
  - Output: To Wrap everything together (input2)  

- **Wrap everything together**  
  - Type: Merge  
  - Role: Combines AI summary and email metadata into one item for templating  
  - Configuration:  
    - Mode: combine  
    - Combination mode: merge by position (pairs corresponding items)  
  - Input: From Ask AI to summarize and Extract data for email  
  - Output: To Prepare email template  

- **Prepare email template**  
  - Type: HTML  
  - Role: Generates HTML email body with meeting time, attendee email, company name, and AI summary message content  
  - Configuration: Inline HTML and CSS with dynamic placeholders for meeting details and summary  
  - Input: From Wrap everything together  
  - Output: To Gmail node  

- **Gmail**  
  - Type: Gmail  
  - Role: Sends personalized emails to configured recipient emails with subject indicating company name and message containing the summary  
  - Configuration:  
    - Send to: emails from Setup node  
    - Subject: Includes company name dynamically  
    - Message: HTML content from previous node  
    - Credentials: OAuth2 Gmail account  
  - Failures: Authentication errors, quota limits, invalid emails  

- **Sticky Note2**  
  - Role: Shows an example of the email received by recipients for clarity and presentation  
  - Content: Screenshot link of sample email  

---

### 3. Summary Table

| Node Name                    | Node Type           | Functional Role                                               | Input Node(s)                 | Output Node(s)                      | Sticky Note                                    |
|------------------------------|---------------------|--------------------------------------------------------------|------------------------------|-----------------------------------|------------------------------------------------|
| Every morning @ 7             | Schedule Trigger    | Triggers workflow daily at 7 AM                              | None                         | Setup                             |                                                |
| Setup                        | Set                 | Stores API keys and recipient emails                         | Every morning @ 7             | Get meetings for today            | Instructions on API registration and keys with RapidAPI links |
| Get meetings for today       | Google Calendar     | Retrieves calendar events for defined date range             | Setup                        | Get attendees email domains       |                                                |
| Get attendees email domains  | Set                 | Extracts domains and emails of attendees                      | Get meetings for today       | Split Out                        |                                                |
| Split Out                   | Split Out            | Splits domain array to process each domain separately         | Get attendees email domains  | Keep only ones with the domain    |                                                |
| Keep only ones with the domain| Filter             | Filters out entries with missing domain                       | Split Out                   | Enrich attendee company           |                                                |
| Enrich attendee company      | Clearbit             | Enriches company info using domain                            | Keep only ones with the domain | Switch                         |                                                |
| Switch                      | Switch               | Branches flow based on presence of LinkedIn handle or Twitter ID | Enrich attendee company      | Get recent LinkedIn posts, Get latest tweets | Advice to add more social media accounts if needed |
| Get recent LinkedIn posts    | HTTP Request         | Fetches recent LinkedIn posts via RapidAPI                    | Switch (linkedin)            | Extract important data            |                                                |
| Get latest tweets            | HTTP Request         | Fetches latest tweets via RapidAPI                            | Switch (twitter)             | Extract important data again      |                                                |
| Extract important data       | Set                  | Extracts relevant LinkedIn post data                          | Get recent LinkedIn posts    | Combine all activity for a company|                                                |
| Extract important data again | Set                  | Extracts relevant Twitter post data                           | Get latest tweets            | Combine all activity for a company|                                                |
| Combine all activity for a company | Merge          | Combines LinkedIn and Twitter data per company               | Extract important data, Extract important data again | Ask AI to summerize, Extract data for email |                                                |
| Ask AI to summerize          | OpenAI (Chat)        | Creates AI summary of combined social media activity          | Combine all activity for a company | Wrap everything together       |                                                |
| Extract data for email       | Set                  | Extracts meeting and attendee email info for email           | Combine all activity for a company | Wrap everything together       |                                                |
| Wrap everything together     | Merge                | Combines AI summary and email metadata                        | Ask AI to summerize, Extract data for email | Prepare email template        |                                                |
| Prepare email template       | HTML                 | Generates personalized HTML email content                     | Wrap everything together     | Gmail                            |                                                |
| Gmail                       | Gmail                | Sends summary email to configured recipients                  | Prepare email template       | None                            |                                                |
| Sticky Note                  | Sticky Note          | Setup instructions and RapidAPI registration links           | None                        | None                            | See "Setup" step for API registration instructions |
| Sticky Note1                 | Sticky Note          | Advice on adding more social media accounts                   | None                        | None                            | Guidance on extending social media data sources |
| Sticky Note2                 | Sticky Note          | Sample email example screenshot                               | None                        | None                            | Shows example email format sent by workflow    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Name: Every morning @ 7  
   - Type: Schedule Trigger  
   - Set trigger to daily at 7:00 AM

2. **Create Set Node for Setup**  
   - Name: Setup  
   - Type: Set  
   - Add fields:  
     - `linkedInAPIKey` (string): Your Fresh LinkedIn Profile Data API key from RapidAPI  
     - `twitterAPIKey` (string): Your Twitter API key from RapidAPI  
     - `emails` (string or array): Email addresses to receive summaries  

3. **Connect Schedule Trigger to Setup**

4. **Create Google Calendar Node**  
   - Name: Get meetings for today  
   - Type: Google Calendar  
   - Operation: getAll events  
   - Calendar: Your Google Calendar email  
   - Options:  
     - timeMin: `{{$today.minus({ days: 3 })}}`  
     - timeMax: `{{$today.plus({ days: 1 })}}`  
     - singleEvents: true  
   - Credentials: OAuth2 Google Calendar account  

5. **Connect Setup to Get meetings for today**

6. **Create Set Node**  
   - Name: Get attendees email domains  
   - Type: Set  
   - Fields:  
     - `domain`: `{{$json.attendees.filter(a => !a.organizer).map(a => a.email.split('@').pop())}}` (array)  
     - `attendeeEmails`: `{{$json.attendees.filter(a => !a.organizer).map(a => a.email)}}` (array)

7. **Connect Get meetings for today to Get attendees email domains**

8. **Create Split Out Node**  
   - Name: Split Out  
   - Type: Split Out  
   - Field to split out: `domain`  
   - Fields to include: `attendeeEmails`, `start`

9. **Connect Get attendees email domains to Split Out**

10. **Create Filter Node**  
    - Name: Keep only ones with the domain  
    - Type: Filter  
    - Condition: `{{$json.domain}}` exists and is not empty  

11. **Connect Split Out to Keep only ones with the domain**

12. **Create Clearbit Node**  
    - Name: Enrich attendee company  
    - Type: Clearbit  
    - Domain: `{{$json.domain}}`  
    - Credentials: Clearbit API key  

13. **Connect Keep only ones with the domain to Enrich attendee company**

14. **Create Switch Node**  
    - Name: Switch  
    - Type: Switch  
    - Rules:  
      - Output “linkedin”: if `{{$json.linkedin.handle !== null}}` is true  
      - Output “twitter”: if `{{$json.twitter.id !== null}}` is true  
    - Set "allMatchingOutputs" to true  

15. **Connect Enrich attendee company to Switch**

16. **Create HTTP Request Node for LinkedIn**  
    - Name: Get recent LinkedIn posts  
    - Type: HTTP Request  
    - Method: GET  
    - URL: `https://fresh-linkedin-profile-data.p.rapidapi.com/get-company-posts`  
    - Query Parameters:  
      - linkedin_url = `https://www.linkedin.com/{{$json.linkedin.handle}}`  
      - sort_by = recent  
    - Headers:  
      - X-RapidAPI-Key: from Setup node's `linkedInAPIKey`  
      - X-RapidAPI-Host: `fresh-linkedin-profile-data.p.rapidapi.com`  

17. **Connect Switch (linkedin output) to Get recent LinkedIn posts**

18. **Create HTTP Request Node for Twitter**  
    - Name: Get latest tweets  
    - Type: HTTP Request  
    - Method: GET  
    - URL: `https://twitter154.p.rapidapi.com/user/tweets`  
    - Query Parameters:  
      - limit=10  
      - user_id = `{{$json.twitter.id}}`  
      - include_replies=false  
      - include_pinned=false  
    - Headers:  
      - X-RapidAPI-Key: from Setup node's `twitterAPIKey`  
      - X-RapidAPI-Host: `twitter154.p.rapidapi.com`  
    - Batching: batch size 1, interval 2000ms  

19. **Connect Switch (twitter output) to Get latest tweets**

20. **Create Set Node for LinkedIn**  
    - Name: Extract important data  
    - Type: Set  
    - Fields:  
      - `linkedin_posts`: top 10 posts from API response mapped to text, likes, comments, postedAt  
      - `name`: company name from Switch node  
      - `meeting`: meeting data from Split Out node  

21. **Connect Get recent LinkedIn posts to Extract important data**

22. **Create Set Node for Twitter**  
    - Name: Extract important data again  
    - Type: Set  
    - Fields:  
      - `twitter_posts`: mapped from Twitter API response including text, favorites, retweets, replies, postedAt  
      - `name`: company name from Switch node  
      - `meeting`: meeting data from Split Out node  

23. **Connect Get latest tweets to Extract important data again**

24. **Create Merge Node**  
    - Name: Combine all activity for a company  
    - Type: Merge  
    - Mode: combine  
    - Join Mode: keep everything  
    - Merge by field: `name`  
    - Clash Handling: prefer input 2  

25. **Connect Extract important data (input 1) and Extract important data again (input 2) to Combine all activity for a company**

26. **Create OpenAI Node**  
    - Name: Ask AI to summerize  
    - Type: OpenAI (Chat)  
    - Model: GPT-4  
    - Prompt: Provide JSON with company name, LinkedIn posts, Twitter posts, and request a sales rep-focused summary  
    - Credentials: OpenAI API key  

27. **Connect Combine all activity for a company to Ask AI to summerize**

28. **Create Set Node**  
    - Name: Extract data for email  
    - Type: Set  
    - Fields:  
      - attendeeEmail: find from meeting.attendeeEmails the email matching the meeting domain  
      - startHour: hour extracted from meeting start datetime  
      - startMinute: minute extracted from meeting start datetime  
      - Include fields: name, html_twitter, html_linkedin  

29. **Connect Combine all activity for a company to Extract data for email**

30. **Create Merge Node**  
    - Name: Wrap everything together  
    - Type: Merge  
    - Mode: combine  
    - Combination Mode: merge by position  

31. **Connect Ask AI to summerize (input 1) and Extract data for email (input 2) to Wrap everything together**

32. **Create HTML Node**  
    - Name: Prepare email template  
    - Type: HTML  
    - HTML Content: Template with meeting time, attendee email, company name, and AI summary in styled HTML format  

33. **Connect Wrap everything together to Prepare email template**

34. **Create Gmail Node**  
    - Name: Gmail  
    - Type: Gmail  
    - Send To: emails from Setup node  
    - Subject: "Latest social activity for: {{company name}}"  
    - Message: HTML content from Prepare email template  
    - Credentials: Gmail OAuth2 account  

35. **Connect Prepare email template to Gmail**

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                 |
|-----------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Register on RapidAPI and subscribe to Fresh LinkedIn Profile Data and Twitter APIs before running workflow             | https://rapidapi.com/freshdata-freshdata-default/api/fresh-linkedin-profile-data                |
| https://rapidapi.com/omarmhaimdat/api/twitter154                                                                      | RapidAPI Twitter API                                                                             |
| Email example screenshot showing how the summary email looks                                                         | https://i.imgur.com/VcZfPpJ.png                                                                 |
| Workflow setup visual guide                                                                                            | https://i.imgur.com/AVy08cl.png                                                                 |
| Consider adding more social media accounts found by Clearbit, process them via additional branches in Switch node    | Comment in Sticky Note1                                                                          |
| Use batching and rate limiting in HTTP Requests to avoid exceeding API quotas                                         | Implemented in Twitter API node with 2-second interval and batch size 1                         |

---

This documentation fully describes the workflow structure, node functions, configuration, and reproduction steps for both human users and AI agents to understand, replicate, and maintain the workflow.