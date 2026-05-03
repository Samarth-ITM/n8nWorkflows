Create an Automated Instagram Content Generator with OpenAI and FB Graph API

https://n8nworkflows.xyz/workflows/create-an-automated-instagram-content-generator-with-openai-and-fb-graph-api-4016


# Create an Automated Instagram Content Generator with OpenAI and FB Graph API

### 1. Workflow Overview

This workflow automates the creation, approval, and posting of Instagram content using AI (OpenAI) and the Facebook Graph API, integrated through n8n. It is designed for digital entrepreneurs, marketing agencies, and content creators to automate virtual influencer content generation, scheduling, and monetization with minimal manual intervention.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Setup**: Manual trigger and initial configuration.
- **1.2 Plan Management and Approval**: Quarterly, monthly, and weekly content planning, including aggregation, generation, approval via Gmail, and conditional branching for changes.
- **1.3 Content Generation and Preparation**: Retrieval of past posts, AI generation of new Instagram posts, image generation, and uploading to Supabase.
- **1.4 Post Approval and Publishing**: Post approval via Gmail, conditional branching on approval status, saving posts, and final publishing through Facebook Graph API.
- **1.5 Auxiliary and Utility Nodes**: Unique ID generation, sticky notes for documentation, and minor data transformations.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Setup

**Overview:**  
This block initiates the workflow manually and sets up initial parameters and configuration.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Configure workflow  

**Node Details:**  

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point for manual workflow execution  
  - Configuration: Default manual trigger, no parameters  
  - Input: None  
  - Output: Connected to "Configure workflow"  
  - Edge cases: None; user must trigger manually  

- **Configure workflow**  
  - Type: Set  
  - Role: Prepares initial workflow data or variables (empty in this configuration)  
  - Configuration: No parameters set, acts as a placeholder or marker node  
  - Input: From manual trigger  
  - Output: Connects to "Get quarterly plan"  
  - Edge cases: None  

---

#### 1.2 Plan Management and Approval

**Overview:**  
Manages quarterly, monthly, and weekly content plans. Retrieves plans from Supabase, aggregates data, generates plans with AI, and handles approval workflows via Gmail nodes. Conditional nodes control flow based on approval status or presence of plans.

**Nodes Involved:**  
- Get quarterly plan  
- Quaterly plan aggregate  
- Has quarterly plan?  
- Get instructions on the quarterly plan  
- Generate quarterly plan  
- Get approval on the quarterly plan  
- Approve quarterly plan?  
- Save quarterly plan  
- Change request fields  
- Get monthly plan  
- Monthly plan aggregate  
- Has monthly plan?  
- Get instructions on the monthly plan  
- Generate monthly plan  
- Montly plan approval  
- Need changes?1  
- Change request fields1  
- Save monthly plan  
- Get weekly plan  
- Weekly plan aggregate  
- Has weekly plan?  
- Get instructions on the weekly plan  
- Generate weekly plan  
- Weekly plan approval  
- Changes to the weekly plan?  
- Weekly plan change fields  
- Save weekly plan  

**Node Details:**

- **Get quarterly plan**  
  - Type: Supabase  
  - Role: Retrieve stored quarterly plan data  
  - Configuration: Connects to Supabase database, queries quarterly plan table  
  - Input: From "Configure workflow"  
  - Output: To "Quaterly plan aggregate"  
  - Edge cases: Connection/auth errors with Supabase, empty data results  

- **Quaterly plan aggregate**  
  - Type: Aggregate  
  - Role: Summarize or process quarterly plan records  
  - Configuration: Aggregation logic (e.g., grouping or counting) unspecified, likely merges multiple entries  
  - Input: From "Get quarterly plan"  
  - Output: To "Has quarterly plan?"  
  - Edge cases: Empty aggregation results  

- **Has quarterly plan?**  
  - Type: If  
  - Role: Checks if quarterly plan data exists to decide next step  
  - Configuration: Conditional expression on aggregated data presence  
  - Input: From "Quaterly plan aggregate"  
  - Output:  
    - True: To "Get monthly plan"  
    - False: To "Get instructions on the quarterly plan"  
  - Edge cases: Expression evaluation errors  

