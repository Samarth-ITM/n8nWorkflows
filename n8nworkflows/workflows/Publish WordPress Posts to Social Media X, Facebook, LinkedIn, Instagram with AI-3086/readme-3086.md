Publish WordPress Posts to Social Media X, Facebook, LinkedIn, Instagram with AI

https://n8nworkflows.xyz/workflows/publish-wordpress-posts-to-social-media-x--facebook--linkedin--instagram-with-ai-3086


# Publish WordPress Posts to Social Media X, Facebook, LinkedIn, Instagram with AI

### 1. Workflow Overview

This workflow automates the creation and publishing of social media posts on multiple platforms (Twitter/X, Facebook, LinkedIn, Instagram) using content sourced from WordPress posts. It leverages AI to generate platform-tailored captions and images, integrates with Google Sheets for data management, and uses native API nodes for publishing content. The workflow is designed to save time, maintain consistent branding, and optimize posts for each social media platform’s audience and style.

**Logical Blocks:**

- **1.1 Input Reception and Data Fetching**  
  Starts with a manual trigger, retrieves the WordPress Post ID from Google Sheets, and fetches the full post content from WordPress.

- **1.2 AI Content Generation**  
  Uses OpenRouter AI to generate platform-specific captions for Twitter/X, Facebook, LinkedIn, and Instagram, parsing the structured output.

- **1.3 AI Image Generation**  
  Generates images for Instagram and for Facebook/LinkedIn posts using OpenAI’s image generation capabilities.

- **1.4 Social Media Publishing**  
  Publishes the generated captions and images on Twitter/X, LinkedIn, Facebook, and Instagram via their respective API nodes.

- **1.5 Google Sheets Update**  
  Marks each social media platform’s post as published in the Google Sheet to track workflow progress.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Data Fetching

**Overview:**  
This block initiates the workflow manually, retrieves the WordPress Post ID from a Google Sheet, and fetches the corresponding WordPress post content.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Google Sheets  
- Get Post

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow on user command.  
  - Configuration: Default manual trigger, no parameters.  
  - Inputs: None  
  - Outputs: Triggers Google Sheets node.  
  - Edge cases: None typical; user must manually trigger.

- **Google Sheets**  
  - Type: Google Sheets node  
  - Role: Reads the WordPress Post ID from a predefined Google Sheet.  
  - Configuration:  
    - Document ID and Sheet Name set to a specific Google Sheet containing social media post data.  
    - Filters to return the first match based on the "TWITTER" column.  
  - Key expressions: None complex; uses sheet and document IDs.  
  - Inputs: Trigger from Manual Trigger  
  - Outputs: Provides Post ID and row number to Get Post node.  
  - Edge cases:  
    - Authentication errors if Google credentials expire.  
    - No matching row found could cause empty data downstream.

- **Get Post**  
  - Type: WordPress node  
  - Role: Fetches the WordPress post content by Post ID.  
  - Configuration:  
    - Operation: Get post by ID.  
    - Post ID dynamically set from Google Sheets output (`{{$json['POST ID']}}`).  
  - Inputs: Google Sheets output  
  - Outputs: WordPress post JSON data (title, content, link).  
  - Edge cases:  
    - Post ID invalid or deleted post returns error or empty.  
    - WordPress API authentication or connectivity issues.

---

#### 2.2 AI Content Generation

**Overview:**  
Generates platform-specific social media captions using an AI language model, parsing the output into structured fields for each platform.

**Nodes Involved:**  
- Social Media Manager (LangChain LLM Chain)  
- OpenRouter Chat Model (AI Model)  
- Structured Output Parser

**Node Details:**

