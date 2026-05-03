Convert WordPress Articles to Multilingual Voiceovers with Google TTS and OpenAI

https://n8nworkflows.xyz/workflows/convert-wordpress-articles-to-multilingual-voiceovers-with-google-tts-and-openai-11789


# Convert WordPress Articles to Multilingual Voiceovers with Google TTS and OpenAI

### 1. Workflow Overview

This workflow automates the process of converting WordPress articles into multilingual voiceovers using Google Text-to-Speech (TTS) and OpenAI for text cleaning and translation. The primary use case is to take newly published WordPress content, clean and translate it into English and Italian, generate synthesized speech audio files for each language, and update WordPress pages with the corresponding voiceover data and Google Sheets records.  

The workflow is logically divided into the following blocks:

- **1.1 Scheduling & Article Retrieval:** Periodic trigger to fetch new WordPress articles.
- **1.2 Text Cleaning & Preparation:** Cleans raw article content and prepares it for translation and TTS.
- **1.3 Translation & Language Processing:** Translates content into Italian and English and further cleans the translated texts.
- **1.4 Text-to-Speech Conversion:** Requests Google Cloud Text-to-Speech API to generate audio files for both languages.
- **1.5 WordPress Voiceover Page Management:** Retrieves existing voiceover pages for each language and updates them with new audio.
- **1.6 Data Logging:** Updates Google Sheets to track workflow progress and data changes.
- **1.7 Synchronization & Wait Handling:** Manages timing between asynchronous requests.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduling & Article Retrieval

- **Overview:**  
  This block initiates the workflow on a defined schedule and fetches WordPress articles to process.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Wordpress (Article Fetch)  
  - Code (Post-processing raw WordPress data)

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: Trigger node  
    - Configuration: Triggers periodically (exact schedule not provided)  
    - Input: None (start node)  
    - Output: Triggers WordPress article fetch  
    - Failures: Trigger misconfiguration or scheduling issues may prevent workflow start.

  - **Wordpress**  
    - Type: WordPress node  
    - Configuration: Set to retrieve articles (parameters unspecified, likely posts endpoint)  
    - Input: Trigger from Schedule Trigger  
    - Output: Raw articles data to Code node  
    - Failures: Authentication errors, WordPress API downtime.

  - **Code**  
    - Type: Code (JavaScript)  
    - Configuration: Processes raw WordPress data to a usable format for next steps  
    - Input: Raw articles  
    - Output: Formatted article data to Google Sheets  
    - Failures: Expression errors if data format unexpected.

#### 1.2 Data Logging & Synchronization

- **Overview:**  
  Stores initial article data in Google Sheets and waits before proceeding for synchronization.

- **Nodes Involved:**  
  - Google Sheets  
  - Wait  
  - Google Sheets2

- **Node Details:**  
  - **Google Sheets**  
    - Type: Google Sheets node  
    - Configuration: Writes article data to a specific sheet (details not given)  
    - Input: Formatted article data from Code  
    - Output: Triggers Wait node  
    - Failures: Google OAuth errors, sheet access issues.

  - **Wait**  
    - Type: Wait node  
    - Configuration: Pauses workflow for a set duration (duration not specified)  
    - Input: Completion from Google Sheets  
    - Output: Triggers Google Sheets2  
    - Failures: None typical, unless workflow timeouts.

  - **Google Sheets2**  
    - Type: Google Sheets node  
    - Configuration: Reads or writes additional data (exact role unclear)  
    - Input: From Wait  
    - Output: Triggers Code | Clean  
    - Failures: Same as other Google Sheets nodes.

#### 1.3 Text Cleaning & Preparation

- **Overview:**  
  Cleans and prepares article text for translation and TTS by removing unwanted formatting or characters.

- **Nodes Involved:**  
  - Code | Clean  
  - Google Translate  
  - OpenAI | IT Clean  
  - OpenAI | EN Clean

