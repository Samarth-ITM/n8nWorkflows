Manage Google Cloud Storage with AI Image Generation using GPT-4 Mini

https://n8nworkflows.xyz/workflows/manage-google-cloud-storage-with-ai-image-generation-using-gpt-4-mini-7502


# Manage Google Cloud Storage with AI Image Generation using GPT-4 Mini

### 1. Workflow Overview

This workflow automates Google Cloud Storage (GCS) bucket and object management, enhanced with AI-powered image generation using OpenAI’s GPT-4 Mini model. It is designed primarily for beginners or developers interested in integrating AI with cloud storage operations.

**Target Use Cases:**  
- Automating creation, listing, and deletion of GCS buckets and objects.  
- Generating creative image prompts with AI and uploading generated images to GCS.  
- Demonstrating an end-to-end example combining cloud storage management with AI-driven content creation.

**Logical Blocks:**  
- **1.1 Workflow Trigger and Input Setup:** Manual start and setting input parameters like project ID and location.  
- **1.2 Google Cloud Storage Bucket Management:** Listing existing buckets, creating new buckets, and a no-operation placeholder.  
- **1.3 AI Prompt Generation and Image Creation:** Using OpenAI GPT-4 Mini to generate a creative prompt, then generating an image from that prompt.  
- **1.4 Object Management in GCS:** Uploading the generated image as an object to a bucket, and optionally deleting that object.  
- **1.5 Documentation and User Guidance:** Sticky notes providing explanations and instructions throughout the workflow.

---

### 2. Block-by-Block Analysis

#### 1.1 Workflow Trigger and Input Setup

**Overview:**  
This block initiates the workflow manually and sets essential input parameters such as the Google Cloud project ID and location for bucket creation.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Edit Fields

**Node Details:**  

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow upon manual execution.  
  - Config: No parameters; triggers on user action.  
  - Inputs: None  
  - Outputs: Connected to Edit Fields node.  
  - Failures: None expected; manual start.

- **Edit Fields**  
  - Type: Set node  
  - Role: Sets workflow variables `project_id` (e.g., "n8n-project-467007") and `location` (e.g., "asia-southeast1").  
  - Config: Assignments are hardcoded string values for these two variables.  
  - Inputs: From manual trigger  
  - Outputs: To ‘Get a list of Buckets for a given project’ and ‘Create a new Bucket’ nodes.  
  - Failures: Misconfiguration of project ID or location can cause downstream API errors.

---

#### 1.2 Google Cloud Storage Bucket Management

**Overview:**  
Lists all existing buckets in the specified project and optionally creates a new bucket if needed.

**Nodes Involved:**  
- Get a list of Buckets for a given project  
- Create a new Bucket  
- No Operation, do nothing

**Node Details:**  

- **Get a list of Buckets for a given project**  
  - Type: Google Cloud Storage node  
  - Role: Retrieves all buckets for the provided project ID.  
  - Config: `projectId` dynamically set from `project_id` variable. Operation: list buckets.  
  - Inputs: From Edit Fields  
  - Outputs: To No Operation node (acts as endpoint for listing flow).  
  - Failures: API auth errors, project ID invalid, network issues.

- **Create a new Bucket**  
  - Type: Google Cloud Storage node  
  - Role: Creates a new bucket within the specified project and location.  
  - Config:  
    - Operation: create bucket  
    - `projectId` from `project_id` variable  
    - Bucket name generated dynamically using workflow ID and current time (format: `bucket{workflowID}{hhmm}`) to ensure uniqueness.  
    - Location set from `location` variable inside `dataLocations` parameter.  
  - Inputs: From Edit Fields  
  - Outputs: To Prompt Generation Agent node.  
  - Failures: Bucket name conflicts, insufficient permissions, invalid location.

- **No Operation, do nothing**  
  - Type: NoOp node  
  - Role: Terminates the bucket listing branch cleanly without further processing.  
  - Inputs: From bucket listing node  
  - Outputs: None  
  - Failures: None.

---

#### 1.3 AI Prompt Generation and Image Creation

**Overview:**  
Generates a creative image prompt using an AI agent, then creates an image based on that prompt using OpenAI's image generation capabilities.

**Nodes Involved:**  
- OpenAI Chat Model  
- Prompt Generation Agent  
- Generate an image

**Node Details:**  

