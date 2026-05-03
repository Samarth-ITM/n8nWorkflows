Create LinkedIn Contributions with AI and Notify Users On Slack

https://n8nworkflows.xyz/workflows/create-linkedin-contributions-with-ai-and-notify-users-on-slack-2491


# Create LinkedIn Contributions with AI and Notify Users On Slack

### 1. Workflow Overview

This workflow automates the weekly discovery and engagement with LinkedIn advice articles by generating unique AI-powered contributions and sharing them via Slack and a NocoDB database. It is designed for professionals and marketers who want to consistently build thought leadership on LinkedIn with minimal manual effort.

The workflow is logically divided into these blocks:

- **1.1 Trigger & Initialization**  
  Initiates the workflow weekly and sets the topic for Google search queries.

- **1.2 Discover LinkedIn Advice Articles**  
  Performs a Google search, extracts LinkedIn article URLs, and splits them for individual processing.

- **1.3 Filter & Merge Unique Articles**  
  Merges newly found article URLs with existing LinkedIn contributions from the NocoDB database to avoid duplicates.

- **1.4 Retrieve and Extract Article Content**  
  Fetches each LinkedIn article's HTML content and extracts key data: article title, topics, and existing contributions.

- **1.5 Generate AI Contributions**  
  Uses an AI model to create unique, thoughtful advice for each topic in the articles.

- **1.6 Post Contributions**  
  Posts the AI-generated advice to a Slack channel and records it in the NocoDB database.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Trigger & Initialization

**Overview:**  
This block triggers the workflow on a schedule (every Monday at 8:00 AM) or manually and sets the search topic for article discovery.

**Nodes Involved:**  
- Schedule Trigger Every Monday, @ 08:00am  
- When clicking ‘Test workflow’  
- Set Topic for Google search

**Node Details:**

- **Schedule Trigger Every Monday, @ 08:00am**  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow automatically on a weekly schedule.  
  - Configuration: Set to trigger every Monday at 8:00 AM.  
  - Inputs: None  
  - Outputs: Triggers subsequent nodes (Get all LinkedIn contributions, Set Topic)  
  - Failures: Rare, but possible if n8n server is down.  
  - Notes: Frequency can be customized.

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual execution for testing.  
  - Inputs: None  
  - Outputs: Same as schedule trigger.  
  - Failures: None.

- **Set Topic for Google search**  
  - Type: Set  
  - Role: Defines the topic string for Google search queries (default: "Paid Advertising").  
  - Configuration: Sets string field “Topic” with a default value.  
  - Inputs: Trigger nodes above.  
  - Outputs: Passes Topic to Google search HTTP request.  
  - Edge cases: Must be updated to reflect the desired search topic.

---

#### 1.2 Discover LinkedIn Advice Articles

**Overview:**  
Performs a Google search for LinkedIn advice articles matching the topic and extracts article links from the search results.

**Nodes Involved:**  
- Get advice articles from a Google search  
- Extract Article links for LinkedIn advice articles  
- Split Out all links for LinkedIn advice articles

**Node Details:**

- **Get advice articles from a Google search**  
  - Type: HTTP Request  
  - Role: Queries Google search with a URL template including the topic.  
  - Configuration: URL uses expression: `https://www.google.com/search?q=site:linkedin.com/advice+{{ $json.Topic }}`  
  - Options: Batching enabled with batch size 25 (to limit request size).  
  - Inputs: Topic from Set Topic node.  
  - Outputs: Raw Google search HTML response.  
  - Failures: Possible HTTP errors, Google blocking requests, or CAPTCHA challenges.

- **Extract Article links for LinkedIn advice articles**  
  - Type: Code (JavaScript)  
  - Role: Parses the Google search HTML response to extract LinkedIn advice article URLs using regex.  
  - Key Logic: Regex `/https:\/\/www\.linkedin\.com\/advice\/[^%&\s"']+/g` matches LinkedIn advice article URLs.  
  - Inputs: Google search HTML text.  
  - Outputs: Array of matched URLs under field `matches`.  
  - Failures: If Google changes result page structure or regex fails, no URLs extracted.

- **Split Out all links for LinkedIn advice articles**  
  - Type: SplitOut  
  - Role: Splits the array of article URLs into individual items for sequential processing downstream.  
  - Inputs: Array of URLs from previous node.  
  - Outputs: Single URL per item.  
  - Failures: None expected.

---

#### 1.3 Filter & Merge Unique Articles