- **Get instructions on the quarterly plan**  
  - Type: Gmail (Webhook)  
  - Role: Receive instructions for creating a quarterly plan from email  
  - Configuration: Gmail IMAP webhook listening to a specific inbox or label  
  - Input: From "Has quarterly plan?" (False branch)  
  - Output: To "Generate quarterly plan"  
  - Edge cases: Email connectivity issues, webhook misconfiguration  

- **Generate quarterly plan**  
  - Type: LangChain Information Extractor (OpenAI)  
  - Role: Generate quarterly content plan text using AI based on instructions  
  - Configuration: Uses AI model to output structured plan data  
  - Input: From "Get instructions on the quarterly plan" and "Change request fields"  
  - Output: To "Get approval on the quarterly plan"  
  - Edge cases: API rate limits, prompt failures, OpenAI service errors  

- **Get approval on the quarterly plan**  
  - Type: Gmail (Webhook)  
  - Role: Waits for approval email on quarterly plan  
  - Configuration: Gmail IMAP webhook listening for approval response  
  - Input: From "Generate quarterly plan"  
  - Output: To "Approve quarterly plan?"  
  - Edge cases: Email delays, webhook failures  

- **Approve quarterly plan?**  
  - Type: If  
  - Role: Conditional logic for plan approval  
  - Configuration: Checks email response content for approval status  
  - Input: From "Get approval on the quarterly plan"  
  - Output:  
    - True: To "Save quarterly plan"  
    - False: To "Change request fields"  
  - Edge cases: Parsing errors, ambiguous email content  

- **Save quarterly plan**  
  - Type: Supabase  
  - Role: Persist approved quarterly plan in database  
  - Configuration: Supabase insert/update operation  
  - Input: From "Approve quarterly plan?" (True branch)  
  - Output: To "Get quarterly plan" (loop for updates)  
  - Edge cases: Database write failures  

- **Change request fields**  
  - Type: Set  
  - Role: Prepare data fields for change requests for quarterly plan  
  - Configuration: Sets variables/fields indicating requested changes  
  - Input: From "Approve quarterly plan?" (False branch)  
  - Output: To "Generate quarterly plan" (retry generation with changes)  
  - Edge cases: Misconfiguration of fields  

- The monthly and weekly plan nodes follow the same pattern:  
  - Supabase retrieval nodes ("Get monthly plan", "Get weekly plan")  
  - Aggregation nodes ("Monthly plan aggregate", "Weekly plan aggregate")  
  - Conditional nodes ("Has monthly plan?", "Has weekly plan?")  
  - Gmail instruction nodes ("Get instructions on the monthly plan", "Get instructions on the weekly plan")  
  - AI generation nodes ("Generate monthly plan", "Generate weekly plan")  
  - Gmail approval nodes ("Montly plan approval", "Weekly plan approval")  
  - Conditional approval nodes ("Need changes?1", "Changes to the weekly plan?")  
  - Change request and save nodes ("Change request fields1", "Weekly plan change fields", "Save monthly plan", "Save weekly plan")  

- Edge cases common to this block include:  
  - Failures in database connectivity (Supabase)  
  - Email webhook latency or failure  
  - AI model errors or rate limits  
  - Conditional expression evaluation failures  
  - Infinite loops if approvals or change requests cycle endlessly  

---

#### 1.3 Content Generation and Preparation

**Overview:**  
Generates new Instagram posts using AI, processes images with OpenAI, retrieves past post data, and uploads images to Supabase storage.

**Nodes Involved:**  
- Get posts from the past 7 days  
- merge previous week's posts  
- Generate new Instagram post  
- Generate unique id  
- Download the reference image  
- Generate image with OpenAI  
- Convert response to image  
- Upload image to Supabase  
- Add share link  

**Node Details:**

- **Get posts from the past 7 days**  
  - Type: Supabase  
  - Role: Retrieve recent Instagram posts for context  
  - Configuration: Query on posts within last 7 days  
  - Input: From "Has weekly plan?" (True branch)  
  - Output: To "merge previous week's posts"  
  - Edge cases: No recent posts, database errors  

