Daily News Digest: Summarize RSS Feeds with OpenAI and Deliver to WhatsApp

https://n8nworkflows.xyz/workflows/daily-news-digest--summarize-rss-feeds-with-openai-and-deliver-to-whatsapp-4709


# Daily News Digest: Summarize RSS Feeds with OpenAI and Deliver to WhatsApp

### 1. Workflow Overview

This workflow automates the daily aggregation, summarization, and delivery of news articles fetched from multiple RSS feeds. It is designed to produce a concise digest of recent news highlights and deliver this summary to a WhatsApp number using an external API.

The workflow is structured into the following logical blocks:

- **1.1 Scheduled Trigger and RSS Feed Collection:** Initiates the workflow on a daily schedule and fetches news articles from four distinct RSS feeds.
- **1.2 Data Normalization and Merging:** Standardizes fields across feeds and merges the multiple RSS feed outputs into a unified collection.
- **1.3 Filtering and Limiting Recent Articles:** Filters articles published within the last day and limits the dataset to the 10 most recent items.
- **1.4 Content Aggregation and AI Summarization:** Aggregates key fields and sends the top articles’ content to OpenAI for generating a concise summary.
- **1.5 Delivery to WhatsApp:** Transmits the AI-generated summary to a specified WhatsApp number via an HTTP API call.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger and RSS Feed Collection

**Overview:**  
This block initiates the workflow once daily at 3 AM and reads news articles from four configured RSS feeds, each potentially providing different content sources.

**Nodes Involved:**  
- Schedule Trigger  
- My RSS 01  
- My RSS 02  
- My RSS 03  
- My RSS 04

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Starts workflow execution daily at 3:00 AM.  
  - *Configuration:* Trigger set at hour 3, recurring every day.  
  - *Input/Output:* No input; outputs trigger event to all RSS feed nodes.  
  - *Edge Cases:* Workflow won't execute if the n8n instance is down at trigger time.

- **My RSS 01, My RSS 02, My RSS 03, My RSS 04**  
  - *Type:* RSS Feed Read  
  - *Role:* Fetches articles from configured RSS feed URLs.  
  - *Configuration:* Each node points to a specific RSS feed URL:
    - My RSS 01: https://uxdesign.cc/feed  
    - My RSS 02: https://www.technologyreview.com/topic/artificial-intelligence/feed/  
    - My RSS 03: https://the-decoder.com/feed/  
    - My RSS 04: https://www.theverge.com/rss/index.xml  
  - *Error Handling:* Set to continue on error to avoid workflow interruption if a feed is down or inaccessible.  
  - *Always Output Data:* Enabled to ensure downstream processes receive data even if partially empty.  
  - *Input/Output:* Receives trigger from Schedule Trigger; outputs RSS items to respective "Edit Fields" nodes.  
  - *Edge Cases:* Possible network issues, invalid feed format, or feed downtime.

---

#### 2.2 Data Normalization and Merging

**Overview:**  
Standardizes the important fields (title, publication date, content) from each RSS feed’s raw data to a uniform schema, then merges all feeds into a single data stream.

**Nodes Involved:**  
- Edit Fields3 (for My RSS 01)  
- Edit Fields (for My RSS 02)  
- Edit Fields1 (for My RSS 03)  
- Edit Fields2 (for My RSS 04)  
- Merge

**Node Details:**

- **Edit Fields (and variants)**  
  - *Type:* Set  
  - *Role:* Extracts and renames key fields to a unified format: `title`, `pubDate`, and `conteudo` (content).  
  - *Configuration:*  
    - Assigns `title` from RSS item title field.  
    - Assigns `pubDate` from RSS item publication date.  
    - Assigns `conteudo` from either `content:encoded`, `contentSnippet`, or `content` depending on feed.  
  - *Input/Output:* Receives RSS feed items; outputs cleaned and standardized items to Merge node.  
  - *Edge Cases:* Some feeds use different field names for content; missing content fields could cause empty `conteudo`.  

- **Merge**  
  - *Type:* Merge  
  - *Role:* Combines outputs from the four Edit Fields nodes into a single stream.  
  - *Configuration:* Set to accept 4 input connections, merging all inputs.  
  - *Input/Output:* Inputs from all Edit Fields nodes; outputs combined data to Filter node.  
  - *Edge Cases:* If any feed outputs no data, merge still proceeds with available inputs.

