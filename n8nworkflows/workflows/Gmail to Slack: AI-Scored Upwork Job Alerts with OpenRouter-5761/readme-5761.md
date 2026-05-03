Gmail to Slack: AI-Scored Upwork Job Alerts with OpenRouter

https://n8nworkflows.xyz/workflows/gmail-to-slack--ai-scored-upwork-job-alerts-with-openrouter-5761


# Gmail to Slack: AI-Scored Upwork Job Alerts with OpenRouter

### 1. Workflow Overview

This workflow automates the process of monitoring Upwork job alert emails received in Gmail, extracting structured job data, evaluating job fit using AI, and sending filtered job opportunity alerts to a Slack channel. It is designed for freelancers who want to automate job discovery and prioritize high-relevance Upwork listings based on a personalized skill and preference profile.  

Logical blocks:  
- **1.1 Input Reception & Preprocessing:** Poll Gmail for new Upwork job alert emails, mark them as read, and convert email content to Markdown for easier AI parsing.  
- **1.2 Job Data Extraction:** Use an AI information extraction model to parse and structure job details from the email content.  
- **1.3 AI-Based Opportunity Scoring:** Score the match quality between extracted job data and the freelancer’s profile using a detailed AI prompt.  
- **1.4 Filtering & Notification:** Filter scored jobs by a threshold score and send notifications to Slack for jobs with a high match score.  

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Preprocessing

- **Overview:**  
  This block fetches new Upwork job alert emails from Gmail every 10 minutes using a search query. Each message is marked as read and its HTML email body converted to Markdown to facilitate AI analysis.  

- **Nodes Involved:**  
  - Get Filtered Messages (Gmail Trigger)  
  - Mark as Read (Gmail)  
  - Convert To Markdown (Markdown Converter)  

- **Node Details:**  

  1. **Get Filtered Messages**  
     - Type: Gmail Trigger  
     - Role: Poll Gmail for new emails matching Upwork job alert criteria  
     - Configuration:  
       - Search query: `from:(donotreply@upwork.com) subject:(New job:)`  
       - Poll interval: every 10 minutes  
       - Credentials: OAuth2 Gmail  
     - Inputs: None (trigger node)  
     - Outputs: New Gmail messages matching query  
     - Edge cases: Gmail quota limits, OAuth token expiration, no new messages  

  2. **Mark as Read**  
     - Type: Gmail  
     - Role: Mark each fetched email as read to avoid duplicate processing  
     - Configuration: Uses message ID from trigger node  
     - Inputs: Email message ID from "Get Filtered Messages"  
     - Outputs: Confirmation of message status update  
     - Edge cases: Gmail API errors, message ID missing or invalid  

  3. **Convert To Markdown**  
     - Type: Markdown Converter  
     - Role: Convert HTML email body to Markdown text for easier AI parsing  
     - Configuration: Input HTML from "Get Filtered Messages" (`textAsHtml` field)  
     - Inputs: HTML content of email body  
     - Outputs: Markdown formatted email body  
     - Edge cases: Malformed HTML, empty email body  

- **Sticky Note:**  
  Explains that the Gmail search query filters Upwork alerts, requires Upwork Freelancer Plus subscription, and notes the polling frequency’s impact on workflow executions.

---

#### 2.2 Job Data Extraction

- **Overview:**  
  Extracts structured job data fields from the Markdown email content using an AI information extractor node with a defined JSON schema.

- **Nodes Involved:**  
  - Job Data Extractor (Information Extractor AI node)  

- **Node Details:**  

  1. **Job Data Extractor**  
     - Type: Information Extractor (AI)  
     - Role: Parse job alert email text to extract job details as JSON  
     - Configuration:  
       - Input text: Subject and Markdown body of email  
       - JSON schema defines required fields: jobName, jobType, jobDetailUrl, price, descriptionSnippet, country, paymentVerified, clientRating, amountSpent, tags, postedAt, experienceLevel  
     - Inputs: Markdown email content, subject line  
     - Outputs: JSON object with parsed job info  
     - Edge cases: AI extraction failures, incomplete data, schema validation errors  
     - Dependencies: Receives markdown from "Convert To Markdown" and email subject from trigger  