- **merge previous week's posts**  
  - Type: Aggregate  
  - Role: Consolidates recent posts into a summary or structured data  
  - Configuration: Aggregation logic to prepare data for AI input  
  - Input: From "Get posts from the past 7 days"  
  - Output: To "Generate new Instagram post"  
  - Edge cases: Empty aggregation results  

- **Generate new Instagram post**  
  - Type: LangChain Information Extractor (OpenAI)  
  - Role: Generate new post content (caption, hashtags, etc.) via AI  
  - Configuration: Uses AI model with prior posts as context  
  - Input: From "merge previous week's posts" and "Add post change request fields"  
  - Output: To "Generate unique id"  
  - Edge cases: API errors, inappropriate content generation  

- **Generate unique id**  
  - Type: Crypto  
  - Role: Create a unique identifier for the post or image  
  - Configuration: Default cryptographic unique ID generation  
  - Input: From "Generate new Instagram post"  
  - Output: To "Download the reference image"  
  - Edge cases: Very rare crypto failures  

- **Download the reference image**  
  - Type: HTTP Request  
  - Role: Download image used as reference or base for generation  
  - Configuration: HTTP GET request to image URL from prior node data  
  - Input: From "Generate unique id"  
  - Output: To "Generate image with OpenAI"  
  - Edge cases: Network errors, invalid URLs  

- **Generate image with OpenAI**  
  - Type: HTTP Request  
  - Role: Calls OpenAI image generation API (e.g., DALL·E)  
  - Configuration: Sends prompt and parameters to image API  
  - Input: From "Download the reference image"  
  - Output: To "Convert response to image"  
  - Edge cases: API limits, service unavailability  

- **Convert response to image**  
  - Type: Convert to File  
  - Role: Converts API response data into file format suitable for upload  
  - Configuration: Converts base64 or JSON response to image file  
  - Input: From "Generate image with OpenAI"  
  - Output: To "Upload image to Supabase"  
  - Edge cases: Conversion failures due to malformed data  

- **Upload image to Supabase**  
  - Type: HTTP Request  
  - Role: Uploads the generated image file to Supabase storage bucket  
  - Configuration: HTTP POST with authentication and image payload  
  - Input: From "Convert response to image"  
  - Output: To "Add share link"  
  - Edge cases: Upload failures, auth errors  

- **Add share link**  
  - Type: Set  
  - Role: Adds or constructs a shareable link for the uploaded image  
  - Configuration: Sets a URL field based on Supabase response  
  - Input: From "Upload image to Supabase"  
  - Output: To "Get approval for the post"  
  - Edge cases: Missing URL in upload response  

---

#### 1.4 Post Approval and Publishing

**Overview:**  
Obtains post approval via Gmail, handles approval decision logic, saves approved posts to Supabase, and publishes them using Facebook Graph API.

**Nodes Involved:**  
- Get approval for the post  
- Post approved?  
- Add post change request fields  
- Save the post  
- Facebook Graph API  
- Facebook Graph API1  

**Node Details:**

- **Get approval for the post**  
  - Type: Gmail (Webhook)  
  - Role: Listens for approval email on generated post  
  - Configuration: Gmail IMAP webhook  
  - Input: From "Add share link"  
  - Output: To "Post approved?"  
  - Edge cases: Email delays, webhook failure  

- **Post approved?**  
  - Type: If  
  - Role: Checks if post was approved or needs changes  
  - Configuration: Checks email content for approval flag  
  - Input: From "Get approval for the post"  
  - Output:  
    - True: To "Save the post"  
    - False: To "Add post change request fields"  
  - Edge cases: Parsing errors, ambiguous responses  

- **Add post change request fields**  
  - Type: Set  
  - Role: Prepare change request data for post regeneration  
  - Configuration: Sets variables indicating requested changes  
  - Input: From "Post approved?" (False branch)  
  - Output: To "Generate new Instagram post" (restart content generation)  
  - Edge cases: Incorrect data setting  

