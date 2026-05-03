Transform Voice Transcripts to LinkedIn Posts with Claude AI and Email Automation

https://n8nworkflows.xyz/workflows/transform-voice-transcripts-to-linkedin-posts-with-claude-ai-and-email-automation-4874


# Transform Voice Transcripts to LinkedIn Posts with Claude AI and Email Automation

### 1. Workflow Overview

This workflow automates the transformation of voice transcription emails into polished LinkedIn posts using Claude AI (Anthropic Claude Sonnet 4) and sends the generated content via email for easy posting or further processing. It is designed for content creators who prefer to dictate ideas via voice memos, transcribe them automatically, and streamline social media content creation with AI assistance.

**Target Use Cases:**  
- Professionals who create content via voice memos and want to automate LinkedIn post creation.  
- Marketing teams seeking to maintain consistent brand voice by integrating inspiration from curated example posts.  
- Users aiming to save time by automating content formatting, style matching, and delivery.

**Logical Blocks:**  
- **1.1 Email Reception and Filtering:** Monitors incoming emails, filters by sender, and extracts relevant transcription content.  
- **1.2 Data Cleaning and Preparation:** Cleans email JSON, extracts transcription text and metadata, and prepares inspiration content.  
- **1.3 Content Merging:** Combines raw transcription with curated example posts from a Google Doc.  
- **1.4 AI Content Generation:** Uses Anthropic Claude model through LangChain to generate a LinkedIn post and an image description in JSON format.  
- **1.5 AI Response Cleaning:** Parses and sanitizes AI output to extract usable content.  
- **1.6 Notification via Email:** Sends the generated LinkedIn post and image description to the user’s email.

---

### 2. Block-by-Block Analysis

#### 2.1 Email Reception and Filtering

**Overview:**  
Receives emails via IMAP, filters them to only process emails from a designated sender (the user’s own email), ensuring the workflow triggers only on relevant transcription emails.

**Nodes Involved:**  
- Email to Monitor  
- Limit email sender  
- Limit email sender1  

**Node Details:**  

- **Email to Monitor**  
  - Type: IMAP Email Read  
  - Role: Continuously monitors a dedicated email inbox for new messages.  
  - Configuration: Uses IMAP credentials linked to the dedicated email address receiving voice transcription emails.  
  - Input/Output: Triggers on new email received; outputs raw email JSON including headers and body.  
  - Edge Cases: Connection failures to IMAP server, authentication errors, or no new emails.  
  - Notes: Requires IMAP access enabled on the email service provider.

- **Limit email sender** and **Limit email sender1**  
  - Type: Filter  
  - Role: Ensures only emails from the specified sender address (user’s email) continue processing.  
  - Configuration: Checks if the “from” field in the incoming email JSON contains the configured email string.  
  - Input: Raw email JSON from Email to Monitor.  
  - Output: Passes filtered emails to the next nodes; blocks others.  
  - Edge Cases: Misconfigured sender email leads to no workflow trigger; case sensitivity enforced.  
  - Notes: Two identical filters exist to feed two parallel paths downstream.

---

#### 2.2 Data Cleaning and Preparation

**Overview:**  
Extracts the transcription text and date from the filtered email JSON and fetches inspiration posts from a Google Doc for style reference.

**Nodes Involved:**  
- Clean JSON  
- Google Doc Post Inspiration  

**Node Details:**  

- **Clean JSON**  
  - Type: Set  
  - Role: Extracts and restructures the email body text (`textPlain`) and the sent date from the email JSON into simplified fields named `Content` and `Date`.  
  - Configuration: Assigns `Content` as `{{ $json.textPlain }}` and `Date` as `{{ $json.date }}`.  
  - Input: Filtered email JSON from Limit email sender node.  
  - Output: Cleaned JSON with only relevant transcription data passed forward.  
  - Edge Cases: Missing or malformed `textPlain` or `date` fields in email JSON.

