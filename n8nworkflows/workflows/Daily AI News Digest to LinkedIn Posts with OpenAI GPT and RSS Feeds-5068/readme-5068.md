Daily AI News Digest to LinkedIn Posts with OpenAI GPT and RSS Feeds

https://n8nworkflows.xyz/workflows/daily-ai-news-digest-to-linkedin-posts-with-openai-gpt-and-rss-feeds-5068


# Daily AI News Digest to LinkedIn Posts with OpenAI GPT and RSS Feeds

### 1. Workflow Overview

This workflow automates the daily creation of LinkedIn posts summarizing the latest AI news by aggregating multiple RSS feeds, filtering for relevance, summarizing key developments using OpenAI GPT models, and preparing a professional LinkedIn post draft. The post is then sent via email for review before publishing.

The workflow can be logically divided into three main blocks:

- **1.1 Research for AI Articles**: Scheduled retrieval of AI news articles from multiple RSS feeds, aggregation, deduplication, and filtering based on recency and AI relevance.

- **1.2 Summarize AI Related Articles**: Preparation of filtered articles into a combined string, then use of OpenAI GPT to generate an expert summary highlighting the most important AI industry developments.

- **1.3 Generate LinkedIn Post and Send to Email**: Creation of an engaging LinkedIn post from the summary by another OpenAI GPT prompt, followed by sending the generated content via Gmail for editorial review.

---

### 2. Block-by-Block Analysis

#### 2.1 Research for AI Articles

**Overview:**  
This block triggers once every 24 hours to fetch AI news from three RSS feeds (VentureBeat, TechCrunch, OpenAI Blog). The articles are merged, filtered to the last 48 hours, and limited to the top 10 relevant AI articles without duplicates.

**Nodes Involved:**  
- Daily AI News Check1 (Schedule Trigger)  
- VentureBeat AI RSS1 (RSS Feed Read)  
- TechCrunch AI RSS1 (RSS Feed Read)  
- OpenAI Blog RSS1 (RSS Feed Read)  
- Merge All Sources1 (Merge)  
- Filter Recent AI Articles1 (Code)

**Node Details:**

- **Daily AI News Check1**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow every 24 hours  
  - Config: Interval set to 24 hours  
  - Input: None  
  - Output: Triggers all three RSS feed reads concurrently  
  - Failures: Trigger misfires rare; time zone considerations may apply

- **VentureBeat AI RSS1**  
  - Type: RSS Feed Read  
  - Role: Fetches latest VentureBeat AI news articles  
  - Config: URL set to VentureBeat AI RSS feed  
  - Input: Trigger from schedule  
  - Output: Articles as JSON items  
  - Failures: Network issues, feed downtime, malformed feed

- **TechCrunch AI RSS1**  
  - Type: RSS Feed Read  
  - Role: Fetches latest TechCrunch AI news articles  
  - Config: URL set to TechCrunch AI RSS feed  
  - Input: Trigger from schedule  
  - Output: Articles as JSON items  
  - Failures: Same as above

- **OpenAI Blog RSS1**  
  - Type: RSS Feed Read  
  - Role: Fetches latest OpenAI blog posts  
  - Config: URL set to OpenAI blog RSS feed  
  - Input: Trigger from schedule  
  - Output: Articles as JSON items  
  - Failures: Same as above

- **Merge All Sources1**  
  - Type: Merge  
  - Role: Combines articles from all three feeds into a single stream  
  - Config: Mode "combine" with multiplex combination mode to merge all incoming items  
  - Input: RSS feed outputs  
  - Output: Combined article list  
  - Failures: None expected unless input data format changes

- **Filter Recent AI Articles1**  
  - Type: Code (JavaScript)  
  - Role: Filters articles published in last 48 hours and relevant to AI keywords  
  - Config:  
    - Checks article publication date against current date minus 48 hours  
    - Filters for AI-related keywords in title or description  
    - Deduplicates articles by link or title  
    - Limits output to top 10 newest articles  
  - Input: Merged articles  
  - Output: Filtered, unique AI-related articles array  
  - Failures: Date parsing errors, missing fields, empty input  
  - Edge Cases: Articles without pubDate or date fields may be excluded; keyword sensitivity might miss some relevant articles or include false positives

