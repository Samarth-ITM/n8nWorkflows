Convert Websites to Audio Summaries via WhatsApp using GPT and TTS

https://n8nworkflows.xyz/workflows/convert-websites-to-audio-summaries-via-whatsapp-using-gpt-and-tts-10297


# Convert Websites to Audio Summaries via WhatsApp using GPT and TTS

---

### 1. Workflow Overview

This workflow automates the process of converting website content shared via WhatsApp into audio summaries using GPT-based AI summarization and text-to-speech (TTS) conversion. It targets users who want quick, hands-free content consumption by simply sending a URL to a WhatsApp business number. The workflow extracts the URL from incoming WhatsApp messages, validates it, fetches and summarizes the webpage content using a sub-workflow, converts the summary into audio using OpenAI's TTS capabilities, and sends the audio back via WhatsApp.

**Logical Blocks:**

- **1.1 Input Reception and URL Extraction:** Receives incoming WhatsApp messages and extracts URLs.
- **1.2 URL Validation and User Feedback:** Checks URL validity, sends error or processing notifications.
- **1.3 Webpage Content Summarization (Sub-Workflow):** Fetches webpage content, extracts text and title, performs GPT summarization.
- **1.4 Audio Generation and Delivery:** Converts text summary to audio and sends audio message via WhatsApp.
- **1.5 Workflow Metadata and Documentation:** Sticky notes provide guidance and usage information.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and URL Extraction

**Overview:**  
This block listens for WhatsApp messages, extracts any URLs contained within them, and prepares the data for validation.

**Nodes Involved:**  
- WhatsApp Trigger  
- Extract URL from Message

**Node Details:**

- **WhatsApp Trigger**  
  - *Type:* Trigger node for WhatsApp messages  
  - *Role:* Starts the workflow upon receiving new WhatsApp messages  
  - *Configuration:* Listens for "messages" update events, requires WhatsApp OAuth credentials  
  - *Input:* Incoming WhatsApp webhook data  
  - *Output:* JSON with the message details  
  - *Edge Cases:* Message might be empty, status update (no message), or malformed data  
  - *Sticky Note:* Explains WhatsApp setup, triggers, and security caveats  

- **Extract URL from Message**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Parses the incoming WhatsApp message JSON to extract the first URL found in the text  
  - *Configuration:* Custom JS code scans message text with regex for URLs; returns extracted URL and sender info or an error message if no URL found  
  - *Input:* Output from WhatsApp Trigger  
  - *Output:* JSON with URL, sender phone number (`from`), original message, and error flag  
  - *Edge Cases:* No messages array, empty message, no URL found, malformed URLs  
  - *Error Handling:* Returns JSON with `error: true` and explanatory message  

---

#### 1.2 URL Validation and User Feedback

**Overview:**  
Validates presence of a URL and sends appropriate WhatsApp messages: an error prompt if invalid or a processing notification if valid.

**Nodes Involved:**  
- Check if Valid URL (If node)  
- Send Error Message  
- Send Processing Message

**Node Details:**

- **Check if Valid URL**  
  - *Type:* If node (conditional branching)  
  - *Role:* Checks if the error flag from the previous node is false (valid URL)  
  - *Configuration:* Condition: `error === false`  
  - *Input:* Extract URL from Message output  
  - *Output:* Two branches - true (valid URL), false (invalid URL)  
  - *Edge Cases:* Expression failures if data missing  

- **Send Error Message**  
  - *Type:* WhatsApp node (send message)  
  - *Role:* Sends error text back to user if no valid URL is found  
  - *Configuration:* Sends `$json.message` text to sender's phone number; uses WhatsApp API credentials  
  - *Input:* False branch from Check if Valid URL  
  - *Output:* None (end of flow for invalid input)  
  - *Edge Cases:* WhatsApp API auth, phone number format errors, network issues  

- **Send Processing Message**  
  - *Type:* WhatsApp node (send message)  
  - *Role:* Notifies user that the webpage is being fetched and processed  
  - *Configuration:* Static text message indicating processing delay; sends to sender's number; uses WhatsApp API credentials  
  - *Input:* True branch from Check if Valid URL  
  - *Output:* Next node (Get URL)  
  - *Edge Cases:* Same as Send Error Message  

