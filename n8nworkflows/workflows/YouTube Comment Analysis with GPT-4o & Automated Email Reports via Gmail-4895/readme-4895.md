YouTube Comment Analysis with GPT-4o & Automated Email Reports via Gmail

https://n8nworkflows.xyz/workflows/youtube-comment-analysis-with-gpt-4o---automated-email-reports-via-gmail-4895


# YouTube Comment Analysis with GPT-4o & Automated Email Reports via Gmail

### 1. Workflow Overview

This workflow automates the analysis of YouTube video comments using OpenAI's GPT-4o model and sends summarized reports via Gmail. It is designed for content creators, marketers, or analysts who want to gain instant, AI-driven insights into viewer feedback and respond with professional email reports. The workflow is structured into four main logical blocks:

1.1 **Input Reception and Filtering**  
- Detects new YouTube videos added to a Google Sheet  
- Filters videos requiring processing based on a status flag  

1.2 **YouTube Data Retrieval**  
- Fetches video metadata (title, channel info, etc.)  
- Retrieves top 100 comments sorted by relevance  

1.3 **AI Comment Analysis and Data Processing**  
- Processes raw comment data (extraction, sentiment, stats)  
- Sends processed comments to GPT-4o via LangChain for analysis  
- Formats AI results into professional HTML for email  

1.4 **Email Dispatch and Status Update**  
- Sends the formatted report by Gmail  
- Updates the video’s status in Google Sheets to prevent duplicate runs  

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Filtering

**Overview:**  
This block triggers when new video IDs are added to a Google Sheet and filters those videos that have a status of "Pending" to be processed.

**Nodes Involved:**  
- Pick Video Ids from Google sheet  
- If  
- Limit  

**Node Details:**  

- **Pick Video Ids from Google sheet**  
  - *Type:* Google Sheets Trigger  
  - *Role:* Watches a Google Sheet for new or updated rows containing YouTube video IDs. Polls every minute.  
  - *Config:* Watch on a specific spreadsheet and sheet; triggers on any data change.  
  - *Inputs:* External Google Sheet data  
  - *Outputs:* New rows to process  
  - *Failures:* Connection/authentication errors with Google Sheets, polling delays  
  - *Notes:* Starting point of the workflow  

- **If**  
  - *Type:* If node  
  - *Role:* Filters videos to process only those with a "Pending" status in the sheet row.  
  - *Config:* Condition checks if `status == "Pending"`  
  - *Inputs:* Rows from Google Sheet trigger  
  - *Outputs:* Passes only filtered rows forward  
  - *Failures:* Expression evaluation errors if field missing or malformed data  

- **Limit**  
  - *Type:* Limit node  
  - *Role:* Restricts processing to one video at a time to avoid API overload.  
  - *Config:* Limit set to 1 item  
  - *Inputs:* Filtered rows from If node  
  - *Outputs:* Single row forwarded  
  - *Failures:* None typical unless input empty  

---

#### 2.2 YouTube Data Retrieval

**Overview:**  
This block collects metadata and comments from the YouTube API for the video selected in the previous block.

**Nodes Involved:**  
- Set Video Details  
- Get Youtube Video Details  
- Get Youtube Video Comments  

**Node Details:**  

- **Set Video Details**  
  - *Type:* Set node  
  - *Role:* Prepares parameters such as video ID and maximum comments count (100) for API calls.  
  - *Config:* Sets variables `videoId` (from input) and `maxComments` to 100  
  - *Inputs:* Single video row from Limit node  
  - *Outputs:* Passes configured parameters  
  - *Failures:* Data missing video ID or incorrect formatting  

- **Get Youtube Video Details**  
  - *Type:* YouTube node  
  - *Role:* Fetches detailed video metadata including title, channel, publish date, etc.  
  - *Config:* Uses YouTube API with video ID from Set Video Details  
  - *Inputs:* Parameters from Set Video Details  
  - *Outputs:* Video metadata JSON  
  - *Failures:* API quota exceeded, invalid video ID, network errors  