- **Google Doc Post Inspiration**  
  - Type: HTTP Request  
  - Role: Retrieves example LinkedIn posts stored in a public Google Doc to guide AI style and tone.  
  - Configuration: HTTP GET request to a configured Google Doc URL that must be public or shareable with link access.  
  - Input: Triggered from Limit email sender1 filter (parallel path).  
  - Output: Raw text data of example posts, to be merged later.  
  - Edge Cases: HTTP request failures, permission errors if the Google Doc is not public, or incorrect URL.

---

#### 2.3 Content Merging

**Overview:**  
Combines user’s transcription content with the inspiration posts fetched from the Google Doc, preparing a composite input to the AI model.

**Nodes Involved:**  
- Merge  
- Aggregate  

**Node Details:**  

- **Merge**  
  - Type: Merge  
  - Role: Joins the cleaned user transcription JSON and the Google Doc inspiration content into a single data stream.  
  - Configuration: Defaults; merges inputs based on index alignment.  
  - Input: Receives cleaned transcription JSON and raw inspiration content.  
  - Output: Combined data for aggregation.  
  - Edge Cases: Mismatched or missing inputs cause incomplete merges.

- **Aggregate**  
  - Type: Aggregate  
  - Role: Aggregates fields `data` (from Google Doc) and `Content` (user transcription) into arrays for batch processing by AI.  
  - Configuration: Aggregates specified fields without additional grouping.  
  - Input: Merged data from Merge node.  
  - Output: Aggregated dataset containing inspiration posts and transcription content arrays.  
  - Edge Cases: Empty inputs lead to no data aggregation.

---

#### 2.4 AI Content Generation

**Overview:**  
Uses Anthropic Claude Sonnet 4 via LangChain to generate a LinkedIn post and an image description based on the merged content, following a strict prompt and JSON output format.

**Nodes Involved:**  
- Anthropic Chat Model  
- Basic LLM Chain  

**Node Details:**  

- **Anthropic Chat Model**  
  - Type: LangChain LM Chat Model (Anthropic)  
  - Role: Interfaces with Anthropic Claude Sonnet 4 API to generate AI text completions.  
  - Configuration: Uses credentials for Anthropic API; model set to `claude-sonnet-4-20250514`.  
  - Input: Text prompt from Basic LLM Chain node; supports streaming and retry on failure.  
  - Output: Raw AI-generated JSON string containing `post_content` and `image_description`.  
  - Edge Cases: API rate limits, authentication failures, network timeouts, malformed responses.

- **Basic LLM Chain**  
  - Type: LangChain Chain LLM  
  - Role: Constructs detailed prompt combining example posts and user transcription, instructing AI to produce JSON with post content and image description.  
  - Configuration:  
    - Prompt includes style examples from Google Doc and user transcription content.  
    - Enforces output to be valid JSON with ASCII only, no special characters.  
    - Word count constrained to 150-300 words for LinkedIn posts.  
  - Input: Aggregated data containing inspiration posts and transcription content.  
  - Output: Text prompt payload for Anthropic Chat Model.  
  - Edge Cases: Expression failures if input data arrays are empty or malformed; prompt misformatting.

---

#### 2.5 AI Response Cleaning

**Overview:**  
Parses the raw JSON string response from the AI, extracts and cleans the LinkedIn post content and image description, handling common formatting issues or partial responses.

**Nodes Involved:**  
- Clean JSON Response  

**Node Details:**  

- **Clean JSON Response**  
  - Type: Code (JavaScript)  
  - Role: Uses regex to extract `post_content` and `image_description` fields from AI response string reliably, avoiding JSON parse errors due to malformed or truncated responses.  
  - Configuration:  
    - Regex extraction for `post_content` and `image_description`.  
    - Cleans escaped characters (`\n`, `\"`, `\t`).  
    - Returns default image description if missing.  
    - Flags if response appears truncated.  
    - Returns error debug info if extraction fails.  
  - Input: Raw AI response JSON string from Basic LLM Chain node.  
  - Output: Clean structured JSON with parsed fields or error info.  
  - Edge Cases: Partial or corrupted AI output, regex misses, missing fields.

---

#### 2.6 Notification via Email

