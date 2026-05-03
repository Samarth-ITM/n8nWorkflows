Turn YouTube Transcripts into Newsletter Drafts using Dumpling AI + GPT-4o

https://n8nworkflows.xyz/workflows/turn-youtube-transcripts-into-newsletter-drafts-using-dumpling-ai---gpt-4o-5798


# Turn YouTube Transcripts into Newsletter Drafts using Dumpling AI + GPT-4o

---

### 1. Workflow Overview

This workflow automates the conversion of YouTube video transcripts into polished newsletter drafts using AI services. It is designed for content creators, marketers, or newsletter editors who want to efficiently repurpose YouTube content into email-ready newsletters without manual transcription or writing effort.

The workflow is logically divided into these blocks:

**1.1 Input Retrieval**  
Reads YouTube video URLs from a Google Sheets document, serving as the source list of videos to process.

**1.2 Iterative Processing Loop**  
Processes each YouTube link one by one through batching, enabling scalable and controlled handling.

**1.3 Transcript Acquisition**  
Uses Dumpling AI’s API to fetch the full transcript for each YouTube video URL.

**1.4 Newsletter Draft Generation**  
Uses GPT-4o (OpenAI) to transform the transcript into a clear, engaging newsletter draft with summarization and formatting optimized for email reading.

**1.5 Output Logging**  
Appends or updates the generated newsletter draft back into the Google Sheet next to the original video link, maintaining record and traceability.

**1.6 Notification**  
Sends an email notification via Gmail to inform stakeholders that a newsletter draft is ready for review.

Additionally, a sticky note summarizing the workflow purpose, tools, and credential requirements is included for user orientation.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Retrieval

- **Overview:**  
  Reads YouTube video URLs from a specified Google Sheets document and sheet. It filters and retrieves rows containing video links to be processed.

- **Nodes Involved:**  
  - Manual Trigger: Start Workflow  
  - Read YouTube Links from Google Sheets

- **Node Details:**  

  1. **Manual Trigger: Start Workflow**  
     - Type: Manual trigger node  
     - Role: Initiates the workflow manually  
     - Configuration: No parameters, simply triggers downstream nodes  
     - Connections: Output to "Read YouTube Links from Google Sheets"  
     - Edge Cases: None; manual initiation  
     - Version: 1

  2. **Read YouTube Links from Google Sheets**  
     - Type: Google Sheets node  
     - Role: Reads rows from Google Sheets containing YouTube video links  
     - Configuration:  
       - Document ID points to a specific Google Sheet containing the "Youtube" data  
       - Sheet name: "Sheet1" (gid=0)  
       - Filters on column "blog post" (though filter set to return first match) — likely used to read links where "blog post" is empty or to get all rows  
       - Operation: Read rows  
     - Credentials: Google Sheets OAuth2  
     - Inputs: From Manual Trigger  
     - Outputs: To Loop node  
     - Edge Cases:  
       - Google Sheets authorization failure  
       - Empty or malformed sheet data  
       - Incorrect sheet or document ID  
     - Version: 4.6

---

#### Block 1.2: Iterative Processing Loop

- **Overview:**  
  Splits the list of YouTube links into batches (here, batches of 1 by default), enabling sequential processing of each video link.

- **Nodes Involved:**  
  - Loop: Process Each YouTube Link

- **Node Details:**  

  1. **Loop: Process Each YouTube Link**  
     - Type: SplitInBatches node  
     - Role: Iterates through each YouTube link individually for processing  
     - Configuration:  
       - Batch size not explicitly specified, defaults to 1 (process one at a time)  
       - Reset option disabled (keeps internal state during execution)  
     - Inputs: From "Read YouTube Links from Google Sheets"  
     - Outputs: Two outputs:  
       - Main output to "Dumpling AI: Get YouTube Transcript" (process video)  
       - Secondary output to "Send Email Notification When Draft is Ready" (likely triggered after all batches or per batch, needs clarification)  
     - Edge Cases:  
       - Empty input array leads to no processing  
       - Batch state persistence issues if workflow retriggered mid-processing  
     - Version: 3

---

#### Block 1.3: Transcript Acquisition

- **Overview:**  
  Sends each YouTube video URL to Dumpling AI API to retrieve the transcript text.