- **Get Youtube Video Comments**  
  - *Type:* HTTP Request node  
  - *Role:* Retrieves top 100 comments ordered by relevance using YouTube Data API v3 (via REST endpoint)  
  - *Config:* HTTP GET request with query parameters set for video ID, max results, order, text format  
  - *Inputs:* Video ID from video details  
  - *Outputs:* Raw comments data  
  - *Failures:* API rate limits, quota issues, malformed requests  

---

#### 2.3 AI Comment Analysis and Data Processing

**Overview:**  
Processes raw comments, extracts meaningful data, performs basic sentiment calculations, limits to 50 comments for efficiency, then sends this data to GPT-4o for in-depth analysis.

**Nodes Involved:**  
- Prepare Comments Data  
- AI Agent  
- Azure OpenAI Chat Model  
- Prepare HTML for Email  

**Node Details:**  

- **Prepare Comments Data**  
  - *Type:* Code (JavaScript)  
  - *Role:* Extracts comment text, calculates statistics (e.g., comment count, sentiment score), filters top 50 comments to optimize AI processing.  
  - *Config:* Custom JS script parsing raw YouTube comment JSON  
  - *Inputs:* Raw comments from HTTP Request node  
  - *Outputs:* Structured comments ready for AI  
  - *Failures:* Parsing errors, unexpected data formats  

- **Azure OpenAI Chat Model**  
  - *Type:* LangChain Azure OpenAI Chat Model node  
  - *Role:* Configures connection to Azure OpenAI service, specifying GPT-4o model for AI analysis  
  - *Config:* Credentials for Azure OpenAI, model specification (GPT-4o)  
  - *Inputs:* Receives prompt and input data from AI Agent node  
  - *Outputs:* AI-generated insights and analysis text  
  - *Failures:* Authentication issues, quota limits, model unavailability  

- **AI Agent**  
  - *Type:* LangChain Agent node  
  - *Role:* Uses GPT-4o model to analyze prepared comments and generate insights, summary, or sentiment analysis based on a customizable prompt.  
  - *Config:* Linked to Azure OpenAI Chat Model node, prompt templates customizable to different use cases  
  - *Inputs:* Processed comments from Prepare Comments Data; AI language model input from Azure OpenAI Chat Model  
  - *Outputs:* AI analysis text data  
  - *Failures:* Prompt errors, API timeouts, unexpected AI outputs  

- **Prepare HTML for Email**  
  - *Type:* Code (JavaScript)  
  - *Role:* Formats AI analysis and statistics into a styled HTML email ready for dispatch, includes tables, highlights, and branding style.  
  - *Config:* Custom JS generating clean HTML email markup  
  - *Inputs:* AI analysis output from AI Agent  
  - *Outputs:* HTML email content  
  - *Failures:* Malformed HTML, missing data fields  

---

#### 2.4 Email Dispatch and Status Update

**Overview:**  
Sends the prepared email report via Gmail and updates the Google Sheet status to "Mail Sent" to avoid duplicate processing.

**Nodes Involved:**  
- Gmail Account Configuration  
- Update Status on Google Sheet  

**Node Details:**  

- **Gmail Account Configuration**  
  - *Type:* Gmail node  
  - *Role:* Sends the formatted HTML email report to a specified recipient using Gmail OAuth2 credentials.  
  - *Config:* Requires OAuth2 credentials for Gmail; sender and recipient email addresses configured; email subject and body set from previous node output  
  - *Inputs:* HTML email content from Prepare HTML for Email  
  - *Outputs:* Confirmation of email sent  
  - *Failures:* Authentication errors, quota limits on sending emails, invalid email format  
  - *Notes:* Replace placeholder email addresses with actual recipients for production  

