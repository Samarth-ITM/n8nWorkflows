Summarize Apple App Store Reviews with Pinecone, GPT-4 Mini & Slack Notifications

https://n8nworkflows.xyz/workflows/summarize-apple-app-store-reviews-with-pinecone--gpt-4-mini---slack-notifications-10666


# Summarize Apple App Store Reviews with Pinecone, GPT-4 Mini & Slack Notifications

### 1. Workflow Overview

This n8n workflow automates the collection, storage, summarization, and reporting of Apple App Store customer reviews for multiple mobile apps. It is designed to run two main schedules: a daily trigger to fetch and store the latest reviews, and a weekly trigger to generate and send a summarized report to a Slack channel.

The workflow is logically divided into the following blocks:

**1.1 Input Reception & Scheduling**  
Defines the apps to monitor and triggers the workflow on a daily basis for fetching reviews and weekly for generating summaries.

**1.2 Fetching and Processing App Store Reviews**  
Loops over each app, generates authentication tokens, calls the App Store Connect API to retrieve recent customer reviews, and filters the reviews by date (e.g., yesterday’s reviews).

**1.3 Storing Reviews in Pinecone Vector Store**  
Processes filtered reviews, converts them into embeddings via OpenAI, enriches them with metadata, and inserts them into a Pinecone vector database, maintaining a separate namespace per app with weekly cleanup.

**1.4 Weekly Summarization with AI Agent**  
Triggered weekly, this block loops over apps, retrieves stored reviews from Pinecone, uses OpenAI GPT-4 Mini to generate a summary categorizing positive and negative reviews with average star ratings, and posts the summary to a designated Slack channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Scheduling

- **Overview:**  
This block initializes the workflow by defining the list of Apple apps and scheduling workflow execution. It includes two triggers: one daily for fetching reviews, and one weekly for generating summaries.

- **Nodes Involved:**  
  - Daily Trigger1 (Schedule Trigger)  
  - Weekly Trigger (Schedule Trigger)  
  - Set App Store app ids (Set)  
  - Set App Store app ids1 (Set)  
  - Split Out6 (Split Out)  
  - Split Out8 (Split Out)  
  - Sticky Note9, Sticky Note10, Sticky Note11, Sticky Note16, Sticky Note17, Sticky Note18 (Informational Sticky Notes)

- **Node Details:**

  - **Daily Trigger1**  
    - Type: Schedule Trigger  
    - Role: Fires workflow once daily at 10:00 AM.  
    - Config: Hour set to 10, no other intervals.  
    - Connections: Starts workflow by triggering Set App Store app ids node.  
    - Edge cases: Timezone issues might affect trigger timing depending on server location.

  - **Weekly Trigger**  
    - Type: Schedule Trigger  
    - Role: Fires workflow weekly on Fridays at 11:00 AM to generate summaries.  
    - Config: Trigger on day 5 (Friday), hour 11.  
    - Connections: Starts workflow by triggering Set App Store app ids1 node.  
    - Edge cases: Same timezone considerations as above.

  - **Set App Store app ids / Set App Store app ids1**  
    - Type: Set  
    - Role: Defines an array of apps with their Apple App Store IDs and app names.  
    - Config: Assigns variable `apps` as an array of objects with keys `apple_app_store_app_id` and `app_name`.  
    - Connections: Feeds into Split Out6 / Split Out8 respectively.  
    - Edge cases: Proper app IDs must be inserted manually; missing or incorrect IDs will cause API request failures.

  - **Split Out6 / Split Out8**  
    - Type: Split Out  
    - Role: Splits the `apps` array into individual items for iterative processing.  
    - Config: Splits field `apps`.  
    - Connections: To Loop Over Items2 (daily flow) and Loop Over Items3 (weekly flow).  
    - Edge cases: Empty app list will cause downstream nodes to receive no data.

  - **Sticky Notes**  
    - Contain important contextual comments and instructions for users (e.g., defining app IDs, scheduling explanations).  
    - No direct functional role but critical for workflow understanding.

#### 2.2 Fetching and Processing App Store Reviews

- **Overview:**  
For each app, this block generates a JWT token for App Store Connect API access, retrieves customer reviews via HTTP requests with pagination, and filters reviews for the previous day.

- **Nodes Involved:**  
  - Loop Over Items2 (Split In Batches)  
  - JWT (JWT Authentication)  
  - HTTP Request - CUSTOMER REVIEWS (HTTP Request)  
  - Split Out9 (Split Out)  
  - Filter1 (Filter)  
  - Sticky Note12, Sticky Note13 (Informational Sticky Notes)

