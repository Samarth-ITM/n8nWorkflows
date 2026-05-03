Generate and Store AI Images with DALL-E and Azure Blob Storage

https://n8nworkflows.xyz/workflows/generate-and-store-ai-images-with-dall-e-and-azure-blob-storage-7648


# Generate and Store AI Images with DALL-E and Azure Blob Storage

### 1. Workflow Overview

This workflow is designed to automate the generation and storage of AI-generated images using OpenAI's DALL·E model, combined with Azure Blob Storage for cloud storage and management. It targets users who want a code-free solution to create creative images based on AI-generated prompts, store those images in Azure containers, and manage storage blobs via n8n automation.

The workflow is logically divided into these blocks:

- **1.1 Input Reception and Initialization:** Manual trigger and setting of input parameters like Azure container name.
- **1.2 Azure Blob Storage Management:** Creation, listing, and deletion of Azure Blob containers and blobs.
- **1.3 AI Prompt Generation:** Using OpenAI’s Chat model agent to create dynamic image prompts.
- **1.4 AI Image Generation:** Calling OpenAI’s image generation (DALL·E) service with the prompt.
- **1.5 Image Upload:** Uploading the generated image into Azure Blob storage.
- **1.6 Blob Management:** Listing blobs and deleting specific blobs as needed.
- **1.7 Optional Cleanup:** Deleting entire containers if required.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception and Initialization

**Overview:**  
This block initiates the workflow manually and sets up the required input fields such as the Azure Blob container name.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Edit Fields

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point for manually starting the workflow.  
  - *Configuration:* No parameters; simply triggers execution on demand.  
  - *Inputs:* None  
  - *Outputs:* Connects to `Edit Fields`  
  - *Potential Failures:* User forgets to click execute, no input validation here.

- **Edit Fields**  
  - *Type:* Set node  
  - *Role:* Defines input data, specifically sets `container_name` (e.g., "trungtrandemon8n1").  
  - *Configuration:* Static assignment of `container_name` string.  
  - *Inputs:* From manual trigger  
  - *Outputs:* To `Create container` and `Get many container` nodes  
  - *Edge Cases:* Hardcoded container name; no validation if container exists or if name is invalid.

---

#### 1.2 Azure Blob Storage Management

**Overview:**  
Handles the creation and deletion of containers and blobs in Azure Blob Storage, enabling management of storage resources.

**Nodes Involved:**  
- Create container  
- Get many container  
- Delete container  
- Create Blob  
- Get many blobs  
- Delete Blob

**Node Details:**

- **Create container**  
  - *Type:* Azure Storage  
  - *Role:* Creates a new container in Azure Blob Storage using the `container_name` from input.  
  - *Configuration:* Operation set to `create`; container name dynamically assigned from `Edit Fields`.  
  - *Inputs:* From `Edit Fields`  
  - *Outputs:* To `Prompt Generation Agent`  
  - *Failures:* Container already exists (operation will not duplicate), Azure auth errors, network timeouts.

- **Get many container**  
  - *Type:* Azure Storage  
  - *Role:* Lists all containers in the storage account.  
  - *Configuration:* Default listing operation.  
  - *Inputs:* From `Edit Fields`  
  - *Outputs:* To `Delete container`  
  - *Failures:* Auth errors, API limits.

- **Delete container**  
  - *Type:* Azure Storage  
  - *Role:* Deletes a specified container by name.  
  - *Configuration:* Container name taken from previous `Get many container` node's JSON `name` property.  
  - *Inputs:* From `Get many container`  
  - *Outputs:* None  
  - *Edge Cases:* Irreversible deletion; deletes all blobs within container; must handle errors if container doesn't exist.

- **Create Blob**  
  - *Type:* Azure Storage  
  - *Role:* Uploads AI generated image to the specified container.  
  - *Configuration:*  
    - Operation: `create`  
    - Resource: `blob`  
    - Container: dynamic from `Edit Fields`  
    - Blob Name: static "mydemoblob" (should be customized for multiple images)  
  - *Inputs:* From `Generate an image`  
  - *Outputs:* To `Get many blobs`  
  - *Failures:* Upload failures, Azure limits, auth errors.