- **Update Status on Google Sheet**  
  - *Type:* Google Sheets node  
  - *Role:* Updates the row corresponding to the processed video to set status as "Mail Sent"  
  - *Config:* Spreadsheet and sheet defined; update row by video ID or unique key  
  - *Inputs:* Confirmation from Gmail node and original video row data  
  - *Outputs:* Updated row confirmation  
  - *Failures:* Google API rate limits, permission errors, row not found  

---

### 3. Summary Table

| Node Name                     | Node Type                      | Functional Role                                  | Input Node(s)                  | Output Node(s)                | Sticky Note                                                                                  |
|-------------------------------|--------------------------------|-------------------------------------------------|-------------------------------|------------------------------|----------------------------------------------------------------------------------------------|
| Workflow Overview             | Sticky Note                    | Documentation placeholder                        |                               |                              |                                                                                              |
| Trigger Documentation         | Sticky Note                    | Documentation placeholder                        |                               |                              |                                                                                              |
| YouTube API Section           | Sticky Note                    | Documentation placeholder                        |                               |                              |                                                                                              |
| AI Analysis Documentation     | Sticky Note                    | Documentation placeholder                        |                               |                              |                                                                                              |
| Email & Status Update         | Sticky Note                    | Documentation placeholder                        |                               |                              |                                                                                              |
| Data Processing Note          | Sticky Note                    | Documentation placeholder                        |                               |                              |                                                                                              |
| Pick Video Ids from Google sheet | Google Sheets Trigger          | Triggers on new videos added to sheet            |                               | If                           | Triggers on new YouTube videos added to spreadsheet. Polls every minute for changes         |
| If                           | If                             | Filters videos with status "Pending"             | Pick Video Ids from Google sheet | Limit                        | Filters rows to process only videos with 'Pending' status                                   |
| Limit                        | Limit                          | Limits processing to 1 item at a time            | If                            | Set Video Details            | Limits processing to 1 item at a time to prevent API overload                               |
| Set Video Details            | Set                            | Prepares video ID and max comments limit         | Limit                         | Get Youtube Video Details    | Prepares video ID and sets max comments limit (100)                                         |
| Get Youtube Video Details    | YouTube                        | Fetches video metadata                            | Set Video Details             | Get Youtube Video Comments   | Fetches video metadata including title, channel name, and other details                     |
| Get Youtube Video Comments   | HTTP Request                   | Retrieves top 100 comments ordered by relevance  | Get Youtube Video Details     | Prepare Comments Data        | Retrieves top 100 comments ordered by relevance using YouTube API                           |
| Prepare Comments Data        | Code                           | Processes raw comments and stats                  | Get Youtube Video Comments    | AI Agent                    | Processes raw comments: extracts text, calculates stats, performs basic sentiment analysis, limits to 50 comments for AI |
| Azure OpenAI Chat Model      | LangChain Azure OpenAI Chat Model | Azure OpenAI GPT-4o model configuration          |                               | AI Agent (ai_languageModel) |                                                                                              |
| AI Agent                    | LangChain Agent                | Uses GPT-4o to analyze comments and generate insights | Prepare Comments Data; Azure OpenAI Chat Model | Prepare HTML for Email       | Uses OpenAI GPT-4o to analyze comments and generate insights. Customize the prompt for different analysis needs |
| Prepare HTML for Email       | Code                           | Formats AI analysis into HTML email               | AI Agent                    | Gmail Account Configuration  | Converts AI analysis into formatted HTML email with statistics, insights, and professional styling |
| Gmail Account Configuration  | Gmail                          | Sends formatted analysis report via Gmail        | Prepare HTML for Email       | Update Status on Google Sheet | Sends formatted analysis report via Gmail. Update SENDER_EMAIL_ADDRESS with actual recipient |
| Update Status on Google Sheet | Google Sheets                  | Updates video status to "Mail Sent"                | Gmail Account Configuration  |                              | Updates video status to 'Mail Sent' to prevent duplicate processing                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger Node**  
   - Type: Google Sheets Trigger  
   - Configure to watch your target spreadsheet and sheet where YouTube video IDs are added  
   - Set polling interval to 1 minute  
   - Output: New or updated rows  

