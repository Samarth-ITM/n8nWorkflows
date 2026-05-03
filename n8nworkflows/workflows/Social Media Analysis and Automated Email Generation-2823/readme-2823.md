Social Media Analysis and Automated Email Generation

https://n8nworkflows.xyz/workflows/social-media-analysis-and-automated-email-generation-2823


# Social Media Analysis and Automated Email Generation

### 1. Workflow Overview

This workflow automates the process of analyzing social media profiles of leads and generating personalized email outreach. It is designed for marketers, lead generation specialists, and business professionals who want to efficiently gather social media data, leverage AI for content creation, and send tailored emails without manual effort.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Initialization:** Triggered by new or updated rows in a Google Sheet containing lead data; sets company-specific variables.
- **1.2 Social Media Data Extraction:** Fetches Twitter user ID and tweets, and LinkedIn posts for each lead using RapidAPI integrations.
- **1.3 Data Processing and Limiting:** Extracts and limits the number of social media posts to a manageable subset for AI analysis.
- **1.4 AI-Powered Content Generation:** Uses OpenAI’s GPT-4o model to analyze social media data and generate a personalized email subject and cover letter.
- **1.5 Email Dispatch and Progress Tracking:** Sends the generated email to the lead and a copy to the sender, then updates the Google Sheet to mark the lead as processed.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

- **Overview:**  
  This block listens for new or updated entries in a Google Sheet containing lead information and initializes company-specific variables for use throughout the workflow.

- **Nodes Involved:**  
  - Google Sheets Trigger  
  - If (condition check)  
  - Set your company's variables

- **Node Details:**

  - **Google Sheets Trigger**  
    - *Type:* Trigger node  
    - *Role:* Watches a specific Google Sheet for new or updated rows.  
    - *Configuration:* Polls every minute on a specified sheet and range; expects columns including linkedin_url, name, twitter_handler, email, and done.  
    - *Inputs:* None (trigger)  
    - *Outputs:* Emits lead data as JSON.  
    - *Edge Cases:* Authentication errors with Google Sheets OAuth2; empty or malformed rows; missing required columns.

  - **If**  
    - *Type:* Conditional node  
    - *Role:* Checks if the "done" field in the lead row is empty to avoid reprocessing completed leads.  
    - *Configuration:* Condition tests if `done` field is empty string.  
    - *Inputs:* Output from Google Sheets Trigger  
    - *Outputs:* Passes data forward only if lead is not yet processed.  
    - *Edge Cases:* If "done" field is missing or contains unexpected values, may skip or reprocess leads.

  - **Set your company's variables**  
    - *Type:* Set node  
    - *Role:* Defines static variables such as company name, activity description, sender name, and sender email for use in AI prompts and email sending.  
    - *Configuration:* Four string variables set: your_company_name, your_company_activity, your_email, your_name.  
    - *Inputs:* Output from If node  
    - *Outputs:* Passes variables downstream.  
    - *Edge Cases:* Variables must be updated by user to reflect actual company info; missing or incorrect values will affect AI output and email sender info.

---

#### 2.2 Social Media Data Extraction

- **Overview:**  
  This block retrieves social media data for each lead by calling Twitter and LinkedIn APIs via RapidAPI. It first obtains the Twitter user ID from the handle, then fetches tweets and LinkedIn posts.

- **Nodes Involved:**  
  - Get twitter ID  
  - Get tweets  
  - Exract and limit X (Twitter posts extraction)  
  - Get linkedin Posts  
  - Extract and limit Linkedin