**Overview:**  
Retrieves all previously logged LinkedIn contributions from the NocoDB database and merges them with newly extracted article URLs to exclude duplicates.

**Nodes Involved:**  
- Get all LinkedIn contributions from database NocoDB (GetRows)  
- Merge data and keep unique google search results

**Node Details:**

- **Get all LinkedIn contributions from database NocoDB (GetRows)**  
  - Type: NocoDB  
  - Role: Retrieves all stored LinkedIn contribution records from the database.  
  - Configuration: Operation "getAll" on specific table `mpagw9n92ran52o` within project `psdqqm1bzphkodc`.  
  - Inputs: Trigger or schedule node.  
  - Outputs: List of stored contributions with URLs and metadata.  
  - Failures: API token invalid, network failure, or permission issues.

- **Merge data and keep unique google search results**  
  - Type: Merge  
  - Role: Combines newly found article URLs and previously stored URLs, keeping only unique, non-duplicate URLs from the new search results.  
  - Configuration: Mode "combine", Join Mode "keepNonMatches", merge by comparing `URL` from database to `matches` from new URLs.  
  - Inputs: Left input - stored contributions; Right input - new URLs (from split).  
  - Outputs: Filtered list of unique new article URLs to process further.  
  - Failures: Merge expression errors or data schema mismatch.

---

#### 1.4 Retrieve and Extract Article Content

**Overview:**  
For each unique LinkedIn article URL, this block fetches the webpage content and extracts specific details (title, topics, existing contributions).

**Nodes Involved:**  
- HTTP Request to get LinkedIn advice articles  
- HTML extract LinkedIn article & other users contributions

**Node Details:**

- **HTTP Request to get LinkedIn advice articles**  
  - Type: HTTP Request  
  - Role: Downloads the HTML content of each LinkedIn article URL.  
  - Configuration: URL set dynamically to `={{ $json.matches }}` from merged unique URLs.  
  - Inputs: URLs from merge node.  
  - Outputs: Raw HTML content of LinkedIn articles.  
  - Failures: HTTP errors, 404 for expired articles, rate limiting by LinkedIn.

- **HTML extract LinkedIn article & other users contributions**  
  - Type: HTML Extract  
  - Role: Parses the article HTML to extract key fields:  
    - ArticleTitle: CSS selector `.pulse-title`  
    - ArticleTopics: CSS selector `.article-main__content`  
    - ArticleContributions: CSS selector `.contribution__text`  
  - Inputs: HTML content from HTTP Request node.  
  - Outputs: JSON with extracted article metadata.  
  - Failures: DOM structure changes on LinkedIn, missing selectors resulting in null fields.

---

#### 1.5 Generate AI Contributions

**Overview:**  
Uses an OpenAI GPT-4o-mini model to generate unique, helpful advice paragraphs for each topic in the LinkedIn article, ensuring originality.

**Nodes Involved:**  
- LinkedIn Contribution Writer

**Node Details:**

- **LinkedIn Contribution Writer**  
  - Type: Langchain OpenAI node  
  - Role: Sends the extracted article data to the AI model with a detailed prompt requesting unique advice for each topic.  
  - Configuration:  
    - Model: GPT-4o-mini  
    - Temperature: 0.7 (for creative but focused output)  
    - Prompt: Includes article title, topics, and existing contributions; instructs AI to output numbered advice paragraphs per topic, ensuring uniqueness.  
  - Inputs: Extracted article data.  
  - Outputs: AI-generated text content with advice per topic.  
  - Failures: API quota exceeded, network errors, prompt formatting errors.

---

#### 1.6 Post Contributions

**Overview:**  
Posts the AI-generated contributions to a Slack channel for team visibility and logs them into the NocoDB database for record-keeping.

**Nodes Involved:**  
- Post new LinkedIn contributions to Slack channel  
- Post new LinkedIn contributions to NocoDB (CreateRows)

**Node Details:**

- **Post new LinkedIn contributions to Slack channel**  
  - Type: Slack  
  - Role: Sends a formatted message to a specific Slack channel with article title, URL, and AI advice.  
  - Configuration:  
    - Channel: ID `C07CFN279HT` (selectable)  
    - Message: Includes article title, URL, AI advice separated by delimiters.  
    - Options: Markdown enabled, unfurl links enabled.  
    - Authentication: OAuth2 with Slack account.  
  - Inputs: AI contribution text and article metadata.  
  - Outputs: Slack message confirmation.  
  - Failures: Authentication errors, channel errors, message formatting issues.