- **OpenAI Chat Model**  
  - Type: Langchain OpenAI Chat Model node  
  - Role: Processes AI chat prompts using GPT-4 Mini.  
  - Config: Model set to `gpt-4.1-mini`, no additional options.  
  - Credentials: OpenAI API Key.  
  - Inputs: From Create a new Bucket node (via ai_languageModel connection).  
  - Outputs: To Prompt Generation Agent node (ai_languageModel input).  
  - Failures: Auth errors, rate limits, invalid API key, model availability.

- **Prompt Generation Agent**  
  - Type: Langchain Agent node  
  - Role: Generates an AI prompt for image creation based on a fixed instruction text.  
  - Config:  
    - Text instructs agent to create a prompt for the OpenAI image generation model with a random theme chosen from Education, Science, Sport, Economy, Health.  
    - Output: prompt text only.  
  - Inputs: From OpenAI Chat Model (or Create a new Bucket for main input).  
  - Outputs: To Generate an image node.  
  - Failures: Expression errors, AI model errors.

- **Generate an image**  
  - Type: Langchain OpenAI Image Generation node  
  - Role: Generates an image based on the prompt text from the AI agent.  
  - Config: Prompt dynamically set to the AI-generated prompt (`{{$json.output}}`).  
  - Credentials: OpenAI API Key.  
  - Inputs: From Prompt Generation Agent  
  - Outputs: To Create an object node.  
  - Failures: API errors, prompt invalidity, image generation limits.

---

#### 1.4 Object Management in Google Cloud Storage

**Overview:**  
Uploads the AI-generated image into the newly created bucket as an object and optionally deletes the object afterward.

**Nodes Involved:**  
- Create an object  
- Delete an object from a bucket

**Node Details:**  

- **Create an object**  
  - Type: Google Cloud Storage node  
  - Role: Uploads a new object into the created bucket.  
  - Config:  
    - Resource: object  
    - Operation: create  
    - Bucket name: dynamically set from the output of "Create a new Bucket" node (`{{$node["Create a new Bucket"].json.name}}`)  
    - Object name: dynamically generated using current date and file extension (`object_{{ $now.format('yyyyMMdd') }}.{{ $json.fileExtension }}`)  
    - Data to upload is empty in node parameters, but expected to be image data from previous step (workflow likely passes image content).  
  - Inputs: From Generate an image node  
  - Outputs: To Delete an object from a bucket node.  
  - Failures: Upload errors, permission issues, bucket not found.

- **Delete an object from a bucket**  
  - Type: Google Cloud Storage node  
  - Role: Deletes the uploaded object from the specified bucket (optional cleanup).  
  - Config:  
    - Resource: object  
    - Operation: delete  
    - Bucket name and object name taken dynamically from previous nodes (`{{$json.bucket}}`, `{{$json.name}}`).  
  - Inputs: From Create an object node  
  - Outputs: None  
  - Failures: Object not found, permission denied, network errors.

---

#### 1.5 Documentation and User Guidance

**Overview:**  
Sticky notes provide detailed explanations, instructions, and contextual information for users operating or modifying the workflow.

**Nodes Involved:**  
- Sticky Note (large, detailed)  
- Sticky Note1 through Sticky Note7

**Node Details:**  

- Sticky Notes contain:  
  - Workflow purpose, audience, and detailed step-by-step explanation.  
  - Setup instructions, requirements, and customization ideas.  
  - Step-specific guidance aligned with functional blocks (e.g., starting the workflow, providing inputs, bucket listing, AI prompt generation, image upload, and cleanup).  
  - Links and formatting preserved for clarity.  
  - They do not affect workflow logic but serve as embedded documentation.

---

### 3. Summary Table

