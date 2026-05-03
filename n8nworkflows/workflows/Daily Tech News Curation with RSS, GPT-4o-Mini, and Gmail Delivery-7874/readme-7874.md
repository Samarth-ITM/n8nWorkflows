Daily Tech News Curation with RSS, GPT-4o-Mini, and Gmail Delivery

https://n8nworkflows.xyz/workflows/daily-tech-news-curation-with-rss--gpt-4o-mini--and-gmail-delivery-7874


# Daily Tech News Curation with RSS, GPT-4o-Mini, and Gmail Delivery

### 1. Workflow Overview

This workflow automates the daily curation and delivery of a technology news newsletter focused on artificial intelligence and related tech trends. It pulls articles from multiple premium RSS feeds, processes and balances the content to avoid source bias, leverages GPT-4o-Mini AI to create a professional HTML newsletter, and sends it via Gmail every day at 8 AM UTC.

The workflow logically divides into the following blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow daily at a set time.
- **1.2 RSS Feed Configuration and Retrieval:** Defines RSS source URLs and fetches articles concurrently.
- **1.3 Article Filtering and Cleaning:** Applies source balancing and standardizes article data.
- **1.4 Aggregation and AI Newsletter Generation:** Combines articles and uses GPT-4o-Mini to create an HTML newsletter.
- **1.5 Email Delivery:** Sends the generated newsletter through Gmail using OAuth2 authentication.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

**Overview:**  
Starts the workflow automatically every day at 8 AM UTC to enable consistent newsletter delivery.

**Nodes Involved:**  
- Daily Newsletter Trigger

**Node Details:**  

- **Daily Newsletter Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Initiates workflow execution daily at a configured hour.  
  - *Configuration:* Set to trigger at 8 AM UTC daily. Options allow changing trigger hour and adding timezone or weekday constraints.  
  - *Inputs:* None (trigger node)  
  - *Outputs:* Connects to "Configure RSS Sources" node.  
  - *Edge Cases:* Misconfiguration of time or timezone can cause incorrect send times.  
  - *Notes:* Provides easy scheduling customization.

---

#### 1.2 RSS Feed Configuration and Retrieval

**Overview:**  
Defines an array of premium tech news RSS feed URLs, splits them for parallel processing, and fetches articles from each feed, continuing processing on individual feed failures.

**Nodes Involved:**  
- Configure RSS Sources  
- Split RSS URLs  
- Fetch RSS Articles

**Node Details:**  

- **Configure RSS Sources**  
  - *Type:* Set  
  - *Role:* Stores the list of RSS feed URLs to be used.  
  - *Configuration:* An array of 10+ URLs from top tech sources like TechCrunch AI, The Verge AI, MIT Tech Review, Wired AI, NY Times Tech, BBC Tech, and others.  
  - *Inputs:* From trigger node  
  - *Outputs:* To "Split RSS URLs"  
  - *Edge Cases:* URLs must be active and properly formatted RSS feeds; outdated or invalid URLs will cause fetch errors.

- **Split RSS URLs**  
  - *Type:* SplitOut  
  - *Role:* Breaks the array of URLs into individual executions for concurrent processing.  
  - *Configuration:* Splits on the "urls" array field.  
  - *Inputs:* From "Configure RSS Sources"  
  - *Outputs:* To "Fetch RSS Articles"  
  - *Edge Cases:* Large number of feeds may increase execution time and resource usage.

- **Fetch RSS Articles**  
  - *Type:* RSS Feed Read  
  - *Role:* Retrieves articles from each RSS feed URL.  
  - *Configuration:* URL dynamically set per execution from split outputs. Continues on errors to avoid workflow failure if a feed is down.  
  - *Inputs:* From "Split RSS URLs"  
  - *Outputs:* To "Filter Articles"  
  - *Edge Cases:* Possible network issues, feed format changes, or empty feeds; "continue on error" ensures resilience.

---

#### 1.3 Article Filtering and Cleaning

**Overview:**  
Balances article sources to avoid dominance by any single domain, limits articles per source to five, and standardizes article fields for further processing.

**Nodes Involved:**  
- Filter Articles (Code)  
- Clean Data (Set)

**Node Details:**  

- **Filter Articles**  
  - *Type:* Code (JavaScript)  
  - *Role:* Groups articles by domain, limits to max 5 per domain, and flattens the result for balanced coverage.  
  - *Configuration:* Custom JS code parsing article URLs to extract domains, enforcing article count limits per source.  
  - *Inputs:* From "Fetch RSS Articles" (all articles)  
  - *Outputs:* To "Clean Data"  
  - *Edge Cases:* Articles without valid URLs default to 'unknown' domain; potential for slight imbalance if many unknowns.

- **Clean Data**  
  - *Type:* Set  
  - *Role:* Maps raw article data to a standardized structure with fields: title, content snippet, link, and publication date.  
  - *Configuration:* Assignments rename and map fields:  
    - title → title  
    - contentSnippet → content  
    - link → link  
    - pubDate → pubDate  
  - *Inputs:* From "Filter Articles"  
  - *Outputs:* To "Combine Articles"  
  - *Edge Cases:* Missing fields in some articles may result in empty values.

