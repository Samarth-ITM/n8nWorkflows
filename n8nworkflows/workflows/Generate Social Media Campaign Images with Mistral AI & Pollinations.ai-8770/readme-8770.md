Generate Social Media Campaign Images with Mistral AI & Pollinations.ai

https://n8nworkflows.xyz/workflows/generate-social-media-campaign-images-with-mistral-ai---pollinations-ai-8770


# Generate Social Media Campaign Images with Mistral AI & Pollinations.ai

---

### 1. Workflow Overview

This workflow automates the generation of a social media campaign visual content package by leveraging AI models and image generation APIs. It is designed for marketing teams or content creators aiming to produce a cohesive set of AI-generated images and campaign messaging based on brand and campaign goals stored in Google Drive.

The workflow’s logic is organized into the following main blocks:

- **1.1 Input Data Retrieval:** Downloads brand profile and campaign goals from Google Drive.
- **1.2 Data Cleaning and Merging:** Processes and merges the retrieved data into a structured JSON format.
- **1.3 Campaign Goal Generation (AI Agent):** Uses Mistral AI to generate refined campaign goals based on cleaned data.
- **1.4 Image Prompt Generation (AI Agent):** Generates 5 detailed image prompts, a caption, and hashtags for the campaign.
- **1.5 Image Generation:** Calls Pollinations.ai API five times in parallel to generate images from the prompts.
- **1.6 Image Naming and Merging:** Renames the binary image data for clarity and merges all images into a single item.
- **1.7 File Upload:** Uploads the merged image file back to Google Drive for storage and distribution.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Data Retrieval

**Overview:**  
This block downloads raw brand profile and campaign goal data files from Google Drive, serving as the initial data input for the workflow.

**Nodes Involved:**
- brand profile
- brand goals
- Sticky Note (describes purpose)

**Node Details:**

- **brand profile**  
  - Type: Google Drive (download operation)  
  - Config: Downloads brand profile data file via provided Google Drive file URL/ID.  
  - Credentials: Google Drive OAuth2 account  
  - Inputs: Manual trigger  
  - Outputs: JSON data containing company profile details  
  - Potential Failures: Auth failure, file not found, rate limits  

- **brand goals**  
  - Type: Google Drive (download operation)  
  - Config: Downloads campaign goal data file via provided Google Drive file URL/ID.  
  - Credentials: Google Drive OAuth2 account  
  - Inputs: Manual trigger  
  - Outputs: JSON data with campaign goal details  
  - Potential Failures: Same as brand profile node  

- **Sticky Note**  
  - Describes this block as “Get brand profile and goal saved in google drive”.

---

#### 2.2 Data Cleaning and Merging

**Overview:**  
Processes the downloaded raw data to extract relevant fields and merge them into a consolidated JSON object representing company and campaign information.

**Nodes Involved:**
- clean retrived data
- Merge profile and goals

**Node Details:**

- **clean retrived data**  
  - Type: Code  
  - Function: Parses incoming data items, extracting summaries, company profile, and campaign information into structured JSON. Handles nested JSON fields safely.  
  - Inputs: Merged brand profile and goal nodes  
  - Outputs: Structured JSON object with keys: summaries, company, campaign  
  - Edge Cases: Missing expected fields, malformed JSON strings, empty data  

- **Merge profile and goals**  
  - Type: Merge  
  - Function: Combines brand profile and brand goals data streams into one for processing.  
  - Inputs: brand profile and brand goals nodes  
  - Outputs: Single merged stream  
  - Requirements: Ensures both inputs arrive before merging  

---

#### 2.3 Campaign Goal Generation (AI Agent)

**Overview:**  
Invokes the Mistral AI language model to generate a refined campaign goal summary using the structured company and campaign data.

**Nodes Involved:**
- Campaign Goal generator
- Mistral Cloud Chat Model4
- Sticky Note1 (describes AI agent role)

**Node Details:**

- **Campaign Goal generator**  
  - Type: Langchain Agent (Mistral AI)  
  - Config: Uses company data as input text; system prompt instructs the AI to produce a structured marketing campaign goal summary including goals, audience, keywords, and success metrics.  
  - Inputs: Output from clean retrived data  
  - Outputs: Textual campaign goal summary  
  - AI Model: mistral-small-latest with temperature 0.9  
  - Edge Cases: AI generating incomplete or off-topic responses, timeout, API errors  

- **Mistral Cloud Chat Model4**  
  - Type: Langchain Mistral Cloud LM  
  - Role: Shared Mistral AI model node used by multiple agents (Campaign Goal and Image Prompt generators).  
  - Credentials: Mistral Cloud API key required  

