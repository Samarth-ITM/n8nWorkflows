Generate & Schedule Social Posts with Gemini/OpenAI for X and LinkedIn

https://n8nworkflows.xyz/workflows/generate---schedule-social-posts-with-gemini-openai-for-x-and-linkedin-5913


# Generate & Schedule Social Posts with Gemini/OpenAI for X and LinkedIn

### 1. Workflow Overview

This workflow automates the generation and scheduling of social media posts tailored for LinkedIn and X (formerly Twitter) using AI models (Google Gemini and OpenAI). It is designed to streamline content creation from receiving post titles, generating platform-specific content, posting automatically, and managing scheduled content ideas for review. The workflow consists of two primary logical blocks:

- **1.1 Content Generation and Publishing Block:** Handles receiving post titles via a form, generating AI content with Google Gemini and LangChain agents, formatting the output, and posting simultaneously on LinkedIn and X. It concludes by showing a confirmation form with links to the published posts.

- **1.2 Scheduled Content Idea Generation and Review Block:** Periodically fetches recent LinkedIn posts, filters for high engagement, generates new post ideas using OpenAI, saves these drafts to Google Sheets, and notifies a reviewer via Slack for manual review and approval.

---

### 2. Block-by-Block Analysis

#### 2.1 Content Generation and Publishing Block

- **Overview:**  
  This block automates the creation and immediate posting of social media content based on a user-submitted post title. It leverages Google Gemini for AI chat generation, formats the output structurally, posts to LinkedIn and X, merges the responses, and provides confirmation to the user.

- **Nodes Involved:**  
  - Receive Post Title  
  - Google Gemini Chat Model  
  - Generate AI Content  
  - Format AI Output  
  - Post to X  
  - Post to LinkedIn  
  - Append Linkedin And X Publishing Responses  
  - Show Confirmation

- **Node Details:**

  **Receive Post Title**  
  - *Type & Role:* Form Trigger node; entry point to receive user input (post title) via a secure form with Basic Authentication.  
  - *Configuration:* Single form field labeled "post title".  
  - *Input/Output:* No input nodes; outputs JSON with post title under `post title`.  
  - *Edge Cases:* Authentication failure, form submission errors, empty or invalid titles.

  **Google Gemini Chat Model**  
  - *Type & Role:* AI language model node using Google Gemini 2.0 Flash for chat-based content generation.  
  - *Configuration:* Model set as `models/gemini-2.0-flash`. No additional options.  
  - *Input/Output:* Receives prompt text from "Generate AI Content" node (via LangChain agent integration). Outputs AI-generated chat response.  
  - *Edge Cases:* API rate limits, model unavailability, network errors.

  **Generate AI Content**  
  - *Type & Role:* LangChain Agent node that orchestrates AI prompt and response processing using the Google Gemini model, requesting minimum 50-word posts for LinkedIn and X separately.  
  - *Configuration:* Prompt uses expression: `write min 50 word about this topic '{{ $json["post title"] }}' for Linkedin and X platform separately`. Output parser enabled.  
  - *Input/Output:* Receives post title JSON from "Receive Post Title"; outputs structured AI content.  
  - *Edge Cases:* Expression evaluation failure, model response errors, timeouts.

  **Format AI Output**  
  - *Type & Role:* LangChain Output Parser Structured node; parses the AI's raw output into a defined JSON schema with fields for event info, platform posts, hashtags, CTAs, and notes.  
  - *Configuration:* Manual JSON schema specifying nested objects for LinkedIn and Twitter posts with required fields.  
  - *Input/Output:* Takes AI text response from "Generate AI Content" and outputs parsed structured data.  
  - *Edge Cases:* Parsing errors if AI output mismatches schema, malformed JSON.

  **Post to X**  
  - *Type & Role:* Twitter node; posts the generated Twitter/X content.  
  - *Configuration:* Text set via expression `={{ $json.output.platform_posts.Twitter.post }}`. Uses configured Twitter credentials.  
  - *Input/Output:* Input from "Generate AI Content"; outputs post metadata (e.g., tweet ID).  
  - *Edge Cases:* Authentication issues, API limits, content length errors.

  **Post to LinkedIn**  
  - *Type & Role:* LinkedIn node; posts the LinkedIn content.  
  - *Configuration:* Text via expression `={{ $json.output.platform_posts.LinkedIn.post }}`; posts as specific person ID "-HtNhNKSsE".  
  - *Input/Output:* Input from "Generate AI Content"; outputs LinkedIn post metadata (e.g., URN).  
  - *Edge Cases:* OAuth token expiration, permission errors, API rate limiting.

  **Append Linkedin And X Publishing Responses**  
  - *Type & Role:* Merge node; combines the outputs from both social media posting nodes into one unified data stream.  
  - *Configuration:* Combine mode set to "combineAll".  
  - *Input/Output:* Inputs from "Post to X" and "Post to LinkedIn"; outputs combined data.  
  - *Edge Cases:* Missing or delayed inputs, merge conflicts.

  **Show Confirmation**  
  - *Type & Role:* Form node; finalizes workflow by showing a confirmation message with clickable links to the published posts on X and LinkedIn.  
  - *Configuration:* Completion title "Your post has been successfully shared". Message template includes dynamic insertion of tweet ID and LinkedIn URN with direct URLs.  
  - *Input/Output:* Input from merged responses; no output.  
  - *Edge Cases:* Missing post IDs, user not receiving confirmation.

