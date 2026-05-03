Ideal Customer Profile (ICP) Generation: AI, Firecrawl, Gemini, Telegram

https://n8nworkflows.xyz/workflows/ideal-customer-profile--icp--generation--ai--firecrawl--gemini--telegram-5047


# Ideal Customer Profile (ICP) Generation: AI, Firecrawl, Gemini, Telegram

# Ideal Customer Profile (ICP) Generation: AI, Firecrawl, Gemini, Telegram  
---

## 1. Workflow Overview

This workflow automates the generation of an Ideal Customer Profile (ICP) by combining web scraping, AI language models, and Telegram messaging for interactive user requests. It is designed to extract relevant data from user-provided URLs, process the content with advanced AI (Google Gemini and LangChain), and deliver structured insights back to the user via Telegram. The workflow efficiently handles single-page and multi-page scraping scenarios, ensuring robust data acquisition before AI processing.

The workflow can be logically divided into the following functional blocks:

- **1.1 Input Reception and Parsing:** Receives user requests from Telegram, extracts the target URL and the number of pages to scrape.
- **1.2 Scraping Decision and Execution:** Determines whether to scrape a single page or multiple pages based on user input, then performs the scraping using appropriate endpoints.
- **1.3 Post-Scraping Wait and Content Retrieval:** Waits for scrape completion (especially for multi-page scrapes) and retrieves the scraped content.
- **1.4 AI Processing and ICP Generation:** Uses Google Gemini and LangChain nodes to analyze scraped data and generate ICP insights.
- **1.5 Output Formatting and Delivery:** Converts AI output to file formats and sends results back to users via Telegram.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception and Parsing

**Overview:**  
This block listens to Telegram incoming messages, extracts relevant parameters like URL and the number of pages, and sets them for downstream processing.

**Nodes Involved:**  
- User Request - Telegram  
- Google Gemini Chat Model4  
- Structured Output Parser  
- Extracts URL and No of Pages  
- Sets url and no of pages

**Node Details:**

- **User Request - Telegram**  
  - *Type:* Telegram Trigger  
  - *Role:* Entry point for user input via Telegram messages.  
  - *Key Configuration:* Listens for messages on a configured Telegram bot webhook.  
  - *Connections:* Outputs to “Extracts URL and No of Pages”.  
  - *Failure Modes:* Telegram API connectivity issues, webhook misconfiguration, malformed messages.

- **Google Gemini Chat Model4**  
  - *Type:* Google Gemini Chat Language Model  
  - *Role:* Processes user input to help parse or understand commands.  
  - *Config:* Uses Google Gemini credentials; no additional parameters specified here (likely default prompt).  
  - *Connections:* Outputs to “Extracts URL and No of Pages”.  
  - *Failures:* API quota limits, auth errors, latency.

- **Structured Output Parser**  
  - *Type:* LangChain Structured Output Parser  
  - *Role:* Parses AI model outputs into structured data (e.g., extracting URL and number of pages).  
  - *Config:* Uses a structured parser template (not detailed in JSON).  
  - *Connections:* Outputs to “Extracts URL and No of Pages” via ai_outputParser channel.  
  - *Failures:* Parsing errors if AI output format changes or unexpected.

- **Extracts URL and No of Pages**  
  - *Type:* LangChain Agent  
  - *Role:* Extracts and confirms URL and page count from inputs.  
  - *Connections:* Outputs to “Sets url and no of pages”.  
  - *Failures:* Expression failures, incomplete data extraction.

- **Sets url and no of pages**  
  - *Type:* Set Node  
  - *Role:* Stores URL and page count into workflow variables for branching logic.  
  - *Connections:* Outputs to “If page 1 (true) or more than 1 (false)”.  
  - *Failures:* Variable misassignment or missing inputs.

---

### 2.2 Scraping Decision and Execution

**Overview:**  
Determines if the scraping task is single-page or multi-page, then routes accordingly to specific endpoints for scraping.

**Nodes Involved:**  
- If page 1 (true) or more than 1 (false)  
- Scrape One Page - user /scrape endpoint  
- Scrapes more than one page - uses /crawl endpoint