- **Nodes Involved:**  
  - Dumpling AI: Get YouTube Transcript

- **Node Details:**  

  1. **Dumpling AI: Get YouTube Transcript**  
     - Type: HTTP Request node  
     - Role: Calls Dumpling AI’s API endpoint to get the transcript of the video  
     - Configuration:  
       - POST method to `https://app.dumplingai.com/api/v1/get-youtube-transcript`  
       - Sends JSON body with parameter `"videoUrl"` dynamically set to the current batch item's link (`={{ $json.link }}`)  
       - Authentication via HTTP header using stored credentials named "Dumpling AI-n8n"  
     - Inputs: From Loop node  
     - Outputs: To "GPT-4o: Write Newsletter Draft from Transcript"  
     - Edge Cases:  
       - API authentication failure (invalid or expired header token)  
       - API rate limits or downtime  
       - Invalid or inaccessible YouTube URL  
       - Empty or partial transcript returned  
     - Version: 4.2

---

#### Block 1.4: Newsletter Draft Generation

- **Overview:**  
  Uses GPT-4o via LangChain n8n node to convert the raw transcript into a concise, engaging newsletter draft suitable for email distribution.

- **Nodes Involved:**  
  - GPT-4o: Write Newsletter Draft from Transcript

- **Node Details:**  

  1. **GPT-4o: Write Newsletter Draft from Transcript**  
     - Type: LangChain OpenAI node  
     - Role: Sends prompt and transcript to GPT-4o to generate newsletter content  
     - Configuration:  
       - Model: `chatgpt-4o-latest`  
       - Messages:  
         - System message instructing GPT-4o to write a newsletter based on the transcript with detailed style and formatting guidelines  
         - User message injecting the transcript dynamically: `=Transcript: {{ $json.transcript }}`  
       - No additional options specified  
     - Credentials: OpenAI API key configured under "OpenAi account 2"  
     - Inputs: From Dumpling AI transcript node  
     - Outputs: To "Log Newsletter Draft to Google Sheets"  
     - Edge Cases:  
       - OpenAI API key issues (invalid, expired, rate limits)  
       - Transcript content too long or malformed causing prompt truncation or errors  
       - API timeouts or network issues  
     - Version: 1.8

---

#### Block 1.5: Output Logging

- **Overview:**  
  Appends or updates the generated newsletter draft back into the Google Sheet alongside the original YouTube link, enabling record keeping and manual review.

- **Nodes Involved:**  
  - Log Newsletter Draft to Google Sheets

- **Node Details:**  

  1. **Log Newsletter Draft to Google Sheets**  
     - Type: Google Sheets node  
     - Role: Writes the newsletter draft text into the "blog post" column next to the matching video link  
     - Configuration:  
       - Operation: Append or update based on matching column "link"  
       - Sheet name: "Sheet1" (gid=0)  
       - Document ID: same as input sheet  
       - Columns mapped:  
         - `link`: pulled from original YouTube link (via expression referencing "Read YouTube Links from Google Sheets" node)  
         - `blog post`: populated with GPT-4o generated newsletter draft (`$json.message.content`)  
       - Matching columns: ["link"] to avoid duplicates and update existing entries  
     - Credentials: Google Sheets OAuth2  
     - Inputs: From GPT-4o node  
     - Outputs: Back to Loop node (allows continued batch processing)  
     - Edge Cases:  
       - Sheet access or permission denied  
       - Concurrent updates causing race conditions  
       - Missing or mismatched link causing update failure  
     - Version: 4.6

---

#### Block 1.6: Notification

- **Overview:**  
  Sends an email notification to a designated recipient once a newsletter draft is created, prompting review or further action.

- **Nodes Involved:**  
  - Send Email Notification When Draft is Ready

- **Node Details:**  

  1. **Send Email Notification When Draft is Ready**  
     - Type: Gmail node  
     - Role: Sends a plain text email notifying that the newsletter draft is ready  
     - Configuration:  
       - Recipient: example@gmail.com (hardcoded)  
       - Subject: "Newsletter Draft Created from YouTube Transcript"  
       - Message: Static text informing that the newsletter draft is available for review  
       - Email type: text (no HTML)  
     - Credentials: Gmail OAuth2 under "Gmail account 2"  
     - Inputs: From Loop node's secondary output  
     - Edge Cases:  
       - Gmail OAuth token expired or invalid  
       - Email sending quota exceeded  
       - Invalid recipient address (hardcoded here)  
     - Version: 2.1