- **OpenRouter Chat Model**  
  - Type: LangChain OpenRouter Chat Model node  
  - Role: Provides AI language model backend for caption generation.  
  - Configuration:  
    - Model: `google/gemini-2.0-flash-exp:free`  
    - Credentials: OpenRouter API key.  
  - Inputs: Receives prompt from Social Media Manager node.  
  - Outputs: AI-generated raw text response.  
  - Edge cases:  
    - API rate limits or key invalidation.  
    - Model downtime or latency.

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses AI raw output into JSON with keys: twitter, facebook, linkedin, instagram.  
  - Configuration:  
    - Manual JSON schema defining string properties for each platform.  
  - Inputs: AI raw text from OpenRouter Chat Model.  
  - Outputs: Structured JSON with captions per platform.  
  - Edge cases:  
    - Parsing failure if AI output does not conform to schema.  
    - Empty or malformed AI responses.

- **Social Media Manager**  
  - Type: LangChain Chain LLM node  
  - Role: Orchestrates prompt construction and output parsing.  
  - Configuration:  
    - Prompt includes WordPress post title and content.  
    - Detailed instructions for tone, style, length, hashtags, and calls to action per platform.  
    - Uses output parser to enforce structured JSON output.  
  - Inputs: WordPress post JSON from Get Post node.  
  - Outputs: Structured captions for all platforms.  
  - Edge cases:  
    - Expression errors if input fields missing.  
    - AI model errors propagate here.

---

#### 2.3 AI Image Generation

**Overview:**  
Generates images tailored for Instagram and for Facebook/LinkedIn posts using OpenAI’s image generation API.

**Nodes Involved:**  
- Image Instagram  
- Image Facebook e Linkedin

**Node Details:**

- **Image Instagram**  
  - Type: LangChain OpenAI Image node  
  - Role: Generates a 1024x1024 image based on Instagram caption text.  
  - Configuration:  
    - Prompt: Uses the Instagram caption from Social Media Manager output (`{{$json.output.instagram}}`).  
    - Image size: 1024x1024  
    - Returns image URLs.  
    - Credentials: OpenAI API key.  
  - Inputs: Social Media Manager output.  
  - Outputs: Image URL(s) for Instagram.  
  - Edge cases:  
    - API quota exceeded or invalid key.  
    - Prompt too vague or empty causing poor image generation.

- **Image Facebook e Linkedin**  
  - Type: LangChain OpenAI Image node  
  - Role: Generates an image sized 1792x1024 for Facebook and LinkedIn posts.  
  - Configuration:  
    - Prompt: Uses Facebook caption text (`{{$json.output.facebook}}`).  
    - Image size: 1792x1024  
    - Does not return image URLs (returns binary data).  
    - Credentials: OpenAI API key.  
  - Inputs: Social Media Manager output.  
  - Outputs: Image binary data for Facebook and LinkedIn.  
  - Edge cases: Same as Instagram image node.

---

#### 2.4 Social Media Publishing

**Overview:**  
Publishes the generated captions and images on Twitter/X, LinkedIn, Facebook, and Instagram using their respective API nodes.

**Nodes Involved:**  
- Publish on X (Twitter)  
- Publish on LinkedIn  
- Publish on Facebook  
- Publish on Instagram

**Node Details:**

- **Publish on X**  
  - Type: Twitter node  
  - Role: Posts the Twitter/X caption text.  
  - Configuration:  
    - Text: Uses Twitter caption from AI output (`{{$json.output.twitter}}`).  
    - Credentials: Twitter OAuth2 credentials.  
  - Inputs: Social Media Manager output.  
  - Outputs: Confirmation data to update Google Sheets.  
  - Edge cases:  
    - Twitter API rate limits or auth errors.  
    - Text length exceeding 280 characters.

- **Publish on LinkedIn**  
  - Type: LinkedIn node  
  - Role: Posts caption and image on LinkedIn as an organization.  
  - Configuration:  
    - Text: LinkedIn caption from AI output.  
    - Organization ID set (needs to be replaced with actual org ID).  
    - Share media category: IMAGE.  
    - Credentials: LinkedIn OAuth2.  
  - Inputs: Image Facebook e Linkedin node output (image binary) and Social Media Manager output.  
  - Outputs: Confirmation for Google Sheets update.  
  - Edge cases:  
    - Invalid organization ID or permissions.  
    - API errors or image upload failures.

