Summarize Google Play Store Reviews with GPT-4-mini, Pinecone & Slack Alerts

https://n8nworkflows.xyz/workflows/summarize-google-play-store-reviews-with-gpt-4-mini--pinecone---slack-alerts-10605


# Summarize Google Play Store Reviews with GPT-4-mini, Pinecone & Slack Alerts

### 1. Workflow Overview

This workflow automates the process of retrieving Google Play Store reviews for multiple apps, summarizing the feedback using AI (GPT-4-mini), storing and managing review data in Pinecone (a vector database), and sending summarized insights to a Slack channel. It is designed for product teams, community managers, or stakeholders who want to monitor app sentiment and review trends efficiently without manually reading individual reviews.

The workflow is logically divided into the following blocks:

- **1.1 App Identification & Scheduling**  
  Defines the Google Play Store app bundle IDs and app names to monitor, and triggers workflow executions on a daily and weekly basis.

- **1.2 Fetching Reviews from Google Play Store**  
  Retrieves the latest user reviews for each app using Google API and a service account credential.

- **1.3 Filtering and Processing Reviews**  
  Filters reviews to only include those from the previous day, splits reviews into individual items for batch processing, and prepares data for storage.

- **1.4 Embedding & Storing Reviews in Pinecone**  
  Converts review texts into vector embeddings with OpenAI, stores them in Pinecone vector store, and manages the Pinecone namespace with weekly clearing.

- **1.5 Summarizing Reviews with AI Agent**  
  Uses an AI agent powered by GPT-4-mini to generate a comprehensive summary of positive and negative reviews, star rating breakdown, average rating, and total processed reviews.

- **1.6 Slack Notification**  
  Posts the generated summary to a specified Slack channel for team visibility.

---

### 2. Block-by-Block Analysis

#### 1.1 App Identification & Scheduling

**Overview:**  
Defines which apps to monitor by their bundle IDs and names. It also sets up triggers that run the workflow daily at 10 AM and weekly on Fridays at 11 AM.

**Nodes Involved:**  
- Set the bundle ids  
- Set app details  
- Daily Trigger  
- Weekly trigger  
- Sticky Notes (9, 5, 8, 10)

**Node Details:**

- **Set the bundle ids**  
  - Type: Set  
  - Role: Holds an array of Google Play app bundle IDs (e.g., "com.bundle.id1") to fetch reviews for.  
  - Config: Array field `apps` with bundle IDs as strings.  
  - Input: Daily Trigger  
  - Output: Split Out  
  - Edge cases: Empty or incorrect bundle IDs would result in failed API requests.

- **Set app details**  
  - Type: Set  
  - Role: Holds an array of objects each containing an app’s bundle ID (`app`) and friendly name (`app_name`).  
  - Used for labeling summaries.  
  - Input: Weekly trigger  
  - Output: Split Out1

- **Daily Trigger**  
  - Type: Schedule Trigger  
  - Role: Triggers workflow daily at 10 AM.  
  - Output: Set the bundle ids

- **Weekly trigger**  
  - Type: Schedule Trigger  
  - Role: Triggers workflow weekly on Fridays at 11 AM.  
  - Output: Set app details

- **Sticky Notes:**  
  - Provide guidance on defining app bundle ids, scheduling, and purpose of nodes.

---

#### 1.2 Fetching Reviews from Google Play Store

**Overview:**  
Fetches app reviews from the Google Play Store API using a Google Service Account. Supports pagination to retrieve multiple pages of reviews.

**Nodes Involved:**  
- Split Out  
- Loop Over Items  
- HTTP Request  
- Split Out4  
- Sticky Notes (1, 2, 10)

**Node Details:**

- **Split Out**  
  - Type: Split Out  
  - Role: Splits the array of app bundle IDs into individual items for processing.  
  - Input: Set the bundle ids  
  - Output: Loop Over Items

- **Loop Over Items**  
  - Type: Split In Batches  
  - Role: Processes each app bundle ID sequentially or in batches.  
  - Input: Split Out  
  - Output: HTTP Request  
  - Error Handling: Continues workflow if errors occur during API calls.