| Node Name                         | Node Type                                      | Functional Role                                  | Input Node(s)                         | Output Node(s)                          | Sticky Note                                                        |
|----------------------------------|------------------------------------------------|-------------------------------------------------|-------------------------------------|----------------------------------------|-------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                                | Starts the workflow manually                      | None                                | Edit Fields                            | Sticky Note1: "### 1. Start the Workflow..."                      |
| Edit Fields                      | Set                                            | Sets project ID and location input variables     | When clicking ‘Execute workflow’    | Get a list of Buckets, Create a new Bucket | Sticky Note2: "### 2. Provide Input Fields..."                    |
| Get a list of Buckets for a given project | Google Cloud Storage                         | Lists all buckets in the specified project       | Edit Fields                        | No Operation                         | Sticky Note3: "### 3. List all the current Google Cloud Storage buckets" |
| No Operation, do nothing          | NoOp                                           | Ends the list buckets branch                      | Get a list of Buckets               | None                                 |                                                                   |
| Create a new Bucket               | Google Cloud Storage                            | Creates a new bucket with dynamic name and location | Edit Fields                      | Prompt Generation Agent               | Sticky Note4: "### 4. Create a New Bucket"                        |
| OpenAI Chat Model                | Langchain OpenAI Chat Model                     | Runs GPT-4 Mini to support prompt generation      | Create a new Bucket                | Prompt Generation Agent (ai_languageModel) | Sticky Note5: "### 5. Generate an AI Prompt..."                   |
| Prompt Generation Agent          | Langchain Agent                                 | Generates a creative prompt for image generation  | OpenAI Chat Model, Create a new Bucket | Generate an image                   | Sticky Note5                                                    |
| Generate an image                | Langchain OpenAI Image Generation               | Creates image from AI-generated prompt            | Prompt Generation Agent            | Create an object                     | Sticky Note5                                                    |
| Create an object                 | Google Cloud Storage                            | Uploads generated image as an object to bucket    | Generate an image                  | Delete an object from a bucket       | Sticky Note6: "### 7. Upload the Image to a Bucket"               |
| Delete an object from a bucket   | Google Cloud Storage                            | Deletes the uploaded object (optional cleanup)    | Create an object                   | None                               | Sticky Note7: "### 8. Delete the Object (Optional)"               |
| Sticky Note                     | Sticky Note                                    | Provides detailed documentation and instructions | None                             | None                               | Large main workflow overview note                                |
| Sticky Note1                    | Sticky Note                                    | Explains manual workflow start                     | None                             | None                               | Covers manual trigger step                                        |
| Sticky Note2                    | Sticky Note                                    | Explains input field setup                          | None                             | None                               | Covers Edit Fields node                                          |
| Sticky Note3                    | Sticky Note                                    | Explains bucket listing                             | None                             | None                               | Covers Get a list of Buckets node                                |
| Sticky Note4                    | Sticky Note                                    | Explains bucket creation                            | None                             | None                               | Covers Create a new Bucket node                                  |
| Sticky Note5                    | Sticky Note                                    | Explains AI prompt generation and image creation  | None                             | None                               | Covers OpenAI Chat Model, Prompt Generation Agent, Generate an image |
| Sticky Note6                    | Sticky Note                                    | Explains image upload to bucket                     | None                             | None                               | Covers Create an object node                                    |
| Sticky Note7                    | Sticky Note                                    | Explains optional object deletion                   | None                             | None                               | Covers Delete an object node                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: "When clicking ‘Execute workflow’"  
   - Type: Manual Trigger (no parameters)  
   - Purpose: To start the workflow manually.

2. **Create a Set Node to Define Input Fields**  
   - Name: "Edit Fields"  
   - Type: Set  
   - Parameters:  
     - Add string field `project_id` with value e.g., `"n8n-project-467007"`  
     - Add string field `location` with value e.g., `"asia-southeast1"`  
   - Connect output from Manual Trigger to this node.

3. **Create Google Cloud Storage Node to List Buckets**  
   - Name: "Get a list of Buckets for a given project"  
   - Type: Google Cloud Storage  
   - Operation: List buckets  
   - Set `projectId` to expression referencing `project_id` from previous node (`={{ $json.project_id }}`)  
   - Connect output from "Edit Fields" node.  
   - Credentials: Configure and select valid Google Cloud Storage OAuth2 credentials.

4. **Create a No Operation Node to End List Buckets Flow**  
   - Name: "No Operation, do nothing"  
   - Type: NoOp  
   - Connect output from "Get a list of Buckets for a given project".

5. **Create Google Cloud Storage Node to Create a New Bucket**  
   - Name: "Create a new Bucket"  
   - Type: Google Cloud Storage  
   - Operation: Create bucket  
   - Set `projectId` as `={{ $json.project_id }}`  
   - Set `bucketName` dynamically with expression: `"bucket" + $workflow.id.toLowerCase() + $now.format('hhmm')` (to ensure uniqueness)  
   - Set `dataLocations` (location) as an array with `[location]`, expression: `=[{{ $json.location }}]`  
   - Connect output from "Edit Fields" node.  
   - Credentials: Use same Google Cloud Storage OAuth2 credentials.