---

#### 1.4 Aggregation and AI Newsletter Generation

**Overview:**  
Aggregates all filtered articles into a single dataset and invokes the GPT-4o-Mini model to analyze and generate an HTML newsletter with top 10 curated AI/tech articles.

**Nodes Involved:**  
- Combine Articles  
- AI Newsletter Creator

**Node Details:**  

- **Combine Articles**  
  - *Type:* Aggregate  
  - *Role:* Combines all individual articles into one array for bulk AI processing.  
  - *Configuration:* Aggregates all input items into a single JSON array under `data`.  
  - *Inputs:* From "Clean Data"  
  - *Outputs:* To "AI Newsletter Creator"  
  - *Edge Cases:* Large article volumes may impact performance or token limits.

- **AI Newsletter Creator**  
  - *Type:* OpenAI (GPT)  
  - *Role:* Uses GPT-4o-Mini to analyze articles and produce a clean HTML newsletter focused on trending AI/tech topics.  
  - *Configuration:*  
    - Model: gpt-4o-mini  
    - Prompt: Instructs AI to create an HTML newsletter with article title, summary, and link for top 10 articles based on relevance and trending topics.  
    - Inputs: Injects combined articles JSON as part of the prompt.  
  - *Credentials:* Requires OpenAI API key configured.  
  - *Inputs:* From "Combine Articles"  
  - *Outputs:* To "Send Newsletter"  
  - *Edge Cases:* API rate limits, token limits, or malformed JSON input can cause errors.

---

#### 1.5 Email Delivery

**Overview:**  
Sends the AI-generated HTML newsletter to a specified recipient via Gmail using OAuth2 authentication.

**Nodes Involved:**  
- Send Newsletter

**Node Details:**  

- **Send Newsletter**  
  - *Type:* Gmail  
  - *Role:* Sends the HTML content as an email with a dynamic subject including the current date.  
  - *Configuration:*  
    - Recipient email: configurable (default "your.email@example.com")  
    - Message body: HTML content from AI node output  
    - Subject: "Daily AI Tech Digest - [weekday, month day, year]" formatted dynamically  
    - Options: Attribution disabled for clean email  
  - *Credentials:* Gmail OAuth2 setup required with API enabled and OAuth client credentials.  
  - *Inputs:* From "AI Newsletter Creator"  
  - *Outputs:* None (final node)  
  - *Edge Cases:* Authentication failures, Gmail API limits, or invalid recipient address.

---

### 3. Summary Table

| Node Name              | Node Type                   | Functional Role                           | Input Node(s)          | Output Node(s)        | Sticky Note                                                                                                   |
|------------------------|-----------------------------|-----------------------------------------|------------------------|-----------------------|--------------------------------------------------------------------------------------------------------------|
| Daily Newsletter Trigger| Schedule Trigger            | Initiates daily workflow execution      | -                      | Configure RSS Sources  | ⏰ Triggers daily at 8 AM UTC. Customize trigger hour, timezone, weekdays.                                    |
| Configure RSS Sources   | Set                         | Defines RSS feed URLs array              | Daily Newsletter Trigger| Split RSS URLs         | 📡 Premium tech news RSS feeds from 10 sources including TechCrunch AI, The Verge AI, Wired AI, etc.          |
| Split RSS URLs          | SplitOut                    | Splits array into individual URLs for parallel fetching | Configure RSS Sources | Fetch RSS Articles     | 🔄 Splits URL array for parallel processing. Creates individual execution paths for each RSS feed URL.        |
| Fetch RSS Articles      | RSS Feed Read               | Fetches articles from each RSS URL       | Split RSS URLs          | Filter Articles        | 📰 Fetches articles with title, snippet, URL, date. Continues on errors to prevent single feed failures.      |
| Filter Articles         | Code                        | Balances articles per source domain      | Fetch RSS Articles      | Clean Data             | ⚖️ Limits max 5 articles per domain to ensure diverse coverage. Groups by domain, then flattens results.      |
| Clean Data             | Set                         | Standardizes article fields               | Filter Articles         | Combine Articles       | 🧹 Maps article fields to title, content snippet, link, and publication date.                                 |
| Combine Articles        | Aggregate                   | Aggregates all articles into single dataset | Clean Data             | AI Newsletter Creator  | 📊 Combines filtered articles from all sources for AI processing.                                            |
| AI Newsletter Creator   | OpenAI (GPT)                | Generates HTML newsletter from articles  | Combine Articles        | Send Newsletter        | 🤖 AI-powered curation using GPT-4o-Mini. Creates top-10 article HTML newsletter focusing on trending AI topics. Requires OpenAI API key. |
| Send Newsletter         | Gmail                       | Sends HTML newsletter via Gmail          | AI Newsletter Creator   | -                      | 📧 Sends HTML email with dynamic date subject. Requires Gmail OAuth2 credentials.                             |
| Sticky Note             | Sticky Note                 | Documentation and workflow overview      | -                      | -                      | # AI Tech News Aggregator & Newsletter. Setup steps and features overview.                                   |
| Sticky Note1            | Sticky Note                 | Customization tips and workflow notes    | -                      | -                      | ## Workflow customization options: timing, content focus, recipients, article limits.                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 8 AM UTC (triggerAtHour: 8)  
   - No inputs  
   - Connect output to next node.