- **Node Details:**

  - **Loop Over Items2**  
    - Type: Split In Batches  
    - Role: Processes each app one by one to avoid API rate limits and manage resource usage.  
    - Config: Batch processing with continuation on errors (`onError: continueRegularOutput`).  
    - Connections: Outputs to JWT node on second output path.  
    - Edge cases: Long app list may prolong processing time; API throttling must be considered.

  - **JWT**  
    - Type: JWT Node  
    - Role: Generates a JWT token required for authenticating requests to App Store Connect API.  
    - Config: Claims include `iss` (Issuer ID), `aud`, `bid` (app bundle ID), and `exp` (expiry 10 minutes from current time).  
    - Credentials: Uses Apple Connect JWT Auth credentials setup with private key.  
    - Connections: Output to HTTP Request - CUSTOMER REVIEWS node.  
    - Edge cases: Expired tokens, incorrect claims, or credential misconfiguration will lead to authentication failures.

  - **HTTP Request - CUSTOMER REVIEWS**  
    - Type: HTTP Request  
    - Role: Calls App Store Connect API to fetch customer reviews for the given app ID, sorted by newest first.  
    - Config: URL constructed dynamically from app ID; uses bearer token authentication (alternative to JWT node). Pagination enabled to fetch up to 5 pages or until no next URL is present.  
    - Credentials: Bearer token configured separately (token placeholder shown).  
    - Connections: Output to Split Out9.  
    - Edge cases: API rate limits, network issues, invalid tokens, or incorrect app IDs can cause failures or empty results.

  - **Split Out9**  
    - Type: Split Out  
    - Role: Splits the returned array of reviews (`data` field) into individual review items.  
    - Config: Splits field `data`.  
    - Connections: Feeds into Filter1 node.  
    - Edge cases: Empty review data results in no further processing.

  - **Filter1**  
    - Type: Filter  
    - Role: Filters reviews to keep only those created exactly one day before the current date (yesterday’s reviews).  
    - Config: Compares `createdDate` attribute of each review to yesterday's date using ISO date format.  
    - Connections: Output goes to Pinecone Vector Store9 node for insertion.  
    - Edge cases: Timezone differences may affect filtering accuracy; reviews without `createdDate` attribute will be excluded.

  - **Sticky Notes**  
    - Explain JWT token creation requirements and HTTP request role.

#### 2.3 Storing Reviews in Pinecone Vector Store

- **Overview:**  
This block converts filtered reviews into vector embeddings using OpenAI, enriches them with metadata, and inserts them into Pinecone. It maintains namespaces per app and clears namespaces weekly to remove outdated data.

- **Nodes Involved:**  
  - Pinecone Vector Store9 (Vector Store Insert)  
  - Embeddings OpenAI9 (OpenAI Embeddings)  
  - Default Data Loader8 (Document Data Loader)  
  - Filter1 (Filter)  
  - Loop Over Items2 (implicit connection)  
  - Sticky Note15 (Informational Sticky Note)

- **Node Details:**

  - **Pinecone Vector Store9**  
    - Type: Vector Store (Pinecone)  
    - Role: Inserts review embeddings into Pinecone vector database.  
    - Config: Mode set to "insert". Namespace dynamically set per app ID with prefix `store_reviews_apple_app_store_`. Clears namespace when current day is Saturday to remove old entries.  
    - Credentials: Pinecone API credentials required.  
    - Connections: Receives embeddings from Embeddings OpenAI9; outputs back to Loop Over Items2 to continue processing.  
    - Edge cases: API quota exceeded, network issues, or credential errors may cause failures.

  - **Embeddings OpenAI9**  
    - Type: OpenAI Embeddings  
    - Role: Converts review text into vector embeddings for Pinecone.  
    - Credentials: OpenAI API key required.  
    - Connections: Outputs embeddings to Pinecone Vector Store9.  
    - Edge cases: API rate limits or invalid API keys may cause errors.

  - **Default Data Loader8**  
    - Type: Document Data Loader  
    - Role: Prepares review data for embeddings, including metadata such as star rating, date, territory, and review ID.  
    - Config: Extracts attributes from filtered review JSON and formats a descriptive text string.  
    - Connections: Outputs to Embeddings OpenAI9.  
    - Edge cases: Missing metadata fields in reviews could cause incomplete data embeddings.

  - **Filter1**  
    - Acts as a filter and source of filtered reviews for this block.

  - **Sticky Note15**  
    - Explains the namespace design and weekly cleanup logic.

#### 2.4 Weekly Summarization with AI Agent