- **Node Details:**

  - **Get twitter ID**  
    - *Type:* HTTP Request  
    - *Role:* Calls Twitter API to get user ID from Twitter handle.  
    - *Configuration:* Uses RapidAPI Twitter endpoint `/v2/user/by-username` with header authentication via RapidAPI key. Query parameter `username` is dynamically set from lead’s twitter_handler field.  
    - *Inputs:* Output from Set your company's variables  
    - *Outputs:* JSON containing Twitter user ID.  
    - *Edge Cases:* Twitter handle missing or invalid; API rate limits; authentication errors.

  - **Get tweets**  
    - *Type:* HTTP Request  
    - *Role:* Fetches recent tweets for the Twitter user ID obtained.  
    - *Configuration:* Calls `/v2/user/tweets` endpoint with userId from previous node. Uses same RapidAPI credentials.  
    - *Inputs:* Output from Get twitter ID  
    - *Outputs:* JSON array of tweets.  
    - *Edge Cases:* User with no tweets; API limits; malformed response.

  - **Exract and limit X**  
    - *Type:* Code (JavaScript)  
    - *Role:* Parses tweets JSON, extracts tweet text, limits to max 10 tweets.  
    - *Configuration:* Loops through tweets, extracts `full_text` from nested JSON, skips tweets without content.  
    - *Inputs:* Output from Get tweets  
    - *Outputs:* JSON object with array "Twitter tweets" limited to 10.  
    - *Edge Cases:* Tweets missing expected fields; empty tweet list.

  - **Get linkedin Posts**  
    - *Type:* HTTP Request  
    - *Role:* Fetches LinkedIn posts for the lead’s LinkedIn URL.  
    - *Configuration:* Calls Fresh LinkedIn Profile Data API endpoint `/get-profile-posts` with query parameter `linkedin_url` from lead data. Uses RapidAPI header authentication.  
    - *Inputs:* Output from Exract and limit X  
    - *Outputs:* JSON array of LinkedIn posts.  
    - *Edge Cases:* Invalid or private LinkedIn URLs; API limits; authentication errors.

  - **Extract and limit Linkedin**  
    - *Type:* Code (JavaScript)  
    - *Role:* Parses LinkedIn posts JSON, extracts title and text, limits to max 10 posts.  
    - *Configuration:* Loops through posts, extracts `article_title` and `text`.  
    - *Inputs:* Output from Get linkedin Posts  
    - *Outputs:* JSON object with array "linkedIn posts" limited to 10.  
    - *Edge Cases:* Posts missing expected fields; empty post list.

---

#### 2.3 AI-Powered Content Generation

- **Overview:**  
  This block uses OpenAI’s GPT-4o model to analyze the extracted social media data and generate a personalized email subject and cover letter in HTML format.

- **Nodes Involved:**  
  - Generate Subject and cover letter based on match  
  - Structured Output Parser  
  - OpenAI Chat Model

- **Node Details:**

  - **Generate Subject and cover letter based on match**  
    - *Type:* LangChain LLM Chain node  
    - *Role:* Defines the prompt for AI to generate email content based on company info, lead info, LinkedIn posts, and Twitter tweets.  
    - *Configuration:*  
      - Prompt includes company variables, lead name, and JSON stringified social media posts.  
      - Instructions specify to find a common activity and generate a JSON output with "subject" and "cover_letter" (HTML).  
      - Uses a system message: "You are a helpful Marketing assistant."  
      - Output parser enabled to expect structured JSON.  
    - *Inputs:* Output from Extract and limit Linkedin  
    - *Outputs:* Raw AI response JSON.  
    - *Edge Cases:* AI may generate malformed JSON; prompt may need tuning; API key limits or errors.

  - **Structured Output Parser**  
    - *Type:* LangChain Output Parser  
    - *Role:* Parses AI response to ensure it matches expected JSON schema with "subject" and "cover_letter".  
    - *Configuration:* Example JSON schema provided.  
    - *Inputs:* Output from OpenAI Chat Model  
    - *Outputs:* Parsed JSON with subject and cover_letter fields.  
    - *Edge Cases:* Parsing failure if AI response is malformed.

  - **OpenAI Chat Model**  
    - *Type:* LangChain LLM Chat Model  
    - *Role:* Executes the GPT-4o model call with the prompt from the chain node.  
    - *Configuration:* Model set to "gpt-4o", no additional options.  
    - *Credentials:* OpenAI API key required.  
    - *Inputs:* Output from Generate Subject and cover letter based on match (as prompt)  
    - *Outputs:* AI-generated response.  
    - *Edge Cases:* API rate limits, authentication errors, network timeouts.

---

#### 2.4 Email Dispatch and Progress Tracking

- **Overview:**  
  This block sends the generated personalized email to the lead and a copy to the sender, then updates the Google Sheet to mark the lead as processed.

- **Nodes Involved:**  
  - Send Cover letter and CC me  
  - Google Sheets

- **Node Details:**

  - **Send Cover letter and CC me**  
    - *Type:* Email Send node  
    - *Role:* Sends an email with the AI-generated subject and HTML cover letter to the lead’s email and CCs the sender.  
    - *Configuration:*  
      - Subject and HTML body dynamically set from AI output.  
      - To: lead email and sender email.  
      - From: static sender email (thomas@pollup.net).  
    - *Credentials:* SMTP credentials configured.  
    - *Inputs:* Output from Generate Subject and cover letter based on match  
    - *Outputs:* Email send status.  
    - *Edge Cases:* SMTP authentication errors; invalid email addresses; email delivery failures.

  - **Google Sheets**  
    - *Type:* Google Sheets node  
    - *Role:* Updates the "done" column in the Google Sheet for the processed lead to "X" indicating completion.  
    - *Configuration:*  
      - Matches rows by linkedin_url.  
      - Updates "done" field to "X".  
    - *Credentials:* Google Sheets OAuth2 credentials.  
    - *Inputs:* Output from Send Cover letter and CC me  
    - *Outputs:* Update confirmation.  
    - *Edge Cases:* Sheet access errors; row matching failures; concurrent updates.

