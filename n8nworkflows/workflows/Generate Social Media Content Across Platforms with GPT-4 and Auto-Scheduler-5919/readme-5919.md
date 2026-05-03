Generate Social Media Content Across Platforms with GPT-4 and Auto-Scheduler

https://n8nworkflows.xyz/workflows/generate-social-media-content-across-platforms-with-gpt-4-and-auto-scheduler-5919


# Generate Social Media Content Across Platforms with GPT-4 and Auto-Scheduler

### 1. Workflow Overview

This workflow, titled **AI Social Media Content Generator & Scheduler**, automates the creation and scheduling of tailored social media posts across multiple platforms using GPT-4 AI. It is designed for marketing teams, content creators, agencies, and personal brands aiming to rapidly generate engaging, platform-optimized content with consistent brand voice and high engagement potential.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception and Validation:** Receives content generation requests via webhook or triggers scheduled daily topics; validates and prepares input parameters.
- **1.2 AI Content Generation and Formatting:** Uses GPT-4 via OpenAI integration to generate platform-specific social media content, then formats and enriches the AI response with metadata.
- **1.3 Response Delivery and Scheduling:** Returns generated content to the requester or triggers further downstream processes (e.g., scheduling posts). Currently, scheduling beyond daily topic preparation is implied as an extension.

Supporting the main logic are multiple sticky notes providing detailed documentation, best practices, content type guidelines, and setup instructions.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Validation

**Overview:**  
This block handles incoming external requests or scheduled triggers for content generation. It validates and normalizes input parameters such as topic, platforms, tone, language, and content preferences.

**Nodes Involved:**  
- Content Request Webhook  
- Process Input  
- Daily Content Schedule  
- Prepare Daily Topic  

**Node Details:**

- **Content Request Webhook**  
  - *Type:* Webhook  
  - *Role:* Entry point for external POST requests containing content generation parameters.  
  - *Configuration:*  
    - HTTP POST method on path `generate-social-content`  
    - CORS enabled (`Access-Control-Allow-Origin: *`)  
    - Responds via downstream `Send Response` node  
  - *Inputs:* None (triggered externally)  
  - *Outputs:* Passes request data to `Process Input`  
  - *Edge Cases:* Missing topic or invalid JSON in request body; CORS errors if misconfigured  

- **Process Input**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Extracts, validates, and enriches input parameters.  
  - *Configuration Highlights:*  
    - Extracts `topic`, `platforms`, `tone`, `language`, `includeHashtags`, `includeEmojis`, and `contentLength` from request body or defaults  
    - Validates presence of `topic`, throws error if missing  
    - Defines platform-specific character limits and hashtag counts  
    - Generates unique request ID and timestamp  
  - *Key Expressions:* Uses null-coalescing and optional chaining to safely access input properties  
  - *Inputs:* Data from webhook or scheduled flow  
  - *Outputs:* Structured JSON with validated parameters forwarded to AI generation  
  - *Potential Failures:* Missing topic causes explicit error; malformed request data may cause runtime errors  

- **Daily Content Schedule**  
  - *Type:* Schedule Trigger  
  - *Role:* Triggers daily automated content generation every 24 hours.  
  - *Configuration:* Interval trigger every 24 hours  
  - *Outputs:* Invokes `Prepare Daily Topic`  
  - *Edge Cases:* Scheduler downtime; time zone considerations for daily trigger  

- **Prepare Daily Topic**  
  - *Type:* Set node  
  - *Role:* Constructs daily content topic and parameters based on current day.  
  - *Configuration:*  
    - Sets `topic` string using current day (e.g., "Monday motivation - Share something inspiring for Monday")  
    - Sets default platforms to LinkedIn, Twitter, Instagram  
    - Sets tone to "inspirational"  
  - *Outputs:* Prepared input forwarded to `Generate Content`  
  - *Edge Cases:* Date/time format errors; limited platform selection hardcoded  

---

#### 2.2 AI Content Generation and Formatting

**Overview:**  
This block uses GPT-4 via the OpenAI API to generate tailored social media content for specified platforms, then formats and enriches the AI response with engagement predictions and metadata.

**Nodes Involved:**  
- Generate Content  
- Format Response  

**Node Details:**

- **Generate Content**  
  - *Type:* OpenAI (LangChain integration)  
  - *Role:* Sends prompt and parameters to GPT-4 to generate social media posts.  
  - *Configuration:*  
    - Model: GPT-4-mini (optimized variant)  
    - Temperature: 0.7 for creative yet coherent output  
    - Max tokens: 3000 (allows for detailed multi-platform content)  
    - System message defines role as a professional social media content creator with platform-specific expertise, tone, hashtags, emojis, and content length based on input JSON parameters  
    - User message requests JSON-formatted content with keys per platform including caption, hashtags, posting time, CTAs, and engagement tips  
    - Credentials: Requires OpenAI API key  
  - *Inputs:* Validated parameters from `Process Input` or `Prepare Daily Topic`  
  - *Outputs:* Raw AI response JSON to `Format Response`  
  - *Potential Failures:* API authentication errors, rate limits, invalid prompt formatting, or incomplete AI responses  