- **Overview:**  
Triggered weekly, this block loops over apps, retrieves stored reviews from Pinecone, generates an AI-powered summary segregating positive and negative reviews and average star rating, then posts the summary to Slack.

- **Nodes Involved:**  
  - Loop Over Items3 (Split In Batches)  
  - Pinecone Vector Store12 (Vector Store Retrieve as Tool)  
  - Embeddings OpenAI12 (OpenAI Embeddings)  
  - OpenAI Chat Model10 (OpenAI GPT-4 Mini Model)  
  - AI Agent - Summariser (Langchain Agent)  
  - Send to Slack channel1 (Slack Node)  
  - Sticky Note7, Sticky Note19 (Informational Sticky Notes)

- **Node Details:**

  - **Loop Over Items3**  
    - Type: Split In Batches  
    - Role: Iterates over each app for summary generation, with error continuation enabled.  
    - Connections: Outputs to AI Agent - Summariser node.  
    - Edge cases: Long app list may slow workflow; failures in one app do not stop others.

  - **Pinecone Vector Store12**  
    - Type: Vector Store (Pinecone)  
    - Role: Retrieves up to 500 most relevant review vectors for the app from Pinecone.  
    - Config: Retrieval mode "retrieve-as-tool", namespace set dynamically per app ID with prefix `store_reviews_apple_app_store_`.  
    - Credentials: Pinecone API credentials.  
    - Connections: Outputs as a tool input to AI Agent - Summariser.  
    - Edge cases: No reviews stored for app results in empty retrieval.

  - **Embeddings OpenAI12**  
    - Type: OpenAI Embeddings  
    - Role: Provides embeddings for query processing within the AI Agent context.  
    - Credentials: OpenAI API key.  
    - Connections: Inputs to Pinecone Vector Store12 as embedding provider.  
    - Edge cases: API limits.

  - **OpenAI Chat Model10**  
    - Type: OpenAI Chat Completion (GPT-4 Mini)  
    - Role: Language model used by the AI Agent to generate the summary text.  
    - Credentials: OpenAI API key.  
    - Connections: AI language model input for AI Agent - Summariser.  
    - Edge cases: Model availability, rate limits.

  - **AI Agent - Summariser**  
    - Type: Langchain Agent  
    - Role: Creates a detailed summary of reviews including number processed, positive and negative highlights, and average star rating.  
    - Config: System message guides the agent’s behavior; prompt instructs summary format.  
    - Connections: Outputs summary text to Slack node.  
    - Edge cases: Summarization errors, malformed input data.

  - **Send to Slack channel1**  
    - Type: Slack Node  
    - Role: Sends the generated summary message to a specified Slack channel.  
    - Config: Channel ID statically defined, text set from AI Agent output.  
    - Credentials: Slack API OAuth2 credentials.  
    - Edge cases: Slack API errors, invalid channel ID.

  - **Sticky Notes**  
    - Indicate that this block posts a summary once a week and explain the summary generation logic.

---

### 3. Summary Table

