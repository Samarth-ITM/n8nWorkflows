Transcribe & Summarize Audio with Whisper and GPT, from Google Drive to Notion

https://n8nworkflows.xyz/workflows/transcribe---summarize-audio-with-whisper-and-gpt--from-google-drive-to-notion-6139


# Transcribe & Summarize Audio with Whisper and GPT, from Google Drive to Notion

### 1. Workflow Overview

This workflow automates the process of transcribing audio files uploaded to a specific Google Drive folder, generating concise summaries of their content using AI, and then creating pages in a Notion database with those summaries. It is designed for users who want to effortlessly convert audio content into summarized text entries in Notion, enabling easier review, organization, and sharing.

The workflow is logically grouped into three main blocks:

- **1.1 Input Reception & Download:** Watches a designated Google Drive folder for new audio files and downloads them when detected.
- **1.2 AI Processing:** Uses OpenAI Whisper to transcribe the downloaded audio, then employs GPT-4 (via LangChain nodes) to summarize the transcription.
- **1.3 Output to Notion:** Creates a new page in a specified Notion database with the AI-generated summary.

Additionally, sticky notes within the workflow provide guidance, context, and contact information.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Download

**Overview:**  
This block triggers the workflow when a new file appears in a specific Google Drive folder and downloads the file for further processing.

**Nodes Involved:**  
- Google Drive Trigger  
- Download file  
- Sticky Note2 (contextual note about downloading from Drive)

**Node Details:**

- **Google Drive Trigger**  
  - *Type:* Trigger node for Google Drive  
  - *Role:* Watches a specific Google Drive folder (ID: `1EMG03GI9ewXAnKj_vT-aAmySU8T-GQZx`) for newly created files of any type.  
  - *Configuration:* Polls every minute; triggers only on new file creation in the specified folder.  
  - *Credentials:* Google Drive OAuth2 account connected.  
  - *Connections:* Outputs file metadata to "Download file" node.  
  - *Edge Cases:* Possible failures include authentication token expiration, folder permissions issues, or API rate limiting.  
  - *Version:* TypeVersion 1.

- **Download file**  
  - *Type:* Google Drive node for file operations  
  - *Role:* Downloads the file identified by the ID received from the trigger node.  
  - *Configuration:* Uses the file ID from the trigger node to perform the download operation.  
  - *Credentials:* Same Google Drive OAuth2 account.  
  - *Connections:* Outputs the binary file data to "Transcribe a recording" node.  
  - *Edge Cases:* Failures may occur due to network issues, file access permissions, or corrupted files.  
  - *Version:* TypeVersion 3.

- **Sticky Note2**  
  - *Content:* "Downloading from Drive(BinarY)"  
  - *Purpose:* Provides visual grouping and context for this block.

---

#### 1.2 AI Processing

**Overview:**  
This block transcribes the downloaded audio file to text using OpenAI Whisper and then summarizes the transcription using GPT-4 via LangChain AI Agent nodes.

**Nodes Involved:**  
- Transcribe a recording  
- AI Agent  
- OpenAI Chat Model  
- Sticky Note (AI Transcriber and Summarizer Nodes)

**Node Details:**

- **Transcribe a recording**  
  - *Type:* LangChain OpenAI Audio node  
  - *Role:* Converts audio binary data into a text transcription using OpenAI Whisper.  
  - *Configuration:* Uses the "transcribe" operation under resource "audio". Input is the downloaded audio file.  
  - *Credentials:* OpenAI API key configured.  
  - *Connections:* Sends the transcription text to the "AI Agent" node.  
  - *Edge Cases:* Possible errors include unsupported audio format, corrupted audio, API limits, or transcription inaccuracies.  
  - *Version:* 1.8.

- **AI Agent**  
  - *Type:* LangChain AI Agent node  
  - *Role:* Receives transcribed text and prompts GPT-4 to generate a concise summary.  
  - *Configuration:* Custom prompt instructs the AI to act as a professional summarizer and return a clear, concise summary of the transcription.  
  - *Prompt Example:*  
    ```
    Just suppose you are a professional Summarizer and helper. 
    This is the data that is coming from a previous node 
    Audio-transcribed : {{ $json.text }} 
    and its the transcribed version of an mp3 file. Just Summarize the text and return to me a clear and a concize summary of the text.
    ```  
  - *Credentials:* OpenAI API key.  
  - *Connections:* Outputs summary text to "Create a page" node.  
  - *Edge Cases:* Failures can include prompt errors, API rate limiting, or incomplete input data.  
  - *Version:* 2.1.

- **OpenAI Chat Model**  
  - *Type:* LangChain OpenAI Chat Model node  
  - *Role:* Acts as the language model backend for the AI Agent using GPT-4.1-mini variant.  
  - *Configuration:* Model selected is "gpt-4.1-mini".  
  - *Credentials:* OpenAI API key.  
  - *Connections:* Feeds AI Agent node.  
  - *Edge Cases:* Model availability changes, API rate limits, or credential issues.  
  - *Version:* 1.2.

- **Sticky Note**  
  - *Content:* "AI Transcriber and Summarizer Nodes"  
  - *Purpose:* Clarifies the purpose of this block.

---

#### 1.3 Output to Notion

**Overview:**  
This block creates a new page in a specified Notion database with the summary generated by the AI, enabling easy access and organization of summarized content.

**Nodes Involved:**  
- Create a page  
- Sticky Note1 (contextual note about Notion upload)

**Node Details:**

- **Create a page**  
  - *Type:* Notion node  
  - *Role:* Creates a new page in a predefined Notion database/page with the summary as a heading.  
  - *Configuration:*  
    - Target Notion page ID: `234c7c0bc348804da681f5d80eca6712`  
    - Adds a heading_1 block containing the summary text from the AI Agent node output.  
  - *Credentials:* Notion API connected.  
  - *Connections:* Final node, no further outputs.  
  - *Edge Cases:* Possible failures include invalid Notion credentials, access permissions, or API throttling.  
  - *Version:* 2.2.