**Node Details:**

- **If page 1 (true) or more than 1 (false)**  
  - *Type:* If Node  
  - *Role:* Branching node that checks if the number of pages is exactly 1 or more.  
  - *Config:* Condition based on the stored page count variable.  
  - *Connections:*  
    - True branch → “Scrape One Page - user /scrape endpoint”  
    - False branch → “Scrapes more than one page - uses /crawl endpoint”  
  - *Failures:* Logic errors if page number is invalid or unset.

- **Scrape One Page - user /scrape endpoint**  
  - *Type:* HTTP Request  
  - *Role:* Makes an HTTP request to a scraping endpoint dedicated to single-page scraping.  
  - *Config:* Parameters likely include URL from previous nodes; endpoint is `/scrape`.  
  - *Connections:* Outputs to “Basic LLM Chain”.  
  - *Failures:* HTTP errors, endpoint unavailability, timeout, invalid URL.

- **Scrapes more than one page - uses /crawl endpoint**  
  - *Type:* HTTP Request  
  - *Role:* Invokes a multi-page scraping endpoint, probably asynchronous.  
  - *Config:* Uses `/crawl` endpoint, with URL and page count as parameters.  
  - *Connections:* Outputs to “Wait 60 seconds - for scraping”.  
  - *Failures:* Same as above, plus potential delays in scrape readiness.

---

### 2.3 Post-Scraping Wait and Content Retrieval

**Overview:**  
Waits for a fixed period to ensure scraping completion (for multi-page scraping), then fetches the scraped content for AI processing.

**Nodes Involved:**  
- Wait 60 seconds - for scraping  
- GET - the scraped content  
- Basic LLM Chain1

**Node Details:**

- **Wait 60 seconds - for scraping**  
  - *Type:* Wait Node  
  - *Role:* Delays workflow execution by 60 seconds to allow scraping to finish.  
  - *Connections:* Outputs to “GET - the scraped content”.  
  - *Failures:* If scrape takes longer, data might be incomplete.

- **GET - the scraped content**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves the scraped data from the scraping service.  
  - *Config:* Uses URL or token from prior steps to GET the content.  
  - *Connections:* Outputs to “Basic LLM Chain1”.  
  - *Failures:* HTTP errors, data unavailability.

- **Basic LLM Chain1**  
  - *Type:* LangChain LLM Chain  
  - *Role:* Processes the scraped content with AI to extract meaningful ICP data.  
  - *Config:* Uses Google Gemini Chat Model3 as AI backend.  
  - *Connections:* Outputs to “Convert to File”.  
  - *Failures:* Model API errors, content format issues.

---

### 2.4 AI Processing and ICP Generation

**Overview:**  
Uses advanced AI chains and models to analyze the scraped content and generate detailed Ideal Customer Profile insights.

**Nodes Involved:**  
- Basic LLM Chain  
- Google Gemini Chat Model2  
- Convert to File1  
- Convert to File

**Node Details:**

- **Basic LLM Chain**  
  - *Type:* LangChain LLM Chain  
  - *Role:* Core AI chain for processing ICP data.  
  - *Config:* Uses Google Gemini Chat Model2 as backend.  
  - *Connections:* Outputs to “Convert to File1”.  
  - *Failures:* AI API errors, data formatting failures.

- **Google Gemini Chat Model2**  
  - *Type:* Google Gemini Chat Model  
  - *Role:* AI Language Model used by Basic LLM Chain.  
  - *Config:* Google Gemini credentials configured.  
  - *Connections:* Feeds “Basic LLM Chain”.  
  - *Failures:* Same as other Gemini nodes.

- **Convert to File1**  
  - *Type:* Convert to File  
  - *Role:* Converts AI output into a file format suitable for Telegram delivery (likely text or JSON).  
  - *Connections:* Outputs to Telegram node for sending results.  
  - *Failures:* File conversion errors.

- **Convert to File**  
  - *Type:* Convert to File  
  - *Role:* Converts another AI output (from Basic LLM Chain1) into file format.  
  - *Connections:* Outputs to “Telegram” node for sending.  
  - *Failures:* Same as above.