---

#### 2.2 Scheduled Content Idea Generation and Review Block

- **Overview:**  
  This block runs on a schedule, fetching recent LinkedIn posts, filters for those with high engagement, uses OpenAI GPT to generate new post ideas, saves drafts to Google Sheets for tracking, and notifies a Slack channel for human review.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Fetch LinkedIn Posts  
  - Filter High Engagement Posts  
  - Generate Post Ideas (OpenAI)  
  - Save Drafts to Google Sheets  
  - Notify Reviewer (Slack)

- **Node Details:**

  **Schedule Trigger**  
  - *Type & Role:* Schedule Trigger node; initiates the workflow periodically based on a configured interval (default unspecified).  
  - *Configuration:* Interval set to run at regular unspecified intervals (daily/hourly).  
  - *Input/Output:* No input; output triggers "Fetch LinkedIn Posts".  
  - *Edge Cases:* Scheduling misconfiguration, missed runs.

  **Fetch LinkedIn Posts**  
  - *Type & Role:* HTTP Request node; retrieves latest LinkedIn posts data from an Apify task dataset.  
  - *Configuration:* URL set to Apify actor task’s last run dataset with token authentication (placeholders present: your-task-id and YOUR_APIFY_TOKEN). JSON response expected.  
  - *Input/Output:* Input from Schedule Trigger; outputs JSON array of posts.  
  - *Edge Cases:* Invalid token, API downtime, malformed JSON.

  **Filter High Engagement Posts**  
  - *Type & Role:* Function node; filters posts to only those with likes greater than 10.  
  - *Configuration:* JavaScript function filtering items by `item.json.engagement.likes > 10`.  
  - *Input/Output:* Input from Fetch LinkedIn Posts; outputs filtered posts.  
  - *Edge Cases:* Missing engagement data, empty filter results.

  **Generate Post Ideas (OpenAI)**  
  - *Type & Role:* OpenAI node; generates new post ideas using the GPT-4o-mini model.  
  - *Configuration:* Model set to `gpt-4o-mini`. No additional options configured.  
  - *Input/Output:* Input from Filter node; outputs AI-generated post ideas.  
  - *Edge Cases:* API rate limits, invalid prompt/data.

  **Save Drafts to Google Sheets**  
  - *Type & Role:* Google Sheets node; appends generated post ideas as drafts to a specified sheet.  
  - *Configuration:* Appends to range `Sheet1!A:B` in a Google Sheet identified by `YOUR_GOOGLE_SHEET_ID`.  
  - *Input/Output:* Input from OpenAI node; outputs appended row info.  
  - *Edge Cases:* Invalid Sheet ID, authentication failures, quota limits.

  **Notify Reviewer (Slack)**  
  - *Type & Role:* Slack node; sends message notification to a Slack channel alerting about new post ideas ready for review.  
  - *Configuration:* Message text includes first choice content from OpenAI response; channel set to `YOUR_SLACK_CHANNEL`.  
  - *Input/Output:* Input from Google Sheets node; no output.  
  - *Edge Cases:* Slack API auth failure, invalid channel, message send failure.