- **Sticky Note1**  
  - *Content:* "Uploading to the Notion Database"  
  - *Purpose:* Provides context for this output step.

---

### 3. Summary Table

| Node Name           | Node Type                          | Functional Role                   | Input Node(s)       | Output Node(s)        | Sticky Note                                 |
|---------------------|----------------------------------|---------------------------------|---------------------|-----------------------|---------------------------------------------|
| Google Drive Trigger | n8n-nodes-base.googleDriveTrigger| Watches for new files in Drive  | -                   | Download file         | Downloading from Drive(BinarY)               |
| Download file       | n8n-nodes-base.googleDrive        | Downloads file from Drive       | Google Drive Trigger | Transcribe a recording | Downloading from Drive(BinarY)               |
| Transcribe a recording| @n8n/n8n-nodes-langchain.openAi | Transcribes audio to text       | Download file        | AI Agent              | AI Transcriber and Summarizer Nodes          |
| AI Agent            | @n8n/n8n-nodes-langchain.agent    | Summarizes transcription        | Transcribe a recording, OpenAI Chat Model | Create a page | AI Transcriber and Summarizer Nodes          |
| OpenAI Chat Model   | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides GPT-4 language model  | -                   | AI Agent              | AI Transcriber and Summarizer Nodes          |
| Create a page       | n8n-nodes-base.notion             | Creates page with summary in Notion | AI Agent            | -                     | Uploading to the Notion Database              |
| Sticky Note         | n8n-nodes-base.stickyNote         | Contextual notes                | -                   | -                     | AI Transcriber and Summarizer Nodes          |
| Sticky Note1        | n8n-nodes-base.stickyNote         | Contextual notes                | -                   | -                     | Uploading to the Notion Database              |
| Sticky Note2        | n8n-nodes-base.stickyNote         | Contextual notes                | -                   | -                     | Downloading from Drive(BinarY)                |
| Sticky Note3        | n8n-nodes-base.stickyNote         | Author and usage instructions   | -                   | -                     | Created by : Abdullah Dilshad ... (full text)|

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger**  
   - Add node: *Google Drive Trigger*  
   - Set event to *fileCreated*  
   - Set file type to *all*  
   - Set folder to watch: use folder ID `1EMG03GI9ewXAnKj_vT-aAmySU8T-GQZx`  
   - Set polling interval to every minute  
   - Connect your Google Drive OAuth2 credentials  

2. **Create Download file node**  
   - Add node: *Google Drive*  
   - Set operation to *download*  
   - For File ID, set expression: `{{$json["id"]}}` from trigger node output  
   - Use the same Google Drive OAuth2 credentials  
   - Connect output of Google Drive Trigger to this node  

3. **Create Transcribe a recording node**  
   - Add node: *OpenAI Audio (LangChain)*  
   - Set resource to *audio*  
   - Set operation to *transcribe*  
   - Connect output of Download file node to this node (binary input)  
   - Use OpenAI API credentials  

4. **Create OpenAI Chat Model node**  
   - Add node: *LangChain OpenAI Chat Model*  
   - Select model *gpt-4.1-mini*  
   - Use OpenAI API credentials  

5. **Create AI Agent node**  
   - Add node: *LangChain AI Agent*  
   - Set prompt type as *define*  
   - Configure prompt text with expression:  
     ```
     Just suppose you are a professional Summarizer and helper. 
     This is the data that is coming from a previous node 
     Audio-transcribed : {{ $json.text }} 
     and its the transcribed version of an mp3 file. Just Summarize the text and return to me a clear and a concize summary of the text.
     ```  
   - Connect output from Transcribe a recording node to this node  
   - Configure AI language model input to connect from OpenAI Chat Model node  
   - Use OpenAI API credentials  

6. **Create Create a page node (Notion)**  
   - Add node: *Notion*  
   - Operation: *Create a page*  
   - Select target page ID: `234c7c0bc348804da681f5d80eca6712`  
   - Add a block: heading_1 with text content from AI Agent node output, expression: `{{$json["output"]}}`  
   - Use Notion API credentials  
   - Connect output of AI Agent node to this node  

7. **Add Sticky Notes (optional but recommended for clarity)**  
   - Add sticky notes near each block with content as per the original workflow for documentation and visual guidance.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                 |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------|
| Created by : Abdullah Dilshad  
Reach me At : iamabdullahdilshad@gmail.com  

Who It's For 🧑‍💻  
This system is ideal for individuals or teams who regularly work with audio content and need an efficient way to convert it into text and extract key information. This includes:

- Researchers: Quickly transcribe interviews or lectures and get summaries for analysis.
- Content Creators: Generate transcripts for podcasts or videos and create quick summaries for social media or show notes.
- Students: Transcribe study materials or lectures and receive summarized notes.
- Business Professionals: Automate the transcription of meeting recordings and get key takeaways.

How to Set Up 🛠️  
Setting up this system in n8n involves configuring each of the described nodes:

- Google Drive Trigger: Connect Google Drive, specify folder.
- OpenAI Whisper (Transcribe Recording node): Provide OpenAI API key, receive audio.
- GPT-4 (Message Model node): Provide OpenAI API key, configure prompt for summary.
- Notion (Create Page node): Connect Notion, specify database, map summary text.

| Full author and usage instructions embedded in Sticky Note3 | See Sticky Note3 in workflow |

---

**Disclaimer:** This documentation is generated from an n8n workflow and strictly follows current content policies. All data handled are legal and publicly accessible.