- **Format Response**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Parses AI response JSON, adds metadata, and generates mock engagement predictions.  
  - *Configuration:*  
    - Parses `message.content` from AI response  
    - Handles JSON parse errors gracefully by returning error object with raw response  
    - Adds metadata such as requestId, topic, timestamp, tone, language, and platforms  
    - Adds mock predictions for engagement rate, estimated reach, and best posting time per platform  
    - Defines optimal posting times per platform as static lookup  
  - *Inputs:* AI response node output  
  - *Outputs:* Structured JSON ready for response or downstream processing  
  - *Edge Cases:* AI returns malformed JSON or unexpected format, fallback error handling activated  

---

#### 2.3 Response Delivery and Scheduling

**Overview:**  
This block returns the generated and formatted social media content to the original requester and lays groundwork for scheduled and bulk content generation paths.

**Nodes Involved:**  
- Send Response  

**Node Details:**

- **Send Response**  
  - *Type:* Respond to Webhook  
  - *Role:* Sends HTTP 200 JSON response containing generated content back to the client.  
  - *Configuration:*  
    - Response code: 200  
    - Content-Type: application/json  
    - Response body: stringified formatted JSON from `Format Response` node  
  - *Inputs:* From `Format Response`  
  - *Outputs:* None (ends HTTP request lifecycle)  
  - *Potential Failures:* Network timeouts, client disconnects, incorrect response formatting  

---

### 3. Summary Table

| Node Name              | Node Type                   | Functional Role                            | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                                             |
|------------------------|-----------------------------|--------------------------------------------|-----------------------|------------------------|-------------------------------------------------------------------------------------------------------------------------|
| Sticky Note            | Sticky Note                 | Documentation/overview                     | —                     | —                      | ## 🚀 AI SOCIAL MEDIA CONTENT GENERATOR ... (overview of features and use cases)                                        |
| Sticky Note1           | Sticky Note                 | Content types documentation                 | —                     | —                      | ## 📝 CONTENT TYPES ... (supported post formats per platform)                                                           |
| Sticky Note2           | Sticky Note                 | Setup guide documentation                   | —                     | —                      | ## ⚙️ SETUP GUIDE ... (required and optional setup steps and tips)                                                      |
| Sticky Note3           | Sticky Note                 | Input prompt examples                        | —                     | —                      | ## 💡 PROMPT EXAMPLES ... (sample input topics for AI content generation)                                               |
| Sticky Note4           | Sticky Note                 | Customization options                        | —                     | —                      | ## 🎨 CUSTOMIZATION ... (brand voice and advanced features)                                                             |
| Sticky Note5           | Sticky Note                 | Best practices documentation                 | —                     | —                      | ## 📈 BEST PRACTICES ... (content strategy, common mistakes, success tips)                                               |
| Sticky Note6           | Sticky Note                 | Workflow path overview                       | —                     | —                      | ## 🔄 WORKFLOW PATHS ... (API, scheduled, bulk generation, and extensions overview)                                      |
| Content Request Webhook| Webhook                     | Receives content generation requests       | —                     | Process Input          | Receives content generation requests with topic and parameters                                                          |
| Process Input          | Code                        | Validates and prepares input parameters     | Content Request Webhook| Generate Content        |                                                                                                                         |
| Generate Content       | OpenAI (LangChain)          | AI generates platform-specific content      | Process Input, Prepare Daily Topic | Format Response         | Uses AI to generate platform-specific content                                                                            |
| Format Response        | Code                        | Parses AI output, adds metadata and predictions | Generate Content       | Send Response           | Structures the AI response and adds metadata                                                                             |
| Send Response          | Respond to Webhook          | Sends JSON response to requester             | Format Response        | —                      | Returns the generated content to the requester                                                                           |
| Daily Content Schedule | Schedule Trigger            | Triggers daily content generation            | —                     | Prepare Daily Topic     | Triggers daily content generation for consistent posting                                                                 |
| Prepare Daily Topic    | Set                         | Prepares daily topic and parameters          | Daily Content Schedule | Generate Content        | Creates daily content topics based on day of week                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note nodes** (optional but recommended) for documentation purposes:  
   - Overview, Content Types, Setup Guide, Prompt Examples, Customization, Best Practices, Workflow Paths.  
   - Copy text content from the provided notes for internal reference.