| Node Name               | Node Type                      | Functional Role                                | Input Node(s)              | Output Node(s)             | Sticky Note                                                                                      |
|-------------------------|--------------------------------|-----------------------------------------------|----------------------------|----------------------------|------------------------------------------------------------------------------------------------|
| Daily Trigger1          | Schedule Trigger                | Triggers daily workflow run                     | —                          | Set App Store app ids       | Fetch reviews once a day                                                                        |
| Set App Store app ids   | Set                            | Defines Apple app IDs and names                 | Daily Trigger1             | Split Out6                 | Define app ids and names                                                                       |
| Split Out6              | Split Out                      | Splits app list into individual apps            | Set App Store app ids       | Loop Over Items2           | Loop over app ids (apps) in the list                                                           |
| Loop Over Items2        | Split In Batches               | Processes each app daily                         | Split Out6                 | JWT (main output)          |                                                                                                |
| JWT                    | JWT                            | Generates JWT token for App Store API auth      | Loop Over Items2           | HTTP Request - CUSTOMER REVIEWS | Generate a JWT token for Apple's App Store Connect API. Required fields: iss, bid, exp          |
| HTTP Request - CUSTOMER REVIEWS | HTTP Request                  | Fetches customer reviews from App Store Connect | JWT                        | Split Out9                 | HTTP call to fetch App Store reviews of the app                                                |
| Split Out9              | Split Out                      | Splits reviews array into single review items   | HTTP Request - CUSTOMER REVIEWS | Filter1                   |                                                                                                |
| Filter1                 | Filter                         | Filters reviews from yesterday                   | Split Out9                 | Default Data Loader8       | Fetch yesterday's reviews                                                                      |
| Default Data Loader8    | Document Data Loader           | Prepares review text and metadata for embeddings | Filter1                    | Embeddings OpenAI9         |                                                                                                |
| Embeddings OpenAI9      | OpenAI Embeddings              | Creates vector embeddings from review text       | Default Data Loader8       | Pinecone Vector Store9     |                                                                                                |
| Pinecone Vector Store9  | Pinecone Vector Store          | Inserts review embeddings into Pinecone          | Embeddings OpenAI9         | Loop Over Items2           | Store the previous day's reviews in the vector store. Clear namespace weekly.                   |
| Weekly Trigger          | Schedule Trigger               | Triggers weekly workflow run for summary         | —                          | Set App Store app ids1     | On Fridays at 11am                                                                             |
| Set App Store app ids1  | Set                            | Defines Apple app IDs and names for weekly flow  | Weekly Trigger             | Split Out8                 | Define app ids and names                                                                       |
| Split Out8              | Split Out                      | Splits app list into individual apps              | Set App Store app ids1     | Loop Over Items3           | Loop over app ids (apps) in the list                                                          |
| Loop Over Items3        | Split In Batches               | Processes each app weekly                         | Split Out8                 | AI Agent - Summariser      |                                                                                                |
| Pinecone Vector Store12 | Pinecone Vector Store          | Retrieves review vectors from Pinecone            | Loop Over Items3           | AI Agent - Summariser      |                                                                                                |
| Embeddings OpenAI12     | OpenAI Embeddings              | Embeddings provider for Pinecone retrieval        | —                          | Pinecone Vector Store12    |                                                                                                |
| OpenAI Chat Model10     | OpenAI Chat Completion (GPT-4 Mini) | Provides language model for summary generation    | —                          | AI Agent - Summariser      |                                                                                                |
| AI Agent - Summariser   | Langchain Agent                | Generates review summary with positive/negative split and average rating | Loop Over Items3, Pinecone Vector Store12, OpenAI Chat Model10 | Send to Slack channel1   | Generate Apple App Store review summary for the app                                           |
| Send to Slack channel1  | Slack                         | Posts summary message to Slack channel            | AI Agent - Summariser      | Loop Over Items3           | Post the summary to a Slack channel                                                           |
| Sticky Note (multiple)  | Sticky Note                   | Informational notes and instructions              | —                          | —                          | See individual sticky notes in workflow for content                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new n8n workflow.**

2. **Add a Schedule Trigger node (`Daily Trigger1`):**  
   - Type: Schedule Trigger  
   - Trigger once daily at 10:00 AM (set hour to 10).  
   - Connect its output to the next node.

3. **Add a Set node (`Set App Store app ids`):**  
   - Define a variable `apps` as an array of objects:  
     ```json
     [
       { "apple_app_store_app_id": "<Apple app id here>", "app_name": "<App name here>" },
       { "apple_app_store_app_id": "<Apple app id here>", "app_name": "<App name here>" }
     ]
     ```  
   - Connect output from Daily Trigger1 to this Set node.

4. **Add a Split Out node (`Split Out6`):**  
   - Configure to split field `apps` to output one app per item.  
   - Connect Set App Store app ids to Split Out6.

5. **Add a Split In Batches node (`Loop Over Items2`):**  
   - Set to batch process items (apps) one by one.  
   - Connect Split Out6 output to Loop Over Items2.

6. **Add a JWT node (`JWT`):**  
   - Configure JWT claims:  
     - `iss`: Your Apple Issuer ID  
     - `aud`: `"appstoreconnect-v1"`  
     - `bid`: Bundle ID of the app (from current item)  
     - `exp`: Current time + 10 minutes (use expression)  
   - Use Apple Connect JWT credentials (set up in n8n with private key).  
   - Connect Loop Over Items2 second output (for per-item processing) to JWT node.

7. **Add an HTTP Request node (`HTTP Request - CUSTOMER REVIEWS`):**  
   - URL: `https://api.appstoreconnect.apple.com/v1/apps/{{ $json.apple_app_store_app_id }}/customerReviews?sort=-createdDate`  
   - Authentication: HTTP Bearer using your Apple Connect Bearer token credentials.  
   - Enable pagination with:  
     - Pagination mode: `responseContainsNextURL`  
     - Next URL expression: `{{$response.body.links["next"]}}`  
     - Limit max 5 pages.  
   - Connect JWT output to this HTTP Request node.

8. **Add a Split Out node (`Split Out9`):**  
   - Split field `data` from HTTP Request response.  
   - Connect HTTP Request output to this node.