- **Post new LinkedIn contributions to NocoDB (CreateRows)**  
  - Type: NocoDB  
  - Role: Creates a new row in the database table with contribution details for future reference.  
  - Configuration:  
    - Table: `mpagw9n92ran52o`  
    - Fields:  
      - Post Title: from extracted article title  
      - URL: from merged unique URLs  
      - Contribution: AI-generated advice text  
      - Topic: set statically as "Lead Generation"  
      - Person: set statically as "Cassie"  
    - Authentication: NocoDB API Token  
  - Inputs: Article title, URL, AI advice.  
  - Outputs: Confirmation of row creation.  
  - Failures: API token invalid, schema mismatch, data validation errors.

---

### 3. Summary Table

| Node Name                                    | Node Type                 | Functional Role                                     | Input Node(s)                                   | Output Node(s)                                       | Sticky Note                                                                                                          |
|----------------------------------------------|---------------------------|----------------------------------------------------|------------------------------------------------|-----------------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’                 | Manual Trigger            | Manual start for testing workflow                   | None                                           | Get all LinkedIn contributions, Set Topic for Google search |                                                                                                                      |
| Schedule Trigger Every Monday, @ 08:00am     | Schedule Trigger          | Weekly automatic start                              | None                                           | Get all LinkedIn contributions, Set Topic for Google search |                                                                                                                      |
| Set Topic for Google search                   | Set                       | Defines search topic string                         | Trigger nodes                                   | Get advice articles from a Google search             | Sets a specific topic to be used in subsequent steps of the workflow                                                 |
| Get advice articles from a Google search      | HTTP Request              | Performs Google search for LinkedIn advice articles | Set Topic for Google search                      | Extract Article links for LinkedIn advice articles   | Uses HTTP request to perform a Google search for LinkedIn advice articles based on a predefined query                 |
| Extract Article links for LinkedIn advice articles | Code                      | Extracts LinkedIn article URLs from search HTML     | Get advice articles from a Google search         | Split Out all links for LinkedIn advice articles     | Extracts LinkedIn article URLs from Google search results using regex                                               |
| Split Out all links for LinkedIn advice articles | SplitOut                  | Splits URLs array to individual items               | Extract Article links for LinkedIn advice articles | Merge data and keep unique google search results    | Splits the list of extracted LinkedIn article links into individual items                                            |
| Get all LinkedIn contributions from database NocoDB (GetRows) | NocoDB                    | Retrieves stored LinkedIn contributions              | Trigger nodes                                   | Merge data and keep unique google search results     | Retrieves all LinkedIn contributions stored in NocoDB table                                                        |
| Merge data and keep unique google search results | Merge                     | Combines new and existing URLs, filters duplicates | Get all LinkedIn contributions, Split Out all links | HTTP Request to get LinkedIn advice articles         | Merges and filters extracted article links, ensuring only unique LinkedIn URLs are processed                         |
| HTTP Request to get LinkedIn advice articles | HTTP Request              | Fetches HTML content of each LinkedIn article        | Merge data and keep unique google search results | HTML extract LinkedIn article & other users contributions | Sends HTTP requests to retrieve HTML content of LinkedIn articles                                                  |
| HTML extract LinkedIn article & other users contributions | HTML Extract              | Extracts article title, topics, and contributions   | HTTP Request to get LinkedIn advice articles     | LinkedIn Contribution Writer                         | Extracts relevant information from LinkedIn article HTML                                                           |
| LinkedIn Contribution Writer                  | OpenAI (Langchain)        | Generates unique AI contributions based on article  | HTML extract LinkedIn article & other users contributions | Post new LinkedIn contributions to Slack channel     | Uses AI to generate unique contributions for each LinkedIn article topic                                           |
| Post new LinkedIn contributions to Slack channel | Slack                     | Posts AI contributions to Slack channel             | LinkedIn Contribution Writer                     | Post new LinkedIn contributions to NocoDB (CreateRows) | Posts generated LinkedIn contributions to Slack for team visibility                                                |
| Post new LinkedIn contributions to NocoDB (CreateRows) | NocoDB                    | Stores AI contributions in NocoDB database          | Post new LinkedIn contributions to Slack channel | None                                                | Stores AI-generated LinkedIn contributions in NocoDB for record-keeping                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure to trigger weekly every Monday at 08:00 AM.

2. **Create a Manual Trigger Node** (optional for testing)  
   - Type: Manual Trigger  

3. **Create a Set Node to Define Search Topic**  
   - Type: Set  
   - Add a string field named `Topic`.  
   - Set its value to your desired search term, e.g., "Paid Advertising".