---

### 3. Summary Table

| Node Name                          | Node Type                          | Functional Role                          | Input Node(s)                    | Output Node(s)                          | Sticky Note                                                                                                                  |
|-----------------------------------|-----------------------------------|----------------------------------------|---------------------------------|---------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Google Sheets Trigger             | Google Sheets Trigger             | Trigger on new/updated lead data       | None                            | If                                    | "Create a Google sheet with columns: linkedin_url, name, twitter_handler, email, done. Populate data except 'done' column."  |
| If                               | If                               | Check if lead is already processed     | Google Sheets Trigger           | Set your company's variables           |                                                                                                                              |
| Set your company's variables      | Set                              | Define company and sender info         | If                             | Get twitter ID                        | "Personalize here: set your name, company name, activity, and email used as sender."                                         |
| Get twitter ID                   | HTTP Request                     | Get Twitter user ID from handle        | Set your company's variables    | Get tweets                           | "Call RapidAPI Twitter API; subscribe and set header auth with x-rapidapi-key."                                              |
| Get tweets                      | HTTP Request                     | Fetch recent tweets for user ID        | Get twitter ID                 | Exract and limit X                   | "Call RapidAPI Twitter API; subscribe and set header auth with x-rapidapi-key."                                              |
| Exract and limit X              | Code                             | Extract and limit tweets to 10         | Get tweets                    | Get linkedin Posts                   |                                                                                                                              |
| Get linkedin Posts              | HTTP Request                     | Fetch LinkedIn posts for lead          | Exract and limit X            | Extract and limit Linkedin           | "Call RapidAPI Fresh LinkedIn Profile Data; subscribe and set header auth with x-rapidapi-key."                              |
| Extract and limit Linkedin      | Code                             | Extract and limit LinkedIn posts to 10| Get linkedin Posts            | Generate Subject and cover letter based on match |                                                                                                                              |
| Generate Subject and cover letter based on match | LangChain Chain LLM             | Generate personalized email content    | Extract and limit Linkedin     | Send Cover letter and CC me           | "Modify the AI prompt here to improve tone, style, or add examples."                                                        |
| OpenAI Chat Model               | LangChain LLM Chat Model         | Execute GPT-4o model call               | Generate Subject and cover letter based on match | Structured Output Parser             |                                                                                                                              |
| Structured Output Parser        | LangChain Output Parser          | Parse AI response to JSON schema       | OpenAI Chat Model             | Generate Subject and cover letter based on match |                                                                                                                              |
| Send Cover letter and CC me     | Email Send                      | Send personalized email to lead and CC sender | Generate Subject and cover letter based on match | Google Sheets                      |                                                                                                                              |
| Google Sheets                  | Google Sheets                    | Update lead row "done" status           | Send Cover letter and CC me    | None                                |                                                                                                                              |
| Sticky Note                    | Sticky Note                     | Instructional notes                     | None                          | None                                | Multiple sticky notes provide setup instructions and API subscription guidance (see detailed notes in section 5).            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheet:**  
   - Columns: linkedin_url, name, twitter_handler, email, done  
   - Populate with lead data; leave "done" column empty.

2. **Add Google Sheets Trigger node:**  
   - Configure with OAuth2 credentials.  
   - Set to poll the Google Sheet every minute on the specified range.

3. **Add If node:**  
   - Condition: Check if `done` field is empty (`""`).  
   - Connect Google Sheets Trigger output to If node input.

4. **Add Set node ("Set your company's variables"):**  
   - Define variables:  
     - your_company_name (e.g., "Pollup Data Services")  
     - your_company_activity (describe your company’s activity)  
     - your_email (sender email)  
     - your_name (sender name)  
   - Connect If node’s true output to this Set node.

5. **Add HTTP Request node ("Get twitter ID"):**  
   - URL: `https://twitter-api47.p.rapidapi.com/v2/user/by-username`  
   - Query parameter: `username` from lead’s twitter_handler field (`={{ $('Google Sheets Trigger').item.json.twitter_handler }}`)  
   - Authentication: Generic HTTP Header Auth with header name `x-rapidapi-key` and your RapidAPI key for Twitter API.  
   - Connect Set node output to this node.

6. **Add HTTP Request node ("Get tweets"):**  
   - URL: `https://twitter-api47.p.rapidapi.com/v2/user/tweets`  
   - Query parameter: `userId` from previous node’s JSON (`={{ $json.rest_id }}`)  
   - Same RapidAPI Twitter API credentials.  
   - Connect "Get twitter ID" output to this node.