- **Node Details:**  
  - **Code | Clean**  
    - Type: Code node  
    - Configuration: Custom JavaScript to clean raw text (e.g., strip HTML, normalize whitespace)  
    - Input: Data from Google Sheets2  
    - Output: Triggers Google Translate and OpenAI IT Clean nodes concurrently  
    - Failures: Parsing errors if unexpected input format.

  - **Google Translate**  
    - Type: Google Translate node  
    - Configuration: Translates cleaned text into English  
    - Input: From Code | Clean  
    - Output: Triggers OpenAI | EN Clean  
    - Failures: API quota limits, translation errors.

  - **OpenAI | IT Clean**  
    - Type: OpenAI node (LangChain)  
    - Configuration: Further cleans or processes Italian text (prompt and model settings not detailed)  
    - Input: From Code | Clean  
    - Output: Triggers Code | IT Clean  
    - Failures: API key/auth, rate limits.

  - **OpenAI | EN Clean**  
    - Type: OpenAI node (LangChain)  
    - Configuration: Further cleans or processes English translated text  
    - Input: From Google Translate  
    - Output: Triggers Code | EN Clean  
    - Failures: Same as IT Clean.

#### 1.4 Text-to-Speech Conversion

- **Overview:**  
  Converts the cleaned English and Italian texts into speech audio files using Google Cloud TTS API.

- **Nodes Involved:**  
  - Code | EN Clean  
  - Code | IT Clean  
  - HTTP GCP TTS EN  
  - HTTP GCP TTS IT

- **Node Details:**  
  - **Code | EN Clean**  
    - Type: Code node  
    - Configuration: Prepares the cleaned English text into the format required by Google TTS API  
    - Input: From OpenAI | EN Clean  
    - Output: Triggers HTTP GCP TTS EN  
    - Failures: Formatting errors.

  - **Code | IT Clean**  
    - Type: Code node  
    - Configuration: Prepares cleaned Italian text for Google TTS API request  
    - Input: From OpenAI | IT Clean  
    - Output: Triggers HTTP GCP TTS IT  
    - Failures: Formatting errors.

  - **HTTP GCP TTS EN**  
    - Type: HTTP Request node  
    - Configuration: Calls Google Cloud Text-to-Speech API for English voiceover  
    - Input: Payload from Code | EN Clean  
    - Output: Triggers Wordpress | GET VoiceOvers Page EN  
    - Failures: Auth errors, API limits, network timeouts.

  - **HTTP GCP TTS IT**  
    - Type: HTTP Request node  
    - Configuration: Calls Google Cloud Text-to-Speech API for Italian voiceover  
    - Input: Payload from Code | IT Clean  
    - Output: Triggers Wordpress | GET VoiceOvers Page IT  
    - Failures: Same as HTTP GCP TTS EN.

#### 1.5 WordPress Voiceover Page Management

- **Overview:**  
  Retrieves current WordPress voiceover pages and updates them with newly generated audio content.

- **Nodes Involved:**  
  - Wordpress | GET VoiceOvers Page EN  
  - Wordpress | GET VoiceOvers Page IT  
  - | Update VoiceOvers Page EN  
  - Wordpress| Update VoiceOvers Page IT  
  - Google Sheets1

- **Node Details:**  
  - **Wordpress | GET VoiceOvers Page EN**  
    - Type: WordPress node  
    - Configuration: Retrieves English voiceover page data for updating  
    - Input: From HTTP GCP TTS EN  
    - Output: Triggers | Update VoiceOvers Page EN  
    - Failures: API errors, missing page.

  - **Wordpress | GET VoiceOvers Page IT**  
    - Type: WordPress node  
    - Configuration: Retrieves Italian voiceover page data  
    - Input: From HTTP GCP TTS IT  
    - Output: Triggers Wordpress| Update VoiceOvers Page IT  
    - Failures: Same as EN.

  - **| Update VoiceOvers Page EN**  
    - Type: WordPress node  
    - Configuration: Updates English voiceover page with new audio or metadata  
    - Input: From Wordpress | GET VoiceOvers Page EN  
    - Output: Triggers Google Sheets1  
    - Failures: Write permission errors.

  - **Wordpress| Update VoiceOvers Page IT**  
    - Type: WordPress node  
    - Configuration: Updates Italian voiceover page similarly  
    - Input: From Wordpress | GET VoiceOvers Page IT  
    - Output: None  
    - Failures: Same as EN update.

  - **Google Sheets1**  
    - Type: Google Sheets node  
    - Configuration: Logs update status or audio metadata for tracking  
    - Input: From | Update VoiceOvers Page EN  
    - Output: None  
    - Failures: Sheet access issues.