---

### 2.5 Output Formatting and Delivery

**Overview:**  
Sends generated ICP files back to users through Telegram.

**Nodes Involved:**  
- Telegram

**Node Details:**

- **Telegram**  
  - *Type:* Telegram Node (Send message/file)  
  - *Role:* Delivers files containing detailed ICP insights back to the user.  
  - *Config:* Uses configured Telegram bot credentials and chat IDs from triggers.  
  - *Connections:* Terminal node for both file conversion outputs.  
  - *Failures:* Telegram API limits, user chat permissions, file size constraints.

---

## 3. Summary Table

| Node Name                                | Node Type                            | Functional Role                          | Input Node(s)                        | Output Node(s)                         | Sticky Note                                       |
|-----------------------------------------|------------------------------------|----------------------------------------|------------------------------------|--------------------------------------|--------------------------------------------------|
| User Request - Telegram                  | Telegram Trigger                   | Entry point for user input              | -                                  | Extracts URL and No of Pages           |                                                  |
| Google Gemini Chat Model4                | Google Gemini Chat Model           | Parse user message                      | -                                  | Extracts URL and No of Pages           |                                                  |
| Structured Output Parser                 | LangChain Structured Output Parser| Parse AI output into structured data   | - (ai_outputParser from Gemini)    | Extracts URL and No of Pages           |                                                  |
| Extracts URL and No of Pages             | LangChain Agent                   | Extract URL and pages from input        | User Request, Gemini, Parser       | Sets url and no of pages               |                                                  |
| Sets url and no of pages                 | Set                              | Store URL and page count                 | Extracts URL and No of Pages       | If page 1 (true) or more than 1 (false)|                                                  |
| If page 1 (true) or more than 1 (false) | If                               | Branch scrape method                     | Sets url and no of pages           | Scrape One Page, Scrapes more than one page |                                                  |
| Scrape One Page - user /scrape endpoint | HTTP Request                     | Single-page scraping                     | If page node (true branch)         | Basic LLM Chain                       |                                                  |
| Scrapes more than one page - /crawl endpoint | HTTP Request                | Multi-page scraping                      | If page node (false branch)        | Wait 60 seconds - for scraping         |                                                  |
| Wait 60 seconds - for scraping           | Wait                             | Delay for multi-page scraping completion| Scrapes more than one page         | GET - the scraped content             |                                                  |
| GET - the scraped content                 | HTTP Request                     | Retrieve scraped data                    | Wait 60 seconds                   | Basic LLM Chain1                      |                                                  |
| Basic LLM Chain1                          | LangChain LLM Chain              | AI processing of scraped content        | GET - the scraped content          | Convert to File                      |                                                  |
| Basic LLM Chain                           | LangChain LLM Chain              | AI processing (single page or final output)| Scrape One Page or others          | Convert to File1                     |                                                  |
| Google Gemini Chat Model2                  | Google Gemini Chat Model           | AI model backend for Basic LLM Chain    | -                                | Basic LLM Chain                      |                                                  |
| Google Gemini Chat Model3                  | Google Gemini Chat Model           | AI model backend for Basic LLM Chain1   | -                                | Basic LLM Chain1                     |                                                  |
| Convert to File1                          | Convert to File                   | Convert AI output to file                | Basic LLM Chain                    | Telegram                            |                                                  |
| Convert to File                           | Convert to File                   | Convert AI output to file                | Basic LLM Chain1                   | Telegram                            |                                                  |
| Telegram                                 | Telegram                         | Send results back to user                | Convert to File, Convert to File1 | -                                  |                                                  |
| Sticky Note (multiple nodes)              | Sticky Note                      | Documentation/comment                    | -                                | -                                  | Multiple sticky notes with no content             |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure with Telegram bot credentials and webhook URL.  
   - This node listens for incoming user messages.

2. **Add Google Gemini Chat Model4 Node**  
   - Type: Google Gemini Chat Model (LangChain)  
   - Configure with Google Gemini credentials.  
   - Connect Telegram Trigger output here to process user input.