7. **Add Code node ("Exract and limit X"):**  
   - JavaScript code to extract up to 10 tweets’ full_text from the API response.  
   - Connect "Get tweets" output to this node.

8. **Add HTTP Request node ("Get linkedin Posts"):**  
   - URL: `https://fresh-linkedin-profile-data.p.rapidapi.com/get-profile-posts`  
   - Query parameter: `linkedin_url` from lead data (`={{ $('Google Sheets Trigger').item.json.linkedin_url }}`)  
   - Authentication: Generic HTTP Header Auth with header name `x-rapidapi-key` and your RapidAPI key for LinkedIn API.  
   - Connect "Exract and limit X" output to this node.

9. **Add Code node ("Extract and limit Linkedin"):**  
   - JavaScript code to extract up to 10 LinkedIn posts (title and text).  
   - Connect "Get linkedin Posts" output to this node.

10. **Add LangChain Chain LLM node ("Generate Subject and cover letter based on match"):**  
    - Set prompt with company variables, lead name, LinkedIn posts, and Twitter tweets.  
    - Instructions to generate JSON with "subject" and "cover_letter" (HTML).  
    - Enable output parser.  
    - Connect "Extract and limit Linkedin" output to this node.

11. **Add LangChain LLM Chat Model node ("OpenAI Chat Model"):**  
    - Model: GPT-4o  
    - Credentials: OpenAI API key  
    - Connect "Generate Subject and cover letter based on match" output to this node.

12. **Add LangChain Output Parser node ("Structured Output Parser"):**  
    - JSON schema example with "subject" and "cover_letter" fields.  
    - Connect "OpenAI Chat Model" output to this node.  
    - Connect parser output back to "Generate Subject and cover letter based on match" node’s input for structured output.

13. **Add Email Send node ("Send Cover letter and CC me"):**  
    - To: lead email and your email (`={{ $('Google Sheets Trigger').item.json.email }}, {{ $('Set your company\'s variables').item.json.your_email }}`)  
    - From: your email (e.g., thomas@pollup.net)  
    - Subject: from AI output (`={{ $json.output.subject }}`)  
    - HTML body: from AI output (`={{ $json.output.cover_letter }}`)  
    - Credentials: SMTP or email service credentials  
    - Connect "Generate Subject and cover letter based on match" output to this node.

14. **Add Google Sheets node:**  
    - Operation: Update  
    - Sheet: same as trigger  
    - Match rows by linkedin_url  
    - Update "done" column to "X"  
    - Credentials: Google Sheets OAuth2  
    - Connect "Send Cover letter and CC me" output to this node.

15. **Test the workflow:**  
    - Add leads to Google Sheet with valid LinkedIn URLs and Twitter handles.  
    - Trigger workflow and verify emails sent and sheet updated.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Create a Google Sheet with columns: linkedin_url, name, twitter_handler, email, done. Populate data except the "done" column which should remain empty.                                                                        | Sticky Note near Google Sheets Trigger node                                                       |
| Personalize company variables: your name, company name, company activity (used to find matches), and your email (used as sender).                                                                                              | Sticky Note near "Set your company's variables" node                                             |
| Subscribe to RapidAPI Twitter API: https://rapidapi.com/restocked-gAGxip8a_/api/twitter-api47. Free tier allows scraping 500 tweets/month. Use Header Auth with header name "x-rapidapi-key".                                   | Sticky Note near "Get twitter ID" and "Get tweets" nodes                                         |
| Subscribe to RapidAPI Fresh LinkedIn Profile Data: https://rapidapi.com. Free tier allows scraping 100 profiles/month. Use Header Auth with header name "x-rapidapi-key".                                                        | Sticky Note near "Get linkedin Posts" node                                                       |
| Modify the AI prompt in the "Generate Subject and cover letter based on match" node to improve tone, style, or add examples following known marketing frameworks.                                                               | Sticky Note near AI prompt node                                                                  |
| Workflow created by Thomas Vie (thomas@pollup.net). Ideal for marketers and lead generation specialists to automate personalized outreach based on social media analysis.                                                        | Large sticky note with workflow description and credits                                         |
| OpenAI GPT-4o model is used for AI content generation; ensure API key is valid and usage limits are monitored.                                                                                                                 | AI nodes configuration                                                                           |
| Email sending requires SMTP or third-party email service credentials; ensure proper authentication and sender email setup to avoid delivery issues or spam filtering.                                                          | Email Send node configuration                                                                   |

---

This document provides a detailed, structured reference to understand, reproduce, and maintain the "Social Media Analysis and Automated Email Generation" workflow in n8n. It covers all nodes, their configurations, dependencies, and potential failure points, enabling both human users and AI agents to work effectively with this automation.