---

### 3. Summary Table

| Node Name                       | Node Type                    | Functional Role                     | Input Node(s)                  | Output Node(s)                     | Sticky Note                         |
|--------------------------------|------------------------------|-----------------------------------|-------------------------------|----------------------------------|-----------------------------------|
| Schedule Trigger               | Schedule Trigger             | Initiate workflow per schedule    | None                          | Wordpress                        |                                   |
| Wordpress                     | WordPress                    | Fetch WordPress articles          | Schedule Trigger              | Code                            |                                   |
| Code                         | Code                         | Format raw WordPress articles     | Wordpress                    | Google Sheets                   |                                   |
| Google Sheets                | Google Sheets                | Log initial article data          | Code                         | Wait                           |                                   |
| Wait                         | Wait                         | Pause for synchronization         | Google Sheets                 | Google Sheets2                 |                                   |
| Google Sheets2               | Google Sheets                | Additional data handling          | Wait                         | Code | Clean                   |                                   |
| Code | Clean                 | Code                         | Clean article text                | Google Sheets2               | Google Translate, OpenAI | IT Clean |                                   |
| Google Translate            | Google Translate            | Translate text to English         | Code | Clean                 | OpenAI | EN Clean               |                                   |
| OpenAI | IT Clean           | OpenAI (LangChain)           | Clean Italian text                | Code | Clean                 | Code | IT Clean                 |                                   |
| OpenAI | EN Clean           | OpenAI (LangChain)           | Clean English text                | Google Translate            | Code | EN Clean                 |                                   |
| Code | EN Clean             | Code                         | Prepare English text for TTS      | OpenAI | EN Clean             | HTTP GCP TTS EN               |                                   |
| Code | IT Clean             | Code                         | Prepare Italian text for TTS      | OpenAI | IT Clean             | HTTP GCP TTS IT               |                                   |
| HTTP GCP TTS EN             | HTTP Request                | Generate English speech audio     | Code | EN Clean               | Wordpress | GET VoiceOvers Page EN |                                   |
| HTTP GCP TTS IT             | HTTP Request                | Generate Italian speech audio     | Code | IT Clean               | Wordpress | GET VoiceOvers Page IT |                                   |
| Wordpress | GET VoiceOvers Page EN | WordPress                    | Fetch English voiceover page data | HTTP GCP TTS EN             | | Update VoiceOvers Page EN     |                                   |
| Wordpress | GET VoiceOvers Page IT | WordPress                    | Fetch Italian voiceover page data | HTTP GCP TTS IT             | Wordpress| Update VoiceOvers Page IT |                                   |
| | Update VoiceOvers Page EN | WordPress                    | Update English voiceover page     | Wordpress | GET VoiceOvers Page EN | Google Sheets1               |                                   |
| Wordpress| Update VoiceOvers Page IT | WordPress                    | Update Italian voiceover page     | Wordpress | GET VoiceOvers Page IT |                               |                                   |
| Google Sheets1              | Google Sheets                | Log update status and metadata    | | Update VoiceOvers Page EN   | None                           |                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Configure to trigger at desired intervals (e.g., hourly/daily).

2. **Add a WordPress node**  
   - Set to fetch posts or articles from your WordPress site.  
   - Configure authentication credentials (OAuth or API key).  
   - Connect Schedule Trigger output to this node.