- **HTTP Request**  
  - Type: HTTP Request  
  - Role: Calls Google Play Developer API to fetch reviews for the given app bundle ID.  
  - Config:  
    - URL dynamically uses the current bundle ID.  
    - Pagination enabled with up to 10 pages, 10 requests max, 10ms interval.  
    - Auth via Google Service Account credential.  
  - Output: Split Out4 (splits reviews)

- **Split Out4**  
  - Type: Split Out  
  - Role: Splits fetched reviews array into individual review objects for filtering and processing.  
  - Input: HTTP Request  
  - Output: Filter

- **Sticky Notes:**  
  - Explain the use of Google Service Account and daily review fetching.

- **Edge cases:**  
  - API quota limits or permissions errors with Google Service Account.  
  - Pagination token missing or malformed causing incomplete data retrieval.

---

#### 1.3 Filtering and Processing Reviews

**Overview:**  
Filters reviews to only include those from the previous day and prepares metadata-enriched documents for embedding and storage.

**Nodes Involved:**  
- Filter  
- Default Data Loader  
- Pinecone Vector Store  
- Loop Over Items (link back)  
- Sticky Notes (3)

**Node Details:**

- **Filter**  
  - Type: Filter  
  - Role: Filters reviews to keep only those whose last modified date equals yesterday’s date.  
  - Condition: Compares review's lastModified timestamp (converted to ISO date) to yesterday’s date.  
  - Input: Split Out4  
  - Output: Pinecone Vector Store

- **Default Data Loader**  
  - Type: Document Default Data Loader (LangChain)  
  - Role: Prepares review data for vector embedding.  
  - Config:  
    - Metadata extracted includes star rating, date, app version, language, and review ID.  
    - Formats review text and metadata into a structured JSON string.  
  - Input: From Filter via embeddings node (Embeddings OpenAI)  
  - Output: Pinecone Vector Store (embedding input)

- **Pinecone Vector Store**  
  - Type: Vector Store (Pinecone)  
  - Role: Inserts embedded review vectors into Pinecone database.  
  - Config:  
    - Namespace dynamically named per app bundle ID.  
    - Clears namespace weekly on Saturdays (day 6).  
  - Input: Embeddings OpenAI (embedding output)  
  - Output: Loop Over Items (continues fetching next reviews or next app)

- **Sticky Notes:**  
  - Explain weekly clearing of old reviews in Pinecone to keep data fresh.

- **Edge cases:**  
  - Date parsing errors in filtering.  
  - Pinecone insertion failures due to network or credential issues.

---

#### 1.4 Embedding & Storing Reviews in Pinecone (Detailed)

**Overview:**  
Generates vector embeddings for reviews using OpenAI embeddings and stores them in Pinecone for efficient retrieval.

**Nodes Involved:**  
- Embeddings OpenAI  
- Pinecone Vector Store

**Node Details:**

- **Embeddings OpenAI**  
  - Type: OpenAI Embeddings Node (LangChain)  
  - Role: Converts text data of reviews into vector embeddings.  
  - Credentials: OpenAI API key configured.  
  - Input: Default Data Loader (document data)  
  - Output: Pinecone Vector Store (embedding input)  

- **Pinecone Vector Store**  
  - See details above.

- **Edge cases:**  
  - API rate limits or failures from OpenAI.  
  - Embedding input data malformed or missing required fields.

---

#### 1.5 Summarizing Reviews with AI Agent

**Overview:**  
Generates a human-readable summary of the app reviews using an AI agent powered by GPT-4-mini, including positive/negative highlights, star rating breakdown, and total reviews processed.

**Nodes Involved:**  
- Set app details (input for app name)  
- Split Out1  
- Loop Over Items1  
- AI Agent - Summary creator  
- OpenAI Chat Model1  
- Embeddings OpenAI1  
- Pinecone Vector Store1  
- Send to Slack channel  
- Sticky Notes (4, 6, 7)

**Node Details:**

- **Split Out1**  
  - Splits app details array into individual app objects.  
  - Input: Set app details  
  - Output: Loop Over Items1

