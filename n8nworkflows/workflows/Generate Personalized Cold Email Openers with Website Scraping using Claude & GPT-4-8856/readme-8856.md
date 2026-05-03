Generate Personalized Cold Email Openers with Website Scraping using Claude & GPT-4

https://n8nworkflows.xyz/workflows/generate-personalized-cold-email-openers-with-website-scraping-using-claude---gpt-4-8856


# Generate Personalized Cold Email Openers with Website Scraping using Claude & GPT-4

---

## 1. Workflow Overview

This workflow automates the generation of personalized cold email opening sentences by enriching lead data with business insights extracted from company websites. It targets sales and marketing teams aiming to improve outreach effectiveness through tailored messaging.

**Use Cases:**
- Automatically process new lead lists uploaded to a specific Google Drive folder.
- Extract and summarize company information from lead domains' websites.
- Generate personalized, emotionally compelling cold email openers using AI models (Claude by Anthropic & GPT-4).
- Update the enriched data back into the Google Sheet.
- Notify stakeholders via Telegram when enrichment completes.

**Logical Blocks:**

- **1.1 Input Reception and Lead Ingestion:** Detects new files in Google Drive, downloads and parses lead data.
- **1.2 Website Content Retrieval and Analysis:** For each lead, scrapes website content using Jina AI, and generates structured business summaries via AI agents.
- **1.3 Website Content Validation:** Uses OpenAI GPT-4 to verify if scraped content is meaningful business data.
- **1.4 Personalized Cold Email Opener Generation:** Branches into two paths based on website data presence—generates personalized openers either with or without website insights.
- **1.5 Data Update and Notification:** Writes enriched data back into the Google Sheet and sends a Telegram notification upon completion.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception and Lead Ingestion

**Overview:**  
This block monitors a Google Drive folder for new lead files, downloads them, and parses the spreadsheet to prepare the data for processing.

**Nodes Involved:**  
- Google Drive Trigger  
- Google Drive_DownLoad  
- Google Sheet Parse  
- Loop Over Items1

**Node Details:**

- **Google Drive Trigger**  
  - Type: Trigger node listening for file creation.  
  - Config: Watches specifically the folder with ID "1-b1yzCQA8RzQmG4vOaGKzYY8iZQxKpvf" every minute.  
  - Role: Initiates workflow when a new lead file is uploaded.  
  - Edge Cases: Missing folder permissions; delayed trigger due to polling interval.

- **Google Drive_DownLoad**  
  - Type: File download node.  
  - Config: Downloads the newly detected file using file ID from trigger.  
  - Role: Retrieves the lead file content.  
  - Edge Cases: File access denied; file deleted before download.

- **Google Sheet Parse**  
  - Type: Spreadsheet parser.  
  - Config: Parses file with header row enabled.  
  - Role: Converts spreadsheet rows into JSON objects for iteration.  
  - Edge Cases: Malformed spreadsheet; missing headers.

- **Loop Over Items1**  
  - Type: Split in batches.  
  - Config: Default batch size (likely 1) to process rows individually.  
  - Role: Allows sequential processing per lead record.  
  - Edge Cases: Empty input array; large batch sizes causing timeouts.

---

### 2.2 Website Content Retrieval and Analysis

**Overview:**  
For each lead, retrieves website markdown content via Jina AI and generates a structured business summary with an AI agent specialized in business analysis.

**Nodes Involved:**  
- AI Agent  
- Fetch Markdown via Jina AI1  
- Think  
- Anthropic Chat Model3  
- Sticky Notes (for documentation purposes)

**Node Details:**

- **Fetch Markdown via Jina AI1**  
  - Type: HTTP request tool node.  
  - Config: Calls `https://r.jina.ai/{url}` with header authentication to fetch website content as markdown.  
  - Role: Scrapes the lead's website domain for content extraction.  
  - Edge Cases: Website inaccessible; Jina API errors; rate limiting.

- **AI Agent**  
  - Type: LangChain agent node with custom prompt.  
  - Config: Uses the scraped markdown content to generate a structured business summary based on a defined template (company overview, products, target market, USP, etc.).  
  - System Message: Instructs agent to produce concise, factual summaries under 300 words.  
  - Role: Converts raw website data into meaningful business insights.  
  - Edge Cases: Poor or incomplete scraped data; agent timeout; API quota.

- **Think**  
  - Type: LangChain Think tool.  
  - Role: Intermediate step to control flow or debugging, triggers AI Agent node.  
  - Edge Cases: None specific.