- **Sticky Note1**  
  - Describes this block as “AI AGENT generated Campaign Goal”.

---

#### 2.4 Image Prompt Generation (AI Agent)

**Overview:**  
Generates five detailed image prompts, a caption, and hashtags using Mistral AI based on the campaign goal summary for visual storytelling.

**Nodes Involved:**
- image prompt generator base on the goal
- Structured Output Parser1
- separate each Prompts
- Sticky Note2

**Node Details:**

- **image prompt generator base on the goal**  
  - Type: Langchain Agent (Mistral AI)  
  - Config: Receives campaign goal summary text; system message instructs to produce JSON output with 5 image prompts, 1 caption, and 4-6 hashtags. Output is strictly structured JSON.  
  - Input: Text from Campaign Goal generator  
  - Output: JSON with image_prompts array, caption string, and hashtags array  
  - Output Parser: Structured Output Parser1 to enforce JSON schema and validate output  
  - Edge Cases: AI output formatting errors, missing fields, invalid JSON  

- **Structured Output Parser1**  
  - Type: Langchain Structured Output Parser  
  - Role: Validates AI output against expected JSON schema and extracts parsed JSON for further processing.  
  - Config: JSON schema example includes keys “image_prompts”, “caption”, and “hashtags”.  

- **separate each Prompts**  
  - Type: Set  
  - Config: Extracts each of the 5 image prompts into separate fields (prompt1 to prompt5) for API calls.  
  - Output: JSON with prompt1, prompt2, prompt3, prompt4, prompt5  

- **Sticky Note2**  
  - Describes this block as “AI AGENT generated prompt for image generation”.

---

#### 2.5 Image Generation

**Overview:**  
Uses Pollinations.ai API to generate five images concurrently, one for each prompt.

**Nodes Involved:**
- pollinations.ai
- pollinations.ai2
- pollinations.ai3
- pollinations.ai4
- pollinations.ai5
- Sticky Note3

**Node Details:**

- **pollinations.ai (and 2,3,4,5)**  
  - Type: HTTP Request  
  - Config: Makes GET requests to https://image.pollinations.ai/prompt/{{ promptX }} replacing promptX with each prompt string.  
  - Options: Retry on failure enabled, max 5 tries, onError set to continue with error output to avoid blocking workflow.  
  - Input: Individual prompt fields from separate each Prompts node  
  - Output: Binary image data in response  
  - Edge Cases: API request failures, rate limits, invalid prompt causing no image, network timeouts  

- **Sticky Note3**  
  - States “image is generated using pollinations.ai”.

---

#### 2.6 Image Naming and Merging

**Overview:**  
Renames each image’s binary data to a unique key for clarity and merges all images into a single item for uploading.

**Nodes Involved:**
- Change name to photo 1
- Change name to photo 2
- Change name to photo 3
- Change name to photo  4
- Change name to photo 5
- Merge image into one item
- Send as 1 merged file1
- Sticky Note4

**Node Details:**

- **Change name to photo [1-5]**  
  - Type: Code  
  - Function: Each node renames the binary data key from “data” to “photoX” where X corresponds to image index (1 to 5). This allows individual image identification after merging.  
  - Input: Binary image from each pollinations.ai node  
  - Output: Item with renamed binary key  
  - Edge Cases: Missing binary data, rename failures  

- **Merge image into one item**  
  - Type: Merge (Number inputs = 5)  
  - Function: Combines the five renamed image items into one single item containing all five binary images.  
  - Input: Outputs of all “Change name to photo X” nodes  
  - Output: Single item with 5 binary images  

- **Send as 1 merged file1**  
  - Type: Code  
  - Function: Final code node that merges all binary image fields into one item (redundant safety), ensuring all images are included in one payload for upload.  
  - Output: Single merged item with all photos in binary  

- **Sticky Note4**  
  - States “each image name is normalized”.

---

#### 2.7 File Upload

**Overview:**  
Uploads the merged image file containing all campaign images to Google Drive for storage and further use.

**Nodes Involved:**
- Upload file
- Sticky Note5

**Node Details:**

- **Upload file**  
  - Type: Google Drive (upload operation)  
  - Config: Uploads binary file item into specified Google Drive folder (My Drive root by default).  
  - Credentials: Google Drive OAuth2 account  
  - Input: Merged image file from “Send as 1 merged file1”  
  - Edge Cases: Auth errors, upload failures, quota exceeded  

- **Sticky Note5**  
  - States “save output to drive”.

---

### 3. Summary Table

