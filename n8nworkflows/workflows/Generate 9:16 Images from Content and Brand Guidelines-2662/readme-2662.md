Generate 9:16 Images from Content and Brand Guidelines

https://n8nworkflows.xyz/workflows/generate-9-16-images-from-content-and-brand-guidelines-2662


# Generate 9:16 Images from Content and Brand Guidelines

### 1. Workflow Overview

This n8n workflow automates the generation of 9:16 aspect ratio images tailored for short-form video content and thumbnails, leveraging brand guidelines and SEO-focused blog content. Its purpose is to streamline the creation of visually consistent, AI-generated images that align with brand identity and SEO strategies, useful for marketing teams, content creators, and social media managers.

The workflow is logically divided into the following blocks:

- **1.1 Manual Trigger**: Starts the workflow manually on demand.
- **1.2 Brand Guidelines Retrieval and Preparation**: Fetches brand style, tone, and other identity elements from Airtable and prepares them for use.
- **1.3 SEO Keywords and Blog Content Retrieval**: Obtains SEO keywords and associated blog posts from Airtable, filters them for relevance, and limits the content scope.
- **1.4 Script and Image Prompt Generation**: Uses GPT-4 to create a four-scene script and corresponding image prompts for short-form video content and thumbnails.
- **1.5 Thumbnail Image Generation**: Improves the thumbnail prompt with Leonardo.ai, generates the thumbnail image, waits for completion, and stores the asset in Airtable.
- **1.6 Scene Images Generation**: Iterates over each scene prompt, improves prompts with Leonardo.ai, generates scene images, waits for completion, and saves the assets in Airtable.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger

**Overview:**  
Entry point to initiate the workflow manually within n8n.

**Nodes Involved:**  
- When clicking ‘Test workflow’

**Node Details:**  
- Type: Manual Trigger  
- Role: Starts the workflow execution on demand.  
- Configuration: Default manual trigger with no parameters.  
- Inputs: None  
- Outputs: Initiates connection to "Get Brand Guidelines" node.  
- Edge Cases: None typical; user must manually trigger, so no automatic scheduling.  

---

#### 1.2 Brand Guidelines Retrieval and Preparation

**Overview:**  
Fetches brand-related data from Airtable, including style elements, tone, tagline, and other guidelines, then prepares this data for downstream use.

**Nodes Involved:**  
- Get Brand Guidelines  
- Set Guidelines  
- Sticky Note (documentation)

**Node Details:**

- **Get Brand Guidelines**  
  - Type: Airtable node  
  - Role: Retrieves brand elements from the "Brand Guidelines" table in Airtable "Content Manager" base.  
  - Configuration: Search operation, no filters applied (fetches all entries). Uses Airtable Personal Access Token credential.  
  - Input: Connects from Manual Trigger  
  - Output: Sends retrieved data to Set Guidelines.  
  - Potential Failures: Airtable API errors (auth, rate limit), network issues.  

- **Set Guidelines**  
  - Type: Set node  
  - Role: Prepares brand guidelines data by assigning the Airtable record ID into the workflow JSON for referencing.  
  - Configuration: Transfers all fields, assigns `id` explicitly for reference.  
  - Input: From Get Brand Guidelines  
  - Output: To Get SEO Keywords node.  
  - Edge Cases: Missing or malformed data could propagate errors downstream.  

---

#### 1.3 SEO Keywords and Blog Content Retrieval

**Overview:**  
Retrieves SEO keywords and their associated content, filters keywords based on a specified search term, removes duplicates, fetches corresponding blog content, and limits the data set.

**Nodes Involved:**  
- Get SEO Keywords  
- Keyword Filter  
- Remove Duplicates  
- Split Out Keywords  
- Get Content  
- Split Out Content  
- Limit  
- Sticky Notes (documentation)

**Node Details:**

- **Get SEO Keywords**  
  - Type: Airtable node  
  - Role: Searches the "SEO Keywords" table for keywords and related content links.  
  - Configuration: Fields retrieved include "Keyword" and "RelatedContent".  
  - Credential: Airtable Personal Access Token.  
  - Input: From Set Guidelines  
  - Output: To Keyword Filter node.  
  - Edge Cases: API failure, empty keyword sets.  