- **Anthropic Chat Model3**  
  - Type: Anthropic Claude Sonnet 4 chat model node.  
  - Config: Used as the AI model backing the AI Agent node.  
  - Role: Executes the language model call for website analysis.  
  - Edge Cases: API errors; authentication failure.

---

### 2.3 Website Content Validation

**Overview:**  
Validates if the scraped website content contains meaningful business information using GPT-4. This determines subsequent personalization logic.

**Nodes Involved:**  
- OpenAI1  
- Think3  
- If  
- Sticky Note5 (documentation)

**Node Details:**

- **OpenAI1**  
  - Type: OpenAI GPT-4 language model node.  
  - Config: Receives website content, outputs JSON indicating whether content is valid business data (`hasWebsite: true/false`) with a brief explanation.  
  - Role: Decision gate for personalization branch.  
  - Edge Cases: Model misclassification; API quota or timeout.

- **Think3**  
  - Type: LangChain Think tool.  
  - Role: Triggers OpenAI1 node.  
  - Edge Cases: None specific.

- **If**  
  - Type: Conditional node.  
  - Config: Checks if `hasWebsite` is false.  
  - Role: Routes workflow to different AI personalization agents depending on website data validity.  
  - Edge Cases: Expression errors; missing field in input JSON.

---

### 2.4 Personalized Cold Email Opener Generation

**Overview:**  
Generates one personalized cold email opening sentence per lead, using either website summary data or fallback logic without website data.

**Nodes Involved:**  
- AI Agent_Personalization_with website  
- AI Agent_Personalization_NO website  
- Anthropic Chat Model1  
- Anthropic Chat Model2  
- Think1  
- Think2  
- Edit Fields1  
- Sticky Note9 (website-based personalization)  
- Sticky Note10 (no personalization)

**Node Details:**

- **AI Agent_Personalization_with website**  
  - Type: LangChain agent node.  
  - Config: Uses the website summary to craft a personalized, emotional cold email opener (single sentence, 15-25 words).  
  - System Message: Sales development expert tone, conversational, avoids generic phrases, includes subtle value hints.  
  - Input variables: Website summary, recipient first and last name.  
  - Role: Produces personalized cold email openers when website data is valid.  
  - Edge Cases: Incomplete website summary; generation errors.

- **AI Agent_Personalization_NO website**  
  - Type: LangChain agent node.  
  - Config: Generates personalized openers without website data, focusing on domain-based assumptions and universal pain points related to lead generation.  
  - System Message: Sales expert tone, focused on saving time, reducing costs, creating curiosity.  
  - Input variables: Lead name, company domain.  
  - Role: Fallback personalization when website data is missing or invalid.  
  - Edge Cases: Limited personalization; generic output risk.

- **Anthropic Chat Model1 & Anthropic Chat Model2**  
  - Type: Anthropic Claude Sonnet 4 model nodes.  
  - Role: Provide language model capacity for personalization agents with and without website data respectively.  
  - Edge Cases: API or authentication issues.

- **Think1 & Think2**  
  - Type: LangChain Think tools.  
  - Role: Trigger respective personalization AI Agents.  
  - Edge Cases: None specific.

- **Edit Fields1**  
  - Type: Set node.  
  - Config: Assigns the generated personalized sentence to the field `personalization`.  
  - Role: Prepare data for final update.  
  - Edge Cases: Expression errors.

---

### 2.5 Data Update and Notification

**Overview:**  
Updates the Google Sheet with enriched data (website summary and personalized sentence) and sends a Telegram notification upon completion.

**Nodes Involved:**  
- Google Sheets  
- End of Loops  
- Lead Enrichment is ready  
- Sticky Note (Update Google Sheet)  
- Sticky Note3 (Send Message to Telegram Bot)

**Node Details:**

- **Google Sheets**  
  - Type: Google Sheets update node.  
  - Config: Updates rows matching on `domain` with new columns: `websiteSummary` and `personalization`.  
  - Sheet and document IDs dynamically taken from the trigger.  
  - Role: Persist enriched data back into the lead spreadsheet.  
  - Edge Cases: Permission errors; row matching failure; API limits.

- **End of Loops**  
  - Type: Code node running JavaScript.  
  - Logic: Counts total leads processed; prepares a success message with file info and timestamp.  
  - Role: Summarizes process completion for notification.  
  - Edge Cases: Reference errors if previous nodes fail.

- **Lead Enrichment is ready**  
  - Type: Telegram node.  
  - Config: Sends message to configured Telegram chat ID with enrichment summary.  
  - Role: Alert user that enrichment workflow finished.  
  - Edge Cases: Invalid chat ID; Telegram API errors.