- **Save the post**  
  - Type: Supabase  
  - Role: Store approved post data in database  
  - Configuration: Insert or update post record in Supabase  
  - Input: From "Post approved?" (True branch)  
  - Output: To "Facebook Graph API"  
  - Edge cases: Database write errors  

- **Facebook Graph API**  
  - Type: Facebook Graph API  
  - Role: Publish Instagram post through Facebook's Graph API for Instagram  
  - Configuration: Authenticated API call to create post on Instagram account  
  - Input: From "Save the post"  
  - Output: To "Facebook Graph API1" (likely a follow-up or confirmation request)  
  - Edge cases: API rate limits, auth token expiration, insufficient permissions  

- **Facebook Graph API1**  
  - Type: Facebook Graph API  
  - Role: Possibly a secondary or follow-up Facebook API call (e.g., for media upload or post confirmation)  
  - Configuration: Similar to above, may handle media or comments  
  - Input: From "Facebook Graph API"  
  - Output: None specified  
  - Edge cases: Same as above  

---

#### 1.5 Auxiliary and Utility Nodes

**Overview:**  
Includes unique ID generation, multiple sticky notes for documentation, and minor data transformation nodes.

**Nodes Involved:**  
- Sticky Note (multiple instances)  
- Change request fields (quarterly, monthly)  
- Change request fields1 (monthly)  
- Add post change request fields  
- Generate unique id  

**Node Details:**  
- Sticky Notes are used for internal documentation and do not affect workflow logic.  
- Change request fields nodes prepare data for retry loops upon rejection or requested changes.  
- Generate unique id node creates identifiers to track posts and assets, ensuring no collisions.  

---

### 3. Summary Table