- **Keyword Filter**  
  - Type: Filter node  
  - Role: Filters keywords containing the phrase "ai automation" (case-insensitive).  
  - Configuration: Condition checks if "Keyword" field contains "ai automation".  
  - Input: From Get SEO Keywords  
  - Output: To Remove Duplicates  
  - Edge Cases: No matches found results in empty output downstream.  

- **Remove Duplicates**  
  - Type: Remove Duplicates node  
  - Role: Ensures unique keywords based on their `id` field to avoid redundant processing.  
  - Input: From Keyword Filter  
  - Output: To Split Out Keywords  
  - Edge Cases: If input is empty, no output.  

- **Split Out Keywords**  
  - Type: Split Out node  
  - Role: Splits array of keyword records to process individually.  
  - Input: From Remove Duplicates  
  - Output: To Get Content node.  
  - Edge Cases: Empty input results in no processing.  

- **Get Content**  
  - Type: Airtable node  
  - Role: Retrieves blog post content linked to each keyword from the "SEO Keywords" table using the "RelatedContent" ID.  
  - Configuration: Search by record ID per input item.  
  - Credential: Airtable Personal Access Token.  
  - Input: From Split Out Keywords  
  - Output: To Split Out Content node.  
  - Edge Cases: Missing content for given ID, API failures.  

- **Split Out Content**  
  - Type: Split Out node  
  - Role: Splits fetched blog content items for individual processing.  
  - Input: From Get Content  
  - Output: To Limit node.  
  - Edge Cases: Empty content results in no further processing.  

- **Limit**  
  - Type: Limit node  
  - Role: Restricts the number of blog posts processed to a default limit (default is usually 1, no explicit setting shown).  
  - Input: From Split Out Content  
  - Output: To Script Prep node.  
  - Edge Cases: Limits workflow load but must be adjusted for large-scale runs.  

---

#### 1.4 Script and Image Prompt Generation

**Overview:**  
Generates a short-form video script with four scenes and corresponding image prompts using GPT-4 (OpenAI), then splits out thumbnail prompts and scene prompts for image generation.

**Nodes Involved:**  
- Script Prep  
- Split Out TN Prompt  
- Split Out Scenes  
- Sticky Note (documentation)

**Node Details:**

- **Script Prep**  
  - Type: OpenAI (LangChain) node  
  - Role: Uses GPT-4O-MINI to prepare a 4-scene video script and image prompts based on blog post title and content.  
  - Configuration:  
    - Model: GPT-4O-MINI  
    - Input messages include blog title and content, request for a script <30 seconds, prompts per scene, and a thumbnail prompt.  
    - Output: JSON with numbered scenes and prompts.  
  - Input: From Limit node  
  - Output: To Split Out TN Prompt and Split Out Scenes nodes.  
  - Credentials: OpenAI API Key  
  - Edge Cases: API errors, content too long, prompt format errors.  

- **Split Out TN Prompt**  
  - Type: Split Out node  
  - Role: Extracts the thumbnail prompt from generated output to feed image generation.  
  - Input: From Script Prep  
  - Output: To Prompt Settings node (for thumbnail).  

- **Split Out Scenes**  
  - Type: Split Out node  
  - Role: Splits the 4 scene prompts into separate items to process them individually for image generation.  
  - Input: From Script Prep  
  - Output: To Loop Over Items node (for scene images).  

---

#### 1.5 Thumbnail Image Generation

**Overview:**  
Improves the thumbnail image prompt via Leonardo.ai, generates the thumbnail image, waits for completion, retrieves the image URL, and stores the asset in Airtable.

**Nodes Involved:**  
- Prompt Settings  
- Leo - Improve Prompt1  
- Leo - Generate Image1  
- Wait 30 Seconds  
- Leo - Get imageId1  
- Add Asset Info  
- Sticky Notes (documentation)

**Node Details:**

- **Prompt Settings**  
  - Type: Set node  
  - Role: Configures parameters for Leonardo.ai image generation (model ID, additional style tips).  
  - Configuration: Sets model to "de7d3faf-762f-48e0-b3b7-9d0ac3a3fcf3" (Phoenix 1.0) and adds "Use the rule of thirds, leading lines, & balance." to prompts.  
  - Input: From Split Out TN Prompt  
  - Output: To Leo - Improve Prompt1.  