**Overview:**  
Sends a formatted email to the user containing the generated LinkedIn post content and image description, ready for posting or further use.

**Nodes Involved:**  
- Send Email  

**Node Details:**  

- **Send Email**  
  - Type: Email Send  
  - Role: Sends an HTML email containing the LinkedIn post and image description generated by AI.  
  - Configuration:  
    - Uses SMTP credentials configured in n8n (not shown in JSON).  
    - Email subject includes current date.  
    - HTML email body uses Handlebars expressions to insert AI-generated content from Clean JSON Response node.  
  - Input: Parsed AI output JSON.  
  - Output: Email sent confirmation.  
  - Edge Cases: SMTP authentication errors, connectivity issues, or email delivery failures.

---

### 3. Summary Table

| Node Name                 | Node Type                      | Functional Role                               | Input Node(s)           | Output Node(s)         | Sticky Note                                            |
|---------------------------|--------------------------------|----------------------------------------------|-------------------------|------------------------|--------------------------------------------------------|
| Email to Monitor          | Email Read (IMAP)              | Monitors incoming emails for transcription   | -                       | Limit email sender, Limit email sender1 | See Sticky Note on configuring email monitoring and filters |
| Limit email sender        | Filter                        | Filters emails from authorized sender        | Email to Monitor        | Clean JSON             | See Sticky Note on configuring email monitoring and filters |
| Limit email sender1       | Filter                        | Filters emails from authorized sender (parallel) | Email to Monitor        | Google Doc Post Inspiration | See Sticky Note on configuring email monitoring and filters |
| Clean JSON                | Set                           | Extracts transcription text and date          | Limit email sender       | Merge                  | See Sticky Note on data processing steps                |
| Google Doc Post Inspiration | HTTP Request                 | Fetches example LinkedIn posts from Google Doc | Limit email sender1      | Merge                  | See Sticky Note on data processing steps                |
| Merge                     | Merge                         | Combines transcription with inspiration posts | Clean JSON, Google Doc Post Inspiration | Aggregate              | See Sticky Note on data processing steps                |
| Aggregate                 | Aggregate                     | Aggregates content fields for AI input        | Merge                   | Basic LLM Chain        | See Sticky Note on data processing steps                |
| Basic LLM Chain           | LangChain LLM Chain           | Constructs AI prompt to generate LinkedIn post | Aggregate                | Clean JSON Response    | See Sticky Note on AI configuration                      |
| Anthropic Chat Model      | LangChain LM Chat Anthropic   | Calls Anthropic Claude AI API for generation  | Basic LLM Chain (ai_languageModel) | Basic LLM Chain (loopback) | See Sticky Note on AI configuration                      |
| Clean JSON Response       | Code (JavaScript)             | Parses and cleans AI JSON response             | Basic LLM Chain          | Send Email             | See Sticky Note on AI configuration                      |
| Send Email                | Email Send                    | Sends formatted email with generated content  | Clean JSON Response      | -                      | See Sticky Note on emailing yourself                     |
| Sticky Note               | Sticky Note                   | Configuration instructions for email setup    | -                       | -                      | Configuration instructions for email monitoring and filters |
| Sticky Note1              | Sticky Note                   | Instructions on data processing and inspiration doc setup | -                       | -                      | Notes on data cleaning and inspiration document usage    |
| Sticky Note2              | Sticky Note                   | AI configuration and output handling notes    | -                       | -                      | Guidance on Anthropic API, prompt engineering, and parsing |
| Sticky Note3              | Sticky Note                   | Instructions on email notification usage      | -                       | -                      | Details on emailing generated content                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create IMAP Email Read Node:**  
   - Type: Email Read (IMAP)  
   - Configure with credentials for a dedicated email account (e.g., linkedin@yourdomain.com).  
   - Set to trigger on new emails.

2. **Add Two Filter Nodes (Limit email sender & Limit email sender1):**  
   - Both filter nodes should check if the email `from` field contains your email address (e.g., "your.email@domain.com").  
   - Connect the IMAP node output to both filters in parallel.