- **Publish on Facebook**  
  - Type: Facebook Graph API node  
  - Role: Posts photo with caption and link on Facebook page.  
  - Configuration:  
    - Edge: photos  
    - Node: Facebook page ID (numeric).  
    - Query parameters: message (Facebook caption), link (WordPress post link).  
    - Sends binary image data from Image Facebook e Linkedin node.  
    - Credentials: Facebook Graph API.  
  - Inputs: Image Facebook e Linkedin output and Social Media Manager output.  
  - Outputs: Confirmation for Google Sheets update.  
  - Edge cases:  
    - Page permissions or token expiry.  
    - Image upload failures.

- **Publish on Instagram**  
  - Type: HTTP Request node (Facebook Graph API for Instagram)  
  - Role: Creates Instagram media object with image URL and caption.  
  - Configuration:  
    - URL: Instagram media endpoint for Facebook page.  
    - Method: POST  
    - Query parameters: image_url (from Image Instagram node), caption (Instagram caption).  
    - Credentials: Facebook Graph API.  
  - Inputs: Image Instagram output and Social Media Manager output.  
  - Outputs: Confirmation for Google Sheets update.  
  - Edge cases:  
    - Instagram API restrictions or token issues.  
    - Image URL accessibility.

---

#### 2.5 Google Sheets Update

**Overview:**  
Updates the Google Sheet to mark posts as published on each platform by setting an "x" in the corresponding column.

**Nodes Involved:**  
- X OK  
- Facebook Ok  
- Linkedin OK  
- Instagram OK

**Node Details:**

- Each node is a Google Sheets node configured to update the respective platform column (TWITTER, FACEBOOK, LINKEDIN, INSTAGRAM) with "x" for the row corresponding to the published post.  
- Uses the `row_number` from the initial Google Sheets read to target the correct row.  
- Inputs: Confirmation from respective publish nodes.  
- Outputs: None (end of workflow branch).  
- Edge cases:  
  - Google Sheets API errors or permission issues.  
  - Row number mismatch causing wrong row update.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                                    | Input Node(s)                 | Output Node(s)                          | Sticky Note                                                                                           |