4. **Create an HTTP Request Node to Search Google**  
   - Type: HTTP Request  
   - URL: `https://www.google.com/search?q=site:linkedin.com/advice+{{ $json.Topic }}`  
   - Enable batching with batch size 25 (optional, to limit data size).  
   - Connect Set node output to this node.

5. **Create a Code Node to Extract LinkedIn Article URLs**  
   - Type: Code  
   - JavaScript code:  
     ```javascript
     const text = $json.data;
     const regexPattern = /https:\/\/www\.linkedin\.com\/advice\/[^%&\s"']+/g;
     const matches = text.match(regexPattern);
     return { matches: matches || [] };
     ```  
   - Input: Output of HTTP request node.

6. **Create a SplitOut Node to Split Article URLs**  
   - Field to split out: `matches`  
   - Input: Code node output.

7. **Create a NocoDB GetRows Node to Fetch Stored Contributions**  
   - Type: NocoDB  
   - Operation: getAll  
   - Configure with your NocoDB credentials and project/table ID.  
   - Input: Connect from Schedule Trigger and Manual Trigger nodes.

8. **Create a Merge Node to Combine and Filter Unique URLs**  
   - Mode: Combine  
   - Join Mode: Keep Non Matches  
   - Merge by fields: Compare `URL` (from DB) with `matches` (from new URLs).  
   - Left input: GetRows node (stored contributions)  
   - Right input: SplitOut node (new article URLs)

9. **Create an HTTP Request Node to Fetch LinkedIn Article HTML**  
   - URL set dynamically: `={{ $json.matches }}`  
   - Input: Merge node output.

10. **Create an HTML Extract Node to Parse Article Content**  
    - Operation: extractHtmlContent  
    - Extraction fields:  
      - ArticleTitle: `.pulse-title`  
      - ArticleTopics: `.article-main__content`  
      - ArticleContributions: `.contribution__text`  
    - Input: HTTP Request node output.

11. **Create an OpenAI Node (Langchain) for AI Contribution Generation**  
    - Model: GPT-4o-mini  
    - Temperature: 0.7  
    - Credentials: Configure OpenAI API key.  
    - Messages: Use the prompt template to provide article title, topics, and existing contributions, asking for unique advice per topic.  
    - Input: HTML Extract node output.

12. **Create a Slack Node to Post AI Contributions**  
    - Authentication: OAuth2 with Slack account  
    - Channel: Set to your Slack channel ID  
    - Message: Format to include article title, URL, and AI advice with markdown enabled.  
    - Input: OpenAI node output.

13. **Create a NocoDB Create Row Node to Store Contributions**  
    - Operation: create  
    - Fields:  
      - Post Title: from HTML Extract node (ArticleTitle)  
      - URL: from Merge node (matches)  
      - Contribution: from OpenAI node (message content)  
      - Topic: Static or dynamic as needed  
      - Person: Static string (e.g., "Cassie")  
    - Authentication: NocoDB API token  
    - Input: Slack node output.

14. **Connect all nodes in the logical order as described above.**

15. **Test the workflow manually, then enable the schedule trigger.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                | Context or Link                                           |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| This workflow was built for the [Let's Automate It Community](http://onlinethinking.io/community) by [Optimus Agency](https://optimus01.co.za/)                            | Project credits and community                             |
| NocoDB can be swapped with other database services such as Airtable or Google Sheets for contribution storage.                                                             | Flexibility in database integration                      |
| To customize the workflow, update the Google search query in the HTTP Request node using the topic string to focus on different LinkedIn advice areas.                     | Search customization                                      |
| The AI prompt in the OpenAI node can be edited to match brand voice or content strategy for personalized advice.                                                           | Contribution quality customization                        |
| Slack channel configuration requires an OAuth2 Slack app with appropriate permissions to post messages.                                                                    | Slack integration setup                                  |
| Google may block or rate-limit automated search requests; consider using a custom API or alternative search methods if issues arise.                                        | Google search reliability warning                         |
| CSS selectors used for HTML extraction are dependent on LinkedIn's page structure and may need updating if LinkedIn redesigns their advice article pages.                    | Maintain extraction accuracy                              |
| Video tutorials and templates for similar workflows can be found at [Online Thinking](https://onlinethinking.io/community)                                                  | Additional learning resources                             |

---

This documentation provides a detailed and structured reference for understanding, reproducing, and customizing the LinkedIn Contribution automation workflow using n8n, ensuring robust application and easy maintenance.