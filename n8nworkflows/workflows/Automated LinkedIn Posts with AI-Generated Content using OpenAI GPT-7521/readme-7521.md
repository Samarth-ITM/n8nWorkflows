Automated LinkedIn Posts with AI-Generated Content using OpenAI GPT

https://n8nworkflows.xyz/workflows/automated-linkedin-posts-with-ai-generated-content-using-openai-gpt-7521


# Automated LinkedIn Posts with AI-Generated Content using OpenAI GPT

### 1. Workflow Overview

This workflow automates the creation and posting of LinkedIn content using AI-generated text from OpenAI's GPT models. It targets users looking to maintain a consistent LinkedIn presence with professionally crafted posts focused on industry insights, professional development, and thought leadership. The workflow includes two trigger mechanisms (scheduled and manual), AI content generation, and automated posting to LinkedIn.

**Logical Blocks:**

- **1.1 Input Reception:** Receives trigger events either on a schedule or manually.
- **1.2 AI Content Generation:** Uses OpenAI GPT to generate LinkedIn post content based on a predefined prompt.
- **1.3 LinkedIn Posting:** Posts the AI-generated content publicly on LinkedIn.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** This block initiates the workflow either automatically at scheduled times or manually by user intervention.
- **Nodes Involved:**  
  - Schedule Trigger  
  - Manual Trigger

##### Node: Schedule Trigger
- **Type and Role:** Schedule node; triggers workflow execution based on a CRON schedule.
- **Configuration:**  
  - CRON expression set to trigger at 9:00 AM every weekday (Monday to Friday).  
- **Key Expressions/Variables:** None (static schedule).
- **Inputs:** None (initiator node).
- **Outputs:** Connects to OpenAI Content Generation node.
- **Version-Specific Requirements:** Uses typeVersion 1.1, which supports updated scheduling options.
- **Edge Cases / Potential Failures:**  
  - Misconfigured CRON expression could prevent triggering.  
  - Workflow disabled or inactive state prevents execution.
- **Sub-Workflow Reference:** None.

##### Node: Manual Trigger
- **Type and Role:** Manual trigger; allows on-demand workflow execution.
- **Configuration:** Default manual trigger with no additional parameters.
- **Inputs:** None.
- **Outputs:** Connects to OpenAI Content Generation node.
- **Version-Specific Requirements:** typeVersion 1.
- **Edge Cases / Potential Failures:**  
  - User must manually trigger; no automatic fallback.
- **Sub-Workflow Reference:** None.

---

#### 1.2 AI Content Generation

- **Overview:** Generates an engaging LinkedIn post using OpenAI GPT-3.5 based on a specific prompt focused on professional growth and learning in tech.
- **Nodes Involved:**  
  - OpenAI Content Generation

##### Node: OpenAI Content Generation
- **Type and Role:** OpenAI node; sends prompt to GPT-3.5-turbo model and receives generated text.
- **Configuration:**  
  - Model: gpt-3.5-turbo.  
  - Prompt: Instructs model to generate a professional LinkedIn post (150-300 characters) about professional growth and continuous learning in tech, including 2-3 hashtags.  
  - Options: temperature 0.7 (balanced creativity), max tokens 200, topP 1.  
  - Operation: `complete` (text completion).  
- **Credentials:** Uses OpenAI API credentials identified as "OpenAI API".
- **Key Expressions/Variables:** None in node config; output text accessed downstream as `$json.choices[0].text`.
- **Inputs:** Triggered from either Schedule Trigger or Manual Trigger.
- **Outputs:** Connects to LinkedIn Post node.
- **Version-Specific Requirements:** typeVersion 1.
- **Edge Cases / Potential Failures:**  
  - API authentication errors if credentials invalid or expired.  
  - Rate limiting or API quota exceeded causing failures.  
  - Network timeouts or OpenAI service downtime.  
  - Output may be empty or irrelevant; downstream handling required.  
- **Retry Policy:** Enabled with up to 3 retries and 1-second wait between attempts.  
- **Continue On Fail:** Enabled to allow workflow continuation even if this node fails.

---

#### 1.3 LinkedIn Posting

- **Overview:** Publishes the generated LinkedIn post content publicly on the user's LinkedIn feed.
- **Nodes Involved:**  
  - LinkedIn Post

