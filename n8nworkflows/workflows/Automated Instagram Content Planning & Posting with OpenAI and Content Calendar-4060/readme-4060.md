Automated Instagram Content Planning & Posting with OpenAI and Content Calendar

https://n8nworkflows.xyz/workflows/automated-instagram-content-planning---posting-with-openai-and-content-calendar-4060


# Automated Instagram Content Planning & Posting with OpenAI and Content Calendar

### 1. Workflow Overview

This workflow automates Instagram content planning and posting using AI-driven content generation, approval processes, and integration with external tools such as Supabase, Gmail, and Facebook Graph API. It targets entrepreneurs, freelancers, and agencies aiming to scale social media management by automating repetitive tasks related to content creation, approval, and publishing.

The workflow is logically divided into four main blocks:

- **1.1 Quarterly Content Planning**: Fetches, generates, approves, and saves quarterly social media plans.
- **1.2 Monthly Content Planning**: Handles monthly plan retrieval, generation, approval, and saving.
- **1.3 Weekly Content Planning**: Manages weekly plans similarly, including change requests.
- **1.4 Instagram Post Generation & Publishing**: Generates individual Instagram posts using AI, manages image creation and upload, approval workflows, and publishes to Facebook/Instagram via API.

Each block interacts with Supabase for data storage, Gmail for communication and approvals, and OpenAI (via LangChain nodes) for AI-powered content generation.

---

### 2. Block-by-Block Analysis

#### 2.1 Quarterly Content Planning

- **Overview**:  
  This block retrieves the quarterly content plan from the database, generates new plans if needed using AI, manages approval via email, and saves approved plans back to Supabase.

- **Nodes Involved**:  
  - `Configure workflow`  
  - `Get quarterly plan`  
  - `Quaterly plan aggregate`  
  - `Has quarterly plan?`  
  - `Get instructions on the quarterly plan`  
  - `Generate quarterly plan`  
  - `Get approval on the quarterly plan`  
  - `Approve quarterly plan?`  
  - `Change request fields`  
  - `Save quarterly plan`  
  - `Sticky Note`, `Sticky Note5`, `Sticky Note7`, `Sticky Note8`, `Sticky Note9` (for documentation)

- **Node Details**:

  - **Configure workflow**  
    - Type: Set node  
    - Role: Initializes or configures starting parameters for the workflow.  
    - Output: Connects to `Get quarterly plan`.  
    - Edge cases: Empty or incorrect initial parameters may cause downstream logic to fail.

  - **Get quarterly plan**  
    - Type: Supabase node  
    - Role: Queries Supabase to fetch the current quarterly plan data.  
    - Always outputs data even if empty, ensuring downstream nodes can handle missing plans.  
    - Outputs to aggregation and conditional check nodes.

  - **Quaterly plan aggregate**  
    - Type: Aggregate node  
    - Role: Aggregates or processes the quarterly plan data fetched to a usable form for decision-making.

  - **Has quarterly plan?**  
    - Type: If node  
    - Role: Branches workflow logic depending on presence or absence of a quarterly plan.  
    - If no plan: triggers instruction retrieval; if yes, proceeds to monthly plan fetching.

  - **Get instructions on the quarterly plan**  
    - Type: Gmail node with webhook  
    - Role: Listens for or sends emails containing instructions related to the quarterly plan.  
    - Handles human input or approval instructions.

  - **Generate quarterly plan**  
    - Type: LangChain Information Extractor (OpenAI)  
    - Role: Generates a quarterly plan based on instructions and previous data.  
    - Depends on AI model credentials and prompt configuration.

  - **Get approval on the quarterly plan**  
    - Type: Gmail node with webhook  
    - Role: Sends or receives approval emails for the quarterly plan.

  - **Approve quarterly plan?**  
    - Type: If node  
    - Role: Decides if the quarterly plan is approved or requires changes.  
    - If approved: saves plan; if not, triggers change request.

  - **Change request fields**  
    - Type: Set node  
    - Role: Modifies or resets fields to handle requested changes and loops back to generation.

  - **Save quarterly plan**  
    - Type: Supabase node  
    - Role: Saves the approved quarterly plan to the database.

- **Edge Cases & Failures**:  
  - Supabase query failures or empty datasets.  
  - Gmail webhook delays or missing approval emails.  
  - OpenAI API timeouts or generation errors.  
  - Approval rejection loops causing infinite cycles if not handled externally.

---

#### 2.2 Monthly Content Planning