3. **Add Structured Output Parser Node**  
   - Type: LangChain Structured Output Parser  
   - Configure parsing schema to extract URL and number of pages from AI response.  
   - Connect Google Gemini Chat Model4 ai_languageModel output to this parser’s input.

4. **Add LangChain Agent Node "Extracts URL and No of Pages"**  
   - Configure to extract and confirm URL and page count from parser output.  
   - Connect outputs from Telegram Trigger, Google Gemini Chat Model4, and Structured Output Parser appropriately.

5. **Add Set Node "Sets url and no of pages"**  
   - Configure to store extracted URL and number of pages into workflow variables.  
   - Connect output of Extracts URL and No of Pages node here.

6. **Add If Node "If page 1 (true) or more than 1 (false)"**  
   - Condition: Check if number of pages == 1.  
   - Connect output of Set Node here.

7. **Add HTTP Request Node "Scrape One Page - user /scrape endpoint"**  
   - Configure HTTP POST or GET to `/scrape` endpoint with URL parameter from variables.  
   - Connect If node’s true branch here.

8. **Add HTTP Request Node "Scrapes more than one page - uses /crawl endpoint"**  
   - Configure HTTP POST or GET to `/crawl` endpoint with URL and page count.  
   - Connect If node’s false branch here.

9. **Add Wait Node "Wait 60 seconds - for scraping"**  
   - Configure fixed delay of 60 seconds.  
   - Connect output of multi-page scrape node here.

10. **Add HTTP Request Node "GET - the scraped content"**  
    - Configure GET request to retrieve scraped content after delay.  
    - Connect Wait node output here.

11. **Add LangChain LLM Chain Node "Basic LLM Chain1"**  
    - Configure with Google Gemini Chat Model3 as backend.  
    - Connect GET node output here.  
    - Configure prompt/template to analyze scraped content for ICP insights.

12. **Add LangChain LLM Chain Node "Basic LLM Chain"**  
    - Configure with Google Gemini Chat Model2 as backend.  
    - Connect output of single-page scrape node and optionally multi-page scrape chain.  
    - Configure prompt/template accordingly.

13. **Add Convert to File Nodes**  
    - Two nodes: Convert to File and Convert to File1.  
    - Connect Basic LLM Chain1 output to Convert to File.  
    - Connect Basic LLM Chain output to Convert to File1.

14. **Add Telegram Node**  
    - Configure with Telegram bot credentials to send messages/files.  
    - Connect both Convert to File and Convert to File1 outputs here for final delivery.

15. **Configure all credentials**  
    - Google Gemini: API keys or OAuth as per Google Cloud configuration.  
    - Telegram: Bot token and webhook URL setup.  
    - HTTP Requests: API endpoints for scraping service with necessary authentication.

16. **Test workflow end-to-end**  
    - Send Telegram messages with URLs and page counts.  
    - Verify scraping, AI processing, and Telegram delivery.

---

## 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                  |
|------------------------------------------------------------------------------|-------------------------------------------------|
| Workflow integrates Google Gemini AI models via LangChain for advanced NLP. | See https://cloud.google.com/vertex-ai/docs/generative-ai for Gemini API details. |
| Telegram bot setup required with webhook URL configured in n8n instance.     | Telegram Bot API: https://core.telegram.org/bots/api |
| Uses Firecrawl scraping service endpoints `/scrape` and `/crawl`.           | Firecrawl documentation may be needed to configure endpoints and parameters. |
| Fixed 60-second wait for multi-page scraping to allow data readiness.        | Consider adjusting wait time based on service response time. |
| Structured Output Parser usage assumes consistent AI output schema.          | Changes in prompt or model version may require parser updates. |
| Sticky notes in workflow contain no content but are placeholders for comments.| Could be used to add documentation or instructions inside n8n editor. |

---

**Disclaimer:** The provided text is exclusively generated from an automated n8n workflow. It strictly complies with content policies and contains no illegal or protected elements. All data handled is legal and publicly accessible.