---

#### 2.3 Filtering and Limiting Recent Articles

**Overview:**  
Filters articles to only those published within the last day and limits the dataset to the 10 most recent articles for summarization.

**Nodes Involved:**  
- Filter  
- Limit  
- Sticky Note (about Limit)

**Node Details:**

- **Filter**  
  - *Type:* Filter  
  - *Role:* Filters articles where `pubDate` is after "today minus 1 day," effectively keeping only articles from the last 24 hours.  
  - *Configuration:* DateTime condition comparing article `pubDate` to current date minus 1 day.  
  - *Input/Output:* Input from Merge; outputs filtered articles to Limit node.  
  - *Edge Cases:* Articles with missing or invalid `pubDate` may be excluded or cause errors. Timezone considerations apply.

- **Limit**  
  - *Type:* Limit  
  - *Role:* Restricts the number of articles forwarded for summarization to 10.  
  - *Configuration:* Max items set to 10.  
  - *Input/Output:* Input from Filter; outputs limited dataset to Aggregate node.  
  - *Sticky Note:* "## About Limit - limits the analysis to the 10 most recent articles" applies here.  
  - *Edge Cases:* If fewer than 10 articles are present, all are passed through.

---

#### 2.4 Content Aggregation and AI Summarization

**Overview:**  
Aggregates key fields from filtered articles and sends the top articles to OpenAI to generate a concise summary of main news highlights.

**Nodes Involved:**  
- Aggregate  
- OpenAI  
- Sticky Note4 (System message context)

**Node Details:**

- **Aggregate**  
  - *Type:* Aggregate  
  - *Role:* Aggregates all item data into a single JSON object containing arrays of `title`, `pubDate`, and `conteudo`.  
  - *Configuration:* Includes specified fields: `title`, `pubDate`, `conteudo`. Aggregates all item data.  
  - *Input/Output:* Input from Limit; outputs aggregated object to OpenAI node.  
  - *Edge Cases:* Empty input leads to empty aggregation, which may affect the downstream prompt.

- **OpenAI**  
  - *Type:* OpenAI (Langchain)  
  - *Role:* Sends article titles and content to OpenAI’s assistant model to generate a concise summary of the top 3-5 main highlights.  
  - *Configuration:*  
    - Prompt text includes a system message: "I've received news articles via RSS. Please analyze them and provide a concise summary of the top 3 to 5 main highlights..."  
    - Dynamically inserts up to 10 articles' titles and content into the prompt using expressions.  
    - Uses a specific assistantId for the OpenAI resource.  
  - *Credentials:* OpenAI API credentials must be configured.  
  - *Input/Output:* Input from Aggregate; outputs summary text to WhatsApp sending node.  
  - *Sticky Note:* "## System message (change if you want)..." provides context for the prompt.  
  - *Edge Cases:*  
    - If fewer than 10 articles exist, expressions accessing `$json.data[n]` could fail or be empty.  
    - API call failures due to authentication, rate limits, or network issues.  
    - Model response may vary in quality.

---

#### 2.5 Delivery to WhatsApp

**Overview:**  
Sends the AI-generated summary message to a specified WhatsApp number through an HTTP POST request to an external API.

**Nodes Involved:**  
- Send resum to Whatsapp  
- No Operation, do nothing  
- Sticky Note2 (about delivery options)

**Node Details:**

- **Send resum to Whatsapp**  
  - *Type:* HTTP Request  
  - *Role:* Posts the summary message to a WhatsApp API endpoint.  
  - *Configuration:*  
    - URL: Placeholder "yoururlapi.com"; should be replaced with actual WhatsApp API endpoint.  
    - Method: POST  
    - Body parameters: `phone` (fixed value, e.g., your phone number), `message` (OpenAI summary output).  
    - Authentication: HTTP Header Authentication using predefined credentials named "Header Zapi".  
  - *Credentials:* HTTP Header Auth credential configured with necessary headers for WhatsApp API.  
  - *Input/Output:* Input from OpenAI; outputs to No Operation node.  
  - *Sticky Note:* "## About Resum - you can send to your whatsapp api, or email, drive..."  
  - *Edge Cases:*  
    - Invalid or missing credentials causing auth failures.  
    - API endpoint downtime or incorrect URL.  
    - Message size or format restrictions from WhatsApp API.