| Node Name                      | Node Type                        | Functional Role                          | Input Node(s)                    | Output Node(s)                 | Sticky Note |
|--------------------------------|---------------------------------|----------------------------------------|---------------------------------|-------------------------------|-------------|
| When clicking ‘Test workflow’   | Manual Trigger                  | Manual start of workflow                | None                            | Configure workflow             |             |
| Configure workflow             | Set                             | Initial configuration (placeholder)    | When clicking ‘Test workflow’   | Get quarterly plan             |             |
| Get quarterly plan             | Supabase                        | Retrieve quarterly plan from DB         | Configure workflow              | Quaterly plan aggregate        |             |
| Quaterly plan aggregate        | Aggregate                      | Aggregate quarterly plan data           | Get quarterly plan              | Has quarterly plan?            |             |
| Has quarterly plan?            | If                             | Check if quarterly plan exists           | Quaterly plan aggregate         | Get monthly plan / Get instructions on the quarterly plan |             |
| Get instructions on the quarterly plan | Gmail (Webhook)           | Receive instructions via email           | Has quarterly plan? (False)     | Generate quarterly plan        |             |
| Generate quarterly plan        | LangChain Info Extractor       | AI-generated quarterly plan              | Get instructions on the quarterly plan, Change request fields | Get approval on the quarterly plan |             |
| Get approval on the quarterly plan | Gmail (Webhook)             | Wait for quarterly plan approval email  | Generate quarterly plan         | Approve quarterly plan?        |             |
| Approve quarterly plan?        | If                             | Determine if quarterly plan approved     | Get approval on the quarterly plan | Save quarterly plan / Change request fields |             |
| Save quarterly plan            | Supabase                       | Save approved quarterly plan             | Approve quarterly plan? (True)  | Get quarterly plan             |             |
| Change request fields          | Set                            | Prepare change request data              | Approve quarterly plan? (False) | Generate quarterly plan        |             |
| Get monthly plan               | Supabase                       | Retrieve monthly plan from DB            | Has quarterly plan? (True)      | Monthly plan aggregate         |             |
| Monthly plan aggregate         | Aggregate                     | Aggregate monthly plan data              | Get monthly plan                | Has monthly plan?              |             |
| Has monthly plan?              | If                            | Check if monthly plan exists              | Monthly plan aggregate          | Get weekly plan / Get instructions on the monthly plan |             |
| Get instructions on the monthly plan | Gmail (Webhook)            | Receive monthly plan instructions email  | Has monthly plan? (False)       | Generate monthly plan          |             |
| Generate monthly plan          | LangChain Info Extractor      | AI-generated monthly plan                 | Get instructions on the monthly plan, Change request fields1 | Montly plan approval           |             |
| Montly plan approval           | Gmail (Webhook)               | Wait for monthly plan approval email     | Generate monthly plan           | Need changes?1                |             |
| Need changes?1                | If                             | Check if monthly plan needs changes       | Montly plan approval            | Save monthly plan / Change request fields1 |             |
| Change request fields1         | Set                            | Prepare monthly plan change request data | Need changes?1 (False)          | Generate monthly plan          |             |
| Save monthly plan              | Supabase                       | Save approved monthly plan                | Need changes?1 (True)           | Get monthly plan              |             |
| Get weekly plan               | Supabase                       | Retrieve weekly plan from DB              | Has monthly plan? (True)        | Weekly plan aggregate          |             |
| Weekly plan aggregate          | Aggregate                    | Aggregate weekly plan data                 | Get weekly plan                 | Has weekly plan?               |             |
| Has weekly plan?              | If                           | Check if weekly plan exists                | Weekly plan aggregate           | Get posts from the past 7 days / Get instructions on the weekly plan |             |
| Get instructions on the weekly plan | Gmail (Webhook)           | Receive weekly plan instructions email    | Has weekly plan? (False)        | Generate weekly plan           |             |
| Generate weekly plan          | LangChain Info Extractor     | AI-generated weekly plan                   | Get instructions on the weekly plan, Weekly plan change fields | Weekly plan approval          |             |
| Weekly plan approval          | Gmail (Webhook)              | Wait for weekly plan approval email       | Generate weekly plan            | Changes to the weekly plan?    |             |
| Changes to the weekly plan?   | If                           | Check if weekly plan needs changes         | Weekly plan approval            | Save weekly plan / Weekly plan change fields |             |
| Weekly plan change fields     | Set                          | Prepare weekly plan change request data   | Changes to the weekly plan? (False) | Generate weekly plan           |             |
| Save weekly plan              | Supabase                     | Save approved weekly plan                  | Changes to the weekly plan? (True) | Get weekly plan              |             |
| Get posts from the past 7 days | Supabase                    | Retrieve posts from last 7 days            | Has weekly plan? (True)         | merge previous week's posts    |             |
| merge previous week's posts  | Aggregate                   | Aggregate recent posts for context         | Get posts from the past 7 days  | Generate new Instagram post    |             |
| Generate new Instagram post  | LangChain Info Extractor    | Generate AI Instagram post content          | merge previous week's posts, Add post change request fields | Generate unique id             |             |
| Generate unique id           | Crypto                      | Generate unique ID for post/image           | Generate new Instagram post     | Download the reference image   |             |
| Download the reference image | HTTP Request                | Download image as reference for generation | Generate unique id              | Generate image with OpenAI     |             |
| Generate image with OpenAI   | HTTP Request                | Generate AI image from prompt                | Download the reference image    | Convert response to image      |             |
| Convert response to image    | Convert to File             | Convert API response to image file           | Generate image with OpenAI      | Upload image to Supabase       |             |
| Upload image to Supabase     | HTTP Request                | Upload image to Supabase storage             | Convert response to image       | Add share link                |             |
| Add share link               | Set                         | Add shareable link to image                   | Upload image to Supabase        | Get approval for the post      |             |
| Get approval for the post    | Gmail (Webhook)             | Wait for post approval email                  | Add share link                 | Post approved?                |             |
| Post approved?               | If                          | Check if post was approved                     | Get approval for the post       | Save the post / Add post change request fields |             |
| Add post change request fields | Set                       | Prepare post change request data               | Post approved? (False)          | Generate new Instagram post    |             |
| Save the post                | Supabase                   | Store approved post in database                 | Post approved? (True)           | Facebook Graph API             |             |
| Facebook Graph API           | Facebook Graph API          | Publish Instagram post                          | Save the post                  | Facebook Graph API1            |             |
| Facebook Graph API1          | Facebook Graph API          | Additional Facebook API interaction (e.g., media) | Facebook Graph API           | None                          |             |
| Sticky Note (various)        | Sticky Note                | Documentation and reminders                     | None                         | None                          | Multiple notes scattered |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: "When clicking ‘Test workflow’"  
   - No parameters required  

