Create a Personalized Daily Newsletter with Google Gemini AI and RSS Feeds

https://n8nworkflows.xyz/workflows/create-a-personalized-daily-newsletter-with-google-gemini-ai-and-rss-feeds-10196


# Create a Personalized Daily Newsletter with Google Gemini AI and RSS Feeds

### 1. Workflow Overview

This workflow, titled **"Create a Personalized Daily Newsletter with Google Gemini AI and RSS Feeds"**, is designed to automatically curate and send a personalized daily newsletter focusing on the latest technology news, tools, and products tailored to user interests. It is ideal for technology enthusiasts, product managers, and developers who want to stay updated with curated content without manual filtering.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Trigger:** Daily scheduled trigger to initiate the workflow.
- **1.2 RSS Feed Sources Setup:** Define and prepare RSS feed URLs from multiple tech news and tools/product sources.
- **1.3 RSS Feeds Processing:** Splitting, batching, and fetching RSS feed content.
- **1.4 Data Combining and Conversion:** Merge all RSS feed data and convert it into a CSV file format for AI processing.
- **1.5 AI Curation:** Use Google Gemini AI to analyze and curate the combined RSS content based on predefined keywords and interests.
- **1.6 Newsletter Formatting and Delivery:** Format the AI-curated content into HTML and send it as a daily email newsletter.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Trigger

- **Overview:** This block initializes the workflow daily at 9:00 AM automatically.
- **Nodes Involved:**  
  - Schedule Trigger1  
  - Sticky Note - Trigger  

- **Node Details:**

  - **Schedule Trigger1**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow daily at 9:00 AM.  
    - Configuration: Trigger set for hour 9 (9 AM).  
    - Inputs: None (start node).  
    - Outputs: Connects to "Tech News Sites" and "Tools Subredit and Sites".  
    - Edge Cases: Timezone misconfiguration could cause trigger to run at unexpected local times.

  - **Sticky Note - Trigger**  
    - Type: Sticky Note  
    - Role: Documentation for the trigger node explaining the daily schedule.  
    - Inputs/Outputs: None.  
    - Edge Cases: None.

---

#### 2.2 RSS Feed Sources Setup

- **Overview:** Defines the RSS feed URLs for technology news and tools/products that the workflow will monitor.
- **Nodes Involved:**  
  - Tech News Sites  
  - Tools Subredit and Sites  
  - Sticky Note - Sources  
  - Sticky Note (Note on two lists)  

- **Node Details:**

  - **Tech News Sites**  
    - Type: Set  
    - Role: Stores an array of RSS feed URLs from popular tech news sites like TechCrunch, The Verge, Wired, etc.  
    - Configuration: JSON with key `url` containing list of RSS URLs.  
    - Inputs: From Schedule Trigger1.  
    - Outputs: Connected to "Split Out".  
    - Edge Cases: Invalid or unreachable RSS URLs would yield empty or error responses downstream.

  - **Tools Subredit and Sites**  
    - Type: Set  
    - Role: Stores RSS feed URLs for tools and product sources including Reddit subreddits and Product Hunt.  
    - Configuration: JSON with key `Url` containing list of RSS URLs.  
    - Inputs: From Schedule Trigger1.  
    - Outputs: Connected to "Split Out1".  
    - Edge Cases: Same as above.

  - **Sticky Note - Sources**  
    - Type: Sticky Note  
    - Role: Describes the purpose of these nodes as the RSS feed source definition.  
    - Inputs/Outputs: None.  

  - **Sticky Note (Note on two lists)**  
    - Type: Sticky Note  
    - Role: Explains that two separate lists (News and Tools) are used and can be customized.  
    - Inputs/Outputs: None.

---

#### 2.3 RSS Feeds Processing

- **Overview:** Splits the feed URL arrays into individual items, processes them in batches, and fetches the RSS content for each feed.
- **Nodes Involved:**  
  - Split Out  
  - Split Out1  
  - Loop Over Items  
  - Loop Over Items1  
  - RSS Read OF Tech News  
  - RSS Read OF Tools  
  - Sticky Note - Processing  

