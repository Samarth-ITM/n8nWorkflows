Social Media Content Generator And Publisher | X, Linkedin

https://n8nworkflows.xyz/workflows/social-media-content-generator-and-publisher---x--linkedin-3082


# Social Media Content Generator And Publisher | X, Linkedin

### 1. Workflow Overview

This workflow automates the generation and publishing of AI-powered social media content to **LinkedIn** and **X (formerly Twitter)**. It is designed for social media managers, content creators, and marketers who want to streamline content creation and posting processes using AI and secure OAuth2 authentication.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives a post title via a secured form submission.
- **1.2 AI Content Generation:** Uses Google Gemini AI to generate tailored social media posts for LinkedIn and X.
- **1.3 AI Output Parsing:** Parses the AI-generated content into a structured JSON format.
- **1.4 Social Media Publishing:** Posts the generated content to X and LinkedIn using OAuth2 credentials.
- **1.5 Post-Publishing Merge:** Combines the publishing responses from both platforms.
- **1.6 Confirmation Display:** Shows a confirmation form with links to the published posts.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block captures the initial input — the post title — from a user-submitted form. It uses Basic Authentication to secure the form endpoint.

- **Nodes Involved:**  
  - Receive Post Title

- **Node Details:**

  - **Receive Post Title**  
    - Type: Form Trigger  
    - Role: Entry point that triggers the workflow upon form submission.  
    - Configuration:  
      - Form titled "post" with a single field labeled "post title".  
      - Basic Authentication enabled to restrict access.  
      - Webhook ID configured for external form integration.  
    - Expressions/Variables: Accesses submitted data via `$json["post title"]`.  
    - Inputs: None (trigger node).  
    - Outputs: Passes form data to the next node (Generate AI Content).  
    - Edge Cases:  
      - Authentication failures if credentials are incorrect.  
      - Missing or empty "post title" field could cause downstream issues.  
      - Network or webhook errors may prevent triggering.  

#### 1.2 AI Content Generation

- **Overview:**  
  Generates AI-powered social media content based on the submitted post title. It uses Google Gemini Chat Model via LangChain integration to create separate posts for LinkedIn and X, including hashtags and CTAs.

- **Nodes Involved:**  
  - Google Gemini Chat Model  
  - Generate AI Content

- **Node Details:**

  - **Google Gemini Chat Model**  
    - Type: Language Model Node (Google Gemini via LangChain)  
    - Role: Provides AI language model capabilities for content generation.  
    - Configuration:  
      - Model: `models/gemini-2.0-flash` (Google Gemini 2.0 Flash).  
      - Credentials: Google PaLM API account configured.  
    - Inputs: Receives prompt from Generate AI Content node (via ai_languageModel connection).  
    - Outputs: AI-generated raw text output to Generate AI Content node.  
    - Edge Cases:  
      - API authentication errors.  
      - Rate limiting or quota exceeded.  
      - Timeout or latency issues.  

  - **Generate AI Content**  
    - Type: LangChain Agent Node  
    - Role: Sends prompt to AI model and handles output parsing.  
    - Configuration:  
      - Prompt: `write min 50 word about this topic '{{ $json["post title"] }}' for Linkedin and X platform separately`  
      - Output parser enabled to structure AI response.  
    - Inputs: Receives post title from Receive Post Title node.  
    - Outputs: Passes AI raw output to Format AI Output node.  
    - Edge Cases:  
      - Expression errors if `post title` is missing or malformed.  
      - AI response may be incomplete or off-topic.  

#### 1.3 AI Output Parsing

- **Overview:**  
  Parses the AI-generated text into a structured JSON format to extract platform-specific posts, hashtags, CTAs, and metadata.

- **Nodes Involved:**  
  - Format AI Output

- **Node Details:**

  - **Format AI Output**  
    - Type: LangChain Structured Output Parser  
    - Role: Converts AI text output into a JSON object with a predefined schema.  
    - Configuration:  
      - Manual JSON schema defining properties:  
        - `event_name` (string)  
        - `event_description` (string)  
        - `platform_posts` (object) with nested LinkedIn and Twitter posts, hashtags, CTAs, character limits  
        - `additional_notes` (string)  
    - Inputs: Receives AI raw output from Generate AI Content node.  
    - Outputs: Structured JSON passed to social media posting nodes.  
    - Edge Cases:  
      - Parsing errors if AI output does not conform to schema.  
      - Missing fields causing downstream failures.  

#### 1.4 Social Media Publishing

- **Overview:**  
  Publishes the structured posts to X and LinkedIn using OAuth2 authentication, ensuring secure and authorized posting.