---

#### Block 1.7: Sticky Note (Documentation)

- **Overview:**  
  Provides a textual summary of the workflow’s purpose, tools used, sheet structure, and credential setup for user reference.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**  

  1. **Sticky Note**  
     - Type: Sticky Note node  
     - Role: Descriptive annotation with key info  
     - Content Summary:  
       - Explains this is a YouTube to newsletter automation using Dumpling AI and GPT-4o  
       - Lists the workflow steps and tools involved  
       - Specifies credential requirements and Google Sheets column expectations (`link`, `blog post`)  
     - Position: Top-left workspace for visibility  
     - No inputs or outputs  
     - Version: 1

---

### 3. Summary Table

| Node Name                             | Node Type                   | Functional Role                         | Input Node(s)                      | Output Node(s)                                     | Sticky Note                                                                                               |
|-------------------------------------|-----------------------------|---------------------------------------|----------------------------------|---------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Manual Trigger: Start Workflow       | Manual Trigger              | Start the workflow manually            | -                                | Read YouTube Links from Google Sheets              |                                                                                                         |
| Read YouTube Links from Google Sheets| Google Sheets               | Read YouTube video URLs from sheet     | Manual Trigger                   | Loop: Process Each YouTube Link                     |                                                                                                         |
| Loop: Process Each YouTube Link      | SplitInBatches              | Iterate through each YouTube link      | Read YouTube Links from Google Sheets | Dumpling AI: Get YouTube Transcript; Send Email Notification When Draft is Ready |                                                                                                         |
| Dumpling AI: Get YouTube Transcript  | HTTP Request                | Retrieve transcript via Dumpling AI    | Loop: Process Each YouTube Link  | GPT-4o: Write Newsletter Draft from Transcript     |                                                                                                         |
| GPT-4o: Write Newsletter Draft from Transcript | LangChain OpenAI          | Generate newsletter draft from transcript | Dumpling AI: Get YouTube Transcript | Log Newsletter Draft to Google Sheets               |                                                                                                         |
| Log Newsletter Draft to Google Sheets| Google Sheets               | Save newsletter draft next to video link | GPT-4o: Write Newsletter Draft from Transcript | Loop: Process Each YouTube Link                     |                                                                                                         |
| Send Email Notification When Draft is Ready | Gmail                      | Notify via email when draft is ready   | Loop: Process Each YouTube Link  | -                                                 |                                                                                                         |
| Sticky Note                         | Sticky Note                 | Documentation and user guidance        | -                                | -                                                 | ### 📨 YouTube → Newsletter Automation\n\nThis workflow turns YouTube videos into newsletter drafts using Dumpling AI and GPT-4o.\n\n**Here’s what it does:**\n1. Reads new YouTube video links from Google Sheets\n2. Fetches the transcript of each video via Dumpling AI\n3. Uses GPT-4o to generate a well-structured, email-friendly newsletter draft\n4. Saves the draft back to Google Sheets next to the original video link\n5. Sends an email notification when the newsletter is ready\n\n🛠️ Tools Used:\n- Dumpling AI (transcript)\n- GPT-4o (newsletter copy)\n- Google Sheets (input + output)\n- Gmail (notification)\n\n✅ Make sure your credentials are securely configured for:\n- Google Sheets\n- Dumpling AI (via header auth)\n- OpenAI GPT\n- Gmail OAuth2\n\n🗒️ Sheet columns required:\n- `link` (YouTube video URL)\n- `blog post` (Newsletter output field) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a "Manual Trigger" node at the start to initiate the workflow.

2. **Add Google Sheets Node to Read YouTube Links**  
   - Set operation to "Read Rows" from Google Sheets.  
   - Configure credentials with Google Sheets OAuth2.  
   - Point to the Google Sheets document containing the video URLs.  
   - Select the sheet "Sheet1" (gid=0).  
   - Optionally filter rows if needed; otherwise, read all rows.  
   - Connect output of Manual Trigger to this node.