2. **Create Set Node for RSS Sources**  
   - Type: Set  
   - Add a field named `urls` of type Array  
   - Assign it an array of RSS feed URLs (see list below)  
     ```
     [
      "https://techcrunch.com/tag/artificial-intelligence/feed/",
      "https://www.theverge.com/artificial-intelligence/rss/index.xml",
      "https://www.technologyreview.com/feed/",
      "https://www.wired.com/feed/category/science/artificial-intelligence/latest/rss",
      "https://venturebeat.com/category/ai/feed/",
      "https://www.zdnet.com/topic/artificial-intelligence/rss.xml",
      "https://towardsdatascience.com/feed",
      "https://rss.nytimes.com/services/xml/rss/nyt/Technology.xml",
      "https://www.theguardian.com/uk/technology/rss",
      "https://feeds.bbci.co.uk/news/technology/rss.xml"
     ]
     ```  
   - Connect the Schedule Trigger's output here.

3. **Create SplitOut Node to split URLs**  
   - Type: SplitOut  
   - Field to split: `urls` (the array from previous node)  
   - Connect Set node output here.

4. **Create RSS Feed Read Node to fetch articles**  
   - Type: RSS Feed Read  
   - URL: use expression `{{$json["urls"]}}` to take one URL per execution  
   - Set "Continue on Fail" to true (to avoid workflow failure on feed errors)  
   - Connect SplitOut output here.

5. **Create Code Node to filter articles by domain**  
   - Type: Code (JavaScript)  
   - Paste code that:  
     - Iterates over all articles  
     - Extracts domain from article link  
     - Limits maximum 5 articles per domain  
     - Returns flattened list of articles  
   - Input from RSS Feed Read node.

6. **Create Set Node to clean and standardize data**  
   - Type: Set  
   - Create fields mapped as:  
     - `title` = `{{$json["title"]}}`  
     - `content` = `{{$json["contentSnippet"]}}`  
     - `link` = `{{$json["link"]}}`  
     - `pubDate` = `{{$json["pubDate"]}}`  
   - Connect output of Code node here.

7. **Create Aggregate Node to combine all articles**  
   - Type: Aggregate  
   - Aggregate mode: Aggregate all item data into one array  
   - Connect Set node output here.

8. **Create OpenAI Node for AI newsletter creation**  
   - Type: OpenAI (GPT) node (requires n8n Langchain package or OpenAI integration)  
   - Model: gpt-4o-mini  
   - Prompt: Instruct AI to analyze the combined articles JSON and create an HTML newsletter with top 10 relevant AI/tech articles including title, summary, and link. Use clean HTML starting with `<!DOCTYPE html>`.  
   - Provide the combined articles as JSON string in the prompt using expression: `{{ JSON.stringify($json.data) }}`  
   - Connect Aggregate node output here.  
   - Configure OpenAI API credentials.

9. **Create Gmail Node to send email**  
   - Type: Gmail  
   - Recipient: set to desired email address (e.g., your.email@example.com)  
   - Subject: `Daily AI Tech Digest - {{ $now.toFormat('EEEE, MMMM d, yyyy') }}` (dynamic date)  
   - Message content: Use the AI node's output field containing the newsletter HTML (e.g., `{{$json["message"]["content"]}}`)  
   - Disable attribution option for clean email  
   - Connect OpenAI node output here.  
   - Configure Gmail OAuth2 credentials with enabled API and client credentials.

10. **Optional: Add Sticky Notes**  
    - Add documentation notes as Sticky Note nodes describing the workflow, setup instructions, and customization tips.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| The workflow relies on premium quality RSS feeds from leading tech publications to ensure top-tier content. | See "Configure RSS Sources" node for URLs. |
| OpenAI GPT-4o-Mini model is used to balance cost and performance for newsletter generation. | Requires valid OpenAI API key in credentials. |
| Gmail API must be enabled and OAuth2 credentials configured for sending emails via Gmail node. | Google Cloud Console Gmail API setup. |
| The workflow continues processing even if some RSS feeds fail, increasing resilience. | "Fetch RSS Articles" node configured to continue on error. |
| The AI prompt can be customized to focus on different tech topics or languages by editing the prompt in "AI Newsletter Creator". | See Sticky Note1 for customization tips. |
| Schedule trigger can be adjusted for time or multiple daily triggers to increase newsletter frequency. | Change "triggerAtHour" in "Daily Newsletter Trigger". |

---

**Disclaimer:**  
The text provided derives exclusively from an automated workflow created with n8n, an integration and automation tool. This workflow strictly complies with content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.