- **Get many blobs**  
  - *Type:* Azure Storage  
  - *Role:* Lists all blobs within the specified container.  
  - *Configuration:*  
    - Resource: `blob`  
    - Container: dynamic from `Edit Fields`  
  - *Inputs:* From `Create Blob`  
  - *Outputs:* To `Delete Blob`  
  - *Failures:* Auth or network errors.

- **Delete Blob**  
  - *Type:* Azure Storage  
  - *Role:* Deletes a specific blob by name in the container.  
  - *Configuration:*  
    - Blob name from input JSON `name` property  
    - Container from `Edit Fields`  
    - Operation: `delete`  
  - *Inputs:* From `Get many blobs`  
  - *Outputs:* None  
  - *Edge Cases:* Deleting non-existent blobs; permission errors.

---

#### 1.3 AI Prompt Generation

**Overview:**  
Generates a creative image prompt for AI image generation based on a selection of predefined topics.

**Nodes Involved:**  
- OpenAI Chat Model  
- Prompt Generation Agent

**Node Details:**

- **OpenAI Chat Model**  
  - *Type:* Langchain Chat OpenAI model  
  - *Role:* Provides the language model backend for prompt generation.  
  - *Configuration:* Model set to `gpt-4.1-mini`; no additional options.  
  - *Inputs:* Feeds AI language model input to `Prompt Generation Agent`  
  - *Outputs:* To `Prompt Generation Agent` (ai_languageModel input)  
  - *Credentials:* OpenAI API key configured  
  - *Failures:* API rate limits, invalid credentials, model unavailability.

- **Prompt Generation Agent**  
  - *Type:* Langchain Agent  
  - *Role:* Uses OpenAI Chat Model to create a prompt for the image generation.  
  - *Configuration:*  
    - Text input instructs it to randomly pick a topic among Education, Science, Sport, Economy, Health and output a prompt only.  
    - Prompt type: `define`  
  - *Inputs:* From `Create container` (main) and OpenAI Chat Model (ai_languageModel)  
  - *Outputs:* To `Generate an image`  
  - *Failures:* Expression errors, unexpected API responses.

---

#### 1.4 AI Image Generation

**Overview:**  
Creates an AI-generated image using DALL·E based on the prompt generated by the previous step.

**Nodes Involved:**  
- Generate an image

**Node Details:**

- **Generate an image**  
  - *Type:* Langchain OpenAI Image generation node  
  - *Role:* Generates an image from the prompt text.  
  - *Configuration:*  
    - Resource: `image`  
    - Prompt: dynamically set to the output from `Prompt Generation Agent`  
  - *Inputs:* From `Prompt Generation Agent`  
  - *Outputs:* To `Create Blob`  
  - *Credentials:* OpenAI API key configured  
  - *Failures:* API limits, invalid prompt, network errors.

---

#### 1.5 Image Upload

**Overview:**  
Uploads the generated AI image to Azure Blob Storage into the specified container.

**Nodes Involved:**  
- Create Blob

**Node Details:**

- **Create Blob**  
  - Detailed above in Block 1.2

---

#### 1.6 Blob Management

**Overview:**  
Lists blobs in a container and deletes specific blobs if needed, enabling management of stored images.

**Nodes Involved:**  
- Get many blobs  
- Delete Blob

**Node Details:**

- **Get many blobs**  
  - Detailed above in Block 1.2

- **Delete Blob**  
  - Detailed above in Block 1.2

---

#### 1.7 Optional Cleanup

**Overview:**  
Provides the option to delete entire containers to clean up storage.

**Nodes Involved:**  
- Delete container

**Node Details:**

- **Delete container**  
  - Detailed above in Block 1.2

---

### 3. Summary Table

