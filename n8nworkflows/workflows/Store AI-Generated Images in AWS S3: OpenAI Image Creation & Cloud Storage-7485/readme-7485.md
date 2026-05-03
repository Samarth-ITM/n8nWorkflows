Store AI-Generated Images in AWS S3: OpenAI Image Creation & Cloud Storage

https://n8nworkflows.xyz/workflows/store-ai-generated-images-in-aws-s3--openai-image-creation---cloud-storage-7485


# Store AI-Generated Images in AWS S3: OpenAI Image Creation & Cloud Storage

### 1. Workflow Overview

This workflow automates the generation of AI-created images using OpenAI models and stores them securely in AWS S3 cloud storage. It is designed for users who want to create, organize, and manage AI-generated images programmatically, combining prompt engineering, image creation, and cloud file management in one seamless process.

**Target Use Cases:**  
- Designers and marketers needing rapid AI-generated visuals  
- Developers building AI + cloud storage pipelines  
- Educators demonstrating AI image generation and cloud integration  
- Businesses automating image content workflows with AWS S3

**Logical Blocks:**

- **1.1 Trigger & Input Setup:** Manual workflow execution and user input fields definition.  
- **1.2 AWS S3 Bucket and Folder Management:** Creation of an S3 bucket and folder to organize image storage.  
- **1.3 AI Prompt Generation:** Using OpenAI Chat Model and an AI agent to generate and refine image prompts.  
- **1.4 Image Generation:** AI model creates images from the refined prompt.  
- **1.5 Upload to AWS S3:** Uploading the generated image to the specified bucket and folder.  
- **1.6 Cleanup (Optional, Disabled):** Nodes to delete files and folders from S3 as cleanup or maintenance steps.


---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Input Setup

**Overview:**  
Starts the workflow manually and sets user-defined parameters such as AWS region and folder name for image storage.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Edit Fields

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual start of the workflow, giving user control over execution timing.  
  - Config: No parameters set.  
  - Connections: Output → Edit Fields  
  - Edge Cases: None typical; user must manually trigger.

- **Edit Fields**  
  - Type: Set Node  
  - Role: Defines input parameters such as AWS region and folder name for organizing images in S3.  
  - Config:  
    - aws_region: "ap-southeast-1" (default AWS region)  
    - folder_name: "my_s3_folder" (default destination folder)  
  - Connections: Input ← Manual Trigger; Output → Create AWS S3 Bucket & Delete uploaded image (conditional)  
  - Edge Cases: User must ensure valid region and folder name; invalid region or folder name can cause AWS API errors.

---

#### 1.2 AWS S3 Bucket and Folder Management

**Overview:**  
Creates an AWS S3 bucket (if it does not exist) and a folder (prefix) inside it for organizing the images.

**Nodes Involved:**  
- Create AWS S3 Bucket  
- Create a folder

**Node Details:**

- **Create AWS S3 Bucket**  
  - Type: AWS S3 node  
  - Role: Creates an S3 bucket named dynamically using the workflow ID for uniqueness (format: `bucket{{ $workflow.id }}`).  
  - Configuration:  
    - Operation: Create Bucket  
    - Region: Taken from `aws_region` field set in Edit Fields  
  - Inputs: From Edit Fields  
  - Outputs: To Create a folder  
  - Error Handling: Continues even if bucket creation fails (e.g., bucket exists)  
  - Edge Cases:  
    - Bucket name conflicts or invalid bucket names can cause AWS errors.  
    - Insufficient IAM permissions to create buckets cause failures.

- **Create a folder**  
  - Type: AWS S3 node  
  - Role: Creates a folder (prefix) inside the bucket using the folder name from Edit Fields.  
  - Configuration:  
    - Operation: Create Folder  
    - Bucket Name: `bucket{{ $workflow.id }}` (same bucket created above)  
    - Folder Name: taken from `folder_name` field  
  - Inputs: From Create AWS S3 Bucket  
  - Outputs: To Prompt Generation Agent  
  - Edge Cases:  
    - Folder creation fails if bucket doesn't exist or permissions are insufficient.