- **Leo - Improve Prompt1**  
  - Type: HTTP Request  
  - Role: Calls Leonardo.ai API to improve the thumbnail prompt for clarity and detail.  
  - Configuration: POST JSON with prompt from Split Out TN Prompt.  
  - Authentication: Leonardo.ai custom HTTP Auth credentials.  
  - Input: From Prompt Settings  
  - Output: To Leo - Generate Image1.  
  - Edge Cases: API errors, invalid prompt structure, auth failures.  

- **Leo - Generate Image1**  
  - Type: HTTP Request  
  - Role: Initiates image generation on Leonardo.ai with improved prompt for thumbnail.  
  - Configuration: 768x1376 resolution, model Phoenix 1.0, dynamic preset style, guidance scale 7, high resolution enabled, single image per call.  
  - Input: From Leo - Improve Prompt1  
  - Output: To Wait 30 Seconds node.  
  - Edge Cases: API limits, generation delays, malformed JSON body.  

- **Wait 30 Seconds**  
  - Type: Wait  
  - Role: Pauses workflow for 30 seconds to allow Leonardo.ai to finish image generation.  
  - Input: From Leo - Generate Image1  
  - Output: To Leo - Get imageId1 node.  

- **Leo - Get imageId1**  
  - Type: HTTP Request  
  - Role: Polls Leonardo.ai with generation ID to retrieve the completed image URL and metadata.  
  - Input: From Wait 30 Seconds  
  - Output: To Add Asset Info node.  
  - Edge Cases: Generation not ready, API timeout, invalid ID.  

- **Add Asset Info**  
  - Type: Airtable node  
  - Role: Saves generated thumbnail image URL and metadata into the "Assets" table in Airtable.  
  - Configuration: Maps the image URL, sets asset name with blog title prefix "TN -", type as "Image", file size set to 0 (can be updated later).  
  - Input: From Leo - Get imageId1  
  - Output: To end (no further nodes)  
  - Edge Cases: Airtable API failures, duplication, missing data.  

---

#### 1.6 Scene Images Generation

**Overview:**  
Iterates over each scene script and prompt, improves prompts, triggers Leonardo.ai image generation for each, waits, retrieves image URLs, and stores assets back in Airtable.

**Nodes Involved:**  
- Loop Over Items  
- Prompt Settings1  
- Leo - Improve Prompt  
- Leo - Generate Image  
- Wait 30 Seconds1  
- Leo - Get imageId  
- Add Asset Info1  
- Sticky Notes (documentation)

**Node Details:**

- **Loop Over Items**  
  - Type: Split In Batches  
  - Role: Processes each scene prompt one at a time for sequential image generation.  
  - Configuration: Default batch size 1, does not reset between runs.  
  - Input: From Split Out Scenes  
  - Output: Two outputs: first empty, second to Prompt Settings1 (used for image generation).  
  - Edge Cases: Empty input means no images generated.  

- **Prompt Settings1**  
  - Type: Set node  
  - Role: Configures Leonardo.ai parameters for scene images with same model and style settings as thumbnail.  
  - Input: From Loop Over Items (second output)  
  - Output: To Leo - Improve Prompt node.  

- **Leo - Improve Prompt**  
  - Type: HTTP Request  
  - Role: Improves each scene prompt for high-quality image generation using Leonardo.ai API.  
  - Input: From Prompt Settings1  
  - Output: To Leo - Generate Image node.  
  - Edge Cases: API errors, prompt errors.  

- **Leo - Generate Image**  
  - Type: HTTP Request  
  - Role: Submits generation requests for scenes with parameters similar to thumbnail but with ultra mode enabled.  
  - Input: From Leo - Improve Prompt  
  - Output: To Wait 30 Seconds1 node.  
  - Edge Cases: Generation queue delays, API rate limits.  

- **Wait 30 Seconds1**  
  - Type: Wait  
  - Role: Pauses for 30 seconds to allow image generation to complete.  
  - Input: From Leo - Generate Image  
  - Output: To Leo - Get imageId node.  