9. **Add a Filter node (`Filter1`):**  
   - Condition: Keep items where `createdDate` equals yesterday’s date (use expression to calculate).  
   - Connect Split Out9 output to Filter1.

10. **Add a Document Default Data Loader node (`Default Data Loader8`):**  
    - Prepare document text from review attributes: title, body, date, star rating, territory.  
    - Add metadata fields for star_rating, date, territory, review_id.  
    - Connect Filter1 output to this node.

11. **Add OpenAI Embeddings node (`Embeddings OpenAI9`):**  
    - No special config needed beyond credentials.  
    - Connect Default Data Loader8 output to Embeddings OpenAI9.

12. **Add Pinecone Vector Store node (`Pinecone Vector Store9`):**  
    - Mode: Insert  
    - Namespace: `store_reviews_apple_app_store_{{ $json.apple_app_store_app_id }}` (dynamic per app)  
    - Clear namespace once a week (e.g., if day is Saturday) using expression.  
    - Connect Embeddings OpenAI9 output to this node.  
    - Connect Pinecone Vector Store9 output back to Loop Over Items2 to continue batch.

13. **Add a Schedule Trigger node (`Weekly Trigger`):**  
    - Trigger weekly on Fridays at 11:00 AM.  
    - Connect output to next node.

14. **Add a Set node (`Set App Store app ids1`):**  
    - Same configuration as Set App Store app ids but used for the weekly flow.  
    - Connect Weekly Trigger to this node.

15. **Add a Split Out node (`Split Out8`):**  
    - Split field `apps`.  
    - Connect Set App Store app ids1 to Split Out8.

16. **Add a Split In Batches node (`Loop Over Items3`):**  
    - Batch process apps for summary generation.  
    - Connect Split Out8 to Loop Over Items3.

17. **Add Pinecone Vector Store node (`Pinecone Vector Store12`):**  
    - Mode: Retrieve as Tool  
    - Top K: 500  
    - Namespace: `store_reviews_apple_app_store_{{ $json.apple_app_store_app_id }}` (dynamic)  
    - Connect Loop Over Items3 output to this node.

18. **Add OpenAI Embeddings node (`Embeddings OpenAI12`):**  
    - Connect as embedding provider to Pinecone Vector Store12.

19. **Add OpenAI Chat Model node (`OpenAI Chat Model10`):**  
    - Model: gpt-4.1-mini  
    - Connect as language model to AI Agent - Summariser.

20. **Add Langchain Agent node (`AI Agent - Summariser`):**  
    - Configure with system message describing summarization role and instructions.  
    - Input text prompt to create summary with positive/negative groupings and average star rating.  
    - Connect Loop Over Items3, Pinecone Vector Store12 (as tool), and OpenAI Chat Model10 to this node.

21. **Add Slack node (`Send to Slack channel1`):**  
    - Configure to send message to a specified Slack channel by channel ID.  
    - Use Slack OAuth2 credentials.  
    - Connect AI Agent - Summariser output to Slack node.

22. **Connect Slack node output back to Loop Over Items3 to continue batch processing.**

23. **Add Sticky Notes for documentation and guidance as desired.**

24. **Set credentials:**  
    - Apple Connect JWT and Bearer credentials with valid keys and tokens.  
    - Pinecone API credentials with access to your Pinecone index.  
    - OpenAI API credentials with embedding and chat completion access.  
    - Slack API credentials with permission to post messages.

25. **Test and deploy the workflow.**

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow automates fetching and summarizing Apple App Store reviews, leveraging OpenAI and Pinecone for embeddings and summarization, and posting results on Slack. | Overview and branding from initial Sticky Note in workflow. |
| Apple App Store Connect API requires JWT tokens with issuer ID, bundle ID, and short expiry for authentication. | See Sticky Note12. |
| Pinecone namespaces are dynamically created per app and cleared weekly to manage storage size. | See Sticky Note15. |
| The AI summarizer splits reviews into positive and negative based on star ratings and includes average rating for insights. | See Sticky Note19. |
| Slack messages are posted to a dedicated channel with OAuth2 credentials. | Node Send to Slack channel1 configuration. |
| For detailed API docs: https://developer.apple.com/documentation/appstoreconnectapi | Apple App Store Connect API documentation. |
| Pinecone docs: https://docs.pinecone.io/ | Pinecone vector database documentation. |
| OpenAI docs: https://platform.openai.com/docs | OpenAI API documentation. |
| Slack API docs: https://api.slack.com/ | Slack API documentation. |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated n8n workflow. The processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.