2. **Create Set Node for Initial Configuration**  
   - Type: Set  
   - Name: "Configure workflow"  
   - Leave parameters empty (acts as placeholder)  
   - Connect input from Manual Trigger  

3. **Create Supabase Node to Get Quarterly Plan**  
   - Type: Supabase  
   - Name: "Get quarterly plan"  
   - Connect input from "Configure workflow"  
   - Configure with your Supabase credentials and query quarterly plan table  
   - Always output data  
 
4. **Create Aggregate Node to Process Quarterly Plan**  
   - Type: Aggregate  
   - Name: "Quaterly plan aggregate"  
   - Connect input from "Get quarterly plan"  
   - Configure aggregation to summarize data as needed  

5. **Create If Node to Check Quarterly Plan Existence**  
   - Type: If  
   - Name: "Has quarterly plan?"  
   - Connect input from "Quaterly plan aggregate"  
   - Condition: Check if aggregated data is not empty  
   - True branch connects to "Get monthly plan"  
   - False branch connects to Gmail webhook node (step 6)  

6. **Create Gmail Webhook Node to Receive Quarterly Plan Instructions**  
   - Type: Gmail (Webhook)  
   - Name: "Get instructions on the quarterly plan"  
   - Configure webhook to listen to your Gmail inbox for instructions  
   - Connect input from "Has quarterly plan?" False branch  
   - Output to AI generation node (step 7)  

7. **Create AI Generation Node for Quarterly Plan**  
   - Type: LangChain Information Extractor (or similar OpenAI node)  
   - Name: "Generate quarterly plan"  
   - Connect input from Gmail webhook and from any change request nodes  
   - Configure with OpenAI credentials and prompt for quarterly plan generation  
   - Output to Gmail webhook for approval (step 8)  

8. **Create Gmail Webhook Node for Quarterly Plan Approval**  
   - Type: Gmail (Webhook)  
   - Name: "Get approval on the quarterly plan"  
   - Configure webhook to listen for approval emails  
   - Connect input from AI generation node  
   - Output to If node for approval check (step 9)  

9. **Create If Node to Check Quarterly Plan Approval**  
   - Type: If  
   - Name: "Approve quarterly plan?"  
   - Connect input from approval Gmail webhook  
   - Condition: Check email content for approval status  
   - True branch to Supabase save node (step 10)  
   - False branch to Set node for change request (step 11)  

10. **Create Supabase Node to Save Quarterly Plan**  
    - Type: Supabase  
    - Name: "Save quarterly plan"  
    - Connect input from approval If node (True)  
    - Configure to save plan data to Supabase  
    - Output loops back to "Get quarterly plan" for updates  

11. **Create Set Node for Quarterly Plan Change Requests**  
    - Type: Set  
    - Name: "Change request fields"  
    - Connect input from approval If node (False)  
    - Configure fields to indicate requested changes  
    - Output to AI generation node for quarterly plan  

12. **Repeat Steps 3–11 for Monthly Plan**  
    - Create nodes: "Get monthly plan", "Monthly plan aggregate", "Has monthly plan?", "Get instructions on the monthly plan", "Generate monthly plan", "Montly plan approval", "Need changes?1", "Change request fields1", "Save monthly plan"  
    - Connect them in similar logic and configure accordingly  

13. **Repeat Steps 3–11 for Weekly Plan**  
    - Create nodes: "Get weekly plan", "Weekly plan aggregate", "Has weekly plan?", "Get instructions on the weekly plan", "Generate weekly plan", "Weekly plan approval", "Changes to the weekly plan?", "Weekly plan change fields", "Save weekly plan"  
    - Connect and configure similarly  