---

#### 2.2 Summarize AI Related Articles

**Overview:**  
This block prepares the filtered articles into a formatted string and sends it to an OpenAI GPT model to generate a concise, expert summary suitable for LinkedIn professionals highlighting key AI industry developments.

**Nodes Involved:**  
- Prepare Articles for Summary1 (Code)  
- Check Articles Exist1 (If)  
- AI News Summarizer1 (OpenAI GPT)

**Node Details:**

- **Prepare Articles for Summary1**  
  - Type: Code (JavaScript)  
  - Role: Converts filtered articles into a single formatted string including title, description, source, date, and link  
  - Input: Filtered articles array  
  - Output: JSON with combined articles string, article count, and timestamp  
  - Failures: Empty input results in empty string, no critical failures expected

- **Check Articles Exist1**  
  - Type: If  
  - Role: Checks if articleCount > 0 to decide next step  
  - Input: JSON from Prepare Articles for Summary1  
  - Output: Routes to summarization if articles exist; otherwise triggers notification  
  - Failures: If expression evaluation misconfigured, could misroute workflow

- **AI News Summarizer1**  
  - Type: OpenAI GPT (Langchain node)  
  - Role: Uses GPT-4.1-mini to analyze combined articles and produce a 3-4 sentence summary focused on major AI developments, business implications, and recent news  
  - Configuration:  
    - Model: GPT-4.1-mini  
    - Prompt instructs to act as expert AI industry analyst, prioritize significant developments, use professional LinkedIn-appropriate style  
    - Injects combined articles string as input variable `{{ $json.articles }}`  
  - Input: Prepared article string  
  - Output: Summary string  
  - Failures: API authentication errors, rate limits, timeout, malformed input  
  - Edge Cases: If articles are too long or improperly formatted, summary quality may degrade

---

#### 2.3 Generate LinkedIn Post and Send to Email

**Overview:**  
This block takes the AI-generated summary, creates a professional and engaging LinkedIn post draft with a specific structure and tone using GPT, then sends the draft via Gmail for human review.

**Nodes Involved:**  
- Generate LinkedIn Post1 (OpenAI GPT)  
- Send for Review1 (Gmail)  
- No Articles Notification1 (Gmail) [Triggered only if no articles found]

**Node Details:**

- **Generate LinkedIn Post1**  
  - Type: OpenAI GPT (Langchain node)  
  - Role: Creates a LinkedIn post text from the AI news summary with a hook, body, call-to-action, emojis, hashtags, and proper style  
  - Configuration:  
    - Model: GPT-4.1-mini  
    - Prompt instructs professional LinkedIn content style, concise engaging paragraphs, specific hashtags, ending with a question to encourage discussion  
    - Uses input from AI News Summarizer1 as message content (`{{ $json.message.content }}`)  
  - Input: Summary text  
  - Output: LinkedIn post draft text  
  - Failures: Same as AI News Summarizer1

- **Send for Review1**  
  - Type: Gmail  
  - Role: Sends generated LinkedIn post content as plain text email to reviewer  
  - Configuration:  
    - Subject: "LinkedIn AI Post Ready for Review"  
    - Message body: Populated with LinkedIn post text  
    - Uses Gmail OAuth2 credentials  
  - Input: LinkedIn post draft  
  - Output: Email sent confirmation  
  - Failures: Authentication failure, quota limits, connectivity issues

- **No Articles Notification1**  
  - Type: Gmail  
  - Role: Sends notification email if no relevant AI articles were found in last 48 hours  
  - Configuration:  
    - Subject: "No AI News Found Today"  
    - Message body: "No AI News Found Today"  
    - Uses Gmail OAuth2 credentials  
  - Input: If node triggers this path  
  - Output: Email sent confirmation  
  - Failures: Same as Send for Review1

---

### 3. Summary Table