---

### 3. Summary Table

| Node Name                      | Node Type                                    | Functional Role                          | Input Node(s)                           | Output Node(s)                     | Sticky Note                                              |
|-------------------------------|----------------------------------------------|----------------------------------------|---------------------------------------|----------------------------------|----------------------------------------------------------|
| Receive Post Title             | Form Trigger                                 | Input Reception of post title          | -                                     | Generate AI Content               |                                                          |
| Google Gemini Chat Model       | LangChain Google Gemini AI Model             | AI language model for chat generation  | Generate AI Content (ai_languageModel) | Generate AI Content               |                                                          |
| Generate AI Content            | LangChain Agent                              | Orchestrates AI prompt & response      | Receive Post Title, Google Gemini Chat Model | Post to X, Post to LinkedIn       |                                                          |
| Format AI Output              | LangChain Output Parser Structured            | Parses AI raw output into JSON schema  | Generate AI Content (ai_outputParser) | Generate AI Content (main)        |                                                          |
| Post to X                     | Twitter Node                                 | Posts generated content to X            | Generate AI Content                   | Append Linkedin And X Publishing Responses |                                                          |
| Post to LinkedIn              | LinkedIn Node                                | Posts generated content to LinkedIn    | Generate AI Content                   | Append Linkedin And X Publishing Responses |                                                          |
| Append Linkedin And X Publishing Responses | Merge Node                                   | Combines LinkedIn and X post responses | Post to X, Post to LinkedIn           | Show Confirmation                |                                                          |
| Show Confirmation             | Form Node                                    | Shows success confirmation with links | Append Linkedin And X Publishing Responses | -                              |                                                          |
| Schedule Trigger              | Schedule Trigger                             | Initiates scheduled content fetch      | -                                     | Fetch LinkedIn Posts             |                                                          |
| Fetch LinkedIn Posts          | HTTP Request                                | Retrieves recent LinkedIn posts        | Schedule Trigger                      | Filter High Engagement Posts     |                                                          |
| Filter High Engagement Posts  | Function Node                               | Filters posts with likes > 10           | Fetch LinkedIn Posts                  | Generate Post Ideas (OpenAI)     |                                                          |
| Generate Post Ideas (OpenAI)  | OpenAI Node                                 | Generates new post ideas                | Filter High Engagement Posts          | Save Drafts to Google Sheets      |                                                          |
| Save Drafts to Google Sheets  | Google Sheets Node                          | Saves post ideas drafts                 | Generate Post Ideas (OpenAI)           | Notify Reviewer (Slack)           |                                                          |
| Notify Reviewer (Slack)       | Slack Node                                  | Sends notification to Slack channel    | Save Drafts to Google Sheets           | -                              |                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node named "Receive Post Title"**  
   - Configure form with title "post" and one field labeled "post title".  
   - Enable Basic Authentication to secure the form.

2. **Add a LangChain Google Gemini Chat Model node named "Google Gemini Chat Model"**  
   - Set Model Name to `models/gemini-2.0-flash`.  
   - No additional options needed.

3. **Add a LangChain Agent node named "Generate AI Content"**  
   - Use the prompt:  
     ```
     write min 50 word about this topic '{{ $json["post title"] }}' for Linkedin and X platform separately
     ```  
   - Enable Output Parser.  
   - Connect "Receive Post Title" main output to "Generate AI Content" main input.  
   - Connect "Google Gemini Chat Model" AI Language Model output to "Generate AI Content" AI Language Model input.

4. **Add a LangChain Output Parser Structured node named "Format AI Output"**  
   - Define a manual JSON schema for expected AI outputs including event info and platform posts for LinkedIn and Twitter with post text, hashtags, and call to action fields.  
   - Connect "Generate AI Content" AI Output Parser output to "Format AI Output" input.