3. **Add SplitInBatches Node for Looping**  
   - Add "SplitInBatches" node to process each YouTube link one by one.  
   - No batch size specified means default 1.  
   - Connect output of Google Sheets node to this node.

4. **Add HTTP Request Node to Call Dumpling AI API**  
   - Add HTTP Request node named "Dumpling AI: Get YouTube Transcript".  
   - Set method to POST.  
   - URL: `https://app.dumplingai.com/api/v1/get-youtube-transcript`  
   - Body: JSON with parameter `"videoUrl"` set dynamically from the current batch item (`={{ $json.link }}`).  
   - Authentication: Set to HTTP Header Auth with credentials for Dumpling AI API key.  
   - Connect output of SplitInBatches node to this node.

5. **Add LangChain OpenAI Node to Generate Newsletter**  
   - Add LangChain OpenAI node named "GPT-4o: Write Newsletter Draft from Transcript".  
   - Select model "chatgpt-4o-latest".  
   - Prepare system message instructing GPT-4o to write a newsletter draft from transcript with specified style and formatting.  
   - Prepare user message injecting transcript dynamically (`=Transcript: {{ $json.transcript }}`).  
   - Configure OpenAI API credentials.  
   - Connect output of HTTP Request node to this node.

6. **Add Google Sheets Node to Log Newsletter Draft**  
   - Add another Google Sheets node named "Log Newsletter Draft to Google Sheets".  
   - Set operation to "Append or Update".  
   - Configure credentials for Google Sheets OAuth2 (same sheet as input).  
   - Map columns:  
     - `link`: from original video link (expression referencing input node).  
     - `blog post`: newsletter draft content (`$json.message.content`).  
   - Set matching column as `link` to avoid duplicates.  
   - Connect output of LangChain node to this node.

7. **Connect Google Sheets Log Node Back to SplitInBatches Node**  
   - Connect output of the Google Sheets log node back to the SplitInBatches node to continue batch processing.

8. **Add Gmail Node for Notification**  
   - Add Gmail node named "Send Email Notification When Draft is Ready".  
   - Configure Gmail OAuth2 credentials.  
   - Set recipient email (e.g., example@gmail.com).  
   - Subject: "Newsletter Draft Created from YouTube Transcript".  
   - Message: Static text informing the draft is ready.  
   - Connect the secondary output of SplitInBatches node to this Gmail node to send notifications after processing each video or batch.

9. **Add Sticky Note for Documentation (Optional)**  
   - Add a sticky note to the canvas with workflow description, purpose, tools used, and credential reminders.

10. **Finalize and Test**  
    - Save the workflow.  
    - Test by triggering the manual node.  
    - Verify each node executes correctly, credentials are valid, and data flows properly.  
    - Check Google Sheets for logged newsletter drafts and email inbox for notifications.

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                          |
|-----------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow leverages Dumpling AI’s API for transcript extraction, which requires an HTTP header authentication token.          | Dumpling AI API documentation: https://app.dumplingai.com/docs/api                                      |
| GPT-4o is used through LangChain n8n node, requiring OpenAI API key with access to chatgpt-4o-latest model.                      | OpenAI API docs: https://platform.openai.com/docs/models/gpt-4o                                         |
| Google Sheets must have at least two columns: `link` for YouTube URLs and `blog post` to store generated newsletter drafts.      | Google Sheets setup instructions: https://support.google.com/docs/answer/6000292                         |
| Gmail node uses OAuth2 for sending notification emails; ensure the Gmail account allows n8n access and has necessary scopes.       | Gmail OAuth2 setup guide: https://developers.google.com/identity/protocols/oauth2                         |
| The workflow assumes YouTube URLs are valid and accessible; invalid or private videos may cause transcript fetching errors.       |                                                                                                         |
| Consider adding error handling or retry mechanisms for network/API failures to improve robustness.                                 | n8n documentation on error workflows: https://docs.n8n.io/nodes/handling-error/                           |
| This workflow is designed for manual trigger but can be scheduled or triggered by webhook for automation.                         |                                                                                                         |

---

**Disclaimer:** The provided text is exclusively derived from an n8n automated workflow. It adheres strictly to content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.

---