| Node Name                   | Node Type                        | Functional Role                          | Input Node(s)                          | Output Node(s)                        | Sticky Note                                |
|-----------------------------|---------------------------------|----------------------------------------|--------------------------------------|-------------------------------------|--------------------------------------------|
| When clicking ‘Test workflow’| Manual Trigger                  | Trigger workflow start                  | -                                    | brand goals, brand profile           |                                            |
| brand profile               | Google Drive                    | Download brand profile data             | When clicking ‘Test workflow’         | Merge profile and goals              |                                            |
| brand goals                 | Google Drive                    | Download campaign goals data            | When clicking ‘Test workflow’         | Merge profile and goals              |                                            |
| Sticky Note                 | Sticky Note                    | Notes on brand data retrieval           | -                                    | -                                   | Get brand profile and goal saved in google drive |
| Merge profile and goals     | Merge                          | Combine brand profile and goals         | brand profile, brand goals            | clean retrived data                  |                                            |
| clean retrived data         | Code                           | Clean and structure input data          | Merge profile and goals               | Campaign Goal generator              |                                            |
| Campaign Goal generator     | Langchain Agent (Mistral AI)   | Generate campaign goal summary          | clean retrived data                   | image prompt generator base on the goal | AI AGENT  generated Campaign Goal           |
| Mistral Cloud Chat Model4   | Langchain LM (Mistral Cloud)   | Shared Mistral AI model node            | -                                    | Campaign Goal generator, image prompt generator base on the goal |                                            |
| image prompt generator base on the goal | Langchain Agent (Mistral AI) | Generate image prompts & caption        | Campaign Goal generator               | separate each Prompts               | AI AGENT  generated prompt for image generation |
| Structured Output Parser1   | Langchain Output Parser        | Validate & parse AI JSON output         | image prompt generator base on the goal | image prompt generator base on the goal |                                            |
| separate each Prompts       | Set                            | Extract 5 image prompts separately      | image prompt generator base on the goal | pollinations.ai, pollinations.ai2, pollinations.ai3, pollinations.ai4, pollinations.ai5 |                                            |
| pollinations.ai             | HTTP Request                   | Generate image #1                       | separate each Prompts                 | Change name to photo 1              | image is generated using pollinations.ai   |
| pollinations.ai2            | HTTP Request                   | Generate image #2                       | separate each Prompts                 | Change name to photo 2              | image is generated using pollinations.ai   |
| pollinations.ai3            | HTTP Request                   | Generate image #3                       | separate each Prompts                 | Change name to photo 3              | image is generated using pollinations.ai   |
| pollinations.ai4            | HTTP Request                   | Generate image #4                       | separate each Prompts                 | Change name to photo  4             | image is generated using pollinations.ai   |
| pollinations.ai5            | HTTP Request                   | Generate image #5                       | separate each Prompts                 | Change name to photo 5              | image is generated using pollinations.ai   |
| Change name to photo 1      | Code                           | Rename binary key to photo1             | pollinations.ai                      | Merge image into one item           | each image name is normalized               |
| Change name to photo 2      | Code                           | Rename binary key to photo2             | pollinations.ai2                     | Merge image into one item           | each image name is normalized               |
| Change name to photo 3      | Code                           | Rename binary key to photo3             | pollinations.ai3                     | Merge image into one item           | each image name is normalized               |
| Change name to photo  4     | Code                           | Rename binary key to photo4             | pollinations.ai4                     | Merge image into one item           | each image name is normalized               |
| Change name to photo 5      | Code                           | Rename binary key to photo5             | pollinations.ai5                     | Merge image into one item           | each image name is normalized               |
| Merge image into one item   | Merge                          | Merge all renamed images into one item | Change name to photo [1-5]           | Send as 1 merged file1              | each image name is normalized               |
| Send as 1 merged file1      | Code                           | Consolidate merged images for upload    | Merge image into one item             | Upload file                        | each image name is normalized               |
| Upload file                 | Google Drive                   | Upload merged image file to Drive       | Send as 1 merged file1                | -                                   | save output to drive                         |
| Sticky Note1                | Sticky Note                    | Notes on AI Campaign Goal generation    | -                                    | -                                   | AI AGENT  generated Campaign Goal           |
| Sticky Note2                | Sticky Note                    | Notes on AI Image Prompt generation     | -                                    | -                                   | AI AGENT  generated prompt for image generation |
| Sticky Note3                | Sticky Note                    | Notes on image generation source        | -                                    | -                                   | image is generated using pollinations.ai   |
| Sticky Note4                | Sticky Note                    | Notes on image naming                    | -                                    | -                                   | each image name is normalized               |
| Sticky Note5                | Sticky Note                    | Notes on upload step                     | -                                    | -                                   | save output to drive                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node:**
   - Name: `When clicking ‘Test workflow’`
   - Purpose: Start the workflow manually.