- **Leo - Get imageId**  
  - Type: HTTP Request  
  - Role: Retrieves generated scene image URL and metadata by generation ID.  
  - Input: From Wait 30 Seconds1  
  - Output: To Add Asset Info1 node.  
  - Edge Cases: Premature polling, API failure.  

- **Add Asset Info1**  
  - Type: Airtable node  
  - Role: Stores scene image URLs and metadata into Airtable "Assets" table, naming assets with scene script text.  
  - Input: From Leo - Get imageId  
  - Output: Loops back to Loop Over Items to process next scene.  
  - Edge Cases: Airtable errors, data mismatches.  

---

### 3. Summary Table

| Node Name                 | Node Type                        | Functional Role                            | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                                 |
|---------------------------|---------------------------------|-------------------------------------------|-----------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                 | Manual start of workflow                   | None                        | Get Brand Guidelines         |                                                                                                             |
| Get Brand Guidelines       | Airtable                        | Fetch brand guidelines from Airtable      | When clicking ‘Test workflow’ | Set Guidelines             |                                                                                                             |
| Set Guidelines            | Set                             | Prepare brand guidelines data              | Get Brand Guidelines          | Get SEO Keywords             |                                                                                                             |
| Get SEO Keywords           | Airtable                        | Retrieve SEO keywords                      | Set Guidelines               | Keyword Filter               |                                                                                                             |
| Keyword Filter             | Filter                         | Filter keywords containing "ai automation" | Get SEO Keywords             | Remove Duplicates            | ## Set keyword filter                                                                                        |
| Remove Duplicates          | Remove Duplicates               | Remove duplicate keywords                  | Keyword Filter               | Split Out Keywords           |                                                                                                             |
| Split Out Keywords         | Split Out                      | Split keywords for individual processing  | Remove Duplicates            | Get Content                  |                                                                                                             |
| Get Content                | Airtable                       | Retrieve blog content by RelatedContent ID | Split Out Keywords           | Split Out Content            |                                                                                                             |
| Split Out Content          | Split Out                      | Split blog content items                   | Get Content                  | Limit                       |                                                                                                             |
| Limit                     | Limit                          | Limit number of blog posts processed       | Split Out Content            | Script Prep                 |                                                                                                             |
| Script Prep               | OpenAI (LangChain)             | Generate 4-scene script and image prompts | Limit                       | Split Out TN Prompt, Split Out Scenes |                                                                                                             |
| Split Out TN Prompt        | Split Out                      | Extract thumbnail prompt                   | Script Prep                  | Prompt Settings             |                                                                                                             |
| Split Out Scenes           | Split Out                      | Extract scene prompts                      | Script Prep                  | Loop Over Items             |                                                                                                             |
| Prompt Settings           | Set                            | Configure Leonardo.ai generation params   | Split Out TN Prompt          | Leo - Improve Prompt1       |                                                                                                             |
| Leo - Improve Prompt1      | HTTP Request                   | Improve thumbnail prompt via Leonardo.ai | Prompt Settings              | Leo - Generate Image1       |                                                                                                             |
| Leo - Generate Image1      | HTTP Request                   | Generate thumbnail image                   | Leo - Improve Prompt1        | Wait 30 Seconds             |                                                                                                             |
| Wait 30 Seconds           | Wait                           | Wait for image generation completion      | Leo - Generate Image1        | Leo - Get imageId1          |                                                                                                             |
| Leo - Get imageId1         | HTTP Request                   | Retrieve generated thumbnail image info   | Wait 30 Seconds             | Add Asset Info              |                                                                                                             |
| Add Asset Info             | Airtable                       | Save thumbnail asset info to Airtable     | Leo - Get imageId1           |                             |                                                                                                             |
| Loop Over Items            | Split In Batches               | Iterate over scene prompts                 | Split Out Scenes             | Prompt Settings1 (main output) |                                                                                                             |
| Prompt Settings1          | Set                            | Configure Leonardo.ai params for scenes   | Loop Over Items              | Leo - Improve Prompt        |                                                                                                             |
| Leo - Improve Prompt       | HTTP Request                   | Improve scene prompt via Leonardo.ai      | Prompt Settings1             | Leo - Generate Image        |                                                                                                             |
| Leo - Generate Image       | HTTP Request                   | Generate scene images                       | Leo - Improve Prompt         | Wait 30 Seconds1            |                                                                                                             |
| Wait 30 Seconds1          | Wait                           | Wait for scene image generation completion | Leo - Generate Image         | Leo - Get imageId           |                                                                                                             |
| Leo - Get imageId          | HTTP Request                   | Retrieve generated scene image info       | Wait 30 Seconds1             | Add Asset Info1             |                                                                                                             |
| Add Asset Info1            | Airtable                       | Save scene asset info to Airtable          | Leo - Get imageId            | Loop Over Items (empty output) |                                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually without parameters.