- **Nodes Involved:**  
  - Post to X  
  - Post to LinkedIn

- **Node Details:**

  - **Post to X**  
    - Type: Twitter Node (configured for X)  
    - Role: Posts the Twitter/X content extracted from AI output.  
    - Configuration:  
      - Text: Extracted from `output.platform_posts.Twitter.post` via expression.  
      - Credentials: OAuth2 Twitter account configured.  
    - Inputs: Receives structured JSON from Format AI Output via Generate AI Content.  
    - Outputs: Publishing response passed to merge node.  
    - Edge Cases:  
      - OAuth token expiration or invalid credentials.  
      - Character limit violations (should be checked by AI but may fail).  
      - API rate limits or posting errors.  

  - **Post to LinkedIn**  
    - Type: LinkedIn Node  
    - Role: Posts LinkedIn content extracted from AI output.  
    - Configuration:  
      - Text: Extracted from `output.platform_posts.LinkedIn.post`.  
      - Person ID: Static value `-HtNhNKSsE` (likely the authenticated user or company page).  
      - Credentials: OAuth2 LinkedIn account configured.  
    - Inputs: Receives structured JSON from Format AI Output via Generate AI Content.  
    - Outputs: Publishing response passed to merge node.  
    - Edge Cases:  
      - OAuth token issues.  
      - API posting errors or content violations.  

#### 1.5 Post-Publishing Merge

- **Overview:**  
  Combines the responses from both social media posting nodes to consolidate publishing results.

- **Nodes Involved:**  
  - Append Linkedin And X Publishing Responses

- **Node Details:**

  - **Append Linkedin And X Publishing Responses**  
    - Type: Merge Node  
    - Role: Combines outputs from Post to X and Post to LinkedIn nodes.  
    - Configuration:  
      - Mode: Combine all inputs into a single output.  
    - Inputs: Two inputs from Post to X and Post to LinkedIn nodes.  
    - Outputs: Combined data passed to confirmation form.  
    - Edge Cases:  
      - One platform failing to post may cause partial data.  
      - Mismatched data structures could cause merge errors.  

#### 1.6 Confirmation Display

- **Overview:**  
  Displays a confirmation form to the user with links to the published posts on both platforms.

- **Nodes Involved:**  
  - Show Confirmation

- **Node Details:**

  - **Show Confirmation**  
    - Type: Form Node (completion type)  
    - Role: Final step showing success message and post URLs.  
    - Configuration:  
      - Completion title: "Your post has been successfully shared"  
      - Completion message includes dynamic links:  
        - X post URL: `https://x.com/x/status/{{ $json.id }}`  
        - LinkedIn post URL: `https://www.linkedin.com/feed/update/{{ $json.urn }}`  
      - Webhook ID configured for form display.  
    - Inputs: Receives merged publishing responses.  
    - Outputs: None (end of workflow).  
    - Edge Cases:  
      - Missing or malformed IDs in responses may break links.  
      - User experience depends on form rendering success.  

---

### 3. Summary Table

| Node Name                              | Node Type                                  | Functional Role                      | Input Node(s)              | Output Node(s)                         | Sticky Note                                                                                      |
|--------------------------------------|--------------------------------------------|------------------------------------|----------------------------|---------------------------------------|-------------------------------------------------------------------------------------------------|
| Receive Post Title                   | Form Trigger                               | Input reception via secured form   | None                       | Generate AI Content                   | Form secured with Basic Authentication                                                         |
| Generate AI Content                  | LangChain Agent                            | AI content generation              | Receive Post Title          | Post to X, Post to LinkedIn           | Uses AI prompt with post title to generate platform-specific content                            |
| Google Gemini Chat Model             | LangChain Language Model (Google Gemini)  | AI language model provider         | Generate AI Content (ai_languageModel) | Generate AI Content (ai_languageModel) | Requires Google PaLM API credentials                                                           |
| Format AI Output                    | LangChain Structured Output Parser         | Parses AI output into JSON         | Generate AI Content (ai_outputParser) | Post to X, Post to LinkedIn           | Uses manual JSON schema for structured parsing                                                 |
| Post to X                          | Twitter Node                               | Posts content to X (Twitter)       | Generate AI Content          | Append Linkedin And X Publishing Responses | OAuth2 authentication required                                                                |
| Post to LinkedIn                   | LinkedIn Node                              | Posts content to LinkedIn          | Generate AI Content          | Append Linkedin And X Publishing Responses | OAuth2 authentication required                                                                |
| Append Linkedin And X Publishing Responses | Merge Node                                | Merges publishing responses        | Post to X, Post to LinkedIn | Show Confirmation                    | Combines results from both platforms                                                          |
| Show Confirmation                  | Form Node (completion)                      | Displays confirmation message      | Append Linkedin And X Publishing Responses | None                                | Shows links to published posts with dynamic URLs                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node: Receive Post Title**  
   - Node Type: Form Trigger  
   - Configure form title as "post".  
   - Add one form field labeled "post title".  
   - Enable Basic Authentication and configure credentials.  
   - Set webhook ID (auto-generated or custom).  