- **Node Details:**

  - **Split Out**  
    - Type: Split Out  
    - Role: Extracts individual RSS feed URLs from the `url` array for tech news sites.  
    - Inputs: From "Tech News Sites".  
    - Outputs: Connects to "Loop Over Items".  
    - Edge Cases: Empty or malformed arrays cause no output.

  - **Split Out1**  
    - Type: Split Out  
    - Role: Extracts individual RSS feed URLs from the `Url` array for tools/product feeds.  
    - Inputs: From "Tools Subredit and Sites".  
    - Outputs: Connects to "Loop Over Items1".  

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Role: Processes tech news feed URLs in batches of 5 to avoid overloading RSS reading.  
    - Inputs: From "Split Out".  
    - Outputs: Two outputs: one to merge list, one to RSS read.  
    - Configuration: Batch size = 5.  
    - Edge Cases: Batch size too large may cause timeout or API rate limits.

  - **Loop Over Items1**  
    - Type: Split In Batches  
    - Role: Processes tools/product feed URLs in batches of 4.  
    - Inputs: From "Split Out1".  
    - Outputs: Two outputs: one to merge list, one to RSS read.  
    - Configuration: Batch size = 4.  

  - **RSS Read OF Tech News**  
    - Type: RSS Feed Read  
    - Role: Fetches the actual feed content from each tech news RSS URL.  
    - Inputs: From "Loop Over Items".  
    - Outputs: Connects back to "Loop Over Items" for processing.  
    - Configuration: URL set dynamically using expression from split URL.  
    - Edge Cases: Unavailable feeds or malformed RSS content may cause empty output. Error handling set to "continueRegularOutput".

  - **RSS Read OF Tools**  
    - Type: RSS Feed Read  
    - Role: Fetches RSS content for tools/products feeds.  
    - Inputs: From "Loop Over Items1".  
    - Outputs: Connects back to "Loop Over Items1".  

  - **Sticky Note - Processing**  
    - Type: Sticky Note  
    - Role: Describes this block as responsible for splitting, looping, and fetching RSS content.  

---

#### 2.4 Data Combining and Conversion

- **Overview:** Merges the two lists of fetched RSS feed contents and converts the combined JSON data into a CSV file for further AI analysis.
- **Nodes Involved:**  
  - Merge Both List  
  - Conver To CSV  
  - Sticky Note - Combine  

- **Node Details:**

  - **Merge Both List**  
    - Type: Merge  
    - Role: Combines RSS feed data from tech news and tools/product lists into a single data stream.  
    - Inputs: Two inputs from "Loop Over Items" (tech news) and "Loop Over Items1" (tools).  
    - Outputs: To "Conver To CSV".  
    - Edge Cases: If one input is empty, only the other’s data will be merged.

  - **Conver To CSV**  
    - Type: Convert To File  
    - Role: Converts the merged JSON feed data into a CSV file, preparing it for AI ingestion.  
    - Inputs: From "Merge Both List".  
    - Outputs: To "Upload a file".  

  - **Sticky Note - Combine**  
    - Type: Sticky Note  
    - Role: Describes this block’s function to merge and convert feed data.  

---

#### 2.5 AI Curation

- **Overview:** Uses Google Gemini AI to analyze the CSV file, curate top items based on keywords, and generate an HTML-formatted newsletter content.
- **Nodes Involved:**  
  - Upload a file  
  - Analyze document  
  - Sticky Note - AI  

- **Node Details:**

  - **Upload a file**  
    - Type: Google Gemini (LangChain)  
    - Role: Uploads the CSV file as a binary input resource to the Google Gemini model.  
    - Inputs: From "Conver To CSV".  
    - Outputs: To "Analyze document".  
    - Configuration: Resource type set to "file" with inputType "binary".  
    - Credentials: Uses Google Gemini (PaLM) API credentials.  
    - Edge Cases: API authentication failure or file size limits.

  - **Analyze document**  
    - Type: Google Gemini (LangChain)  
    - Role: Sends a detailed prompt to Google Gemini to curate the top 10-15 items from tools and news lists based on specified keywords and format the output as an HTML newsletter.  
    - Inputs: Receives file URI from "Upload a file".  
    - Outputs: To "Code in JavaScript".  
    - Configuration:  
      - Text prompt includes detailed instructions on selection criteria, formatting, tone, and HTML template.  
      - Model used: "models/gemini-2.5-flash-preview-05-20".  
    - Credentials: Uses Google Gemini (PaLM) API credentials.  
    - Edge Cases:  
      - AI model errors or rate limiting.  
      - Incorrect prompt formatting causing unexpected output.

  - **Sticky Note - AI**  
    - Type: Sticky Note  
    - Role: Explains AI curation step and prompt customization advice.  