2. **Create Webhook Node:**  
   - Name: `Content Request Webhook`  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `generate-social-content`  
   - Response Mode: `responseNode`  
   - Add response header: `Access-Control-Allow-Origin: *`  
   - No credentials needed  
   - Position appropriately

3. **Create Code Node:**  
   - Name: `Process Input`  
   - Type: Code  
   - Language: JavaScript  
   - Paste JS code to extract and validate input parameters (topic, platforms, tone, language, hashtags, emojis, length), generate requestId and timestamp, define platform character limits and hashtag counts.  
   - Connect output of `Content Request Webhook` to this node.

4. **Create OpenAI Node:**  
   - Name: `Generate Content`  
   - Type: OpenAI (LangChain)  
   - Credentials: Configure with valid OpenAI API key  
   - Model: `gpt-4-mini`  
   - Temperature: 0.7  
   - Max tokens: 3000  
   - System message: Define AI role as professional social media content creator, taking parameters tone, hashtags, emojis, content length from input JSON.  
   - User message: Request JSON-formatted multi-platform content including captions, hashtags, posting times, CTAs, engagement tips, with platform character limits and hashtag recommendations injected from input JSON.  
   - Connect output of `Process Input` to this node.

5. **Create Code Node:**  
   - Name: `Format Response`  
   - Type: Code  
   - Language: JavaScript  
   - Paste JS code to parse AI response, handle errors, add metadata (requestId, topic, timestamp, tone, language, platforms), and add mock engagement predictions (engagement rate, estimated reach, best posting time) per platform.  
   - Connect output of `Generate Content` to this node.

6. **Create Respond to Webhook Node:**  
   - Name: `Send Response`  
   - Type: Respond to Webhook  
   - Response Code: 200  
   - Response Headers: Content-Type: application/json  
   - Response Body: `={{ JSON.stringify($json, null, 2) }}`  
   - Connect output of `Format Response` to this node.

7. **Create Schedule Trigger Node:**  
   - Name: `Daily Content Schedule`  
   - Type: Schedule Trigger  
   - Set interval: every 24 hours  

8. **Create Set Node:**  
   - Name: `Prepare Daily Topic`  
   - Type: Set  
   - Assign values:  
     - `topic`: `"={{ $now.toFormat('EEEE') }} motivation - Share something inspiring for {{ $now.toFormat('EEEE') }}"`  
     - `platforms`: `["linkedin", "twitter", "instagram"]`  
     - `tone`: `"inspirational"`  
   - Connect output of `Daily Content Schedule` to this node.

9. **Connect `Prepare Daily Topic` output to the `Generate Content` node** to enable scheduled daily content generation.

10. **Validate all connections:**  
    - `Content Request Webhook` → `Process Input` → `Generate Content` → `Format Response` → `Send Response`  
    - `Daily Content Schedule` → `Prepare Daily Topic` → `Generate Content`

11. **Credential Setup:**  
    - Create and configure OpenAI API credential with valid API key.  
    - Assign credential to `Generate Content` node.

12. **Test Workflow:**  
    - Trigger webhook with sample POST payload containing `topic` and optional parameters.  
    - Confirm valid JSON response with generated content per platform.  
    - Verify daily scheduled run triggers content generation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                     | Context or Link                                                                                       |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The workflow supports multi-language content generation, hashtag optimization, emoji inclusion, and tone customization for brand voice consistency.                                            | Workflow features overview in Sticky Note                                                          |
| Suggested posting times per platform are based on general social media best practices (e.g., LinkedIn mornings midweek, Twitter peak hours).                                                    | Defined in `Format Response` node’s helper function                                                |
| Extensions can include image generation APIs, social media platform posting APIs, analytics tracking webhooks, and approval workflows for content moderation.                                  | Workflow Paths sticky note outlines potential extensions                                           |
| For improved AI response reliability, consider implementing retry logic and advanced error handling on OpenAI API calls.                                                                      | Best practice suggestion for production workflows                                                  |
| Branding and customization parameters (tone, style, emoji usage) enable adaptable content for different audiences and campaign types.                                                        | Customization sticky note                                                                          |
| Ensure CORS is properly configured on the webhook node for browser-based integrations.                                                                                                          | Webhook node CORS headers                                                                          |
| For bulk content generation, a CSV input node and loop can be added, as noted in the workflow paths sticky note.                                                                                | Bulk generation path reference                                                                     |
| Monitor API usage and cost associated with OpenAI GPT-4 calls, especially for large-scale or scheduled content generation.                                                                     | General operational consideration                                                                 |
| The workflow JSON is compatible with n8n versions supporting OpenAI LangChain integration and webhook response nodes (version 1.1+ recommended).                                               | Version-specific note                                                                              |

---

**Disclaimer:** The provided text and workflow are generated using n8n automation, fully compliant with content policies, handling legal and public data only.