2. **Create Airtable Node - Get Brand Guidelines**  
   - Operation: Search  
   - Base: Content Manager (appRDq3E42JNtruIP)  
   - Table: Brand Guidelines (tblF8Ye2g0gPdpsaI)  
   - Credentials: Airtable Personal Access Token  
   - Connect input from Manual Trigger.

3. **Create Set Node - Set Guidelines**  
   - Assign field `id` from input JSON `id`.  
   - Include all other fields.  
   - Connect input from Get Brand Guidelines.

4. **Create Airtable Node - Get SEO Keywords**  
   - Operation: Search  
   - Base: Content Manager  
   - Table: SEO Keywords (tblU1fgGH1LXwnWRb)  
   - Retrieve fields: Keyword, RelatedContent  
   - Credentials: Airtable Personal Access Token  
   - Connect input from Set Guidelines.

5. **Create Filter Node - Keyword Filter**  
   - Condition: Keyword field contains (case-insensitive) "ai automation"  
   - Connect input from Get SEO Keywords.

6. **Create Remove Duplicates Node**  
   - Compare field: `id`  
   - Connect input from Keyword Filter.

7. **Create Split Out Node - Split Out Keywords**  
   - Field to split: RelatedContent  
   - Connect input from Remove Duplicates.

8. **Create Airtable Node - Get Content**  
   - Operation: Search by ID  
   - Base: Content Manager  
   - Table: SEO Keywords  
   - Credentials: Airtable Personal Access Token  
   - Connect input from Split Out Keywords.

9. **Create Split Out Node - Split Out Content**  
   - Field to split: id  
   - Connect input from Get Content.

10. **Create Limit Node**  
    - Default limit (e.g. 1) to restrict processed blog posts.  
    - Connect input from Split Out Content.

11. **Create OpenAI Node - Script Prep**  
    - Model: GPT-4O-MINI  
    - Input Messages:  
      - Content contains blog Title and Content from previous node.  
      - Instruction to create a 4-scene script <30 seconds, image prompts per scene, and thumbnail prompt for 9:16 video.  
    - JSON output enabled.  
    - Credentials: OpenAI API Key.  
    - Connect input from Limit node.

12. **Create Split Out Node - Split Out TN Prompt**  
    - Field to split: message.content.thumbnail_prompt  
    - Connect input from Script Prep.

13. **Create Split Out Node - Split Out Scenes**  
    - Field to split: message.content.scenes  
    - Connect input from Script Prep.

14. **Create Set Node - Prompt Settings**  
    - Assignments:  
      - model: "de7d3faf-762f-48e0-b3b7-9d0ac3a3fcf3" (Leonardo.ai Phoenix 1.0)  
      - additional: "Use the rule of thirds, leading lines, & balance."  
    - Include other fields.  
    - Connect input from Split Out TN Prompt.

15. **Create HTTP Request Node - Leo - Improve Prompt1**  
    - POST to "https://cloud.leonardo.ai/api/rest/v1/prompt/improve"  
    - Body JSON: `{ "prompt": "{{ $json['message.content[\'Thumbnail Prompt\']'] }}" }`  
    - Headers: Accept application/json  
    - Authentication: Leonardo.ai HTTP Custom Auth  
    - Connect input from Prompt Settings.

16. **Create HTTP Request Node - Leo - Generate Image1**  
    - POST to "https://cloud.leonardo.ai/api/rest/v1/generations"  
    - JSON Body includes:  
      - alchemy: true  
      - width: 768, height: 1376  
      - modelId from Prompt Settings  
      - num_images: 1  
      - presetStyle: DYNAMIC  
      - prompt from improved prompt response  
      - guidance_scale: 7  
      - highResolution: true  
      - Other flags as per original node.  
    - Authentication: Leonardo.ai HTTP Custom Auth  
    - Connect input from Leo - Improve Prompt1.

