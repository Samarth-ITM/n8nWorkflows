Generate Podcast Transcript Summaries & Keywords with OpenAI and Gmail

https://n8nworkflows.xyz/workflows/generate-podcast-transcript-summaries---keywords-with-openai-and-gmail-6661


# Generate Podcast Transcript Summaries & Keywords with OpenAI and Gmail

### 1. Workflow Overview

This workflow automates the process of generating concise podcast episode summaries and extracting relevant keywords from provided transcripts using OpenAI’s language models, then emails the results via Gmail. It is designed for content creators or marketers who want to repurpose podcast transcripts into useful metadata for SEO, show notes, or promotional material.

The workflow is logically divided into six functional blocks:

- **1.1 Input Reception:** Manual or automated provision of the raw podcast transcript.
- **1.2 AI Processing:** Two parallel OpenAI nodes generate a summary and extract keywords from the transcript.
- **1.3 Output Consolidation:** Aggregation of AI-generated data into a structured format.
- **1.4 Result Distribution:** Sending the summary and keywords via email.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception

**Overview:**  
This block initiates the workflow and inputs the raw transcript data. It supports manual triggering for testing and can be replaced or extended for automation (e.g., RSS feed updates, webhooks).

**Nodes Involved:**  
- Manual Trigger  
- Input Raw Transcript

**Node Details:**

- **Manual Trigger**  
  - *Type & Role:* Trigger node; initiates workflow manually for testing.  
  - *Configuration:* Default manual trigger with no parameters.  
  - *Expressions:* None.  
  - *Connections:* Output connects to “Input Raw Transcript.”  
  - *Edge Cases:* None, manual action required; no automatic retries.  
  - *Sub-workflow:* None.

- **Input Raw Transcript**  
  - *Type & Role:* Set node; holds the full unedited podcast transcript text.  
  - *Configuration:* A single field `rawTranscript` set with sample transcript text (can be replaced with dynamic input).  
  - *Expressions:* Static text input for testing; can be dynamically set for production.  
  - *Connections:* Input from “Manual Trigger”; outputs to two AI nodes (“AI: Generate Summary” and “AI: Extract Keywords”).  
  - *Edge Cases:* Transcript must be complete and high-quality for good AI results; empty or partial input will degrade output.  
  - *Sub-workflow:* None.

---

#### Block 1.2: AI Processing

**Overview:**  
Two parallel OpenAI nodes process the transcript: one generates a concise summary, the other extracts keywords and topics relevant for SEO or tagging.

**Nodes Involved:**  
- AI: Generate Summary  
- AI: Extract Keywords

**Node Details:**

- **AI: Generate Summary**  
  - *Type & Role:* OpenAI node; generates a concise, engaging summary of the transcript.  
  - *Configuration:*  
    - Model: `gpt-3.5-turbo` (default), upgradeable to `gpt-4o` or `gpt-4` for enhanced output.  
    - Messages: System prompt instructs summarization style and focus (no conversational openings); user prompt includes transcript via expression `{{ $json.rawTranscript }}`.  
    - Credential: OpenAI API key must be configured under credentials.  
  - *Expressions:* Uses transcript from “Input Raw Transcript.”  
  - *Connections:* Outputs summary to “Consolidate Output.”  
  - *Edge Cases:*  
    - API key invalid or quota exceeded causes authentication errors.  
    - Transcript too long may hit API token limits.  
    - Model unavailability or timeouts possible.  
  - *Sub-workflow:* None.

- **AI: Extract Keywords**  
  - *Type & Role:* OpenAI node; extracts 5-10 relevant keywords and topics as a comma-separated list.  
  - *Configuration:*  
    - Model: Matches “AI: Generate Summary” for consistency.  
    - Messages: System prompt requests expert SEO keyword extraction; user prompt includes transcript.  
    - Credential: Same OpenAI API key as above.  
  - *Expressions:* Uses transcript from “Input Raw Transcript.”  
  - *Connections:* Outputs keywords to “Consolidate Output.”  
  - *Edge Cases:* Same as summary node; additionally, output format may vary if AI output is not strictly comma-separated.  
  - *Sub-workflow:* None.

---

#### Block 1.3: Output Consolidation

**Overview:**  
Aggregates summary and keyword outputs into a single structured JSON for easier downstream use.

**Nodes Involved:**  
- Consolidate Output

**Node Details:**

- **Consolidate Output**  
  - *Type & Role:* Set node; maps AI-generated summary and keywords into named fields `episodeSummary` and `episodeKeywords`.  
  - *Configuration:*  
    - `episodeSummary` is set from `{{ $node["AI: Generate Summary"].json.choices[0].message.content }}`  
    - `episodeKeywords` is set from `{{ $node["AI: Extract Keywords"].json.choices[0].message.content }}`  
  - *Expressions:* Pulls data from previous AI nodes.  
  - *Connections:* Feeds “Email Results” node.  
  - *Edge Cases:* If prior AI nodes fail, these fields may be empty or undefined, causing failure in email content.  
  - *Sub-workflow:* None.

---

#### Block 1.4: Result Distribution

**Overview:**  
Sends the consolidated AI outputs via email using Gmail API.

**Nodes Involved:**  
- Email Results

**Node Details:**