- **Overview**:  
  This block retrieves, generates, and approves monthly content plans, following a similar process to the quarterly planning block.

- **Nodes Involved**:  
  - `Get monthly plan`  
  - `Monthly plan aggregate`  
  - `Has monthly plan?`  
  - `Get instructions on the monthly plan`  
  - `Generate monthly plan`  
  - `Montly plan approval`  
  - `Need changes?1`  
  - `Change request fields1`  
  - `Save monthly plan`

- **Node Details**:

  - **Get monthly plan**  
    - Supabase: Fetch monthly plans; always outputs data.

  - **Monthly plan aggregate**  
    - Aggregates monthly plan data for decision-making.

  - **Has monthly plan?**  
    - If node: Checks existence.

  - **Get instructions on the monthly plan**  
    - Gmail with webhook: Manages email instructions.

  - **Generate monthly plan**  
    - LangChain: AI-driven plan generation.

  - **Montly plan approval**  
    - Gmail with webhook: Manages approval emails.

  - **Need changes?1**  
    - If node: Branches on approval or change requests.

  - **Change request fields1**  
    - Set node: Adjusts fields for change loop.

  - **Save monthly plan**  
    - Supabase: Saves approved monthly plan.

- **Edge Cases & Failures**:  
  - Similar to quarterly block: database, email, or AI failures.  
  - Approval rejection loops.

---

#### 2.3 Weekly Content Planning

- **Overview**:  
  This block manages weekly content plans, including data retrieval, generation, approval, and handling change requests.

- **Nodes Involved**:  
  - `Get weekly plan`  
  - `Weekly plan aggregate`  
  - `Has weekly plan?`  
  - `Get instructions on the weekly plan`  
  - `Generate weekly plan`  
  - `Weekly plan approval`  
  - `Changes to the weekly plan?`  
  - `Weekly plan change fields`  
  - `Save weekly plan`

- **Node Details**:

  - **Get weekly plan**  
    - Supabase: Fetch weekly plans.

  - **Weekly plan aggregate**  
    - Aggregates data.

  - **Has weekly plan?**  
    - If node: Checks presence.

  - **Get instructions on the weekly plan**  
    - Gmail webhook node: Instructions handling.

  - **Generate weekly plan**  
    - LangChain AI node: Generates plan.

  - **Weekly plan approval**  
    - Gmail webhook node: Handles approval.

  - **Changes to the weekly plan?**  
    - If node: Determines if changes requested.

  - **Weekly plan change fields**  
    - Set node: Adjusts fields for change processing.

  - **Save weekly plan**  
    - Supabase: Saves weekly plan.

- **Edge Cases & Failures**:  
  - Same as monthly and quarterly blocks.  
  - Potential for loops if approval never granted.

---

#### 2.4 Instagram Post Generation & Publishing

- **Overview**:  
  This block generates individual Instagram posts using AI, creates images, uploads media to Supabase, requests final approval via email, and posts approved content to Instagram via Facebook Graph API.

- **Nodes Involved**:  
  - `Get posts from the past 7 days`  
  - `merge previous week's posts`  
  - `Generate new Instagram post`  
  - `Generate unique id`  
  - `Download the reference image`  
  - `Generate image with OpenAI`  
  - `Convert response to image`  
  - `Upload image to Supabase`  
  - `Add share link`  
  - `Get approval for the post`  
  - `Post approved?`  
  - `Add post change request fields`  
  - `Save the post`  
  - `Facebook Graph API`  
  - `Facebook Graph API1`

- **Node Details**:

  - **Get posts from the past 7 days**  
    - Supabase: Retrieves recent posts for context in generation.

  - **merge previous week's posts**  
    - Aggregate node: Combines recent posts data.

  - **Generate new Instagram post**  
    - LangChain Information Extractor: Creates post content.

  - **Generate unique id**  
    - Crypto node: Produces unique identifier for media or post.

  - **Download the reference image**  
    - HTTP Request: Downloads reference image—likely for image generation context.

  - **Generate image with OpenAI**  
    - HTTP Request: Calls OpenAI API for image generation.

  - **Convert response to image**  
    - Convert To File node: Converts API response to image file for upload.

  - **Upload image to Supabase**  
    - HTTP Request: Uploads generated image to Supabase storage.

  - **Add share link**  
    - Set node: Adds accessible link to uploaded image.

  - **Get approval for the post**  
    - Gmail webhook: Requests human approval for the post.

  - **Post approved?**  
    - If node: Branches depending on approval or request for changes.

  - **Add post change request fields**  
    - Set node: Handles post change requests leading to re-generation.

  - **Save the post**  
    - Supabase: Saves approved post data.

  - **Facebook Graph API & Facebook Graph API1**  
    - Facebook Graph API nodes: Post content and media to Instagram via Facebook API.