17. **Create Wait Node - Wait 30 Seconds**  
    - Amount: 30 seconds  
    - Connect input from Leo - Generate Image1.

18. **Create HTTP Request Node - Leo - Get imageId1**  
    - GET from "https://cloud.leonardo.ai/api/rest/v1/generations/{{ $json.body.sdGenerationJob.generationId }}"  
    - Authentication: Leonardo.ai HTTP Custom Auth  
    - Connect input from Wait 30 Seconds.

19. **Create Airtable Node - Add Asset Info**  
    - Operation: Create  
    - Base: Content Manager  
    - Table: Assets (tblqoaJ7bRLBgENED)  
    - Map fields:  
      - Asset URL from Leonardo.ai image URL  
      - Asset Name: "TN - " plus blog Title  
      - Asset Type: Image  
      - File Size: 0 (placeholder)  
    - Credentials: Airtable Personal Access Token  
    - Connect input from Leo - Get imageId1.

20. **Create Split In Batches Node - Loop Over Items**  
    - No reset on completion, batch size default 1  
    - Connect input from Split Out Scenes.

21. **Create Set Node - Prompt Settings1**  
    - Same assignments as Prompt Settings for scene images.  
    - Connect input from Loop Over Items (main output).

22. **Create HTTP Request Node - Leo - Improve Prompt**  
    - Similar to Leo - Improve Prompt1 but uses scene prompt (`{{ $json.image_prompt }}`)  
    - Connect input from Prompt Settings1.

23. **Create HTTP Request Node - Leo - Generate Image**  
    - POST to Leonardo.ai generations with scene prompt and ultra mode = true  
    - Connect input from Leo - Improve Prompt.

24. **Create Wait Node - Wait 30 Seconds1**  
    - 30 second wait  
    - Connect input from Leo - Generate Image.

25. **Create HTTP Request Node - Leo - Get imageId**  
    - Poll generation status using generationId  
    - Connect input from Wait 30 Seconds1.

26. **Create Airtable Node - Add Asset Info1**  
    - Save scene image asset info to Airtable Assets table, naming assets with "Scene - " plus script text  
    - Connect input from Leo - Get imageId.

27. **Connect Add Asset Info1 back to Loop Over Items (empty output)**  
    - This loops to process next scene image.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow created by AlexK1919, AI-Native Workflow Architect.                                                                                                                                                                              | [AlexK1919 n8n Templates](https://n8n.io/creators/alexk1919)                                       |
| Example Airtable base used for brand guidelines, SEO keywords, and assets management.                                                                                                                                                     | [Airtable Base Example](https://airtable.com/appRDq3E42JNtruIP/shrnc9EzlxpCq7Vxe)                  |
| Video overview of the workflow is available on YouTube channel.                                                                                                                                                                          | [AlexK1919 YouTube Channel](https://www.youtube.com/@alexk1919_)                                   |
| Leonardo.ai API usage includes prompt improvement and image generation using the Phoenix 1.0 model.                                                                                                                                        | [Leonardo.ai](https://app.leonardo.ai/?via=alexk1919)                                              |
| The workflow uses GPT-4O-MINI for efficient script and prompt generation optimized for short-form video content under 30 seconds.                                                                                                        | OpenAI API documentation                                                                         |
| API credentials required: OpenAI API Key, Leonardo.ai API Key, Airtable API Token.                                                                                                                                                         | Configure securely in n8n Credentials manager                                                     |
| The workflow can be adjusted by modifying keyword filters, batch sizes, or image generation parameters for scalability and customization.                                                                                                |                                                                                                    |
| Generated images are stored with metadata in Airtable for asset management and reuse in marketing workflows.                                                                                                                             |                                                                                                    |
| Sticky notes in the workflow provide contextual guidance about node groups and usage, including model versions and optional configurations.                                                                                               | See sticky notes in workflow editor                                                               |

---

This detailed reference document enables users and AI agents to fully understand, reproduce, and extend the "Generate 9:16 Images from Content and Brand Guidelines" workflow, ensuring robust integration and error anticipation across all parts.