| Node Name                   | Node Type                             | Functional Role                              | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                               |
|-----------------------------|-------------------------------------|----------------------------------------------|--------------------------------|--------------------------------|----------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                      | Manual start of the workflow                   | None                           | Edit Fields                    | 1. 🖱️ Trigger the Workflow - Manual trigger node for interactive start.                                  |
| Edit Fields                 | Set                                 | Input parameter setup (container name)        | When clicking ‘Execute workflow’| Create container, Get many container | 2. ✍️ Input Your Fields - Defines container name input for subsequent steps.                            |
| Create container            | Azure Storage                       | Creates Azure Blob container                    | Edit Fields                   | Prompt Generation Agent        | 3. 📦 Create Blob Container - Creates container in Azure Storage.                                        |
| Get many container          | Azure Storage                       | Lists all containers                            | Edit Fields                   | Delete container               |                                                                                                          |
| Delete container            | Azure Storage                       | Deletes specified container                      | Get many container            | None                          | 9. 🗑️ (Optional) Delete the Container - Deletes entire container and contents.                           |
| OpenAI Chat Model           | Langchain Chat OpenAI model         | Provides AI language model for prompt generation| Prompt Generation Agent (ai_languageModel input) | Prompt Generation Agent (ai_languageModel output) |                                                                                                          |
| Prompt Generation Agent     | Langchain Agent                    | Generates AI prompt text                         | Create container, OpenAI Chat Model | Generate an image             | 4. 🤖 Generate AI Prompt - Generates prompt for image generation.                                        |
| Generate an image           | Langchain OpenAI Image generation   | Generates AI image from prompt                   | Prompt Generation Agent       | Create Blob                   | 5. 🎨 Generate the Image - Uses DALL·E to create AI image from prompt.                                   |
| Create Blob                 | Azure Storage                       | Uploads generated image to Azure Blob container | Generate an image             | Get many blobs                | 6. ☁️ Upload Image to Azure Blob - Uploads image to Azure Blob Storage.                                  |
| Get many blobs              | Azure Storage                       | Lists blobs in container                         | Create Blob                   | Delete Blob                  | 7. 📂 List All Blobs - Retrieves blobs inside container.                                                |
| Delete Blob                 | Azure Storage                       | Deletes specific blob                             | Get many blobs                | None                          | 8. 🧹 Delete Specific Blob - Deletes specified blob from container.                                      |
| Sticky Note                 | Sticky Note                        | Documentation and instructions                   | None                         | None                          | Contains full tutorial and setup instructions with video link.                                          |
| Sticky Note1                | Sticky Note                        | Documentation for manual trigger                  | None                         | None                          | Notes about manual trigger node.                                                                         |
| Sticky Note2                | Sticky Note                        | Documentation for input fields                     | None                         | None                          | Notes about the input fields node.                                                                       |
| Sticky Note3                | Sticky Note                        | Documentation for create container node           | None                         | None                          | Notes about container creation.                                                                           |
| Sticky Note4                | Sticky Note                        | Documentation for image generation node           | None                         | None                          | Notes about image generation with DALL·E.                                                                |
| Sticky Note5                | Sticky Note                        | Documentation for prompt generation agent         | None                         | None                          | Notes about AI prompt generation.                                                                         |
| Sticky Note6                | Sticky Note                        | Documentation for blob upload node                 | None                         | None                          | Notes about uploading image to Azure Blob Storage.                                                       |
| Sticky Note7                | Sticky Note                        | Documentation for container deletion node          | None                         | None                          | Notes about container deletion caution.                                                                  |
| Sticky Note8                | Sticky Note                        | Documentation for listing blobs                     | None                         | None                          | Notes about listing blobs in the container.                                                              |
| Sticky Note9                | Sticky Note                        | Documentation for deleting blobs                    | None                         | None                          | Notes about deleting specific blobs.                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named `When clicking ‘Execute workflow’`.  
   - No configuration needed.

2. **Create Set Node for Input Fields**  
   - Add a **Set** node named `Edit Fields`.  
   - Add a string field `container_name` with the desired Azure Blob container name, e.g., `"trungtrandemon8n1"`.  
   - Connect `When clicking ‘Execute workflow’` → `Edit Fields`.

3. **Create Azure Storage Node to Create Container**  
   - Add an **Azure Storage** node named `Create container`.  
   - Set operation to `create`.  
   - For container name, use expression: `{{$json.container_name}}` to get from `Edit Fields`.  
   - Configure with Azure Storage Shared Key credentials (account name, access key).  
   - Connect `Edit Fields` → `Create container`.

4. **Create Azure Storage Node to List Containers**  
   - Add an **Azure Storage** node named `Get many container`.  
   - Set operation to list containers (default).  
   - Use same Azure credentials.  
   - Connect `Edit Fields` → `Get many container`.

5. **Create Azure Storage Node to Delete Container**  
   - Add an **Azure Storage** node named `Delete container`.  
   - Set operation to `delete`.  
   - Set container name dynamically from previous `Get many container` node’s JSON property `name`: `{{$json.name}}`.  
   - Connect `Get many container` → `Delete container`.