---

#### 1.3 AI Prompt Generation

**Overview:**  
Generates an AI image prompt using an OpenAI Chat Model and an AI agent that picks a random topic and formulates a prompt for image generation.

**Nodes Involved:**  
- OpenAI Chat Model  
- Prompt Generation Agent

**Node Details:**

- **OpenAI Chat Model**  
  - Type: LangChain AI Chat Model (OpenAI GPT-4.1-mini)  
  - Role: Provides natural language processing to refine or generate prompts.  
  - Configuration: Model set to "gpt-4.1-mini"  
  - Credentials: OpenAI API key  
  - Input: From Prompt Generation Agent (agent uses this model internally)  
  - Output: To Prompt Generation Agent  
  - Edge Cases:  
    - API rate limits  
    - Invalid API key or connectivity issues

- **Prompt Generation Agent**  
  - Type: LangChain Agent Node  
  - Role: Acts as a prompt generation agent to produce an image prompt from a randomly selected topic among Education, Science, Sport, Economy, and Health.  
  - Configuration: Text prompt instructs the agent to output only the prompt for image generation.  
  - Input: From Create a folder node (trigger to start prompt generation)  
  - Output: To Generate an image node  
  - Edge Cases:  
    - Expression errors in prompt text  
    - API call failures

---

#### 1.4 Image Generation

**Overview:**  
Generates an AI image based on the prompt created by the agent.

**Nodes Involved:**  
- Generate an image

**Node Details:**

- **Generate an image**  
  - Type: LangChain OpenAI Image Generation Node  
  - Role: Creates an AI-generated image using the prompt from the previous step.  
  - Configuration:  
    - Prompt: Uses expression to fetch refined prompt (`={{ $json.output }}`)  
    - Resource: Image generation  
  - Credentials: OpenAI API key  
  - Input: From Prompt Generation Agent  
  - Output: To Upload a file  
  - Edge Cases:  
    - Image generation failures due to invalid prompt, API limits, or model issues  
    - Output file format and size considerations

---

#### 1.5 Upload to AWS S3

**Overview:**  
Uploads the generated image to the previously created S3 bucket and folder with a unique timestamped filename.

**Nodes Involved:**  
- Upload a file

**Node Details:**

- **Upload a file**  
  - Type: AWS S3 node  
  - Role: Uploads the AI-generated image to the S3 bucket inside the specified folder.  
  - Configuration:  
    - Operation: Upload  
    - Bucket Name: `bucket{{ $workflow.id }}`  
    - File Name: Dynamic, format `{{folder_name}}/image_{{ current_date }}.{{ fileExtension }}` (e.g., `my_s3_folder/image_2024-06-28.png`)  
  - Credentials: AWS credentials with permissions to write to S3  
  - Input: From Generate an image  
  - Output: None (end of upload flow)  
  - Edge Cases:  
    - Upload failures due to permissions, bucket accessibility, or network errors  
    - File naming conflicts or invalid characters

---

#### 1.6 Cleanup (Optional, Disabled)

**Overview:**  
Nodes designed for deleting uploaded images or folders from S3, useful for maintenance or testing cleanup. These nodes are disabled by default.

**Nodes Involved:**  
- Delete uploaded image (disabled)  
- Delete a folder (disabled)  

**Node Details:**

- **Delete uploaded image**  
  - Type: AWS S3 node  
  - Role: Deletes a specific uploaded image file from the S3 bucket.  
  - Configuration:  
    - Operation: Delete  
    - Bucket Name: `bucket{{ $workflow.id }}`  
    - File Key: `{{folder_name}}/image_{{ current_date }}.png`  
  - Credentials: AWS with delete permissions  
  - Input: From Edit Fields  
  - Output: To Delete a folder  
  - Edge Cases:  
    - Deletion fails if file does not exist or insufficient permissions