14. **Create Supabase Node to Get Posts From Past 7 Days**  
    - Type: Supabase  
    - Name: "Get posts from the past 7 days"  
    - Connect from "Has weekly plan?" (True branch)  
    - Configure to retrieve recent posts  

15. **Create Aggregate Node to Merge Previous Week's Posts**  
    - Type: Aggregate  
    - Name: "merge previous week's posts"  
    - Connect input from "Get posts from the past 7 days"  
    - Configure aggregation  

16. **Create AI Node to Generate New Instagram Post**  
    - Type: LangChain Information Extractor  
    - Name: "Generate new Instagram post"  
    - Connect input from aggregate node and any post change request sets  
    - Configure with OpenAI credentials and prompt  

17. **Create Crypto Node to Generate Unique ID**  
    - Type: Crypto  
    - Name: "Generate unique id"  
    - Connect input from AI post generation node  

18. **Create HTTP Request Node to Download Reference Image**  
    - Type: HTTP Request  
    - Name: "Download the reference image"  
    - Connect input from unique ID node  
    - Configure GET request to image URL from AI output  

19. **Create HTTP Request Node to Generate Image with OpenAI**  
    - Type: HTTP Request  
    - Name: "Generate image with OpenAI"  
    - Connect input from image download node  
    - Configure to call OpenAI image generation API (DALL·E)  

20. **Create Convert to File Node to Convert Response to Image**  
    - Type: Convert to File  
    - Name: "Convert response to image"  
    - Connect input from OpenAI image generation node  

21. **Create HTTP Request Node to Upload Image to Supabase**  
    - Type: HTTP Request  
    - Name: "Upload image to Supabase"  
    - Connect input from convert to file node  
    - Configure POST to Supabase storage with image and authentication  

22. **Create Set Node to Add Share Link**  
    - Type: Set  
    - Name: "Add share link"  
    - Connect input from upload node  
    - Set URL field from upload response  

23. **Create Gmail Webhook Node to Get Approval for Post**  
    - Type: Gmail (Webhook)  
    - Name: "Get approval for the post"  
    - Connect input from "Add share link"  

24. **Create If Node to Check Post Approval**  
    - Type: If  
    - Name: "Post approved?"  
    - Connect input from post approval Gmail webhook  
    - True branch to "Save the post"  
    - False branch to "Add post change request fields"  

25. **Create Set Node for Post Change Requests**  
    - Type: Set  
    - Name: "Add post change request fields"  
    - Connect input from If node (False)  
    - Configure fields to request post changes  
    - Output back to "Generate new Instagram post" node  

26. **Create Supabase Node to Save Post**  
    - Type: Supabase  
    - Name: "Save the post"  
    - Connect input from If node (True)  

27. **Create Facebook Graph API Nodes to Publish Post**  
    - Two nodes of type Facebook Graph API  
    - Names: "Facebook Graph API" and "Facebook Graph API1"  
    - Connect input from Supabase save node and chain output respectively  
    - Configure with Facebook/Instagram OAuth2 credentials and API parameters for posting content  

28. **Add Sticky Notes** to document steps or add reminders as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow enables fully automated Instagram content creation and posting with AI and APIs.   | Workflow purpose                                                                                 |
| Requires API credentials for OpenAI, Supabase, Gmail (IMAP/SMTP), and Facebook Graph API.       | Credential setup                                                                                |
| Supports iterative approval cycles for quarterly, monthly, weekly plans, and individual posts. | Workflow logic                                                                                  |
| Can be extended with integrations for Gumroad, PayPal for monetization (not included in this workflow). | Suggested expansion                                                                             |
| For Facebook Graph API setup see: https://developers.facebook.com/docs/graph-api                 | Official API documentation                                                                      |
| For OpenAI image and chat models API: https://platform.openai.com/docs/api-reference             | OpenAI API documentation                                                                       |
| For Supabase usage: https://supabase.com/docs                                                   | Supabase documentation                                                                          |

---

This structured documentation covers all nodes and their connections to help advanced users and AI agents understand, reproduce, and maintain the workflow effectively.