Web Content to Telegram Publisher with AI Enhancement & Image Watermarking

https://n8nworkflows.xyz/workflows/web-content-to-telegram-publisher-with-ai-enhancement---image-watermarking-9441


# Web Content to Telegram Publisher with AI Enhancement & Image Watermarking

### 1. Workflow Overview

This workflow automates the process of fetching news content from a specified website, enhancing it using AI, watermarking associated images, and publishing the final enriched posts to a Telegram channel. It is designed for users who want to keep their Telegram channels regularly updated with fresh, AI-polished content sourced from web pages, ensuring unique presentation and branding with image watermarking.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Web Content Fetching:** Periodic trigger to fetch the news webpage, extract relevant article links and images, and clean these links for further processing.

- **1.2 Link Management and Filtering:** Check extracted links against a Google Sheet to avoid reposting old content and decide whether to process new articles.

- **1.3 Article Extraction and Cleaning:** If a new link is detected, open the article page, extract the title and text, and clean unwanted elements from the text.

- **1.4 AI-Based Text Processing:** Use an AI language model to rewrite or condense the article text into a well-formatted, Telegram-ready message.

- **1.5 Image Retrieval and Watermarking:** Fetch the article's main image, apply a watermark text to it for branding purposes.

- **1.6 Publishing to Telegram:** Send the final formatted text along with the watermarked image as a photo post to the Telegram channel.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Web Content Fetching

- **Overview:**  
This block triggers the workflow on an hourly schedule, fetches the target news webpage, extracts article links and images from the HTML content, and cleans these extracted URLs to generate usable links.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Fetch News Page  
  - Extract Links from HTML  
  - Clean and Trim Links  

- **Node Details:**

1. **Schedule Trigger**  
   - Type: Trigger  
   - Role: Initiates workflow execution every hour.  
   - Configuration: Interval set to 1 hour.  
   - Input: None (triggered automatically)  
   - Output: Triggers "Fetch News Page" node.  
   - Edge Cases: Workflow may not trigger if n8n instance is down or scheduling is disabled.

2. **Fetch News Page**  
   - Type: HTTP Request  
   - Role: Retrieves the raw HTML content of the news page at https://www.film.ru/topic/news.  
   - Configuration:  
     - Sends custom headers including User-Agent, Accept-Language, Referer, and Accept to mimic a browser request.  
     - Method: GET (default).  
   - Input: Trigger from Schedule Trigger.  
   - Output: HTML content forwarded to "Extract Links from HTML".  
   - Edge Cases: Possible HTTP errors such as 404, 503, or timeouts; site structure changes could break downstream parsing.

3. **Extract Links from HTML**  
   - Type: HTML Extract  
   - Role: Parses the fetched HTML to extract article links and image references using CSS selectors.  
   - Configuration:  
     - Extracts 'link' using selector ".redesign_topic_main strong".  
     - Extracts 'image' using selector ".wrapper_block_stack var".  
   - Input: HTML content from "Fetch News Page".  
   - Output: Raw extracted link and image data to "Clean and Trim Links".  
   - Edge Cases: If the page structure changes or selectors become invalid, extraction will fail or return empty values.

4. **Clean and Trim Links**  
   - Type: Code (JavaScript)  
   - Role: Cleans extracted strings by parsing out URLs from bracketed patterns and prepends the base URL "https://www.film.ru".  
   - Key Expressions:  
     - Uses regex to extract URLs inside square brackets.  
     - Constructs absolute URLs for both article link and image.  
   - Input: Raw extracted link and image from previous node.  
   - Output: Cleaned full URLs passed to "Check Links in Google Sheet".  
   - Edge Cases: Malformed input strings could cause regex failures; missing or unexpected formats lead to errors.

---

#### 1.2 Link Management and Filtering

- **Overview:**  
This block checks if the extracted article link is already present in a Google Sheet to avoid reposting old content. If the link is new, it proceeds to fetch the article content; otherwise, it halts processing.

- **Nodes Involved:**  
  - Check Links in Google Sheet  
  - Check: Should We Parse?  
  - No Operation, do nothing (used to stop processing if link exists)  
  - Open Article by Link  

- **Node Details:**