- **Delete a folder**  
  - Type: AWS S3 node  
  - Role: Deletes an entire folder (prefix) and all objects within from the S3 bucket.  
  - Configuration:  
    - Operation: Delete  
    - Bucket Name: `bucket{{ $workflow.id }}`  
    - Folder Key: `{{folder_name}}`  
  - Credentials: AWS with delete permissions  
  - Input: From Delete uploaded image node  
  - Edge Cases:  
    - Deletion fails if folder does not exist or permission issues  
    - Caution: deletes all files under the prefix permanently

---

### 3. Summary Table

| Node Name                | Node Type                                   | Functional Role                           | Input Node(s)             | Output Node(s)         | Sticky Note                                                                                                           |
|--------------------------|---------------------------------------------|------------------------------------------|---------------------------|------------------------|-----------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                              | Starts the workflow manually              | —                         | Edit Fields            | ## Step 1: Trigger Workflow - The workflow begins when you **click “Execute Workflow”** in n8n.                        |
| Edit Fields              | Set Node                                    | Defines input parameters (AWS region, folder name) | When clicking ‘Execute workflow’ | Create AWS S3 Bucket, Delete uploaded image (disabled) | ## Step 2: Edit Fields - This node allows you to **manually provide input fields** for the workflow.                   |
| Create AWS S3 Bucket     | AWS S3                                      | Creates S3 bucket if not exists           | Edit Fields                | Create a folder        | ## Step 3: Create AWS S3 Bucket - The workflow will create a new **AWS S3 bucket** to store your images.                |
| Create a folder          | AWS S3                                      | Creates a folder (prefix) inside bucket   | Create AWS S3 Bucket       | Prompt Generation Agent | ## Step 4: Create a Folder - Inside your S3 bucket, a **folder** (sometimes called a “prefix”) is created.               |
| OpenAI Chat Model        | LangChain AI Chat Model (OpenAI GPT-4.1-mini) | Provides AI language model for prompt refining | Prompt Generation Agent (ai_languageModel) | Prompt Generation Agent | ## Step 5: Prompt Generation Agent - This node uses the **OpenAI Chat Model** to **refine your prompt** before generating the image. |
| Prompt Generation Agent  | LangChain Agent                             | Generates refined prompt for image generation | Create a folder            | Generate an image      | ## Step 5: Prompt Generation Agent - This node uses the **OpenAI Chat Model** to **refine your prompt** before generating the image.|
| Generate an image        | LangChain OpenAI Image Generation           | Generates AI image from prompt            | Prompt Generation Agent    | Upload a file          | ## Step 6: Generate an Image - Using the refined prompt, the **OpenAI Image Generation model** creates an image.         |
| Upload a file            | AWS S3                                      | Uploads generated image to S3 bucket      | Generate an image          | —                      | ## Step 7: Upload File to AWS S3 - The generated image is uploaded into your S3 bucket and folder.                      |
| Delete uploaded image    | AWS S3 (disabled)                           | Deletes specific uploaded image file      | Edit Fields                | Delete a folder        | ## Step 8: Delete an Object - This step lets you **remove a specific file (object) from your S3 bucket** (disabled).     |
| Delete a folder          | AWS S3 (disabled)                           | Deletes entire folder and contents        | Delete uploaded image      | —                      | ## Step 9: Delete a Folder - This step allows you to **delete an entire folder (prefix) in S3** (disabled).               |
| Sticky Note              | Sticky Note                                 | Documentation and explanation notes       | —                         | —                      | See detailed notes section below                                                                                       |
| Sticky Note1             | Sticky Note                                 | Step 1 explanation                        | —                         | —                      | See detailed notes section below                                                                                       |
| Sticky Note2             | Sticky Note                                 | Step 2 explanation                        | —                         | —                      | See detailed notes section below                                                                                       |
| Sticky Note3             | Sticky Note                                 | Step 3 explanation                        | —                         | —                      | See detailed notes section below                                                                                       |
| Sticky Note4             | Sticky Note                                 | Step 4 explanation                        | —                         | —                      | See detailed notes section below                                                                                       |
| Sticky Note5             | Sticky Note                                 | Step 5 explanation                        | —                         | —                      | See detailed notes section below                                                                                       |
| Sticky Note6             | Sticky Note                                 | Step 6 explanation                        | —                         | —                      | See detailed notes section below                                                                                       |
| Sticky Note7             | Sticky Note                                 | Step 7 explanation                        | —                         | —                      | See detailed notes section below                                                                                       |
| Sticky Note8             | Sticky Note                                 | Step 8 explanation (Delete object)       | —                         | —                      | See detailed notes section below                                                                                       |
| Sticky Note9             | Sticky Note                                 | Step 9 explanation (Delete folder)       | —                         | —                      | See detailed notes section below                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**  
   - Name: When clicking ‘Execute workflow’  
   - No parameters needed.