- **No Operation, do nothing**  
  - *Type:* No Operation  
  - *Role:* Acts as an endpoint to gracefully finish the workflow after sending the message.  
  - *Input/Output:* Input from HTTP Request node; no output.  

---

### 3. Summary Table

| Node Name              | Node Type                 | Functional Role                          | Input Node(s)                    | Output Node(s)                | Sticky Note                                                                                      |
|------------------------|---------------------------|----------------------------------------|---------------------------------|------------------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger       | Schedule Trigger          | Starts workflow daily at 3 AM           | None                            | My RSS 01, My RSS 02, My RSS 03, My RSS 04 | ## About Trgger - Set trigger how you wish                                                        |
| My RSS 01              | RSS Feed Read             | Fetches articles from UX Design RSS     | Schedule Trigger                | Edit Fields3                  | ## About RSS - You can add how many RSS you want                                                |
| My RSS 02              | RSS Feed Read             | Fetches articles from Technology Review | Schedule Trigger                | Edit Fields                  | ## About RSS - You can add how many RSS you want                                                |
| My RSS 03              | RSS Feed Read             | Fetches articles from The Decoder       | Schedule Trigger                | Edit Fields1                 | ## About RSS - You can add how many RSS you want                                                |
| My RSS 04              | RSS Feed Read             | Fetches articles from The Verge          | Schedule Trigger                | Edit Fields2                 | ## About RSS - You can add how many RSS you want                                                |
| Edit Fields3           | Set                       | Normalizes RSS 01 fields                 | My RSS 01                      | Merge                        |                                                                                                |
| Edit Fields            | Set                       | Normalizes RSS 02 fields                 | My RSS 02                      | Merge                        |                                                                                                |
| Edit Fields1           | Set                       | Normalizes RSS 03 fields                 | My RSS 03                      | Merge                        |                                                                                                |
| Edit Fields2           | Set                       | Normalizes RSS 04 fields                 | My RSS 04                      | Merge                        |                                                                                                |
| Merge                  | Merge                     | Combines all normalized feeds            | Edit Fields3, Edit Fields, Edit Fields1, Edit Fields2 | Filter                       |                                                                                                |
| Filter                 | Filter                    | Keeps articles from last 24 hours        | Merge                         | Limit                        |                                                                                                |
| Limit                  | Limit                     | Limits to 10 most recent articles        | Filter                        | Aggregate                    | ## About Limit - limits the analysis to the 10 most recent articles                             |
| Aggregate              | Aggregate                 | Aggregates article fields for AI input   | Limit                         | OpenAI                       |                                                                                                |
| OpenAI                 | OpenAI (Langchain)        | Generates summary from aggregated articles| Aggregate                     | Send resum to Whatsapp       | ## System message (change if you want): I've received 10 news articles via RSS...               |
| Send resum to Whatsapp | HTTP Request              | Sends summary to WhatsApp API            | OpenAI                       | No Operation, do nothing     | ## About Resum - you can send to your whatsapp api, or email, drive...                          |
| No Operation, do nothing| No Operation             | Ends workflow gracefully                  | Send resum to Whatsapp         | None                        |                                                                                                |
| Sticky Note            | Sticky Note               | Notes on Limit node                       | None                         | None                        | ## About Limit - limits the analysis to the 10 most recent articles                             |
| Sticky Note1           | Sticky Note               | Notes on RSS feeds                        | None                         | None                        | ## About RSS - You can add how many RSS you want                                               |
| Sticky Note2           | Sticky Note               | Notes on summary delivery options        | None                         | None                        | ## About Resum - you can send to your whatsapp api, or email, drive...                         |
| Sticky Note3           | Sticky Note               | Notes on trigger                          | None                         | None                        | ## About Trgger - Set trigger how you wish                                                    |
| Sticky Note4           | Sticky Note               | Notes on system prompt for OpenAI        | None                         | None                        | ## System message (change if you want): I've received 10 news articles via RSS...               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Set to run daily at 3 AM.

2. **Create four RSS Feed Read nodes:**  
   - Name: My RSS 01, 02, 03, 04.  
   - Configure URLs:  
     - My RSS 01: https://uxdesign.cc/feed  
     - My RSS 02: https://www.technologyreview.com/topic/artificial-intelligence/feed/  
     - My RSS 03: https://the-decoder.com/feed/  
     - My RSS 04: https://www.theverge.com/rss/index.xml  
   - Set error handling to continue on error.  
   - Enable “Always Output Data.”  
   - Connect Schedule Trigger output to all four RSS nodes.