- **Loop Over Items1**  
  - Processes each app’s data for summarization sequentially.  
  - Input: Split Out1  
  - Output: AI Agent - Summary creator

- **AI Agent - Summary creator**  
  - Type: LangChain Agent Node  
  - Role: Uses GPT-4-mini to generate a detailed summary based on retrieved reviews.  
  - Config:  
    - System message sets context explaining star ratings and review sentiment.  
    - Prompt instructs agent to include number of reviews, positive/negative summaries, star rating breakdown, and average rating.  
  - Input: Data retrieved from Pinecone Vector Store1 (retrieved review vectors) and app details.  
  - Output: Send to Slack channel

- **OpenAI Chat Model1**  
  - Language model GPT-4-mini used by AI Agent for text generation.  
  - Credentials: OpenAI API

- **Embeddings OpenAI1 & Pinecone Vector Store1**  
  - Used for retrieval-as-tool to fetch relevant review vectors for summarization.  
  - Pinecone namespace matches app bundle ID.  
  - Top 500 vectors retrieved.

- **Send to Slack channel**  
  - Sends generated summary text to a configured Slack channel using Slack Webhook.  
  - Config: Slack channel ID set, OAuth2 credential configured.

- **Sticky Notes:**  
  - Explain looping over apps for summary generation and posting.  
  - Emphasize Slack notification purpose.

- **Edge cases:**  
  - AI agent timeouts or errors.  
  - Slack API failures or misconfigured channel IDs.

---

#### 1.6 Slack Notification

**Overview:**  
Sends the AI-generated summary to a predefined Slack channel for team visibility.

**Nodes Involved:**  
- Send to Slack channel

**Node Details:**

- **Send to Slack channel**  
  - Type: Slack node  
  - Role: Posts message text to a Slack channel.  
  - Configuration: Uses a Slack OAuth2 credential, channel ID is set explicitly.  
  - Input: AI Agent - Summary creator output  
  - Output: None (end of workflow for that path)  
  - Edge cases: Slack API rate limits, invalid webhook or channel ID errors.

---

### 3. Summary Table

| Node Name               | Node Type                                   | Functional Role                                      | Input Node(s)                          | Output Node(s)                          | Sticky Note                                                                                      |
|-------------------------|---------------------------------------------|-----------------------------------------------------|--------------------------------------|---------------------------------------|-------------------------------------------------------------------------------------------------|
| Set the bundle ids       | Set                                         | Define list of Google Play app bundle IDs           | Daily Trigger                        | Split Out                             | Define the list of bundle ids of the Google Play Store apps you want to summarise reviews for... |
| Split Out               | Split Out                                   | Splits apps array into individual bundle IDs        | Set the bundle ids                   | Loop Over Items                      | Loop over bundle ids (apps) in the list                                                        |
| Loop Over Items         | Split In Batches                            | Processes each app bundle ID for review fetching    | Split Out                          | HTTP Request                         |                                                                                                |
| HTTP Request            | HTTP Request                               | Fetch reviews from Google Play Store API             | Loop Over Items                     | Split Out4                          | Fetch reviews from Google with a service account                                               |
| Split Out4              | Split Out                                   | Splits reviews array into individual review objects | HTTP Request                       | Filter                              |                                                                                                |
| Filter                  | Filter                                      | Filters reviews to only include yesterday’s reviews | Split Out4                        | Pinecone Vector Store               | Store the previous day's reviews in the vector store. Create a different namespace for each app |
| Default Data Loader     | Document Default Data Loader (LangChain)   | Prepares review text + metadata for embedding       | Filter                            | Embeddings OpenAI                   |                                                                                                |
| Embeddings OpenAI       | OpenAI Embeddings (LangChain)               | Creates vector embeddings for reviews                | Default Data Loader               | Pinecone Vector Store               |                                                                                                |
| Pinecone Vector Store   | Vector Store (Pinecone)                      | Inserts embedded vectors; manages namespace clearing | Embeddings OpenAI                | Loop Over Items                    |                                                                                                |
| Set app details         | Set                                         | Defines app bundle IDs and friendly names            | Weekly trigger                   | Split Out1                         | Define app bundle ids and names                                                                 |
| Split Out1              | Split Out                                   | Splits app details array into individual apps        | Set app details                  | Loop Over Items1                   | Loop over bundle ids (apps) in the list                                                        |
| Loop Over Items1        | Split In Batches                            | Processes each app for AI summarization              | Split Out1                     | AI Agent - Summary creator         |                                                                                                |
| Pinecone Vector Store1  | Vector Store (Pinecone)                      | Retrieves review vectors for summarization           | Embeddings OpenAI1              | AI Agent - Summary creator         |                                                                                                |
| Embeddings OpenAI1      | OpenAI Embeddings (LangChain)               | Embeds query or documents for retrieval              | Default Data Loader             | Pinecone Vector Store1             |                                                                                                |
| AI Agent - Summary creator | LangChain Agent Node                        | Generates summary text of reviews using GPT-4-mini  | Loop Over Items1, Pinecone Vector Store1 | Send to Slack channel          | Generate Google Play Store review summary for the app                                          |
| OpenAI Chat Model1      | OpenAI Chat Model (LangChain)                | Language model GPT-4-mini used by AI Agent           | AI Agent - Summary creator (AI languageModel) | AI Agent - Summary creator       |                                                                                                |
| Send to Slack channel   | Slack Node                                  | Sends summary text to Slack channel                   | AI Agent - Summary creator       | None                              | Post the summary to a Slack channel                                                             |
| Daily Trigger           | Schedule Trigger                            | Triggers workflow daily at 10 AM                      | None                           | Set the bundle ids                | Daily at 10am                                                                                   |
| Weekly trigger          | Schedule Trigger                            | Triggers workflow weekly on Fridays at 11 AM         | None                           | Set app details                  | On Fridays at 11am                                                                              |
| Sticky Note             | Sticky Note                                 | Multiple notes with instructions and explanations     | N/A                            | N/A                              | See section 1.1 and 1.2 for detailed notes                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Daily Trigger node**  
   - Type: Schedule Trigger  
   - Configure to run daily at 10:00 AM.