3. **Add a Set node:**  
   - Name: Edit Fields  
   - Set the following variables:  
     - `aws_region`: string, e.g., "ap-southeast-1"  
     - `folder_name`: string, e.g., "my_s3_folder"  
   - Connect Manual Trigger output to Edit Fields input.

4. **Add an AWS S3 node:**  
   - Name: Create AWS S3 Bucket  
   - Operation: Create Bucket  
   - Bucket Name: Expression: `bucket{{$workflow.id}}` (dynamic unique bucket name)  
   - Additional Fields > Region: Expression: `{{$json["aws_region"]}}`  
   - Credentials: AWS S3 (configure with Access Key, Secret, Region)  
   - Connect Edit Fields output to Create AWS S3 Bucket input.  
   - Set "On Error" to "Continue Regular Output" to skip if bucket exists.

5. **Add another AWS S3 node:**  
   - Name: Create a folder  
   - Operation: Create Folder  
   - Bucket Name: Expression: `bucket{{$workflow.id}}`  
   - Folder Name: Expression: `{{$json["folder_name"]}}`  
   - Credentials: AWS S3  
   - Connect Create AWS S3 Bucket output to Create a folder input.

6. **Add LangChain OpenAI Chat Model node:**  
   - Name: OpenAI Chat Model  
   - Model: Select `gpt-4.1-mini` or equivalent  
   - Credentials: OpenAI API key  
   - No direct input connection (used as AI language model for agent).

7. **Add LangChain Agent node:**  
   - Name: Prompt Generation Agent  
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
   - Prompt Type: define  
   - Connect Create a folder output to Prompt Generation Agent input.  
   - Under "AI Language Model," select the OpenAI Chat Model node.

8. **Add LangChain OpenAI Image Generation node:**  
   - Name: Generate an image  
   - Resource: Image  
   - Prompt: Expression: `{{$json["output"]}}` (output from Prompt Generation Agent)  
   - Credentials: OpenAI API key  
   - Connect Prompt Generation Agent output to Generate an image input.

9. **Add AWS S3 node:**  
   - Name: Upload a file  
   - Operation: Upload  
   - Bucket Name: Expression: `bucket{{$workflow.id}}`  
   - File Name: Expression: `{{$json["folder_name"]}}/image_{{$now.format("yyyy-MM-dd")}}.{{$json["fileExtension"]}}`  
   - Credentials: AWS S3  
   - Connect Generate an image output to Upload a file input.

10. **(Optional) Add AWS S3 node for deleting a file:**  
    - Name: Delete uploaded image  
    - Operation: Delete  
    - Bucket Name: Expression: `bucket{{$workflow.id}}`  
    - File Key: Expression: `{{$json["folder_name"]}}/image_{{$now.format("yyyy-MM-dd")}}.png`  
    - Credentials: AWS S3  
    - Connect Edit Fields output to Delete uploaded image input.  
    - Disable this node by default.

11. **(Optional) Add AWS S3 node for deleting a folder:**  
    - Name: Delete a folder  
    - Operation: Delete  
    - Bucket Name: Expression: `bucket{{$workflow.id}}`  
    - Folder Key: Expression: `{{$json["folder_name"]}}`  
    - Credentials: AWS S3  
    - Connect Delete uploaded image output to Delete a folder input.  
    - Disable this node by default.