- **Sticky Note:**  
  Notes this block extracts key job information like country and price.

---

#### 2.3 AI-Based Opportunity Scoring

- **Overview:**  
  Evaluates how well each job matches the freelancer’s profile using a custom AI scoring prompt, producing a score and reasoning.

- **Nodes Involved:**  
  - Opportunity Scorer (Information Extractor AI node)  
  - Edit Fields (Set node)  
  - OpenRouter Chat Model1 (AI Language Model node)  

- **Node Details:**  

  1. **OpenRouter Chat Model1**  
     - Type: AI Language Model (OpenRouter Chat)  
     - Role: Provide language model inference for scoring prompt  
     - Configuration: Connected as AI model for "Opportunity Scorer"  
     - Credentials: OpenRouter API key  
     - Edge cases: API rate limits, request timeouts  

  2. **Opportunity Scorer**  
     - Type: Information Extractor (AI)  
     - Role: Generate a match score (1-10) and detailed reasoning comparing job data to freelancer profile  
     - Configuration:  
       - Custom prompt details professional profile placeholder (to be replaced by user)  
       - Input includes extracted job JSON  
       - Output schema: object with integer "score" and string "reasoning"  
     - Inputs: Job JSON from "Job Data Extractor," AI model from "OpenRouter Chat Model1"  
     - Outputs: Evaluation object with score and reasoning  
     - Edge cases: AI model misinterpretation, incomplete profile info, invalid JSON output  

  3. **Edit Fields**  
     - Type: Set  
     - Role: Combine job data and evaluation into unified JSON for downstream use  
     - Configuration: Assigns `evaluation` field from the scorer output and `job` field from job extractor output  
     - Inputs: Evaluation JSON and job JSON  
     - Outputs: Combined JSON object for filtering and notification  

- **Sticky Note:**  
  Instructs user to replace placeholder profile text with detailed personal freelancer profile for accurate scoring.

---

#### 2.4 Filtering & Notification

- **Overview:**  
  Filters jobs to only those with a match score ≥ 7 and sends formatted notifications to a specified Slack channel.

- **Nodes Involved:**  
  - Filter By Score (Filter node)  
  - Send Slack Alert (Slack node)  

- **Node Details:**  

  1. **Filter By Score**  
     - Type: Filter  
     - Role: Allow only jobs with AI evaluation score 7 or higher to proceed  
     - Configuration: Condition `evaluation.score >= 7`  
     - Inputs: Combined job and evaluation object from "Edit Fields"  
     - Outputs: Filtered jobs passing score threshold  
     - Edge cases: Missing score field, score below threshold  

  2. **Send Slack Alert**  
     - Type: Slack  
     - Role: Send message to Slack channel alerting user of matching job opportunity  
     - Configuration:  
       - Message text uses advanced templating with job fields and evaluation details  
       - Channel ID hardcoded (modifiable)  
       - OAuth2 Slack credentials provided  
     - Inputs: Filtered high-score jobs  
     - Outputs: Slack message confirmation  
     - Edge cases: Slack API errors, invalid channel ID, rate limits  

- **Sticky Note:**  
  Notes this block only notifies for jobs scoring 7+ and reminds to select the correct Slack channel.

---

### 3. Summary Table