5. **Add a Twitter node named "Post to X"**  
   - Set the text field to: `={{ $json.output.platform_posts.Twitter.post }}`.  
   - Configure Twitter OAuth credentials.  
   - Connect "Generate AI Content" main output to "Post to X".

6. **Add a LinkedIn node named "Post to LinkedIn"**  
   - Set the text to `={{ $json.output.platform_posts.LinkedIn.post }}`.  
   - Set the person ID to the specific LinkedIn user (e.g., "-HtNhNKSsE").  
   - Configure LinkedIn OAuth credentials.  
   - Connect "Generate AI Content" main output to "Post to LinkedIn".

7. **Add a Merge node named "Append Linkedin And X Publishing Responses"**  
   - Set mode to "combineAll".  
   - Connect "Post to X" and "Post to LinkedIn" outputs to this Merge node’s inputs.

8. **Add a Form node named "Show Confirmation"**  
   - Set operation to "completion".  
   - Title: "Your post has been successfully shared".  
   - Message:  
     ```
     🔗 View your posts:

     X (Twitter): 
     [https://x.com/x/status/{{ $json.id }}]

     LinkedIn:
     [https://www.linkedin.com/feed/update/{{ $json.urn }}]
     ```  
   - Connect "Append Linkedin And X Publishing Responses" output to "Show Confirmation" input.

9. **Create a Schedule Trigger node named "Schedule Trigger"**  
   - Configure the desired interval (e.g., daily at a specific time).

10. **Add an HTTP Request node named "Fetch LinkedIn Posts"**  
    - URL:  
      ```
      https://api.apify.com/v2/actor-tasks/your-task-id/runs/last/dataset/items?token=YOUR_APIFY_TOKEN
      ```  
    - Set to expect JSON response.  
    - Connect "Schedule Trigger" output to this node.

11. **Add a Function node named "Filter High Engagement Posts"**  
    - Use this code to filter posts with likes > 10:  
      ```js
      return items.filter(item => item.json.engagement && item.json.engagement.likes > 10);
      ```  
    - Connect "Fetch LinkedIn Posts" output to this node.

12. **Add an OpenAI node named "Generate Post Ideas (OpenAI)"**  
    - Select model `gpt-4o-mini`.  
    - Connect "Filter High Engagement Posts" output to this node.  
    - Configure OpenAI credentials.

13. **Add a Google Sheets node named "Save Drafts to Google Sheets"**  
    - Operation: Append  
    - Range: `Sheet1!A:B`  
    - Specify your Google Sheet ID (`YOUR_GOOGLE_SHEET_ID`).  
    - Connect "Generate Post Ideas (OpenAI)" output to this node.  
    - Configure Google Sheets OAuth2 credentials.

14. **Add a Slack node named "Notify Reviewer (Slack)"**  
    - Set channel to `YOUR_SLACK_CHANNEL`.  
    - Message text:  
      ```
      New LinkedIn post ideas are ready for review:

      {{$json["choices"][0].message.content}}
      ```  
    - Connect "Save Drafts to Google Sheets" output to this node.  
    - Configure Slack credentials.

---

### 5. General Notes & Resources

| Note Content                                                                                                       | Context or Link                                  |
|--------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Workflow leverages Google Gemini 2.0 Flash and OpenAI GPT-4o-mini models for content generation.                     | AI model details                                |
| Uses LangChain nodes for AI prompt orchestration and structured output parsing.                                     | LangChain for n8n integration                   |
| Apify actor task must be configured and token provided for LinkedIn post fetching.                                  | https://apify.com                                |
| Google Sheets and Slack integrations require OAuth2 credential setup with appropriate permissions.                  | Google Sheets and Slack API docs                 |
| Form Trigger node uses Basic Authentication to secure user input for post titles.                                   | n8n Form Trigger documentation                   |
| LinkedIn node requires specific user person ID to post on behalf of that user.                                      | LinkedIn API documentation                        |
| Confirmation message includes direct clickable links to social posts for user convenience.                          | Links formatted dynamically with post IDs       |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.