---

#### 2.6 Newsletter Formatting and Delivery

- **Overview:** Processes the AI-generated HTML content and sends the formatted newsletter email to the recipient.
- **Nodes Involved:**  
  - Code in JavaScript  
  - Send email  
  - Sticky Note - Email  

- **Node Details:**

  - **Code in JavaScript**  
    - Type: Code (JavaScript)  
    - Role: Cleans the HTML text returned from the AI node by removing newline characters and trimming, preparing it for the email body.  
    - Inputs: From "Analyze document".  
    - Outputs: To "Send email".  
    - Code Summary:  
      - Extracts HTML content from the AI node’s JSON response.  
      - Removes "\n" characters and trims whitespace.  
      - Outputs cleaned HTML.  
    - Edge Cases:  
      - AI output missing expected fields causing exceptions.

  - **Send email**  
    - Type: Email Send  
    - Role: Sends the daily tech digest email with the curated HTML content.  
    - Inputs: From "Code in JavaScript".  
    - Outputs: None (end node).  
    - Configuration:  
      - Subject line includes current date dynamically.  
      - Email addresses for sender and receiver configurable.  
      - SMTP credentials required and configured.  
    - Credentials: SMTP account configured.  
    - Edge Cases:  
      - SMTP authentication failure.  
      - Email delivery issues or spam filtering.

  - **Sticky Note - Email**  
    - Type: Sticky Note  
    - Role: Explains the function of this block and notes to configure SMTP and email addresses.  

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                         | Input Node(s)                     | Output Node(s)                     | Sticky Note                                                                                         |
|-------------------------|----------------------------------|---------------------------------------|----------------------------------|----------------------------------|---------------------------------------------------------------------------------------------------|
| Sticky Note - Overview  | Sticky Note                      | Workflow high-level overview           | None                             | None                             | # 📬 Personal Daily Tech Newsletter... Customize RSS sources, keywords, email, and schedule time.  |
| Schedule Trigger1       | Schedule Trigger                 | Daily 9:00 AM trigger                   | None                             | Tech News Sites, Tools Subredit and Sites | ## 📅 Daily Trigger Scheduled to run every day at 9:00 AM                                         |
| Sticky Note - Trigger   | Sticky Note                      | Documentation for the trigger node     | None                             | None                             | ## 📅 Daily Trigger Scheduled to run every day at 9:00 AM                                         |
| Tech News Sites         | Set                             | Defines tech news RSS feed URLs        | Schedule Trigger1                | Split Out                       | ## 📰 RSS Feed Sources Define which RSS feeds to monitor for tech news and tools                   |
| Tools Subredit and Sites| Set                             | Defines tools and product RSS feed URLs| Schedule Trigger1                | Split Out1                      | ## 📰 RSS Feed Sources Define which RSS feeds to monitor for tech news and tools                   |
| Split Out               | Split Out                       | Splits tech news RSS feed URLs         | Tech News Sites                  | Loop Over Items                 | ## 🔄 RSS Feed Processing Split, loop through feeds in batches, and fetch content                   |
| Split Out1              | Split Out                       | Splits tools/product RSS feed URLs     | Tools Subredit and Sites         | Loop Over Items1                | ## 🔄 RSS Feed Processing Split, loop through feeds in batches, and fetch content                   |
| Loop Over Items         | Split In Batches                | Batch processing tech news RSS feeds   | Split Out                       | Merge Both List, RSS Read OF Tech News | ## 🔄 RSS Feed Processing Split, loop through feeds in batches, and fetch content                   |
| Loop Over Items1        | Split In Batches                | Batch processing tools/product RSS feeds| Split Out1                      | Merge Both List, RSS Read OF Tools | ## 🔄 RSS Feed Processing Split, loop through feeds in batches, and fetch content                   |
| RSS Read OF Tech News   | RSS Feed Read                  | Fetches tech news RSS feed content     | Loop Over Items                 | Loop Over Items                | ## 🔄 RSS Feed Processing Split, loop through feeds in batches, and fetch content                   |
| RSS Read OF Tools       | RSS Feed Read                  | Fetches tools/product RSS feed content | Loop Over Items1                | Loop Over Items1               | ## 🔄 RSS Feed Processing Split, loop through feeds in batches, and fetch content                   |
| Merge Both List         | Merge                          | Combines tech news and tools feed data | Loop Over Items, Loop Over Items1 | Conver To CSV                  | ## 📦 Combine & Convert Merge all feeds and convert to CSV for AI                                  |
| Conver To CSV           | Convert To File                | Converts merged JSON feed data to CSV | Merge Both List                | Upload a file                 | ## 📦 Combine & Convert Merge all feeds and convert to CSV for AI                                  |
| Upload a file           | Google Gemini (LangChain)      | Uploads CSV file to Google Gemini AI   | Conver To CSV                  | Analyze document              | ## 🤖 AI Curation Google Gemini analyzes content and selects top items based on interests          |
| Analyze document        | Google Gemini (LangChain)      | AI curation and HTML newsletter creation| Upload a file                  | Code in JavaScript            | ## 🤖 AI Curation Google Gemini analyzes content and selects top items based on interests          |
| Code in JavaScript      | Code                          | Cleans AI-generated HTML content       | Analyze document               | Send email                   | ## 📧 Newsletter Delivery Format and send the personalized newsletter                              |
| Send email              | Email Send                    | Sends the curated newsletter via email | Code in JavaScript             | None                        | ## 📧 Newsletter Delivery Format and send the personalized newsletter                              |
| Sticky Note - Sources   | Sticky Note                  | Describes RSS feed source nodes         | None                          | None                        | ## 📰 RSS Feed Sources Define which RSS feeds to monitor for tech news and tools                   |
| Sticky Note - Processing| Sticky Note                  | Describes RSS feed processing steps     | None                          | None                        | ## 🔄 RSS Feed Processing Split, loop through feeds in batches, and fetch content                   |
| Sticky Note - Combine   | Sticky Note                  | Describes data merging and conversion   | None                          | None                        | ## 📦 Combine & Convert Merge all feeds and convert to CSV for AI                                  |
| Sticky Note - AI        | Sticky Note                  | Describes AI curation step and prompt   | None                          | None                        | ## 🤖 AI Curation Google Gemini analyzes content and selects top items based on interests          |
| Sticky Note - Email     | Sticky Note                  | Notes about email sending configuration | None                          | None                        | ## 📧 Newsletter Delivery Format and send the personalized newsletter                              |
| Sticky Note (Note on two lists) | Sticky Note          | Explains two separate lists for news and tools | None                     | None                        | **Note:** Two different sources of interest (News and Tools) used; change as needed               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Configure to trigger daily at 9:00 AM (set triggerAtHour = 9).  
   - Name it "Schedule Trigger1".