6. **Add Langchain Chat OpenAI Model Node**  
   - Add a **Langchain Chat OpenAI** node named `OpenAI Chat Model`.  
   - Select model `gpt-4.1-mini`.  
   - Configure with OpenAI API credentials.  
   - Connect `OpenAI Chat Model` ai_languageModel output → `Prompt Generation Agent` ai_languageModel input.

7. **Add Langchain Agent Node for Prompt Generation**  
   - Add a **Langchain Agent** node named `Prompt Generation Agent`.  
   - Set prompt text to:  
     ```
     You're prompt generator agent, you will create prompt for the open-ai image generation model with a random topic among:
     - Education
     - Science 
     - Sport
     - Economy
     - Health
     Output the prompt only, nothing else
     ```  
   - Set promptType to `define`.  
   - Connect `Create container` main output → `Prompt Generation Agent` main input.  
   - Connect `OpenAI Chat Model` ai_languageModel → `Prompt Generation Agent` ai_languageModel input.

8. **Add Langchain OpenAI Node to Generate Image**  
   - Add a **Langchain OpenAI** node named `Generate an image`.  
   - Set resource to `image`.  
   - Set prompt dynamically from `Prompt Generation Agent` output: `{{$json.output}}`.  
   - Use OpenAI API credentials.  
   - Connect `Prompt Generation Agent` → `Generate an image`.

9. **Add Azure Storage Node to Create Blob**  
   - Add an **Azure Storage** node named `Create Blob`.  
   - Set operation to `create`.  
   - Resource set to `blob`.  
   - Container name dynamically from `Edit Fields`: `{{$json.container_name}}`.  
   - Set Blob name (e.g., "mydemoblob") — consider dynamic naming for multiple images.  
   - Connect `Generate an image` → `Create Blob`.

10. **Add Azure Storage Node to List Blobs**  
    - Add an **Azure Storage** node named `Get many blobs`.  
    - Set resource to `blob`.  
    - Container name dynamically from `Edit Fields`: `{{$json.container_name}}`.  
    - Connect `Create Blob` → `Get many blobs`.

11. **Add Azure Storage Node to Delete Blob**  
    - Add an **Azure Storage** node named `Delete Blob`.  
    - Set operation to `delete`.  
    - Blob name from input JSON `name` property: `{{$json.name}}`.  
    - Container name from `Edit Fields`: `{{$json.container_name}}`.  
    - Connect `Get many blobs` → `Delete Blob`.

12. **Finalizing Credentials Setup**  
    - Configure **OpenAI API credentials** with your OpenAI API key.  
    - Configure **Azure Blob Storage Shared Key credentials** with Storage Account Name and Access Key.

13. **Test the Workflow**  
    - Manually trigger the workflow.  
    - Ensure container is created, prompt generated, image created, uploaded, blobs listed.  
    - Optionally delete blobs or containers to test cleanup.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                                                |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| Beginner’s Tutorial: Manage Azure Storage Account Container & Blob with n8n. This workflow shows how to generate AI images using OpenAI, store them in Azure Blob Storage, and manage blob containers with zero code. Suitable for beginners, no-code developers, and cloud learners.                                                                                                                                                                                                                                             | [YouTube Video Tutorial](https://www.youtube.com/watch?v=vh06fpMkalw)                                                          |
| Setup instructions for OpenAI and Azure Blob Storage credentials, including creating storage accounts and setting up access keys in n8n.                                                                                                                                                                                                                                                                                                                                                                                          | Included in Sticky Note content and workflow description.                                                                       |
| Customization ideas: Adjust prompt generation (styles, branding, languages), organize container names by date/user, send image outputs to Slack/Telegram/Email, and add cleanup logic like auto-deletion or versioning.                                                                                                                                                                                                                                                                                                                | Workflow description section.                                                                                                   |
| OpenAI charges tokens for image generation; monitor usage accordingly.                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Workflow description warning.                                                                                                  |
| This demo uses Access Key authentication for Azure Blob Storage; consider SAS or OAuth for production.                                                                                                                                                                                                                                                                                                                                                                                                                              | Workflow description notes.                                                                                                    |

---

*Disclaimer:*  
The provided text originates exclusively from an automated workflow created with n8n, a no-code integration and automation platform. The process strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data manipulated is legal and public.