| Node Name               | Node Type                | Functional Role                          | Input Node(s)                  | Output Node(s)                     | Sticky Note                                               |
|-------------------------|--------------------------|----------------------------------------|-------------------------------|----------------------------------|-----------------------------------------------------------|
| Daily AI News Check1     | Schedule Trigger         | Initiate daily workflow run             | None                          | VentureBeat AI RSS1, TechCrunch AI RSS1, OpenAI Blog RSS1 | ## 1) Research for AI articles                             |
| VentureBeat AI RSS1      | RSS Feed Read            | Fetch VentureBeat AI news               | Daily AI News Check1           | Merge All Sources1                | ## 1) Research for AI articles                             |
| TechCrunch AI RSS1       | RSS Feed Read            | Fetch TechCrunch AI news                | Daily AI News Check1           | Merge All Sources1                | ## 1) Research for AI articles                             |
| OpenAI Blog RSS1         | RSS Feed Read            | Fetch OpenAI blog news                  | Daily AI News Check1           | Merge All Sources1                | ## 1) Research for AI articles                             |
| Merge All Sources1       | Merge                    | Combine all RSS feed articles           | VentureBeat AI RSS1, TechCrunch AI RSS1, OpenAI Blog RSS1 | Filter Recent AI Articles1         | ## 1) Research for AI articles                             |
| Filter Recent AI Articles1 | Code                   | Filter last 48h AI-relevant & unique articles | Merge All Sources1           | Prepare Articles for Summary1    | ## 1) Research for AI articles                             |
| Prepare Articles for Summary1 | Code                | Format articles into single string      | Filter Recent AI Articles1     | Check Articles Exist1             | ## 2) Summarise AI related articles                        |
| Check Articles Exist1    | If                       | Check if articles exist to continue     | Prepare Articles for Summary1  | AI News Summarizer1 (true), No Articles Notification1 (false) | ## 2) Summarise AI related articles                        |
| AI News Summarizer1      | OpenAI GPT (Langchain)   | Summarize AI news articles               | Check Articles Exist1 (true)   | Generate LinkedIn Post1           | ## 2) Summarise AI related articles                        |
| Generate LinkedIn Post1  | OpenAI GPT (Langchain)   | Create LinkedIn post from summary        | AI News Summarizer1            | Send for Review1                 | ## 3) Generate Linkedin Post and send to email            |
| Send for Review1         | Gmail                    | Email LinkedIn post for review           | Generate LinkedIn Post1        | None                            | ## 3) Generate Linkedin Post and send to email            |
| No Articles Notification1| Gmail                    | Notify if no AI news articles found      | Check Articles Exist1 (false)  | None                            | ## 3) Generate Linkedin Post and send to email            |
| Sticky Note              | Sticky Note              | Workflow annotation                      | None                          | None                            | ## 1) Research for AI articles                             |
| Sticky Note1             | Sticky Note              | Workflow annotation                      | None                          | None                            | ## 2) Summarise AI related articles                        |
| Sticky Note2             | Sticky Note              | Workflow annotation                      | None                          | None                            | ## 3) Generate Linkedin Post and send to email            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Name: `Daily AI News Check1`  
   - Type: Schedule Trigger  
   - Set interval to every 24 hours (hoursInterval = 24).

2. **Create RSS Feed Read Nodes (3 total):**  
   - Name: `VentureBeat AI RSS1`  
     - URL: `https://feeds.feedburner.com/venturebeat/SZYF`  
   - Name: `TechCrunch AI RSS1`  
     - URL: `https://techcrunch.com/category/artificial-intelligence/feed/`  
   - Name: `OpenAI Blog RSS1`  
     - URL: `https://openai.com/blog/rss.xml`  
   - All: Leave default RSS options.  
   - Connect the output of `Daily AI News Check1` main to each RSS node’s input.

3. **Create Merge Node:**  
   - Name: `Merge All Sources1`  
   - Type: Merge  
   - Mode: Combine  
   - Combination Mode: Multiplex  
   - Connect the outputs of all three RSS Feed nodes to `Merge All Sources1`.