---

## 3. Summary Table

| Node Name                     | Node Type                                | Functional Role                          | Input Node(s)                | Output Node(s)               | Sticky Note                                    |
|-------------------------------|-----------------------------------------|----------------------------------------|-----------------------------|-----------------------------|------------------------------------------------|
| Google Drive Trigger           | Google Drive Trigger                     | Detect new lead file upload            |                             | Google Drive_DownLoad        |                                                |
| Google Drive_DownLoad          | Google Drive                            | Download uploaded file                  | Google Drive Trigger         | Google Sheet Parse           |                                                |
| Google Sheet Parse             | Spreadsheet Parser                      | Parse spreadsheet rows into JSON       | Google Drive_DownLoad        | Loop Over Items1             |                                                |
| Loop Over Items1               | Split in Batches                        | Process leads one by one                 | Google Sheet Parse           | End of Loops, AI Agent       |                                                |
| End of Loops                  | Code                                   | Summarize processing and prepare message| Loop Over Items1             | Lead Enrichment is ready     |                                                |
| Lead Enrichment is ready       | Telegram                               | Notify user of workflow completion      | End of Loops                |                             | ## Send Message to Telegram Bot                 |
| AI Agent                      | LangChain Agent                        | Generate structured business summary    | Think                       | OpenAI1                     | ## Website Summary                               |
| Fetch Markdown via Jina AI1    | LangChain HTTP Request Tool            | Scrape website markdown content         | AI Agent                    |                             |                                                |
| Think                         | LangChain Think Tool                   | Trigger AI Agent                         |                             | AI Agent                    |                                                |
| OpenAI1                      | OpenAI GPT-4 Model                    | Validate if website content is meaningful| AI Agent                    | If                          | ## Determines if we have scraped the website successfully |
| Think3                        | LangChain Think Tool                   | Trigger OpenAI1                         |                             | OpenAI1                     |                                                |
| If                            | Conditional                           | Branch based on website content validity| OpenAI1                     | AI Agent_Personalization_NO website, AI Agent_Personalization_with website |                                                |
| AI Agent_Personalization_NO website | LangChain Agent                 | Generate cold email opener without website data | Think1                      | Edit Fields1                | ## No Personalization                            |
| AI Agent_Personalization_with website | LangChain Agent              | Generate cold email opener with website summary | Think2                      | Edit Fields1                | ## Personalization based on the website summary |
| Think1                        | LangChain Think Tool                   | Trigger AI Agent_Personalization_NO website |                             | AI Agent_Personalization_NO website |                                                |
| Think2                        | LangChain Think Tool                   | Trigger AI Agent_Personalization_with website |                             | AI Agent_Personalization_with website |                                                |
| Edit Fields1                  | Set                                    | Assign personalized sentence to field  | AI Agent_Personalization_NO website, AI Agent_Personalization_with website | Google Sheets               | ## Update Google Sheet                            |
| Google Sheets                 | Google Sheets                         | Update lead spreadsheet with enrichment| Edit Fields1                | Loop Over Items1             |                                                |
| Sticky Note1                  | Sticky Note                           | Documentation: Website Summary          |                             |                             | ## Website Summary                               |
| Sticky Note3                  | Sticky Note                           | Documentation: Telegram notification    |                             |                             | ## Send Message to Telegram Bot                  |
| Sticky Note5                  | Sticky Note                           | Documentation: Website scraping success |                             |                             | ## Determines if we have scraped the website successfully |
| Sticky Note9                  | Sticky Note                           | Documentation: Personalization with website |                             |                             | ## Personalization based on the website summary |
| Sticky Note10                 | Sticky Note                           | Documentation: No Personalization        |                             |                             | ## No Personalization                             |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger node**  
   - Type: Google Drive Trigger  
   - Configure event: `File Created`  
   - Poll every minute  
   - Limit trigger to folder ID: `1-b1yzCQA8RzQmG4vOaGKzYY8iZQxKpvf` (Clean Leads_Enrichment folder)  

2. **Add Google Drive node (Download)**  
   - Operation: Download  
   - File ID: Use expression `{{$json["id"]}}` from trigger output  
   - Connect: Google Drive Trigger → Google Drive_DownLoad  

3. **Add Spreadsheet File node (Parse)**  
   - Operation: Parse  
   - Header row enabled (true)  
   - Connect: Google Drive_DownLoad → Google Sheet Parse  

4. **Add Split In Batches node (Loop Over Items1)**  
   - Default batch size (1) to process leads one by one  
   - Connect: Google Sheet Parse → Loop Over Items1  