3. **Add a Code node** (formatting raw WordPress data)  
   - Use JavaScript to parse and format article data for further processing.  
   - Connect WordPress node output to this Code node.

4. **Add a Google Sheets node** (first instance)  
   - Configure to append or update initial article data in a Google Sheets document.  
   - Set up Google Sheets OAuth2 credentials.  
   - Connect Code node output to this node.

5. **Add a Wait node**  
   - Configure a wait time (e.g., a few seconds) to synchronize processes.  
   - Connect Google Sheets node output to Wait node.

6. **Add a second Google Sheets node**  
   - Configure as needed to read/write additional data or flags for workflow control.  
   - Connect Wait node output to this node.

7. **Add a Code node labeled "Code | Clean"**  
   - Write JavaScript to clean text from Google Sheets2 output (e.g., remove HTML tags, fix whitespace).  
   - Connect Google Sheets2 output to this node.

8. **Add a Google Translate node**  
   - Translate cleaned text to English.  
   - Configure Google Cloud Translate OAuth credentials.  
   - Connect Code | Clean node output to this node.

9. **Add two OpenAI nodes** (LangChain OpenAI integration)  
   - One labeled "OpenAI | IT Clean" connected from Code | Clean to clean/process Italian text.  
   - One labeled "OpenAI | EN Clean" connected from Google Translate node to clean English text.  
   - Configure OpenAI API credentials.

10. **Add two Code nodes for TTS preparation**  
    - "Code | IT Clean" connected from OpenAI | IT Clean node.  
    - "Code | EN Clean" connected from OpenAI | EN Clean node.  
    - These nodes prepare request payloads for Google TTS API.

11. **Add two HTTP Request nodes for Google Cloud TTS**  
    - "HTTP GCP TTS IT" connected from Code | IT Clean.  
    - "HTTP GCP TTS EN" connected from Code | EN Clean.  
    - Configure OAuth2 credentials for Google Cloud API.  
    - Set method to POST with appropriate headers and body for TTS API.

12. **Add two WordPress nodes to GET existing voiceover pages**  
    - "Wordpress | GET VoiceOvers Page IT" connected from HTTP GCP TTS IT.  
    - "Wordpress | GET VoiceOvers Page EN" connected from HTTP GCP TTS EN.  
    - Configure with API credentials.

13. **Add two WordPress nodes to UPDATE voiceover pages**  
    - "Wordpress| Update VoiceOvers Page IT" connected from GET VoiceOvers Page IT node.  
    - "| Update VoiceOvers Page EN" connected from GET VoiceOvers Page EN node.  
    - Configure to update pages with new audio URLs or metadata.

14. **Add a final Google Sheets node**  
    - "Google Sheets1" connected from "| Update VoiceOvers Page EN" node.  
    - Configure to log the update status or voiceover metadata.

15. **Set up all credentials carefully:**  
    - WordPress OAuth or API keys  
    - Google Sheets OAuth2  
    - Google Cloud API keys for Translate and TTS  
    - OpenAI API key

16. **Test the workflow end-to-end**  
    - Verify data flows correctly and that audio files are generated and linked properly.  
    - Handle errors by adding error workflows or alerts as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                       | Context or Link                                      |
|--------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Workflow tags include PROD, Website, creator-paid-release, and creator-to-publish indicating production readiness. | Workflow metadata                                    |
| Uses LangChain OpenAI node integration for advanced text cleaning and processing.                                   | https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-langchain.openai/ |
| Google Cloud Text-to-Speech and Translate API require proper OAuth2 credential setup with quota management.         | https://cloud.google.com/text-to-speech/docs/quickstart-client-libraries |
| WordPress API calls need proper authentication and permissions to read and update pages.                            | https://developer.wordpress.org/rest-api/           |
| Consider adding error handling nodes (e.g., IF, ErrorTrigger) to manage API rate limits and failures gracefully.    | n8n documentation on error workflows                  |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with relevant content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.