---

#### 1.3 Webpage Content Summarization (Sub-Workflow)

**Overview:**  
Fetches the webpage content from the URL, extracts text and title, performs chunking, loads documents, and uses GPT to generate a concise summary.

**Nodes Involved:**  
- Get URL (Set node)  
- Get Webpage Summary (Execute Workflow node)  
- [Sub-Workflow: Get Webpage Summary] nodes (detailed below)

**Node Details:**

- **Get URL**  
  - *Type:* Set node  
  - *Role:* Passes the validated URL to the sub-workflow  
  - *Configuration:* Sets `URL` field with value from Check if Valid URL node  
  - *Input:* Output from Send Processing Message  
  - *Output:* Passes URL to Get Webpage Summary node  

- **Get Webpage Summary**  
  - *Type:* Execute Workflow (Sub-workflow trigger)  
  - *Role:* Calls a dedicated sub-workflow named "[SUB] Get Webpage Summary" that handles webpage fetching and summarization  
  - *Configuration:* References sub-workflow by ID; passes URL as input  
  - *Input:* URL from Get URL node  
  - *Output:* Summary text after processing  
  - *Edge Cases:* Sub-workflow failures, timeouts, HTTP errors, AI API errors  

---

**Sub-Workflow: [SUB] Get Webpage Summary**

- **Sub Nodes:**

  - **Sub Execute**  
    - *Type:* Execute Workflow Trigger  
    - *Role:* Starts the sub-workflow with passed URL  
    - *Input:* URL  

  - **Fetch site texts**  
    - *Type:* HTTP Request  
    - *Role:* Retrieves raw HTML content of the URL with user-agent headers and redirect support  
    - *Edge Cases:* HTTP errors, redirects, timeouts  

  - **Extract title**  
    - *Type:* HTML Extract  
    - *Role:* Extracts `<title>` from HTML for context in summary  

  - **Extract Text Only**  
    - *Type:* HTML Extract  
    - *Role:* Extracts textual content from `<body>`, excluding images and navigation elements  

  - **Summarization Chain**  
    - *Type:* LangChain Summarization Chain  
    - *Role:* Uses GPT model to summarize the extracted text  
    - *Configuration:* Uses GPT-4o-mini model with document loader operation mode  
    - *Input:* Loaded and split text chunks  

  - **Recursive Character Text Splitter**  
    - *Type:* Text Splitter  
    - *Role:* Splits large text into manageable chunks (6000 characters) for GPT processing  

  - **Default Data Loader**  
    - *Type:* Document Loader  
    - *Role:* Loads split text chunks as documents for summarization chain  

  - **OpenAI Chat Model**  
    - *Type:* GPT model node  
    - *Role:* Processes summarization prompts  

  - **Merge**  
    - *Type:* Merge node  
    - *Role:* Combines title and summary outputs into one JSON  

  - **Clean up**  
    - *Type:* Set node  
    - *Role:* Refines output fields to `title` and `summary`  

- **Edge Cases:**  
  - HTTP request failures, malformed HTML, empty content, GPT API limits or errors  

---

#### 1.4 Audio Generation and Delivery

**Overview:**  
Converts the generated text summary to an audio file in opus format using OpenAI's audio resource and sends it back as an audio WhatsApp message.

**Nodes Involved:**  
- Convert Summary to Audio  
- Send Audio Summary

**Node Details:**

- **Convert Summary to Audio**  
  - *Type:* OpenAI Audio (LangChain node)  
  - *Role:* Text-to-Speech conversion of summary to audio in opus format  
  - *Configuration:* Input set as summary text, speed set to 1, response format "opus"  
  - *Credentials:* OpenAI API  
  - *Input:* Summary text from Get Webpage Summary output  
  - *Output:* Audio binary data  
  - *Edge Cases:* API quota, network errors, unsupported text length  

- **Send Audio Summary**  
  - *Type:* WhatsApp node (send audio message)  
  - *Role:* Sends the audio file back to the user on WhatsApp  
  - *Configuration:* Sends audio media from previous node output to sender's phone number  
  - *Input:* Converted audio file, sender info from URL extraction node  
  - *Credentials:* WhatsApp API  
  - *Edge Cases:* WhatsApp media size limits, network errors, authentication  