5. **Add LangChain HTTP Request Tool node (Fetch Markdown via Jina AI1)**  
   - URL: `https://r.jina.ai/{url}`  
   - Parameter `url`: expression using lead domain  
   - Authentication: HTTP Header Auth with Jina AI credentials  
   - Connect: AI Agent node (below) → Fetch Markdown via Jina AI1  

6. **Add LangChain Agent node (AI Agent)**  
   - Text: `"Please analyze the website at {{ $json.domain }} and create a structured business summary according to the template. Use the "Fetch Markdown via Jina AI" tool to get the website content.\n{{ $json.chatInput }}"`  
   - System message: Define detailed instructions to extract company overview, products, target market, USPs, size, pain points; limit to 300 words.  
   - Connect: Think → AI Agent → OpenAI1  

7. **Add LangChain Think node (Think)**  
   - Connect: Loop Over Items1 → Think → AI Agent  

8. **Add OpenAI GPT-4 node (OpenAI1)**  
   - Model: GPT-4.1  
   - Prompt: Validate website content from AI Agent output, return JSON with `hasWebsite: true/false` and explanation.  
   - Connect: AI Agent → OpenAI1 → If  

9. **Add LangChain Think node (Think3)**  
   - Connect as intermediary trigger for OpenAI1 node if needed.  

10. **Add If node**  
    - Condition: Check if `{{$json["message"]["content"]["hasWebsite"]}}` is false  
    - Connect: OpenAI1 → If  
    - True branch → AI Agent_Personalization_NO website  
    - False branch → AI Agent_Personalization_with website  

11. **Add LangChain Agent node (AI Agent_Personalization_NO website)**  
    - Text: Create personalized cold email opener without website data using recipient name and domain.  
    - System message: Sales development expert tone emphasizing time-saving and curiosity.  
    - Connect: Think1 → AI Agent_Personalization_NO website → Edit Fields1  

12. **Add LangChain Agent node (AI Agent_Personalization_with website)**  
    - Text: Use website summary to craft personalized cold email opener.  
    - System message: Sales development expert with personalization instructions.  
    - Connect: Think2 → AI Agent_Personalization_with website → Edit Fields1  

13. **Add LangChain Think nodes (Think1 and Think2)**  
    - Connect If node branches to respective Think nodes which trigger personalization agents.  

14. **Add Set node (Edit Fields1)**  
    - Assign field `personalization` to the output of the AI agents (`{{$json["output"]}}`)  
    - Connect both personalization agents → Edit Fields1 → Google Sheets  

15. **Add Google Sheets node**  
    - Operation: Update  
    - Sheet Name: Use file name from Google Drive Trigger (`{{$json["name"]}}`)  
    - Document ID: Use file ID from Google Drive Trigger (`{{$json["id"]}}`)  
    - Mapping: Update rows by matching `domain`, update columns `websiteSummary` and `personalization`.  
    - Connect: Edit Fields1 → Google Sheets → Loop Over Items1 (to process next lead)  

16. **Add Code node (End of Loops)**  
    - JavaScript: Count total leads processed, prepare success message with file info and timestamp.  
    - Connect: Loop Over Items1 (after processing all items) → End of Loops → Lead Enrichment is ready  

17. **Add Telegram node (Lead Enrichment is ready)**  
    - Configure with Telegram API OAuth2 credentials  
    - Chat ID: Your Telegram chat ID  
    - Message: Use expressions to include file name and URL from End of Loops output.  

18. **Add Sticky Notes for Documentation**  
    - Place sticky notes near relevant nodes with content describing block purposes, e.g., "Website Summary", "No Personalization", "Update Google Sheet", "Send Message to Telegram Bot", "Determines if we have scraped the website successfully".  

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                        | Context or Link                               |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------|
| This workflow leverages the Anthropic Claude Sonnet 4 model and OpenAI GPT-4.1 for advanced language understanding and generation. Credentials for these APIs must be properly configured in n8n. | AI models usage and credential setup          |
| The Jina AI web scraping tool extracts markdown content from websites to facilitate structured summarization.                                                                                     | https://jina.ai/                              |
| Telegram API integration requires a valid bot and chat ID to send notifications.                                                                                                                    | https://core.telegram.org/bots/api           |
| The cold email personalization logic emphasizes emotional connection, brevity (15-25 words), and avoiding salesy or generic phrases.                                                             | Sales best practices for cold outreach        |
| Folder ID for Google Drive trigger is specific; update accordingly if you use a different folder for lead uploads.                                                                                 | Google Drive folder setup                      |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, which respects all applicable content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.

---