##### Node: LinkedIn Post
- **Type and Role:** LinkedIn node; creates a new LinkedIn post.
- **Configuration:**  
  - Resource: `post`.  
  - Operation: `create`.  
  - Text: Uses expression to insert AI-generated text: `={{ $json.choices[0].text }}`.  
  - Additional Fields: Visibility set to `public`.  
- **Credentials:** Uses LinkedIn OAuth2 credentials named "LinkedIn OAuth2".
- **Inputs:** Receives from OpenAI Content Generation node.
- **Outputs:** None (end of workflow).
- **Version-Specific Requirements:** typeVersion 1.
- **Edge Cases / Potential Failures:**  
  - OAuth token expiry or permission errors.  
  - LinkedIn API rate limits or downtime.  
  - Empty or malformed post content causing API rejection.  
- **Retry Policy:** Enabled with up to 3 retries and 1-second wait between attempts.  
- **Continue On Fail:** Enabled to prevent workflow halt if posting fails.

---

### 3. Summary Table

| Node Name               | Node Type               | Functional Role                   | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                                                        |
|-------------------------|-------------------------|---------------------------------|------------------------|---------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger        | Automatic scheduled initiation   | None                   | OpenAI Content Generation |                                                                                                                                    |
| Manual Trigger          | Manual Trigger          | Manual initiation                | None                   | OpenAI Content Generation |                                                                                                                                    |
| OpenAI Content Generation | OpenAI Node             | AI-generated LinkedIn post text  | Schedule Trigger, Manual Trigger | LinkedIn Post             | Retry enabled to handle temporary API failures; continue on fail to keep workflow running.                                         |
| LinkedIn Post           | LinkedIn Node           | Publish post on LinkedIn          | OpenAI Content Generation | None                      | Retry enabled for resilience; continue on fail to avoid stoppage on posting errors.                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set Cron Expression: `0 9 * * 1-5` (9 AM every weekday)  
   - No credentials required  
   - Position it on canvas (e.g., left side)

2. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Default settings (no parameters)  
   - Position near Schedule Trigger

3. **Create OpenAI Content Generation Node**  
   - Type: OpenAI Node  
   - Operation: Text Completion (`complete`)  
   - Model: `gpt-3.5-turbo`  
   - Prompt:  
     ```
     You are a professional LinkedIn content creator. Generate an engaging, professional post that is 150-300 characters long with relevant hashtags. Focus on industry insights, professional development, or thought leadership topics.

     Create a LinkedIn post about professional growth and continuous learning in the tech industry. Make it inspiring and include 2-3 relevant hashtags.
     ```  
   - Options: temperature 0.7, max tokens 200, topP 1  
   - Attach OpenAI API credentials (with valid API key)  
   - Enable Retry on Fail: max retries 3, wait time 1000ms  
   - Enable Continue On Fail  

4. **Create LinkedIn Post Node**  
   - Type: LinkedIn Node  
   - Resource: `post`  
   - Operation: `create`  
   - Text: Set expression to `={{ $json.choices[0].text }}` to use generated content  
   - Additional fields: set `visibility` to `public`  
   - Attach LinkedIn OAuth2 credentials with posting scope  
   - Enable Retry on Fail: max retries 3, wait time 1000ms  
   - Enable Continue On Fail  

5. **Connect Nodes**  
   - Connect Schedule Trigger output to OpenAI Content Generation input  
   - Connect Manual Trigger output to OpenAI Content Generation input  
   - Connect OpenAI Content Generation output to LinkedIn Post input  

6. **Activate Workflow**  
   - Save and activate workflow for scheduled runs  
   - Use Manual Trigger to test ad-hoc post generation  

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                       |
|-----------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| LinkedIn OAuth2 credentials must have posting permissions and be authorized for the user.      | LinkedIn Developer Portal OAuth2 setup documentation.                |
| OpenAI API key with GPT-3.5-turbo access required; watch usage limits and costs.               | https://platform.openai.com/docs/api-reference/introduction          |
| Schedule set to business days at 9 AM to target optimal posting times.                         | Common LinkedIn engagement best practices.                           |
| Retry with continue on fail settings ensures workflow robustness, avoiding full stoppage.      | n8n documentation on error handling and retry policies.              |

---

*Disclaimer:* The provided content derives exclusively from an automated workflow created with n8n, following current content policies. All data processed is legal and public.