1. **Check Links in Google Sheet**  
   - Type: Google Sheets  
   - Role: Reads from a preconfigured Google Sheet to verify if the article URL exists.  
   - Configuration:  
     - Connects to a specific Google Sheet titled "film.ru test" using OAuth2 credentials.  
     - Reads from a sheet with ID 1839495031.  
   - Input: Cleaned link from "Clean and Trim Links".  
   - Output: Passes data to "Check: Should We Parse?" for conditional logic.  
   - Edge Cases: Google Sheets API quota limits or credential expiry; sheet structure changes may cause failures.

2. **Check: Should We Parse?**  
   - Type: If node  
   - Role: Compares the current link against URLs in the Google Sheet to decide whether to parse the article.  
   - Configuration:  
     - Condition checks if the current JSON url equals the cleaned full URL from the previous block.  
     - If equal (link exists), routes to "No Operation, do nothing" (stop).  
     - If not equal (new link), routes to "Open Article by Link".  
   - Input: Data from Google Sheets.  
   - Output: Conditional branch to either no-op or article fetching.  
   - Edge Cases: Expression evaluation errors if input data is missing or malformed.

3. **No Operation, do nothing**  
   - Type: NoOp  
   - Role: Terminates workflow branch when the article link already exists.  
   - Configuration: None.  
   - Input: From "Check: Should We Parse?" negative branch.  
   - Output: None (ends this branch).  
   - Edge Cases: None.

4. **Open Article by Link**  
   - Type: HTTP Request  
   - Role: Fetches the full article page content from the cleaned URL for further extraction.  
   - Configuration:  
     - URL dynamically set to the article link from "Clean and Trim Links".  
     - Method: GET (default).  
   - Input: Triggered only if link is new.  
   - Output: Sends HTML to "Extract Article Description".  
   - Edge Cases: HTTP errors, site structure changes, slow response.

---

#### 1.3 Article Extraction and Cleaning

- **Overview:**  
Extracts the article title and text from the fetched HTML, then cleans unwanted parts (e.g., bracketed annotations) from the text.

- **Nodes Involved:**  
  - Extract Article Description  
  - Clean and Trim Text  
  - Update Google Sheet with Clean Links  

- **Node Details:**

1. **Extract Article Description**  
   - Type: HTML Extract  
   - Role: Parses the article HTML to extract the main title and text content.  
   - Configuration:  
     - Extracts "Title" using CSS selector ".wrapper_articles_left h1".  
     - Extracts "Text" using ".wrapper_articles_text".  
   - Input: HTML from "Open Article by Link".  
   - Output: Passes extracted raw title and text to "Clean and Trim Text".  
   - Edge Cases: If selectors fail due to page redesign, extraction yields empty or incorrect data.

2. **Clean and Trim Text**  
   - Type: Code (JavaScript)  
   - Role: Cleans the extracted article text by removing all substrings within square brackets (e.g., references or annotations).  
   - Key Expressions:  
     - Uses regex to globally remove bracketed text from the article body.  
   - Input: Extracted article text.  
   - Output: Cleaned text forwarded to "Update Google Sheet with Clean Links".  
   - Edge Cases: Unexpected text formats might cause partial cleaning or errors.

3. **Update Google Sheet with Clean Links**  
   - Type: Google Sheets  
   - Role: Updates the Google Sheet to mark the article URL as processed, preventing future duplication.  
   - Configuration:  
     - Updates row number 2 with the new URL (fixed row in this config).  
     - Uses OAuth2 credentials.  
   - Input: Cleaned text and URL data.  
   - Output: Triggers the AI text processing block.  
   - Edge Cases: API errors, concurrency conflicts in sheet updates.

---

#### 1.4 AI-Based Text Processing

- **Overview:**  
This block uses an AI language model (OpenAI GPT-4) to rewrite or summarize the cleaned article text into a polished, Telegram-friendly message with formatting and emojis, respecting character limits.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - Text processing  

- **Node Details:**

1. **OpenAI Chat Model**  
   - Type: Langchain OpenAI Chat Model  
   - Role: Provides AI-powered text rewriting and summarization.  
   - Configuration:  
     - Model: "gpt-4.1-mini" (a GPT-4 variant).  
     - No additional options enabled.  
   - Input: Raw text prompt from "Text processing" node.  
   - Output: AI-generated formatted text passed back to "Text processing".  
   - Edge Cases: API rate limits, invalid API key, prompt errors.