- **Edge Cases & Failures**:  
  - API rate limits or failures (OpenAI, Facebook).  
  - Image generation errors or invalid files.  
  - Approval delays or missing responses.  
  - Upload failures to Supabase.

---

### 3. Summary Table

| Node Name                      | Node Type                          | Functional Role                         | Input Node(s)                | Output Node(s)                       | Sticky Note                                      |
|--------------------------------|----------------------------------|---------------------------------------|-----------------------------|------------------------------------|-------------------------------------------------|
| When clicking ‘Test workflow’   | Manual Trigger                   | Workflow entry point                   | -                           | Configure workflow                 |                                                 |
| Configure workflow              | Set                             | Workflow configuration start          | When clicking ‘Test workflow’| Get quarterly plan                 |                                                 |
| Get quarterly plan             | Supabase                        | Fetch quarterly plan data              | Configure workflow           | Quaterly plan aggregate, Has quarterly plan? |                                                 |
| Quaterly plan aggregate         | Aggregate                      | Aggregate quarterly plan data          | Get quarterly plan           | Has quarterly plan?                |                                                 |
| Has quarterly plan?             | If                             | Check if quarterly plan exists         | Quaterly plan aggregate      | Get monthly plan or Get instructions on quarterly plan |                                                 |
| Get instructions on the quarterly plan | Gmail (webhook)              | Receive/send email instructions        | Has quarterly plan?          | Generate quarterly plan            |                                                 |
| Generate quarterly plan         | LangChain Information Extractor | AI generates quarterly plan            | Get instructions on quarterly plan, Change request fields1, Change request fields | Get approval on the quarterly plan              |                                                 |
| Get approval on the quarterly plan | Gmail (webhook)              | Manage email approval process           | Generate quarterly plan      | Approve quarterly plan?            |                                                 |
| Approve quarterly plan?         | If                             | Branch on approval status               | Get approval on the quarterly plan | Save quarterly plan or Change request fields |                                                 |
| Change request fields           | Set                             | Prepare data for change request loop   | Approve quarterly plan?      | Generate quarterly plan            |                                                 |
| Save quarterly plan             | Supabase                        | Save approved quarterly plan           | Approve quarterly plan?      | Get quarterly plan                 |                                                 |
| Get monthly plan               | Supabase                        | Fetch monthly plan data                 | Has quarterly plan?          | Monthly plan aggregate, Has monthly plan? |                                                 |
| Monthly plan aggregate          | Aggregate                      | Aggregate monthly plan data             | Get monthly plan             | Has monthly plan?                  |                                                 |
| Has monthly plan?               | If                             | Check if monthly plan exists            | Monthly plan aggregate       | Get weekly plan or Get instructions on monthly plan |                                                 |
| Get instructions on the monthly plan | Gmail (webhook)              | Receive/send email instructions        | Has monthly plan?            | Generate monthly plan             |                                                 |
| Generate monthly plan           | LangChain Information Extractor | AI generates monthly plan               | Get instructions on monthly plan, Change request fields1 | Montly plan approval                 |                                                 |
| Montly plan approval            | Gmail (webhook)                | Manage monthly plan approval            | Generate monthly plan        | Need changes?1                    |                                                 |
| Need changes?1                  | If                             | Branch on monthly plan approval status | Montly plan approval         | Save monthly plan or Change request fields1 |                                                 |
| Change request fields1          | Set                             | Prepare data for monthly change request| Need changes?1               | Generate monthly plan             |                                                 |
| Save monthly plan               | Supabase                        | Save approved monthly plan              | Need changes?1               | Get monthly plan                  |                                                 |
| Get weekly plan                | Supabase                        | Fetch weekly plan data                  | Has monthly plan?            | Weekly plan aggregate, Has weekly plan? |                                                 |
| Weekly plan aggregate           | Aggregate                      | Aggregate weekly plan data              | Get weekly plan              | Has weekly plan?                  |                                                 |
| Has weekly plan?                | If                             | Check if weekly plan exists             | Weekly plan aggregate        | Get posts from the past 7 days or Get instructions on the weekly plan |                                                 |
| Get posts from the past 7 days | Supabase                        | Retrieve recent posts                   | Has weekly plan?             | merge previous week's posts       |                                                 |
| merge previous week's posts     | Aggregate                      | Aggregate recent posts                  | Get posts from the past 7 days | Generate new Instagram post       |                                                 |
| Get instructions on the weekly plan | Gmail (webhook)              | Receive/send instructions               | Has weekly plan?             | Generate weekly plan              |                                                 |
| Generate weekly plan            | LangChain Information Extractor | AI generates weekly plan                | Get instructions on weekly plan, Weekly plan change fields | Weekly plan approval             |                                                 |
| Weekly plan approval            | Gmail (webhook)                | Manage weekly plan approval             | Generate weekly plan         | Changes to the weekly plan?       |                                                 |
| Changes to the weekly plan?     | If                             | Branch on weekly plan approval status  | Weekly plan approval         | Save weekly plan or Weekly plan change fields |                                                 |
| Weekly plan change fields       | Set                             | Prepare data for weekly change requests| Changes to the weekly plan?  | Generate weekly plan              |                                                 |
| Save weekly plan               | Supabase                        | Save approved weekly plan               | Changes to the weekly plan?  | Get weekly plan                  |                                                 |
| Generate new Instagram post     | LangChain Information Extractor | AI generates Instagram post content    | merge previous week's posts  | Generate unique id                |                                                 |
| Generate unique id              | Crypto                         | Generates unique ID for media           | Generate new Instagram post  | Download the reference image      |                                                 |
| Download the reference image    | HTTP Request                   | Downloads image reference for generation| Generate unique id           | Generate image with OpenAI        |                                                 |
| Generate image with OpenAI      | HTTP Request                   | Calls OpenAI API to generate image     | Download the reference image | Convert response to image         |                                                 |
| Convert response to image       | Convert To File                | Converts API response to image file    | Generate image with OpenAI   | Upload image to Supabase          |                                                 |
| Upload image to Supabase        | HTTP Request                   | Uploads image file to Supabase storage | Convert response to image    | Add share link                   |                                                 |
| Add share link                 | Set                             | Adds shareable URL to post data         | Upload image to Supabase     | Get approval for the post         |                                                 |
| Get approval for the post       | Gmail (webhook)                | Requests human approval for post       | Add share link               | Post approved?                   |                                                 |
| Post approved?                 | If                             | Branch on post approval status          | Get approval for the post    | Save the post or Add post change request fields |                                                 |
| Add post change request fields | Set                             | Prepares data for post change requests | Post approved? (No branch)   | Generate new Instagram post       |                                                 |
| Save the post                 | Supabase                        | Saves approved post data                 | Post approved? (Yes branch)  | Facebook Graph API               |                                                 |
| Facebook Graph API             | Facebook Graph API             | Publishes post to Instagram              | Save the post                | Facebook Graph API1              |                                                 |
| Facebook Graph API1            | Facebook Graph API             | Secondary Facebook Graph API call       | Facebook Graph API           | -                              |                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger**  
   - Type: Manual Trigger  
   - Name: `When clicking ‘Test workflow’`  
   - Purpose: Entry point to run the workflow manually.