2. **Add two Set nodes for RSS feed URLs**  
   - Node 1: "Tech News Sites"  
     - Mode: Raw JSON  
     - JSON: `{ "url": [ "https://techcrunch.com/feed/", "https://www.theverge.com/rss/index.xml", "https://analyticsindiamag.com/feed/", "https://www.wired.com/feed/rss", "https://thenextweb.com/feed" ] }`  
   - Node 2: "Tools Subredit and Sites"  
     - Mode: Raw JSON  
     - JSON: `{ "Url": [ "https://www.reddit.com/r/SideProject/.rss", "https://www.reddit.com/r/InternetIsBeautiful/.rss", "https://dev.to/feed/tag/tools", "https://www.producthunt.com/feed" ] }`  
   - Connect “Schedule Trigger1” to both nodes.

3. **Add Split Out nodes to split RSS URLs into individual items**  
   - For "Tech News Sites": Add "Split Out" node, fieldToSplitOut = "url". Connect from "Tech News Sites".  
   - For "Tools Subredit and Sites": Add "Split Out1" node, fieldToSplitOut = "Url". Connect from "Tools Subredit and Sites".

4. **Add Split In Batches nodes for batch processing**  
   - "Loop Over Items" for tech news feeds, batchSize = 5, connect from "Split Out".  
   - "Loop Over Items1" for tools feeds, batchSize = 4, connect from "Split Out1".