| Node Name              | Node Type                     | Functional Role                          | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                              |
|------------------------|-------------------------------|----------------------------------------|-------------------------|-------------------------|--------------------------------------------------------------------------------------------------------------------------|
| Get Filtered Messages   | Gmail Trigger                 | Fetch Upwork job alert emails           | None                    | Mark as Read            | Explains Gmail search query, Upwork Freelancer Plus needed, and polling frequency impact                                  |
| Mark as Read           | Gmail                         | Mark email as read                      | Get Filtered Messages    | Convert To Markdown      | See above                                                                                                                |
| Convert To Markdown    | Markdown Converter            | Convert email HTML body to Markdown    | Mark as Read            | Job Data Extractor       | See above                                                                                                                |
| Job Data Extractor     | AI Information Extractor      | Extract structured job data             | Convert To Markdown      | Opportunity Scorer       | Notes extraction of key job info fields                                                                                 |
| OpenRouter Chat Model  | AI Language Model             | Provide LM inference for extractor     | N/A (used by Job Data Extractor) | Job Data Extractor | None                                                                                                                    |
| Opportunity Scorer     | AI Information Extractor      | Score job match quality                  | Job Data Extractor, OpenRouter Chat Model1 | Edit Fields | Reminder to replace profile placeholder text                                                                             |
| OpenRouter Chat Model1 | AI Language Model             | Provide LM inference for scoring       | N/A (used by Opportunity Scorer) | Opportunity Scorer    | See above                                                                                                                |
| Edit Fields            | Set                          | Combine job and evaluation data         | Opportunity Scorer      | Filter By Score          | See above                                                                                                                |
| Filter By Score        | Filter                       | Filter jobs by score threshold          | Edit Fields             | Send Slack Alert         | Only notify for score 7+                                                                                                |
| Send Slack Alert       | Slack                        | Send job alerts to Slack channel        | Filter By Score         | None                    | Reminder to select correct Slack channel                                                                                |
| Sticky Note            | Sticky Note                  | Info on Gmail polling and subscription  | N/A                     | N/A                     | See detailed note content                                                                                                |
| Sticky Note1           | Sticky Note                  | Info on job data extraction              | N/A                     | N/A                     | See detailed note content                                                                                                |
| Sticky Note2           | Sticky Note                  | Info on scoring and profile placeholder | N/A                     | N/A                     | See detailed note content                                                                                                |
| Sticky Note3           | Sticky Note                  | Info on Slack notification filtering    | N/A                     | N/A                     | See detailed note content                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node ("Get Filtered Messages"):**  
   - Type: Gmail Trigger  
   - Credentials: OAuth2 Gmail  
   - Set Polling: Every 10 minutes  
   - Set Search Query: `from:(donotreply@upwork.com) subject:(New job:)`  
   - Enable output of full message details (not simplified)  

2. **Create Gmail Node ("Mark as Read"):**  
   - Type: Gmail  
   - Credentials: OAuth2 Gmail  
   - Operation: Mark as Read  
   - Set Message ID parameter to: `={{ $json.id }}` (from trigger)  
   - Connect "Get Filtered Messages" output to this node’s input  

3. **Create Markdown Node ("Convert To Markdown"):**  
   - Type: Markdown  
   - Input HTML: `={{ $('Get Filtered Messages').item.json.textAsHtml }}`  
   - Output key: `markdown`  
   - Connect "Mark as Read" output to this node’s input  

4. **Create OpenRouter Chat Model Node ("OpenRouter Chat Model"):**  
   - Type: AI Language Model (OpenRouter Chat)  
   - Credentials: OpenRouter API key  
   - No additional parameters needed  
   - This node will be referenced later by Job Data Extractor  

5. **Create Information Extractor Node ("Job Data Extractor"):**  
   - Type: Information Extractor  
   - Input Text:  
     ```
     Below is a job alert email received from Upwork. Please extract the requested information in valid JSON format.

     Subject: {{ $('Get Filtered Messages').item.json.subject }}
     Body: {{ $json.markdown }}
     ```  
   - Input JSON Schema: Define required fields including jobName, jobType, jobDetailUrl, price, descriptionSnippet, country, paymentVerified, clientRating, amountSpent, tags, postedAt, experienceLevel  
   - Connect input from "Convert To Markdown" and set AI language model to "OpenRouter Chat Model" node  

6. **Create Second OpenRouter Chat Model Node ("OpenRouter Chat Model1"):**  
   - Same type and credentials as the first OpenRouter node  
   - This will be connected to the Opportunity Scorer node  

7. **Create Information Extractor Node ("Opportunity Scorer"):**  
   - Type: Information Extractor  
   - Input Text: Custom prompt including:  
     - Role description as expert job matching analyst  
     - Instructions to score match from 1-10 with reasoning  
     - Placeholder `<my_profile>` block to be replaced with user’s detailed profile  
     - Job JSON injected with: `{{ JSON.stringify($json.output, null, 2) }}`  
   - Output JSON Schema: Object with integer `score` and string `reasoning`  
   - Connect input from "Job Data Extractor" and set AI language model to "OpenRouter Chat Model1"  