2. **Configure Workflow Node**  
   - Type: Set  
   - Name: `Configure workflow`  
   - Purpose: Initialize basic parameters or variables.  
   - Connect: From `When clicking ‘Test workflow’`.

3. **Quarterly Content Planning**  
   - Create Supabase node: `Get quarterly plan`  
     - Configure to query quarterly plans.  
     - Set to always output data to avoid empty errors.  
   - Aggregate node: `Quaterly plan aggregate`  
     - Aggregate query results as needed.  
   - If node: `Has quarterly plan?`  
     - Condition: Check if quarterly plan exists.  
     - True branch: Connect to `Get monthly plan`.  
     - False branch: Connect to `Get instructions on the quarterly plan`.  
   - Gmail node (webhook): `Get instructions on the quarterly plan`  
     - Configure Gmail OAuth2 credentials.  
     - Set webhook to listen for incoming instructions.  
   - LangChain node: `Generate quarterly plan`  
     - Configure OpenAI API credentials.  
     - Set model and prompt for extracting quarterly plan.  
   - Gmail node (webhook): `Get approval on the quarterly plan`  
     - Setup for sending/receiving approval email.  
   - If node: `Approve quarterly plan?`  
     - Condition: Check approval status.  
     - True: Connect to `Save quarterly plan`.  
     - False: Connect to `Change request fields`.  
   - Set node: `Change request fields`  
     - Prepare data for re-generation.  
   - Supabase node: `Save quarterly plan`  
     - Configure to save approved plan.