4. **Create Code Node for Filtering Articles:**  
   - Name: `Filter Recent AI Articles1`  
   - Type: Code (JavaScript)  
   - Paste the provided JS code that:  
     - Filters articles from last 48 hours  
     - Checks for AI-related keywords in title/description  
     - Deduplicates by link or title  
     - Sorts by date, limits to 10  
   - Connect `Merge All Sources1` output to this node.

5. **Create Code Node to Prepare Articles for Summary:**  
   - Name: `Prepare Articles for Summary1`  
   - Type: Code (JavaScript)  
   - Combine filtered articles into a single string with titles, descriptions, source, date, and link separated by delimiters.  
   - Include article count and timestamp in output JSON.  
   - Connect `Filter Recent AI Articles1` output to this node.

6. **Create If Node to Check Articles Exist:**  
   - Name: `Check Articles Exist1`  
   - Type: If  
   - Condition: Check if `articleCount` (from JSON) > 0  
   - Connect `Prepare Articles for Summary1` output to this node.

7. **Create OpenAI Langchain Node for Summarization:**  
   - Name: `AI News Summarizer1`  
   - Type: OpenAI (Langchain)  
   - Model: `gpt-4.1-mini`  
   - Prompt: Use the detailed prompt instructing expert AI analyst summarization of articles; insert `{{ $json.articles }}` for article text.  
   - Connect `Check Articles Exist1` “true” output to this node.  
   - Set credentials for OpenAI API.

8. **Create OpenAI Langchain Node for LinkedIn Post Generation:**  
   - Name: `Generate LinkedIn Post1`  
   - Type: OpenAI (Langchain)  
   - Model: `gpt-4.1-mini`  
   - Prompt: Use detailed prompt for professional LinkedIn post creation, using `{{ $json.message.content }}` from summarizer output.  
   - Connect `AI News Summarizer1` output to this node.  
   - Use same OpenAI credentials.

9. **Create Gmail Node to Send Post for Review:**  
   - Name: `Send for Review1`  
   - Type: Gmail  
   - Subject: `LinkedIn AI Post Ready for Review`  
   - Message: `={{ $json.message.content }}` (text body)  
   - Use Gmail OAuth2 credentials.  
   - Connect `Generate LinkedIn Post1` output to this node.

10. **Create Gmail Node for No Articles Notification:**  
    - Name: `No Articles Notification1`  
    - Type: Gmail  
    - Subject: `No AI News Found Today`  
    - Message: `No AI News Found Today` (plain text)  
    - Use Gmail OAuth2 credentials.  
    - Connect `Check Articles Exist1` “false” output to this node.

11. **Create Sticky Notes (optional):**  
    - Add three sticky notes annotating each block for clarity if desired.

12. **Verify all connections:**  
    - `Daily AI News Check1` → all three RSS feeds  
    - RSS feeds → `Merge All Sources1`  
    - `Merge All Sources1` → `Filter Recent AI Articles1`  
    - `Filter Recent AI Articles1` → `Prepare Articles for Summary1`  
    - `Prepare Articles for Summary1` → `Check Articles Exist1`  
    - `Check Articles Exist1` true → `AI News Summarizer1` → `Generate LinkedIn Post1` → `Send for Review1`  
    - `Check Articles Exist1` false → `No Articles Notification1`

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                          |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------|
| Workflow tag: AI Automation                                                                       | Categorizes workflow within AI automation projects      |
| Sticky notes divide workflow into 3 clear blocks: Research, Summarize, Generate & Send           | Useful for workflow documentation and visualization     |
| OpenAI GPT model used: gpt-4.1-mini                                                              | Requires OpenAI API credentials and sufficient quota    |
| Gmail OAuth2 credentials required for email sending                                              | Setup Gmail OAuth2 for n8n with proper scopes           |
| RSS feed URLs used are publicly accessible AI news sources from VentureBeat, TechCrunch, OpenAI  | Can be replaced or extended with other AI news sources  |
| The summarization prompt is carefully crafted to balance professional tone and informative style | Ensures LinkedIn-ready content with business relevance  |
| Post generation prompt includes emoji and hashtag usage guidelines                               | Enhances engagement on social media platforms           |
| Edge cases: No articles found triggers notification email to avoid silent failures               | Ensures user awareness of daily news absence            |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.