5. **Add RSS Feed Read nodes**  
   - "RSS Read OF Tech News" connected from "Loop Over Items". URL configured dynamically: `={{ $('Split Out').item.json.url }}`. Set onError to "continueRegularOutput".  
   - "RSS Read OF Tools" connected from "Loop Over Items1". URL: `={{ $json.Url }}`.

6. **Connect the RSS Read outputs back to their respective Split In Batches nodes**  
   - "RSS Read OF Tech News" output connects to "Loop Over Items" (to continue batch processing).  
   - "RSS Read OF Tools" output connects to "Loop Over Items1".

7. **Add a Merge node "Merge Both List"**  
   - Connect the first input to the first output of "Loop Over Items" (tech news).  
   - Connect the second input to the first output of "Loop Over Items1" (tools).  
   - Merge mode defaults to “append”.

8. **Add Convert To File node "Conver To CSV"**  
   - Connect from "Merge Both List".  
   - Set file format to CSV.

9. **Add Google Gemini node "Upload a file"**  
   - Resource: file  
   - InputType: binary  
   - Connect from "Conver To CSV".  
   - Add Google Gemini (PaLM) API credentials.

10. **Add Google Gemini node "Analyze document"**  
    - Resource: document  
    - Model: "models/gemini-2.5-flash-preview-05-20"  
    - Text prompt: Use the detailed prompt defined in the workflow (includes instructions for selecting top items, keywords, and HTML formatting).  
    - Document URLs: `={{ $json.fileUri }}` from "Upload a file".  
    - Connect from "Upload a file".  
    - Add Google Gemini credentials.

11. **Add Code node "Code in JavaScript"**  
    - JavaScript code to clean AI-generated HTML:  
      ```javascript
      let htmlText = $input.first().json.content.parts[0].text;
      htmlText = htmlText.replace(/\n/g, "").trim();
      return [{ json: { html: htmlText } }];
      ```  
    - Connect from "Analyze document".

12. **Add Email Send node "Send email"**  
    - Subject: `=⚡ Daily Tech Digest - {{ new Date().toLocaleDateString("en-US", { year: "numeric", month: "long", day: "numeric" }) }}`  
    - To Email: Set your recipient email address.  
    - From Email: Set your sender email address.  
    - HTML: `={{ $json.html.replace(/^```html/, '').replace(/```$/, '').trim() }}`  
    - Connect from "Code in JavaScript".  
    - Configure SMTP credentials.

13. **Add Sticky Notes for documentation** (optional but recommended)  
    - Overview of workflow  
    - Trigger description  
    - RSS sources explanation  
    - Processing explanation  
    - Combining and conversion explanation  
    - AI curation explanation  
    - Email sending instructions  

14. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                             |
|--------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| The workflow uses two different source lists (Tech News and Tools/Product feeds) for targeted curation.| Sticky note near "Tools Subredit and Sites" and "Tech News Sites" nodes.                                   |
| Google Gemini AI’s prompt can be customized by changing keywords and HTML template inside the "Analyze document" node. | Allows tailoring the newsletter content and format.                                                        |
| SMTP credentials must be configured with valid sender and recipient email addresses for delivery.      | Refer to "Send email" node sticky note for setup instructions.                                             |
| The RSS Feed Read nodes are configured to continue on error, minimizing workflow failure due to unreachable feeds.| Enhances robustness against transient feed issues.                                                         |
| The workflow is scheduled to run daily at 9 AM; adjust Schedule Trigger node for different timing if needed.| See "Schedule Trigger1" node and sticky note on trigger.                                                   |
| The HTML template used by AI is optimized for readability and modern styling suitable for email clients.| Included in the prompt text of "Analyze document" node.                                                    |

---

This detailed analysis and reconstruction guide will enable users and AI agents to understand, reproduce, and modify the personalized daily newsletter workflow efficiently and reliably.