4. **Monthly Content Planning**  
   - Supabase node: `Get monthly plan`  
   - Aggregate node: `Monthly plan aggregate`  
   - If node: `Has monthly plan?`  
     - True: Connect to `Get weekly plan`.  
     - False: Connect to `Get instructions on the monthly plan`.  
   - Gmail node (webhook): `Get instructions on the monthly plan`  
   - LangChain node: `Generate monthly plan`  
   - Gmail node (webhook): `Montly plan approval`  
   - If node: `Need changes?1`  
     - True: Connect to `Change request fields1`.  
     - False: Connect to `Save monthly plan`.  
   - Set node: `Change request fields1`  
   - Supabase node: `Save monthly plan`

5. **Weekly Content Planning**  
   - Supabase node: `Get weekly plan`  
   - Aggregate node: `Weekly plan aggregate`  
   - If node: `Has weekly plan?`  
     - True: Connect to `Get posts from the past 7 days`.  
     - False: Connect to `Get instructions on the weekly plan`.  
   - Supabase node: `Get posts from the past 7 days`  
   - Aggregate node: `merge previous week's posts`  
   - Gmail node (webhook): `Get instructions on the weekly plan`  
   - LangChain node: `Generate weekly plan`  
   - Gmail node (webhook): `Weekly plan approval`  
   - If node: `Changes to the weekly plan?`  
     - True: Connect to `Weekly plan change fields` then back to `Generate weekly plan`.  
     - False: Connect to `Save weekly plan`.  
   - Set node: `Weekly plan change fields`  
   - Supabase node: `Save weekly plan`

6. **Instagram Post Generation & Publishing**  
   - LangChain node: `Generate new Instagram post`  
     - Connected from `merge previous week's posts`.  
   - Crypto node: `Generate unique id`  
   - HTTP Request node: `Download the reference image`  
   - HTTP Request node: `Generate image with OpenAI`  
     - Configure OpenAI image generation API.  
   - Convert To File node: `Convert response to image`  
   - HTTP Request node: `Upload image to Supabase`  
     - Configure API endpoint and credentials.  
   - Set node: `Add share link`  
   - Gmail node (webhook): `Get approval for the post`  
   - If node: `Post approved?`  
     - True: Connect to `Save the post`.  
     - False: Connect to `Add post change request fields`.  
   - Set node: `Add post change request fields`  
     - Connect back to `Generate new Instagram post` for regeneration.  
   - Supabase node: `Save the post`  
   - Facebook Graph API node: `Facebook Graph API`  
     - Configure Facebook OAuth credentials and Instagram account.  
   - Facebook Graph API node: `Facebook Graph API1`  
     - Secondary call for publishing or confirmation.

7. **Credential Setup**  
   - Add OpenAI credentials for LangChain and HTTP Request nodes.  
   - Add Gmail OAuth2 credentials for email nodes.  
   - Add Supabase API credentials (URL and keys).  
   - Add Facebook Graph API OAuth2 credentials.

8. **Workflow Testing**  
   - Use manual trigger to test.  
   - Verify each email webhook receives/sends emails properly.  
   - Confirm data saved and retrieved from Supabase.  
   - Confirm image generation and upload works.  
   - Confirm post publishes to Instagram via Facebook API.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow integrates OpenAI's GPT model via LangChain nodes for content generation at multiple plan levels. | n8n LangChain documentation: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-langchain/ |
| Gmail nodes use webhooks for real-time email handling and approvals.                                             | Gmail API: https://developers.google.com/gmail/api                                                       |
| Supabase is used as the primary backend database and file storage system.                                        | Supabase Docs: https://supabase.com/docs                                                       |
| Facebook Graph API nodes handle Instagram posting using OAuth2 credentials.                                      | Facebook Graph API: https://developers.facebook.com/docs/graph-api/                              |
| To avoid infinite loops on approvals, external human intervention or timeout mechanisms should be considered.   |                                                                                                   |
| The workflow is designed to automate up to 80% of social media content planning and posting tasks.               |                                                                                                   |

---

**Disclaimer:**  
The provided text and workflow are exclusively generated from an automated n8n workflow. All content complies with applicable content policies and contains no illegal or protected elements. All data processed is lawful and publicly accessible.