---

#### 1.5 Workflow Metadata and Documentation

**Overview:**  
Contains sticky notes with documentation, usage instructions, and WhatsApp integration caveats.

**Nodes Involved:**  
- Sticky Note (main workflow)  
- Sticky Note1 (sub-workflow)  
- Upload & Path Processing1 (sticky note on input block)  
- Sharing Link Management1 (sticky note on output block)

**Node Details:**

- **Sticky Note (main)**  
  - Describes overall workflow purpose, WhatsApp requirements, OAuth setup, and security caveats about WhatsApp business accounts and message limits.

- **Sticky Note1**  
  - Brief description of the sub-workflow’s role: summarizing website text with GPT.

- **Upload & Path Processing1**  
  - Explains initial input reception and validation steps.

- **Sharing Link Management1**  
  - Describes the summary generation and audio response logic.

---

### 3. Summary Table

| Node Name                 | Node Type                                  | Functional Role                                | Input Node(s)              | Output Node(s)           | Sticky Note                                                                                                                  |
|---------------------------|--------------------------------------------|------------------------------------------------|----------------------------|--------------------------|------------------------------------------------------------------------------------------------------------------------------|
| WhatsApp Trigger          | WhatsApp Trigger                           | Entry point: receives WhatsApp messages         | -                          | Extract URL from Message  | Explains WhatsApp setup, OAuth requirements, and message sending policy                                                      |
| Extract URL from Message  | Code                                       | Extracts URL from incoming WhatsApp messages    | WhatsApp Trigger           | Check if Valid URL        |                                                                                                                              |
| Check if Valid URL        | If                                         | Validates presence of URL                        | Extract URL from Message   | Send Processing Message, Send Error Message |                                                                                                                              |
| Send Error Message        | WhatsApp Send Message                       | Sends error message if no valid URL             | Check if Valid URL (false) | -                        |                                                                                                                              |
| Send Processing Message   | WhatsApp Send Message                       | Notifies user that processing has started       | Check if Valid URL (true)  | Get URL                  |                                                                                                                              |
| Get URL                   | Set                                         | Sets URL field for sub-workflow input           | Send Processing Message    | Get Webpage Summary       |                                                                                                                              |
| Get Webpage Summary       | Execute Workflow (Sub-Workflow)             | Calls sub-workflow to summarize webpage         | Get URL                   | Convert Summary to Audio  | Calls "[SUB] Get Webpage Summary" workflow                                                                                   |
| Convert Summary to Audio  | OpenAI Audio (LangChain)                     | Converts summary text to audio (opus)            | Get Webpage Summary        | Send Audio Summary        |                                                                                                                              |
| Send Audio Summary        | WhatsApp Send Message (audio)                | Sends audio summary back to user                 | Convert Summary to Audio   | -                        |                                                                                                                              |
| Sticky Note               | Sticky Note                                 | Documentation and workflow overview              | -                          | -                        | Detailed explanation of WhatsApp Voice Research Assistant and integration caveats                                            |
| Sticky Note1              | Sticky Note                                 | Sub-workflow description                          | -                          | -                        | Brief about "[SUB] Get Webpage Summary"                                                                                        |
| Upload & Path Processing1 | Sticky Note                                 | Input block description                           | -                          | -                        | Explains WhatsApp Trigger, URL extraction, and validation steps                                                              |
| Sharing Link Management1  | Sticky Note                                 | Output block description                          | -                          | -                        | Describes summary generation and audio response process                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WhatsApp Trigger Node**  
   - Type: WhatsApp Trigger  
   - Configure to listen for "messages" update events  
   - Connect WhatsApp OAuth2 credentials  
   - Position as start node  

2. **Add Code Node "Extract URL from Message"**  
   - Paste JS code to parse incoming WhatsApp message JSON for URLs  
   - Input: WhatsApp Trigger output  
   - Output: JSON with URL, sender phone number, original message, error flag  

3. **Add If Node "Check if Valid URL"**  
   - Condition: `$json.error === false`  
   - Input: Extract URL from Message output  
   - Two branches:  
     - True: proceed to processing message  
     - False: send error message  