2. **Text processing**  
   - Type: Langchain Agent  
   - Role: Defines the AI prompt and formatting instructions for the OpenAI model.  
   - Configuration:  
     - System message instructs the AI to act as a Telegram news editor, applying bold, italics, and emojis, condensing text over 700 characters.  
     - Input text includes the article title and cleaned article text with separators.  
   - Input: Cleaned article text and title from prior nodes.  
   - Output: Final AI-processed text ready for Telegram caption.  
   - Edge Cases: Incorrect prompt formatting may yield suboptimal AI responses.

---

#### 1.5 Image Retrieval and Watermarking

- **Overview:**  
Fetches the article's main image and applies a consistent watermark text for branding.

- **Nodes Involved:**  
  - Get Article Image  
  - Add Watermark Text to Image  

- **Node Details:**

1. **Get Article Image**  
   - Type: HTTP Request (binary)  
   - Role: Downloads the article image from the cleaned image URL.  
   - Configuration:  
     - URL dynamically set from "Clean and Trim Links".  
     - Binary data retrieval enabled.  
   - Input: AI-processed text output triggers image fetch.  
   - Output: Binary image data forwarded to watermarking node.  
   - Edge Cases: Broken image URLs, large image sizes causing timeouts.

2. **Add Watermark Text to Image**  
   - Type: Edit Image  
   - Role: Adds the watermark text "KINO ЛЕГЕНДЫ NET" onto the downloaded image.  
   - Configuration:  
     - Text: "KINO ЛЕГЕНДЫ NET"  
     - Font: Verdana Bold Italic (path: /usr/share/fonts/truetype/msttcorefonts/Verdana_Bold_Italic.ttf)  
     - Font size: 48  
     - Font color: Light gray (#F3ECEC)  
     - Position on image: Y offset 150 pixels  
     - Operates on binary data property named dynamically (=data)  
   - Input: Binary image from "Get Article Image".  
   - Output: Watermarked image sent to Telegram posting node.  
   - Edge Cases: Missing font files, image format incompatibilities.

---

#### 1.6 Publishing to Telegram

- **Overview:**  
Sends the final AI-enhanced text and watermarked image as a photo post to a Telegram channel using a configured Telegram bot.

- **Nodes Involved:**  
  - Send Post to Telegram  

- **Node Details:**

1. **Send Post to Telegram**  
   - Type: Telegram node  
   - Role: Publishes the photo and caption to the Telegram channel.  
   - Configuration:  
     - Operation: "sendPhoto" with binary image data.  
     - Caption: Uses AI-processed formatted text.  
     - Parse mode: Markdown for Telegram formatting.  
     - Uses stored Telegram API credentials linked to the bot.  
   - Input: Watermarked image and AI-processed text.  
   - Output: Final publishing action, end of workflow branch.  
   - Edge Cases: Telegram API limits, invalid bot token, network errors.

---

### 3. Summary Table

| Node Name                    | Node Type                      | Functional Role                           | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                                               |
|------------------------------|--------------------------------|------------------------------------------|-------------------------------|-------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger             | Schedule Trigger               | Triggers workflow hourly                  | None                          | Fetch News Page               | **1️. Fetch & Parse Website** Runs on schedule, fetches a webpage via HTTP, extracts and cleans article links using HTML and JavaScript. |
| Fetch News Page             | HTTP Request                  | Fetches news webpage HTML                  | Schedule Trigger              | Extract Links from HTML       |                                                                                                                            |
| Extract Links from HTML     | HTML Extract                  | Extracts raw article links and images      | Fetch News Page               | Clean and Trim Links          |                                                                                                                            |
| Clean and Trim Links        | Code                         | Cleans extracted strings to full URLs      | Extract Links from HTML       | Check Links in Google Sheet   |                                                                                                                            |
| Check Links in Google Sheet | Google Sheets                | Checks if article link is already processed | Clean and Trim Links          | Check: Should We Parse?       | **2️. Check & Update Links** Reads the Google Sheet to skip old links. If new → extracts article text and updates the sheet. |
| Check: Should We Parse?     | If                           | Decides whether to parse article or stop   | Check Links in Google Sheet   | No Operation, do nothing / Open Article by Link |                                                                                                                            |
| No Operation, do nothing    | NoOp                         | Stops processing for existing links        | Check: Should We Parse?       | None                         |                                                                                                                            |
| Open Article by Link        | HTTP Request                 | Fetches full article HTML                   | Check: Should We Parse?       | Extract Article Description   |                                                                                                                            |
| Extract Article Description | HTML Extract                 | Extracts article title and text             | Open Article by Link          | Clean and Trim Text           |                                                                                                                            |
| Clean and Trim Text         | Code                         | Cleans article text of unwanted elements   | Extract Article Description   | Update Google Sheet with Clean Links |                                                                                                                            |
| Update Google Sheet with Clean Links | Google Sheets          | Updates sheet marking link as processed    | Clean and Trim Text           | Text processing              |                                                                                                                            |
| OpenAI Chat Model           | Langchain OpenAI Chat Model | AI model for rewriting and summarizing     | Text processing              | Text processing              | **3️. Process Content with AI** Uses an AI Agent to rewrite or summarize the article text for clarity and tone.              |
| Text processing             | Langchain Agent              | Defines AI prompt and formats the text     | Update Google Sheet with Clean Links / OpenAI Chat Model | Get Article Image            |                                                                                                                            |
| Get Article Image           | HTTP Request (binary)        | Downloads article image                      | Text processing              | Add Watermark Text to Image  | **4️. Prepare & Publish** Fetches the article image, adds a watermark, and sends the final post to your Telegram channel.    |
| Add Watermark Text to Image | Edit Image                   | Adds watermark text to downloaded image     | Get Article Image            | Send Post to Telegram        |                                                                                                                            |
| Send Post to Telegram       | Telegram                     | Sends photo post with caption to Telegram  | Add Watermark Text to Image  | None                         |                                                                                                                            |
| No Operation, do nothing    | NoOp                         | Stops branch execution on duplicate links  | Check: Should We Parse?       | None                         |                                                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set interval to every 1 hour.

2. **Create an HTTP Request node "Fetch News Page"**  
   - URL: https://www.film.ru/topic/news  
   - Add headers:  
     - User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/127.0.0.1 Safari/537.36  
     - Accept-Language: ru-RU,ru;q=0.9,en-US;q=0.8,en;q=0.7  
     - Referer: https://www.kinonews.ru/  
     - Accept: text/html,application/xhtml+xml,application/xml;q=0.9,/;q=0.8  
   - Connect Schedule Trigger → Fetch News Page.

3. **Create an HTML Extract node "Extract Links from HTML"**  
   - Operation: Extract HTML content.  
   - Extraction keys:  
     - link: CSS selector ".redesign_topic_main strong"  
     - image: CSS selector ".wrapper_block_stack var"  
   - Connect Fetch News Page → Extract Links from HTML.

4. **Create a Code node "Clean and Trim Links"**  
   - JavaScript code:  
   ```js
   const baseUrl = "https://www.film.ru";
   const latest = items[0].json;

   const cleanLink = latest.link.match(/\[([^\]]+)\]/)[1];
   const cleanImage = latest.image.match(/\[([^\]]+)\]/)[1];

   return [{
     json: {
       image: baseUrl + cleanImage,
       fullUrl: baseUrl + cleanLink
     }
   }];
   ```
   - Connect Extract Links from HTML → Clean and Trim Links.

5. **Create a Google Sheets node "Check Links in Google Sheet"**  
   - Operation: Read rows from sheet named "film.ru test".  
   - Credentials: Setup OAuth2 for Google Sheets.  
   - Connect Clean and Trim Links → Check Links in Google Sheet.

6. **Create an If node "Check: Should We Parse?"**  
   - Condition: Check if the current item URL equals the fullUrl from "Clean and Trim Links".  
   - True branch: Connect to a No Operation node to stop (create No Operation node "No Operation, do nothing").  
   - False branch: Connect to "Open Article by Link".  
   - Connect Check Links in Google Sheet → Check: Should We Parse?.

7. **Create HTTP Request node "Open Article by Link"**  
   - URL: Expression from "Clean and Trim Links" → fullUrl.  
   - Connect False branch of Check: Should We Parse? → Open Article by Link.

8. **Create HTML Extract node "Extract Article Description"**  
   - Extract keys:  
     - Title: CSS selector ".wrapper_articles_left h1"  
     - Text: CSS selector ".wrapper_articles_text"  
   - Connect Open Article by Link → Extract Article Description.

9. **Create Code node "Clean and Trim Text"**  
   - JavaScript code:  
   ```js
   return $input.all().map(item => {
     const text = item.json.Text;
     const cleanText = text.replace(/\[[^\]]*\]/g, '');
     return { json: { cleanText } };
   });
   ```
   - Connect Extract Article Description → Clean and Trim Text.

10. **Create Google Sheets node "Update Google Sheet with Clean Links"**  
    - Operation: Update operation on sheet "film.ru test".  
    - Update row 2 with new url from "Clean and Trim Links".  
    - Connect Clean and Trim Text → Update Google Sheet with Clean Links.

11. **Create Langchain OpenAI Chat Model node "OpenAI Chat Model"**  
    - Model: gpt-4.1-mini (ensure API key and credentials configured).  
    - Connect this node AI input to "Text processing".

12. **Create Langchain Agent node "Text processing"**  
    - Text prompt:  
      ```
      You are a news editor for a Telegram channel. Your responsibilities include only adding beautiful formatting to the news text—without changing the actual text. If the original text exceeds 700 characters, condense it in a way that preserves the meaning and ensures the text never exceeds 700 characters.

      Use Telegram's built-in formatting options, such as **Bold** and *Italic*, as well as appropriate emojis.

      Create a beautifully formatted version of the news titled:
      "{{ $('Extract Article Description').item.json.Title }}"

      ---
      {{ $('Clean and Trim Text').item.json.cleanText }}

      ---
      The response should be a ready-to-publish text.
      ```
    - Connect Update Google Sheet with Clean Links → Text processing.  
    - Connect OpenAI Chat Model → Text processing (AI language model input).

13. **Create HTTP Request node "Get Article Image"**  
    - URL expression: `{{ $('Clean and Trim Links').item.json.image }}`  
    - Binary data enabled to fetch image.  
    - Connect Text processing → Get Article Image.

14. **Create Edit Image node "Add Watermark Text to Image"**  
    - Operation: Add text watermark.  
    - Text: "KINO ЛЕГЕНДЫ NET"  
    - Font: Verdana Bold Italic (ensure font file exists at `/usr/share/fonts/truetype/msttcorefonts/Verdana_Bold_Italic.ttf`)  
    - Font size: 48  
    - Font color: #F3ECEC  
    - Position Y: 150  
    - Data property: set dynamically to binary image data.  
    - Connect Get Article Image → Add Watermark Text to Image.

15. **Create Telegram node "Send Post to Telegram"**  
    - Operation: sendPhoto  
    - Binary Data: true (send image)  
    - Caption: use AI processed text output from "Text processing" node.  
    - Parse mode: Markdown  
    - Credentials: Configure Telegram bot API credentials.  
    - Connect Add Watermark Text to Image → Send Post to Telegram.

---

### 5. General Notes & Resources

| Note Content                                                                                                                 | Context or Link                                                                                 |
|------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| AI Telegram Web Parser: Automatically parses a website on schedule, rewrites the article text with AI, adds a watermark to image, and publishes it to Telegram. Flow: Fetch → Parse → Rewrite → Add watermark → Publish | Sticky Note7 node content                                                                        |
| Use Case: Keep your Telegram channel updated with AI-enhanced posts from any website. Requirements: Schedule trigger, Google Sheet, AI Agent, Telegram Bot | Sticky Note7 node content                                                                        |
| **1️. Fetch & Parse Website** Runs on schedule, fetches a webpage via HTTP, extracts and cleans article links using HTML and JavaScript. | Sticky Note node near Schedule Trigger and Fetch News Page                                      |
| **2️. Check & Update Links** Reads the Google Sheet to skip old links. If new → extracts article text and updates the sheet. | Sticky Note1 node near Google Sheets nodes                                                     |
| **3️. Process Content with AI** Uses an AI Agent to rewrite or summarize the article text for clarity and tone.               | Sticky Note2 node near AI processing nodes                                                    |
| **4️. Prepare & Publish** Fetches the article image, adds a watermark, and sends the final post to your Telegram channel.    | Sticky Note3 node near image and Telegram nodes                                               |
| Workflow Bot Credentials: Ensure the Telegram API credentials used have permissions to send images and messages to your target channel. | Telegram node "Send Post to Telegram"                                                          |
| Google Sheets Credentials: Use OAuth2 credentials with access to the specific Google Sheet for link management.              | Google Sheets nodes "Check Links in Google Sheet" and "Update Google Sheet with Clean Links"   |
| OpenAI API Key: Required for Langchain OpenAI Chat Model node to enable AI rewriting.                                         | OpenAI Chat Model node                                                                         |
| Font file for watermarking must be present on the n8n host machine at /usr/share/fonts/truetype/msttcorefonts/Verdana_Bold_Italic.ttf | Add Watermark Text to Image node                                                              |

---

**Disclaimer:**  
The text provided is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.