8. **Create Set Node ("Edit Fields"):**  
   - Assign two new fields:  
     - `evaluation` = `={{ $json.output }}` (output from Opportunity Scorer)  
     - `job` = `={{ $('Job Data Extractor').item.json.output }}` (output from Job Data Extractor)  
   - Connect input from "Opportunity Scorer"  

9. **Create Filter Node ("Filter By Score"):**  
   - Condition: `evaluation.score >= 7` (number greater or equal to 7)  
   - Connect input from "Edit Fields"  

10. **Create Slack Node ("Send Slack Alert"):**  
    - Type: Slack  
    - Credentials: Slack OAuth2  
    - Channel: Select appropriate Slack channel from workspace (e.g., "C090EHUFGEN")  
    - Message Text: Use templating to display job info and AI evaluation, for example:  
      ```
      🚨 *New Upwork Job Opportunity!*
      ----

      *{{ $json.job.jobName.toUpperCase() }}*
      _{{ $json.job.descriptionSnippet }}..._

      🗺️ {{ $json.job.country }}
      {{ $json.output.paymentVerified ? "✅ Payment verified" : "❌ Payment not verified"  }}
      🕧 {{ $json.job.jobType }} | {{ $json.job.price }}
      🏷️ {{ $json.job.tags.map(tag => `\`${tag}\``).join(', ') }}
      ⭐ {{ $json.job.clientRating }}
      💰 {{ $json.job.amountSpent }}
      🏆 {{ $json.job.experienceLevel }}
      📆 Posted {{ $json.job.postedAt }}

      ----
      *AI EVALUATION*
      Score: *{{ $json.evaluation.score }}*
      _{{ $json.evaluation.reasoning }}_
      ---

      _View this job:_ {{ $json.job.jobDetailUrl }}
      ```  
    - Connect input from "Filter By Score"  

11. **Connect all nodes sequentially:**  
    - "Get Filtered Messages" → "Mark as Read" → "Convert To Markdown" → "Job Data Extractor" → "Opportunity Scorer" → "Edit Fields" → "Filter By Score" → "Send Slack Alert"  

12. **Final setup:**  
    - Replace the placeholder profile text inside the Opportunity Scorer’s prompt with your detailed freelancer profile for best scoring accuracy.  
    - Adjust Slack channel ID as needed.  
    - Verify all credentials (Gmail OAuth2, OpenRouter API, Slack OAuth2) are valid and connected.  
    - Adjust Gmail polling interval per usage needs and quota.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow requires an Upwork Freelancer Plus subscription to receive job alert emails via Gmail. Adjust the Gmail polling frequency based on your workflow execution limits to avoid excessive runs.                                                                                                                                                             | Sticky note near Gmail Trigger node                                                               |
| Replace the placeholder `<my_profile>` text in the Opportunity Scorer node with your own detailed freelancer profile including skills, experience, tools, and preferences to ensure accurate AI scoring.                                                                                                                                                         | Sticky note near Opportunity Scorer node                                                          |
| Slack notifications are filtered to only send alerts for jobs with AI match score ≥ 7, but you can customize this threshold or filtering logic per your preferences. Ensure you select the correct Slack channel in the Slack node configuration.                                                                                                              | Sticky note near Filter and Slack nodes                                                           |
| This workflow uses OpenRouter API as the AI language model provider; ensure your OpenRouter API key is active and has sufficient quota.                                                                                                                                                                                                                         | Credentials setup for OpenRouter AI nodes                                                         |
| The job data extraction schema is strict to ensure consistent structured data; if Upwork changes their email format significantly, the extraction prompt and schema may require updates.                                                                                                                                                                        | Job Data Extractor node information                                                               |
| Slack message formatting uses markdown and templating expressions for clear presentation of job details and AI evaluation.                                                                                                                                                                                                                                     | Slack node message text configuration                                                             |

---

**Disclaimer:** The text provided is extracted solely from an n8n automation workflow and respects all relevant content policies. It contains no illegal or offensive material. All data processed is legal and publicly accessible.