|-------------------------|----------------------------------|---------------------------------------------------|------------------------------|---------------------------------------|-----------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                   | Starts the workflow manually                       | None                         | Google Sheets                         |                                                                                                     |
| Google Sheets           | Google Sheets                    | Retrieves WordPress Post ID from Google Sheet     | When clicking ‘Test workflow’ | Get Post                             | Get the Post ID of the Wordpress article on which you want to generate the caption for social media |
| Get Post                | WordPress                       | Fetches WordPress post content by Post ID         | Google Sheets                | Social Media Manager                  |                                                                                                     |
| Social Media Manager    | LangChain Chain LLM             | Generates platform-specific captions using AI     | Get Post                    | Image Instagram, Image Facebook e Linkedin, Publish on X | The SMM Chain analyses the content of the post and creates the most suitable caption based on the destination social network. |
| OpenRouter Chat Model   | LangChain OpenRouter Chat Model | AI backend for caption generation                  | Social Media Manager         | Structured Output Parser              |                                                                                                     |
| Structured Output Parser| LangChain Output Parser         | Parses AI output into structured JSON captions    | OpenRouter Chat Model        | Social Media Manager                  |                                                                                                     |
| Image Instagram         | LangChain OpenAI Image          | Generates Instagram-specific image                 | Social Media Manager          | Publish on Instagram                  |                                                                                                     |
| Image Facebook e Linkedin| LangChain OpenAI Image          | Generates Facebook and LinkedIn image              | Social Media Manager          | Publish on LinkedIn, Publish on Facebook |                                                                                                     |
| Publish on X            | Twitter                        | Posts caption on Twitter/X                         | Social Media Manager          | X OK                                |                                                                                                     |
| Publish on LinkedIn     | LinkedIn                       | Posts caption and image on LinkedIn                | Image Facebook e Linkedin     | Linkedin OK                         |                                                                                                     |
| Publish on Facebook     | Facebook Graph API             | Posts photo with caption on Facebook page          | Image Facebook e Linkedin     | Facebook Ok                        |                                                                                                     |
| Publish on Instagram    | HTTP Request (Facebook Graph)  | Posts image and caption on Instagram               | Image Instagram              | Instagram OK                       |                                                                                                     |
| X OK                    | Google Sheets                 | Marks Twitter post as published in Google Sheet   | Publish on X                 | None                                |                                                                                                     |
| Facebook Ok             | Google Sheets                 | Marks Facebook post as published in Google Sheet  | Publish on Facebook          | None                                |                                                                                                     |
| Linkedin OK             | Google Sheets                 | Marks LinkedIn post as published in Google Sheet  | Publish on LinkedIn          | None                                |                                                                                                     |
| Instagram OK            | Google Sheets                 | Marks Instagram post as published in Google Sheet | Publish on Instagram         | None                                |                                                                                                     |
| Sticky Note             | Sticky Note                   | Instructional note about Post ID                   | None                         | None                                | Get the Post ID of the Wordpress article on which you want to generate the caption for social media |
| Sticky Note1            | Sticky Note                   | Explains Social Media Manager role                 | None                         | None                                | The SMM Chain analyses the content of the post and creates the most suitable caption based on the destination social network. |
| Sticky Note2            | Sticky Note                   | Setup instructions and API key links               | None                         | None                                | ## STEP 1: API keys and authentication links for social media platforms                             |
| Sticky Note3            | Sticky Note                   | Google Sheet setup instructions                     | None                         | None                                | ## STEP 2: Clone the Google Sheet and set WordPress Post ID only                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow on demand.

2. **Add Google Sheets Node**  
   - Type: Google Sheets  
   - Operation: Read rows  
   - Document ID: Set to your Google Sheet containing social media post data.  
   - Sheet Name: Set to the appropriate sheet (e.g., "gid=0").  
   - Options: Return first match only.  
   - Filters: Use column "TWITTER" or other to identify the row.  
   - Credentials: Configure Google Sheets OAuth2 credentials.

3. **Add WordPress Node (Get Post)**  
   - Type: WordPress  
   - Operation: Get post by ID  
   - Post ID: Expression from Google Sheets node `{{$json["POST ID"]}}`  
   - Credentials: Configure WordPress API credentials.

4. **Add LangChain OpenRouter Chat Model Node**  
   - Type: LangChain OpenRouter Chat Model  
   - Model: `google/gemini-2.0-flash-exp:free`  
   - Credentials: OpenRouter API key.

5. **Add LangChain Structured Output Parser Node**  
   - Type: Structured Output Parser  
   - Schema: JSON object with string properties: twitter, facebook, linkedin, instagram.

6. **Add LangChain Chain LLM Node (Social Media Manager)**  
   - Type: Chain LLM  
   - Prompt:  
     ```
     Generate social content from the following text with title "{{ $json.title.rendered }}" (in the same language):

     '''
     {{ $json.content.rendered }}
     '''

     [Insert detailed platform-specific instructions as per workflow description]
     ```  
   - Output parser: Use the Structured Output Parser node.  
   - Connect input from WordPress Get Post node.  
   - Connect AI model to OpenRouter Chat Model node.

7. **Add LangChain OpenAI Image Node for Instagram**  
   - Type: OpenAI Image  
   - Prompt: `{{$json.output.instagram}}`  
   - Image size: 1024x1024  
   - Return image URLs: true  
   - Credentials: OpenAI API key.