2. **Add Google Gemini Chat Model Node**  
   - Node Type: LangChain Language Model (Google Gemini)  
   - Set model name to `models/gemini-2.0-flash`.  
   - Configure Google PaLM API credentials.  

3. **Add Generate AI Content Node**  
   - Node Type: LangChain Agent  
   - Set prompt text:  
     ```
     write min 50 word about this topic '{{ $json["post title"] }}' for Linkedin and X platform separately
     ```  
   - Enable output parser.  
   - Connect input from Receive Post Title node.  
   - Connect ai_languageModel input to Google Gemini Chat Model node.  

4. **Add Format AI Output Node**  
   - Node Type: LangChain Structured Output Parser  
   - Set schema type to manual.  
   - Paste the JSON schema defining the expected AI output structure (see section 2.3).  
   - Connect ai_outputParser input from Generate AI Content node.  

5. **Add Post to X Node**  
   - Node Type: Twitter Node (configured for X)  
   - Set text parameter to: `={{ $json.output.platform_posts.Twitter.post }}`  
   - Configure OAuth2 Twitter credentials.  
   - Connect input from Generate AI Content node (main output).  

6. **Add Post to LinkedIn Node**  
   - Node Type: LinkedIn Node  
   - Set text parameter to: `={{ $json.output.platform_posts.LinkedIn.post }}`  
   - Set person ID to `-HtNhNKSsE` (replace with your LinkedIn user or page ID).  
   - Configure OAuth2 LinkedIn credentials.  
   - Connect input from Generate AI Content node (main output).  

7. **Add Merge Node: Append Linkedin And X Publishing Responses**  
   - Node Type: Merge  
   - Set mode to "combine".  
   - Connect inputs from Post to X and Post to LinkedIn nodes.  

8. **Add Show Confirmation Node**  
   - Node Type: Form (completion)  
   - Set completion title: "Your post has been successfully shared".  
   - Set completion message with dynamic links:  
     ```
     =🔗 View your posts:

     X (Twitter): 
     [https://x.com/x/status/{{ $json.id }}]

     LinkedIn:
     [https://www.linkedin.com/feed/update/{{ $json.urn }}]
     ```  
   - Configure webhook ID.  
   - Connect input from Merge node.  

9. **Connect the nodes in this order:**  
   - Receive Post Title → Generate AI Content  
   - Generate AI Content (ai_languageModel) → Google Gemini Chat Model  
   - Generate AI Content (ai_outputParser) → Format AI Output  
   - Generate AI Content (main) → Post to X and Post to LinkedIn  
   - Post to X and Post to LinkedIn → Merge node  
   - Merge node → Show Confirmation  

10. **Verify all credentials are properly configured:**  
    - Google PaLM API for Gemini model  
    - OAuth2 for Twitter (X) account  
    - OAuth2 for LinkedIn account  
    - Basic Auth credentials for form trigger  

11. **Test the workflow:**  
    - Submit a form with a post title.  
    - Confirm AI generates content for both platforms.  
    - Confirm posts are published and confirmation form displays correct links.  

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The workflow uses Google Gemini AI via LangChain integration for advanced content generation.      | Requires Google PaLM API credentials.                                                           |
| OAuth2 authentication is mandatory for secure posting to LinkedIn and X (Twitter).                  | Ensure developer accounts and apps are properly configured with required permissions.            |
| Basic Authentication secures the form submission endpoint to prevent unauthorized access.          | Credentials must be managed securely in n8n.                                                    |
| The confirmation form dynamically generates URLs to the published posts using response IDs.       | URLs depend on correct API responses from LinkedIn and X.                                      |
| For more information on n8n social media integrations, visit: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.twitter/ | Official n8n documentation for Twitter node.                                                    |
| LinkedIn API documentation: https://learn.microsoft.com/en-us/linkedin/marketing/integrations/community-management/shares/share-api | Useful for understanding LinkedIn post parameters and limitations.                              |

---

This detailed reference document provides a comprehensive understanding of the workflow’s structure, node configurations, and operational logic, enabling users and AI agents to reproduce, modify, and troubleshoot the automation effectively.