3. **Create Set Node (Clean JSON):**  
   - Extract and assign two fields:  
     - `Content`: set to the plain text body of the email (`{{ $json.textPlain }}`).  
     - `Date`: set to the email date field (`{{ $json.date }}`).  
   - Connect output of first filter (Limit email sender) to this node.

4. **Create HTTP Request Node (Google Doc Post Inspiration):**  
   - Configure GET request to your publicly accessible Google Doc URL containing example LinkedIn posts.  
   - Connect output of second filter (Limit email sender1) to this node.

5. **Create Merge Node:**  
   - Connect outputs of Clean JSON and Google Doc Post Inspiration nodes as inputs to this node (respectively as second and first inputs).  
   - Use default merging mode.

6. **Create Aggregate Node:**  
   - Configure to aggregate the fields `data` (Google Doc) and `Content` (email transcription) into arrays.  
   - Connect output of Merge node to this node.

7. **Create LangChain Chain LLM Node (Basic LLM Chain):**  
   - Configure prompt with:  
     - Style examples from the aggregated Google Doc data.  
     - User transcription content from aggregated Content field.  
     - Instructions to generate only JSON with `post_content` and `image_description` fields.  
     - Enforce ASCII-only, 150-300 word count for LinkedIn post.  
   - Connect output of Aggregate node to this node.

8. **Create LangChain LM Chat Anthropic Node (Anthropic Chat Model):**  
   - Set model to Claude Sonnet 4 (e.g., `claude-sonnet-4-20250514`).  
   - Add your Anthropic API credentials.  
   - Connect `ai_languageModel` input of Basic LLM Chain to this node’s main output.  
   - Enable retry on failure.

9. **Create Code Node (Clean JSON Response):**  
   - Add JavaScript code to regex-extract `post_content` and `image_description` from AI response string.  
   - Clean escape sequences and handle truncated or malformed responses gracefully.  
   - Connect output of Basic LLM Chain to this node.

10. **Create Email Send Node (Send Email):**  
    - Configure SMTP credentials for sending email.  
    - Subject: `LinkedIn Post Ready - {{ new Date().toLocaleDateString() }}`.  
    - HTML body includes:  
      - Post content from `post_content` field.  
      - Image description from `image_description` field.  
    - Connect output of Clean JSON Response node to this node.

11. **Set Up Sticky Notes (Optional):**  
    - Add sticky notes in the editor to document configuration steps for email setup, Google Doc preparation, AI prompt guidelines, and email sending instructions.

12. **Test Workflow:**  
    - Send a test email from your configured sender with a voice transcription in the body to the dedicated email.  
    - Verify email is received, filtered, processed by AI, and you get a formatted email with generated post content.

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Configure Email Monitoring with a dedicated email address for receiving voice transcriptions; update filters to your sending email.   | Sticky Note on email setup; critical for triggering workflow only on your emails.                                   |
| Use a publicly accessible Google Doc URL containing 10-15 example LinkedIn posts as inspiration for AI style and tone.                  | Sticky Note on data processing; Google Doc must be public or shareable.                                             |
| Sign up and configure Anthropic Claude API with valid credentials; use Claude Sonnet 4 model for content generation.                    | Sticky Note on AI configuration; includes prompt engineering tips.                                                  |
| AI output is strictly JSON formatted; code node uses regex to reliably parse AI responses to avoid JSON parsing errors.                 | Sticky Note on AI configuration; ensures robust content extraction.                                                 |
| Email output includes the formatted LinkedIn post and image description for easy copy-pasting into LinkedIn or other applications.      | Sticky Note on emailing yourself; can be extended to other delivery systems like Airtable or Slack.                  |
| Workflow saves significant time (1–2 hours daily) by automating content creation from voice memos to LinkedIn posts.                   | Workflow description and user testimonial included in email body.                                                   |
| For further customization, integrate nodes to push content to databases or social media scheduling tools.                              | Mentioned in Sticky Note on email notification usage.                                                                |

---

**Disclaimer:**  
The provided text and workflow are generated exclusively using n8n automation. The processing strictly follows current content policies and contains no illegal or offensive content. All handled data is legal and public.