2. **Create a Weekly Trigger node**  
   - Type: Schedule Trigger  
   - Configure to run weekly on Friday at 11:00 AM.

3. **Create a Set node ("Set the bundle ids")**  
   - Connect input from Daily Trigger.  
   - Add an array field named `apps` with your Google Play Store bundle IDs as strings (e.g., ["com.bundle.id1", "com.bundle.id2"]).  

4. **Create a Set node ("Set app details")**  
   - Connect input from Weekly trigger.  
   - Define an array of objects with keys `app` (bundle ID) and `app_name` (friendly app name).

5. **Create a Split Out node ("Split Out")**  
   - Connect input from "Set the bundle ids".  
   - Configure to split the `apps` array into individual items.

6. **Create a Split In Batches node ("Loop Over Items")**  
   - Connect input from "Split Out".  
   - Default batch size (process one app at a time).  
   - Error handling: Continue on error.

7. **Create an HTTP Request node ("HTTP Request")**  
   - Connect input from "Loop Over Items".  
   - Configure URL: `https://androidpublisher.googleapis.com/androidpublisher/v3/applications/{{$json.apps}}/reviews`  
   - Enable pagination with:  
     - Pagination token parameter: `token` from response `tokenPagination.nextPageToken`  
     - Max requests: 10  
     - Request interval: 10ms  
     - Stop when `nextPageToken` is null  
   - Authentication: Google Service Account credential with access to Google Play Developer API.

8. **Create a Split Out node ("Split Out4")**  
   - Connect input from HTTP Request.  
   - Split `reviews` array into individual review items.

9. **Create a Filter node ("Filter")**  
   - Connect input from "Split Out4".  
   - Filter condition: Include only reviews where review lastModified date equals yesterday's date. Use expression comparing review date converted to ISO date and `$today.minus({days:1})`.

10. **Create a Document Default Data Loader node ("Default Data Loader")**  
    - Connect input from Filter.  
    - Configure metadata extraction fields: star_rating, date, app_version, language, review_id.  
    - Format review text and metadata into JSON string for embedding.