6. **Create Langchain OpenAI Chat Model Node**  
   - Name: "OpenAI Chat Model"  
   - Type: Langchain OpenAI Chat Model  
   - Model: Select `gpt-4.1-mini` from the list  
   - Credentials: Provide OpenAI API key credential  
   - Connect output (ai_languageModel) from "Create a new Bucket" node.

7. **Create Langchain Agent Node for Prompt Generation**  
   - Name: "Prompt Generation Agent"  
   - Type: Langchain Agent  
   - Prompt Type: Define  
   - Text:  
     ```
     You're prompt generator agent, you will create prompt for the open-ai image generation model with a random topic among:
     - Education
     - Science 
     - Sport
     - Economy
     - Health
     Output the prompt only, nothing else
     ```  
   - Connect ai_languageModel input from "OpenAI Chat Model" node  
   - Also connect main input from "Create a new Bucket" to allow for branching if needed.

8. **Create Langchain OpenAI Image Generation Node**  
   - Name: "Generate an image"  
   - Type: Langchain OpenAI Image Generation  
   - Prompt: Set to dynamic expression referencing output of "Prompt Generation Agent": `={{ $json.output }}`  
   - Credentials: OpenAI API key  
   - Connect output from "Prompt Generation Agent" node.

9. **Create Google Cloud Storage Node to Upload Object**  
   - Name: "Create an object"  
   - Type: Google Cloud Storage  
   - Resource: Object  
   - Operation: Create  
   - Bucket Name: Expression pulling bucket name from "Create a new Bucket" node: `={{ $('Create a new Bucket').item.json.name }}`  
   - Object Name: Expression generating name with date and extension: `=object_{{ $now.format('yyyyMMdd') }}.{{ $json.fileExtension }}`  
   - Upload data: Map image content from previous node output (ensure image binary data is passed here)  
   - Credentials: Google Cloud Storage OAuth2  
   - Connect output from "Generate an image" node.

10. **Create Google Cloud Storage Node to Delete Object (Optional)**  
    - Name: "Delete an object from a bucket"  
    - Type: Google Cloud Storage  
    - Resource: Object  
    - Operation: Delete  
    - Bucket Name: From previous node output, `={{ $json.bucket }}`  
    - Object Name: From previous node output, `={{ $json.name }}`  
    - Credentials: Google Cloud Storage OAuth2  
    - Connect output from "Create an object" node.

11. **Add Sticky Notes for Documentation**  
    - Add sticky notes with content describing each step, matching the content and placement from the original workflow for user guidance.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                                                     |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| This workflow is a beginner-friendly demonstration combining Google Cloud Storage bucket/object management with AI-powered prompt and image generation using OpenAI’s GPT-4 Mini model. It illustrates the full lifecycle: listing buckets, creating buckets, generating AI prompts, generating images, uploading images as objects, and optionally deleting objects. | Main workflow purpose and overview.                                                                                               |
| Requirements include a Google Cloud account with Cloud Storage API enabled, a service account key configured as OAuth2 credentials in n8n, and an OpenAI API key configured for AI nodes.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Setup prerequisites.                                                                                                               |
| Customization ideas: use different object types (PDF, logs), schedule triggers instead of manual, accept user input dynamically, manage multiple buckets/projects, or add notification steps (e.g., Slack or Email).                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Suggestions for extending or adapting the workflow.                                                                               |
| Workflow uses the Langchain n8n nodes for OpenAI integration — ensure these nodes are installed and updated to compatible versions (Chat Model v1.2+, Agent v2.1+, Image v1.8+).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Version-specific notes for AI nodes.                                                                                              |
| For best results, ensure bucket names are globally unique in GCS; the workflow uses a dynamic name with workflow ID and timestamp to avoid collisions.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Bucket naming considerations.                                                                                                    |
| The prompt generation agent creates randomized thematic prompts, enhancing creativity for image generation. This approach can be replaced or extended to accept user inputs for dynamic content creation.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | AI prompt logic explanation.                                                                                                     |

---

**Disclaimer:** The text above is derived exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.