8. **Add LangChain OpenAI Image Node for Facebook and LinkedIn**  
   - Type: OpenAI Image  
   - Prompt: `{{$json.output.facebook}}`  
   - Image size: 1792x1024  
   - Return image URLs: false (binary data)  
   - Credentials: OpenAI API key.

9. **Add Twitter Node (Publish on X)**  
   - Type: Twitter  
   - Text: `{{$json.output.twitter}}`  
   - Credentials: Twitter OAuth2.

10. **Add LinkedIn Node (Publish on LinkedIn)**  
    - Type: LinkedIn  
    - Text: `{{$json.output.linkedin}}`  
    - Post as: Organization  
    - Organization ID: Set your LinkedIn organization ID  
    - Share media category: IMAGE  
    - Credentials: LinkedIn OAuth2.

11. **Add Facebook Graph API Node (Publish on Facebook)**  
    - Type: Facebook Graph API  
    - Edge: photos  
    - Node: Your Facebook page ID  
    - Query parameters:  
      - message: `{{$json.output.facebook}}`  
      - link: `{{$json.link}}` from WordPress post  
    - Send binary data: true (from Facebook/LinkedIn image node)  
    - Credentials: Facebook Graph API.

12. **Add HTTP Request Node (Publish on Instagram)**  
    - Type: HTTP Request  
    - URL: `https://graph.facebook.com/v20.0/{page-id}/media` (replace `{page-id}`)  
    - Method: POST  
    - Query parameters:  
      - image_url: `{{$json.url}}` from Instagram image node  
      - caption: `{{$json.output.instagram}}`  
    - Authentication: Use Facebook Graph API credentials.

13. **Add Google Sheets Update Nodes for Each Platform**  
    - For Twitter (X OK), Facebook (Facebook Ok), LinkedIn (Linkedin OK), Instagram (Instagram OK)  
    - Operation: Update row  
    - Columns: Set respective platform column to "x"  
    - Matching column: Use `row_number` from initial Google Sheets read  
    - Credentials: Google Sheets OAuth2.

14. **Connect Nodes in Order:**  
    - Manual Trigger → Google Sheets → Get Post → Social Media Manager →  
      - Image Instagram → Publish on Instagram → Instagram OK  
      - Image Facebook e Linkedin → Publish on LinkedIn → Linkedin OK  
      - Image Facebook e Linkedin → Publish on Facebook → Facebook Ok  
      - Social Media Manager → Publish on X → X OK

15. **Test the Workflow**  
    - Trigger manually.  
    - Verify posts appear on all platforms.  
    - Confirm Google Sheet updates.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                                         |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Get the API Keys of the social networks you want to publish on: X (Twitter), LinkedIn, Facebook, Instagram.    | [X API](https://docs.x.com/x-api/getting-started/getting-access), [LinkedIn API](https://learn.microsoft.com/en-us/linkedin/marketing/community-management/shares/posts-api?view=li-lms-2025-02&tabs=http), [Facebook API](https://developers.facebook.com/docs/facebook-login/guides/access-tokens#portabletokens), [Instagram API](https://developers.facebook.com/docs/instagram-platform/instagram-api-with-facebook-login/content-publishing/) |
| Clone the Google Sheet template to set WordPress Post IDs for social media post generation.                     | [Google Sheet Template](https://docs.google.com/spreadsheets/d/1suPQNdgoAzrklleN4ok2mZnsq0GK1dt59oIHv8JWX5U/edit?usp=sharing) |
| The Social Media Manager node uses detailed instructions to generate platform-optimized captions with AI.      | See node prompt content in Block 2.2 for tone, style, hashtags, and calls to action per platform.                       |
| Ensure WordPress REST API is accessible and credentials are valid for fetching posts.                          |                                                                                                                        |
| Image generation sizes differ per platform to optimize visual presentation.                                    | Instagram: 1024x1024; Facebook/LinkedIn: 1792x1024                                                                      |

---

This document provides a comprehensive understanding of the workflow, enabling reproduction, modification, and troubleshooting for advanced users and AI agents alike.