3. **Create four Set nodes to normalize fields:**  
   - Name accordingly: Edit Fields3 (My RSS 01), Edit Fields (My RSS 02), Edit Fields1 (My RSS 03), Edit Fields2 (My RSS 04).  
   - For each node:  
     - Assign `title` to `{{$json.title}}`  
     - Assign `pubDate` to `{{$json.pubDate}}`  
     - Assign `conteudo` to the appropriate content field:  
       - My RSS 01: `{{$json.content}}`  
       - My RSS 02: `{{$json['content:encoded']}}`  
       - My RSS 03: `{{$json.contentSnippet}}`  
       - My RSS 04: `{{$json.contentSnippet}}`  
   - Connect each RSS Feed Read node output to its respective Set node.

4. **Create a Merge node:**  
   - Set number of inputs to 4.  
   - Connect all four Set nodes to the Merge node.

5. **Add a Filter node:**  
   - Condition: Filter items where `pubDate` is after `{{$today.minus({ days: 1 })}}`.  
   - Connect Merge node output to Filter node input.

6. **Add a Limit node:**  
   - Max Items: 10.  
   - Connect Filter node output to Limit node input.

7. **Add an Aggregate node:**  
   - Aggregate all item data including fields: `title`, `pubDate`, `conteudo`.  
   - Connect Limit node output to Aggregate node input.

8. **Add an OpenAI node (Langchain integration):**  
   - Configure OpenAI API credentials.  
   - Set resource to ‘assistant’, specify assistantId as per your setup.  
   - Text parameter:  
     ```
     I've received news articles via RSS. Please analyze them and provide a concise summary of the top 3 to 5 main highlights. My goal is to get a quick overview of what's most relevant in these articles.
     1.{{ $json.data[0].title }}{{ $json.data[0].conteudo }}
     2.{{ $json.data[1].title }}{{ $json.data[1].conteudo }}
     3.{{ $json.data[2].title }}{{ $json.data[2].conteudo }}
     4.{{ $json.data[3].title }}{{ $json.data[3].conteudo }}
     5.{{ $json.data[4].title }}{{ $json.data[4].conteudo }}
     6.{{ $json.data[5].title }}{{ $json.data[5].conteudo }}
     7.{{ $json.data[6].title }}{{ $json.data[6].conteudo }}
     8.{{ $json.data[7].title }}{{ $json.data[7].conteudo }}
     9.{{ $json.data[8].title }}{{ $json.data[8].conteudo }}
     10.{{ $json.data[9].title }}{{ $json.data[9].conteudo }}
     ```
   - Connect Aggregate node output to OpenAI node input.

9. **Add an HTTP Request node to send the summary to WhatsApp:**  
   - URL: Set your WhatsApp API endpoint URL.  
   - Method: POST.  
   - Authentication: HTTP Header Auth with credentials configured for your WhatsApp API.  
   - Body parameters:  
     - `phone`: Your target phone number as string.  
     - `message`: Expression `{{$json.output}}` to send OpenAI’s summary.  
   - Connect OpenAI node output to HTTP Request node input.

10. **Add a No Operation node:**  
    - Connect HTTP Request node output to No Operation node to end workflow gracefully.

11. **Optional: Add Sticky Notes for documentation:**  
    - Notes about limits, RSS feeds, trigger, system message, and delivery as per workflow comments.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                      |
|--------------------------------------------------------------------------------------------------|------------------------------------|
| You can add as many RSS feeds as desired to scale the news sources.                              | Workflow flexibility                |
| Set the trigger time as preferred to control when the daily digest runs.                         | Schedule Trigger configuration     |
| The system message in the OpenAI prompt can be customized to alter summary style or details.     | Sticky Note4 content                |
| The summary can be sent not only to WhatsApp but also via email, cloud storage, or other APIs.  | Sticky Note2 content                |
| Ensure OpenAI API credentials are correctly configured to avoid authentication failures.         | OpenAI node credential requirement |
| WhatsApp API endpoint and authentication must be valid and tested to guarantee message delivery. | HTTP Request node credential setup |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.