2. **Add two Google Drive nodes for input:**
   - `brand profile`:
     - Operation: Download
     - Credentials: Google Drive OAuth2
     - Parameter: Set file ID or URL to brand profile file in Google Drive.
   - `brand goals`:
     - Operation: Download
     - Credentials: Google Drive OAuth2
     - Parameter: Set file ID or URL to campaign goals file in Google Drive.
   - Connect both from the manual trigger node.

3. **Merge brand profile and brand goals:**
   - Add a Merge node `Merge profile and goals`
   - Configure to wait for both inputs before merging (default mode).
   - Connect `brand profile` and `brand goals` nodes to this merge node.

4. **Add a Code node `clean retrived data`:**
   - Purpose: Parse and structure input data into summaries, company profile, and campaign info.
   - Copy the JavaScript code that extracts and consolidates relevant JSON fields.
   - Connect the output of the merge node to this code node.

5. **Create Langchain Mistral Cloud LM node:**
   - Name: `Mistral Cloud Chat Model4`
   - Credentials: Provide Mistral Cloud API credentials.
   - Model: `mistral-small-latest`
   - Temperature: 0.9
   - This node will be reused by AI agent nodes.

6. **Add Langchain Agent node `Campaign Goal generator`:**
   - Set the input text to `{{$json.company}}`
   - Use the system message that defines the marketing strategist instructions for campaign goal generation.
   - Connect the `clean retrived data` node output to this node.
   - Assign `Mistral Cloud Chat Model4` as its language model.
   - Connect AI output to next step.

7. **Add Langchain Agent node `image prompt generator base on the goal`:**
   - Input: `{{$json.output}}` from `Campaign Goal generator`
   - System message: instruct to create 5 image prompts, 1 caption, hashtags in strict JSON format.
   - Enable structured output parsing.
   - Assign `Mistral Cloud Chat Model4` as LM.
   - Connect its output to Structured Output Parser node.

8. **Add Langchain Structured Output Parser node `Structured Output Parser1`:**
   - Use JSON schema for image prompts, caption, hashtags.
   - Connect output back to `image prompt generator base on the goal` node.

9. **Add Set node `separate each Prompts`:**
   - Define variables `prompt1` to `prompt5` extracting each prompt from parsed output JSON.
   - Connect from `image prompt generator base on the goal`.

10. **Add five HTTP Request nodes `pollinations.ai` to `pollinations.ai5`:**
    - For each, configure:
      - Method: GET
      - URL: `https://image.pollinations.ai/prompt/{{ $json.promptX }}` where X=1 to 5
      - Retry on fail: enabled, max 5 tries
      - On error: Continue
    - Connect each to `separate each Prompts` node.

11. **Add five Code nodes `Change name to photo 1` through `Change name to photo 5`:**
    - Each node renames the binary data key from `data` to `photoX` accordingly.
    - Copy JavaScript code that maps binary data key rename.
    - Connect each HTTP Request node to corresponding Code node.

12. **Add Merge node `Merge image into one item`:**
    - Set number of inputs to 5.
    - Connect outputs of all five `Change name to photo X` nodes into this merge node.

13. **Add Code node `Send as 1 merged file1`:**
    - JavaScript code to merge all binary data into a single item.
    - Connect from `Merge image into one item`.

14. **Add Google Drive node `Upload file`:**
    - Operation: Upload file
    - Credentials: Google Drive OAuth2
    - Drive: My Drive, Folder: root (or any target folder)
    - Connect from `Send as 1 merged file1`.

15. **Add Sticky Notes as described at relevant points for clarity and documentation.**

---

### 5. General Notes & Resources

| Note Content                                                                        | Context or Link                                             |
|-------------------------------------------------------------------------------------|-------------------------------------------------------------|
| Workflow uses Mistral AI via Langchain integration for advanced text generation.     | Requires valid Mistral Cloud API credentials.                |
| Images are generated using Pollinations.ai, which can sometimes fail or time out.   | https://image.pollinations.ai                               |
| Google Drive OAuth2 credentials are required for downloading input files and saving output images. | Setup in n8n credentials manager                              |
| The workflow includes retry and error continuation settings to improve robustness.  | HTTP Request nodes with retryOnFail and continueErrorOutput |
| Structured Output Parser node ensures AI JSON outputs conform to expected schema.   | Prevents downstream errors due to malformed AI responses.   |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or restricted elements. All data processed is legal and public.

---