11. **Create an OpenAI Embeddings node ("Embeddings OpenAI")**  
    - Connect input from "Default Data Loader".  
    - Use your OpenAI API credentials.

12. **Create a Pinecone Vector Store node ("Pinecone Vector Store")**  
    - Connect input from "Embeddings OpenAI".  
    - Mode: Insert  
    - Namespace: `google_play_store_reviews_{{ $('Split Out').item.json.apps }}` (dynamic per app)  
    - Clear namespace weekly on Saturday (day=6) via expression.  
    - Use your Pinecone API credentials.

13. **Connect Pinecone Vector Store output back to "Loop Over Items"** to continue processing other apps.

14. **Create a Split Out node ("Split Out1")**  
    - Connect input from "Set app details".  
    - Split the app array into individual items.

15. **Create a Split In Batches node ("Loop Over Items1")**  
    - Connect input from "Split Out1".  
    - Batch size 1, process each app for summarization.

16. **Create an OpenAI Embeddings node ("Embeddings OpenAI1")**  
    - Configure similarly to previous embeddings node; used for retrieval.

17. **Create a Pinecone Vector Store node ("Pinecone Vector Store1")**  
    - Mode: Retrieve-as-tool  
    - Top K: 500  
    - Namespace: `google_play_store_reviews_{{ $('Loop Over Items1').item.json.app }}`  
    - Use Pinecone API credentials.

18. **Create an OpenAI Chat Model node ("OpenAI Chat Model1")**  
    - Model: GPT-4.1-mini  
    - Use OpenAI API credentials.

19. **Create an AI Agent node ("AI Agent - Summary creator")**  
    - Input: Connect to "Loop Over Items1" and "Pinecone Vector Store1" (retrieved vectors).  
    - Configure system message with context about star ratings and sentiment.  
    - Prompt: Ask agent to create a summary including positive/negative review highlights, star rating breakdown, average rating, and review count.  
    - Set error handling to continue on error.

20. **Create a Slack node ("Send to Slack channel")**  
    - Connect input from AI Agent output.  
    - Configure Slack channel ID and OAuth2 credentials.  
    - Map message text to AI Agent output.

21. **Connect AI Agent output (main) to Slack node input.**

22. **Add Sticky Notes** for documentation and guidance on each block (optional but recommended).

23. **Activate workflow and test** with your app bundle IDs, credentials, and Slack channel.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                | Context or Link                                                                                                                       |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| This workflow automates fetching, summarizing, and sharing Google Play Store app reviews using Pinecone for vector storage, OpenAI GPT-4-mini for summarization, and Slack for notifications.                                                                                                  | Overview of workflow capabilities                                                                                                |
| Bundle IDs can be found in the Google Play Store URL, e.g., `com.tripledot.blockbash` in `https://play.google.com/store/apps/details?id=com.tripledot.blockbash`                                                                                                                                | Guidance on finding app bundle IDs                                                                                                |
| The workflow clears Pinecone namespaces weekly to prevent stale data accumulation, keeping review storage up to date.                                                                                                                                                                        | Data management best practice                                                                                                    |
| The AI summarization separates positive and negative reviews based on star ratings and provides statistical breakdowns for comprehensive insights.                                                                                                                                          | AI agent prompt design                                                                                                           |
| Slack node requires OAuth2 credentials with permission to post to the selected channel. Verify the channel ID is correct to avoid posting errors.                                                                                                                                             | Slack integration note                                                                                                           |
| Google Service Account used must have permission to access Google Play Developer API for the targeted apps.                                                                                                                                                                                 | Google API access requirement                                                                                                   |
| Pagination in HTTP Request node ensures retrieval of multiple pages of reviews, with limit set to avoid excessive requests.                                                                                                                                                                   | Pagination details                                                                                                               |
| AI Agent node uses LangChain integration within n8n, which requires correct version compatibility (n8n >= certain version). Verify node versions if errors are encountered.                                                                                                                    | Version-specific note                                                                                                           |

---

This completes the detailed documentation and reproduction instructions for the "Summarize Google Play Store Reviews with GPT-4-mini, Pinecone & Slack Alerts" workflow.