- **Email Results**  
  - *Type & Role:* Gmail node; sends an email with the summary and keywords.  
  - *Configuration:*  
    - Credential: Gmail API OAuth2 credential must be set up with appropriate scopes.  
    - From Email: Must match authenticated Gmail account.  
    - To Email: User must replace placeholder (`YOUR_RECIPIENT_EMAIL@example.com`) with actual recipient email.  
    - Subject: Predefined as “New Podcast Content: Summary & Keywords Ready!”  
    - Text: Uses expressions to include `episodeSummary` and `episodeKeywords` from “Consolidate Output.”  
  - *Expressions:*  
    - `{{ $json.episodeSummary }}`  
    - `{{ $json.episodeKeywords }}`  
  - *Connections:* Receives input from “Consolidate Output.”  
  - *Edge Cases:*  
    - Gmail credential misconfiguration leads to auth errors.  
    - Incorrect “To Email” results in undelivered messages.  
    - API rate limits or network issues may cause failures.  
  - *Sub-workflow:* None.

---

### 3. Summary Table

| Node Name           | Node Type          | Functional Role                | Input Node(s)         | Output Node(s)          | Sticky Note                                                                 |
|---------------------|--------------------|-------------------------------|-----------------------|-------------------------|----------------------------------------------------------------------------|
| Manual Trigger      | Manual Trigger     | Workflow initiation/testing    | —                     | Input Raw Transcript    | This manual trigger is for testing; replace with RSS feed or webhook for production. |
| Input Raw Transcript | Set                | Holds raw podcast transcript   | Manual Trigger        | AI: Generate Summary, AI: Extract Keywords | Paste or receive transcript here; quality of input affects AI output.        |
| AI: Generate Summary | OpenAI             | Generates episode summary      | Input Raw Transcript  | Consolidate Output       | Use OpenAI with gpt-3.5-turbo or better; system prompt guides summary style.  |
| AI: Extract Keywords | OpenAI             | Extracts keywords from transcript | Input Raw Transcript  | Consolidate Output       | Extracts 5-10 SEO-relevant keywords; output as comma-separated list.           |
| Consolidate Output   | Set                | Aggregates summary & keywords  | AI: Generate Summary, AI: Extract Keywords | Email Results           | Maps AI outputs to `episodeSummary` and `episodeKeywords` fields.             |
| Email Results       | Gmail              | Sends email with summary & keywords | Consolidate Output     | —                       | Configure Gmail API credential; replace recipient email placeholder.           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add `Manual Trigger` node for testing workflow execution.

2. **Create Input Raw Transcript Node (Set)**  
   - Add `Set` node named “Input Raw Transcript.”  
   - Add a field `rawTranscript` (String) and paste full podcast transcript text for testing.  
   - Connect output of “Manual Trigger” to this node’s input.

3. **Create AI: Generate Summary Node (OpenAI)**  
   - Add `OpenAI` node named “AI: Generate Summary.”  
   - Select model: `gpt-3.5-turbo` (or `gpt-4o`/`gpt-4` if available).  
   - Configure credential: Create and select OpenAI API Credential with your API key.  
   - Messages setup:  
     - System role message: instruct to provide concise, comprehensive summary without conversational openings.  
     - User role message: include transcript with expression `{{ $json.rawTranscript }}`.  
   - Connect output of “Input Raw Transcript” to this node.

4. **Create AI: Extract Keywords Node (OpenAI)**  
   - Add another `OpenAI` node named “AI: Extract Keywords.”  
   - Use same model and credentials as summary node.  
   - Messages setup:  
     - System role message: instruct to extract 5-10 SEO-relevant keywords as a comma-separated list.  
     - User role message: include transcript with expression `{{ $json.rawTranscript }}`.  
   - Connect output of “Input Raw Transcript” to this node.

5. **Create Consolidate Output Node (Set)**  
   - Add `Set` node named “Consolidate Output.”  
   - Add fields:  
     - `episodeSummary` with value `={{ $node["AI: Generate Summary"].json.choices[0].message.content }}`  
     - `episodeKeywords` with value `={{ $node["AI: Extract Keywords"].json.choices[0].message.content }}`  
   - Connect outputs of both AI nodes to this node’s input (parallel inputs).

6. **Create Email Results Node (Gmail)**  
   - Add `Gmail` node named “Email Results.”  
   - Configure Gmail API Credential by creating a new credential with OAuth2 for your Gmail account.  
   - Set “From Email” to your authenticated Gmail address.  
   - Set “To Email” to your desired recipient email address (replace placeholder).  
   - Set “Subject” to `New Podcast Content: Summary & Keywords Ready!`.  
   - Set “Text” with the following expression:  
     ```
     Hello!

     Your automated podcast content repurposer has finished its work.

     ### Episode Summary:

     {{ $json.episodeSummary }}

     ### Keywords:

     {{ $json.episodeKeywords }}

     ---

     *This content was generated automatically by n8n.*
     ```  
   - Connect output of “Consolidate Output” to this node.

7. **Save and Execute**  
   - Save the workflow.  
   - Use the “Manual Trigger” node’s Execute button to test entire flow.  
   - Verify email delivery of summary and keywords.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                               |
|------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| For production, replace “Manual Trigger” and “Input Raw Transcript” nodes with dynamic input sources such as RSS feeds, webhooks, or transcription APIs. | Workflow overview and node notes.                             |
| OpenAI API keys start with `sk-`; ensure you keep them secure and monitor usage to avoid quota issues. | OpenAI credential setup.                                      |
| Gmail API OAuth2 credentials require proper scopes and consent screen setup in Google Cloud Console. | Gmail credential setup instructions within n8n documentation.|
| Using GPT-4 models can improve output quality but will increase API costs. Adjust accordingly.        | AI node notes.                                                |
| Ensure the recipient email in the Gmail node is updated to avoid sending messages to a placeholder.   | Email Results node notes.                                     |

---

**Disclaimer:**  
The provided text originates solely from an automated n8n workflow. It complies strictly with applicable content policies and contains no illegal, offensive, or protected elements. All processed data are legal and public.