4. **Add WhatsApp Node "Send Error Message"**  
   - Operation: send message  
   - Text Body: `{{$json.message}}` (error message from extraction)  
   - Recipient: `{{$json.from}}`  
   - Use WhatsApp API credentials  
   - Connect from false branch of Check if Valid URL  

5. **Add WhatsApp Node "Send Processing Message"**  
   - Operation: send message  
   - Text Body: `"🔍 Fetching and analyzing webpage...\nThis may take 10-30 seconds."`  
   - Recipient: `{{$json.from}}`  
   - Use WhatsApp API credentials  
   - Connect from true branch of Check if Valid URL  

6. **Add Set Node "Get URL"**  
   - Assign field `URL` with value: `{{$json.URL}}` from Check if Valid URL  
   - Input: Send Processing Message output  

7. **Add Execute Workflow Node "Get Webpage Summary"**  
   - Reference your sub-workflow that summarizes webpages (create separately)  
   - Pass `URL` as input parameter  
   - Input: Get URL node output  

8. **Create Sub-Workflow "[SUB] Get Webpage Summary"**  
   - Start with Execute Workflow Trigger node (passthrough input)  
   - Add HTTP Request node "Fetch site texts"  
     - URL: from input  
     - Set headers: User-Agent (Mozilla/5.0), Accept headers  
     - Enable redirects (max 5)  
   - Add HTML Extract node "Extract title"  
     - CSS Selector: `title`  
   - Add HTML Extract node "Extract Text Only"  
     - CSS Selector: `body`  
     - Skip selectors: `img,nav`  
   - Add Recursive Character Text Splitter node  
     - Chunk size: 6000 characters  
   - Add Default Data Loader node  
     - Input: output of text splitter  
   - Add OpenAI Chat Model node  
     - Model: `gpt-4o-mini` (or equivalent)  
     - Use OpenAI API credentials  
   - Add Summarization Chain node  
     - Operation mode: documentLoader  
     - Connect OpenAI Chat Model, Data Loader, and Text Splitter accordingly  
   - Add Merge node to combine extracted title and GPT summary  
   - Add Set node "Clean up"  
     - Fields: `title` and `summary` from merged output  
   - Output: formatted JSON with title and summary  

9. **Back to Main Workflow: Add OpenAI Audio Node "Convert Summary to Audio"**  
   - Input: summary text from sub-workflow output  
   - Set speed: 1  
   - Response format: `opus` (audio)  
   - Use OpenAI API credentials  

10. **Add WhatsApp Node "Send Audio Summary"**  
    - Operation: send audio message  
    - Media path: use binary data from previous node  
    - Recipient: phone number from original extracted URL node (`from`)  
    - Use WhatsApp API credentials  

11. **Connect all nodes as per logical order:**  
    - WhatsApp Trigger → Extract URL → Check if Valid URL →  
      - False → Send Error Message  
      - True → Send Processing Message → Get URL → Get Webpage Summary → Convert Summary to Audio → Send Audio Summary  

12. **Add Sticky Notes for documentation:**  
    - Add notes describing input process, summarization sub-workflow, and audio response logic for ease of maintenance  

13. **Configure Credentials:**  
    - WhatsApp OAuth2 credentials for trigger and send nodes  
    - OpenAI API key for GPT and audio conversion nodes  

14. **Set webhook URLs and ensure WhatsApp webhook is properly set in Facebook Developer portal**  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                    | Context or Link                                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| WhatsApp business OAuth setup requires creating an app, registering phone number, and creating a System User with a token for automated use.   | https://docs.n8n.io/integrations/builtin/credentials/whatsapp                                                   |
| WhatsApp policy restricts sending auto-generated messages after 24 hours unless user interaction or approved message templates occur.          | See WhatsApp Business API official documentation                                                                |
| The workflow uses a natural AI voice (Alloy) and auto-detects language from content for TTS conversion.                                        | Provided by OpenAI audio resource node                                                                           |
| Sub-workflow modularity allows easy replacement or enhancement of summarization logic separately from main flow.                               | N8N executeWorkflow node referencing "[SUB] Get Webpage Summary" workflow                                        |
| HTTP request headers mimic browsers to avoid blocked requests and enhance content retrieval reliability.                                       | User-Agent and Accept headers in HTTP Request node                                                              |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---