12. **Add Sticky Notes as needed:** To document each step for clarity, matching the content described in section 5 below.

13. **Set workflow settings:**  
    - Execution Order: v1 (sequential)  
    - Activate credentials for OpenAI and AWS S3.

14. **Test the workflow manually:**  
    - Click "Execute Workflow"  
    - Confirm images are generated and uploaded to AWS S3 bucket/folder.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                                   |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| This workflow is a hands-on example combining AI prompt engineering, image generation, and cloud storage automation into a single streamlined process.                                                                                                                                                                                                                                       | Main workflow description (Sticky Note)                                                                         |
| Step 1: Trigger Workflow - The workflow begins when you **click “Execute Workflow”** in n8n. You can replace this with Webhook or Scheduler triggers later for automation.                                                                                                                                                                                                                   | Sticky Note1                                                                                                     |
| Step 2: Edit Fields - Allows manual input fields such as image description, size, and folder name. Customize for your use case (e.g., campaign name).                                                                                                                                                                                                                                         | Sticky Note2                                                                                                     |
| Step 3: Create AWS S3 Bucket - Creates a new bucket if it doesn't exist. Recommended bucket name format: `ai-images-demo-[yourname]`. Ensure AWS credentials have bucket creation permissions.                                                                                                                                                                                                 | Sticky Note3                                                                                                     |
| Step 4: Create a Folder - Creates a folder (prefix) inside the bucket to organize images (e.g., by project or campaign). Folder names can be dynamic from input fields.                                                                                                                                                                                                                         | Sticky Note4                                                                                                     |
| Step 5: Prompt Generation Agent - Uses OpenAI Chat Model to refine and generate creative prompts from predefined topics, ensuring better image quality.                                                                                                                                                                                                                                       | Sticky Note5                                                                                                     |
| Step 6: Generate an Image - Generates AI images using OpenAI Image Generation model based on refined prompt. Supports parameterization of resolution and style. Output is PNG or JPEG.                                                                                                                                                                                                         | Sticky Note6                                                                                                     |
| Step 7: Upload File to AWS S3 - Uploads image to S3 bucket/folder with unique timestamped filename. Ensure bucket access policies (public/private) are configured properly.                                                                                                                                                                                                                      | Sticky Note7                                                                                                     |
| Step 8: Delete an Object - Allows removal of specific files from S3 bucket. Use carefully as deletion is permanent. Typical use cases include replacing outdated images or cleanup. Disabled by default.                                                                                                                                                                                        | Sticky Note8                                                                                                     |
| Step 9: Delete a Folder - Deletes an entire folder (prefix) and all contained objects from S3. Useful for clearing old assets or test cleanup. Disabled by default. Note that AWS S3 folders are prefixes, so this deletes all objects with matching prefix.                                                                                                                                   | Sticky Note9                                                                                                     |
| AWS Credentials must have appropriate permissions for bucket creation, folder creation, file upload, and deletion operations. OpenAI API keys are required for the Chat and Image Generation models.                                                                                                                                                                                          | Credential requirements                                                                                          |
| This workflow requires n8n version supporting LangChain nodes and AWS S3 node version 2 or higher for folder and bucket operations.                                                                                                                                                                                                                                                           | Version prerequisite                                                                                             |
| For customization, consider replacing manual trigger with webhook or scheduled trigger, multiple image variations via looping, customized file naming, or alternative cloud storage providers like Google Cloud Storage or Azure Blob.                                                                                                                                                            | Customization suggestions                                                                                        |
| For more details on using AWS S3 with n8n, see the official docs: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.awsS3/                                                                                                                                                                                                                                                       | Official n8n AWS S3 Node Documentation                                                                           |
| For OpenAI image generation and prompt engineering, see: https://platform.openai.com/docs/guides/images and https://www.openai.com/blog/chatgpt                                                                                                                                                                                                                                            | OpenAI Documentation                                                                                            |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.