2. **Add If Node**  
   - Type: If  
   - Condition: Check if `status` field in the row equals "Pending"  
   - Connect input from Google Sheets Trigger  
   - Output: Pass only rows where status is "Pending"  

3. **Add Limit Node**  
   - Type: Limit  
   - Set limit to 1 item  
   - Connect input from If node  
   - Purpose: Process one video at a time to prevent API throttling  

4. **Add Set Node (Set Video Details)**  
   - Type: Set  
   - Map the video ID from input data to a parameter `videoId`  
   - Set a static parameter `maxComments` to 100  
   - Connect input from Limit node  

5. **Add YouTube Node (Get Youtube Video Details)**  
   - Type: YouTube  
   - Configure credentials with YouTube API key or OAuth  
   - Set operation to get video details by `videoId` (from Set node)  
   - Connect input from Set Video Details node  

6. **Add HTTP Request Node (Get Youtube Video Comments)**  
   - Type: HTTP Request  
   - Configure to call YouTube Data API v3 `/commentThreads` endpoint  
   - Query parameters: videoId, maxResults=100, order=relevance, textFormat=plainText  
   - Set authentication with your YouTube API key  
   - Connect input from Get Youtube Video Details node  

7. **Add Code Node (Prepare Comments Data)**  
   - Type: Code (JavaScript)  
   - Script to parse raw comments JSON, extract text, calculate basic sentiment and stats, select top 50 comments  
   - Connect input from Get Youtube Video Comments node  

8. **Add LangChain Azure OpenAI Chat Model Node**  
   - Type: LangChain Azure OpenAI Chat Model  
   - Configure credentials for Azure OpenAI service  
   - Specify GPT-4o model  
   - No direct input connection; this node is linked as AI language model for the Agent node  

9. **Add LangChain Agent Node (AI Agent)**  
   - Type: LangChain Agent  
   - Link to Azure OpenAI Chat Model node as AI language model  
   - Configure prompt template to analyze comments and generate insights  
   - Connect input from Prepare Comments Data node  
   - Output: AI analysis text  

10. **Add Code Node (Prepare HTML for Email)**  
    - Type: Code (JavaScript)  
    - Script formats AI analysis and stats into HTML email content with professional styling  
    - Connect input from AI Agent node  

11. **Add Gmail Node (Gmail Account Configuration)**  
    - Type: Gmail  
    - Configure OAuth2 credentials for Gmail account  
    - Set email recipient and sender addresses (replace placeholders with actual emails)  
    - Subject: e.g., "YouTube Comment Analysis Report for [Video Title]"  
    - Body: Use HTML content from Prepare HTML for Email node  
    - Connect input from Prepare HTML for Email node  

12. **Add Google Sheets Node (Update Status on Google Sheet)**  
    - Type: Google Sheets  
    - Configure to update the `status` field to "Mail Sent" for the processed video row  
    - Connect input from Gmail node  

**Connect nodes in the order described above to form the processing chain.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                               |
|----------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------|
| Replace `SENDER_EMAIL_ADDRESS` in Gmail node with actual sending and recipient email addresses before production use.           | Gmail Account Configuration node               |
| Customize AI prompt in the LangChain Agent node to tailor analysis to specific needs (sentiment, summary, action items).       | AI Agent node                                  |
| Google API quotas and YouTube Data API rate limits may affect workflow performance; use Limit node to mitigate overload.        | Limit node                                     |
| For Gmail OAuth2 setup, refer to Google’s OAuth 2.0 documentation to properly configure credentials in n8n.                     | https://developers.google.com/identity/protocols/oauth2 |
| The workflow processes one video at a time to comply with API limits and avoid hitting quota ceilings.                           | Limit node notes                               |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.