Create Character-Consistent Images with FLUX Kontext & Post to Social via Upload Post

https://n8nworkflows.xyz/workflows/create-character-consistent-images-with-flux-kontext---post-to-social-via-upload-post-4798


# Create Character-Consistent Images with FLUX Kontext & Post to Social via Upload Post

### 1. Workflow Overview

This workflow automates the creation of character-consistent images using the FLUX Kontext API and subsequently posts the generated images to social media platforms via Upload Post. It targets use cases where a base image and a set of related prompts are used to iteratively generate variations or scenes featuring the same character, preserving consistent visual features. The workflow is structured into the following logical blocks:

- **1.1 Input Reception and Initialization:** Accepts manual or external workflow triggers to start the process, loads the initial character image from a GitHub repository, and defines an array of prompts for iterative image generation.
- **1.2 Iterative Image Generation with FLUX Kontext:** For each prompt, sends the initial or previously generated image to the FLUX Kontext API to produce new images that maintain character consistency across different scenarios.
- **1.3 Polling and Status Checking:** Waits and checks the status of each FLUX Kontext image generation job to ensure readiness before proceeding.
- **1.4 Image Retrieval and Management:** Downloads and manages the generated images, transforming binary data as needed to prepare for the next iteration or final output.
- **1.5 Posting Images to Social Media via Upload Post:** After completing all iterations, posts the collection of generated images to social media through the Upload Post service.
- **1.6 Workflow Control and Merging:** Manages data merging, iteration control, and branching logic to connect all processing steps.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

**Overview:**  
This block initiates workflow execution either manually or when triggered by another workflow. It loads the initial mascot image from a GitHub repository and sets up the array of prompts to be used in iterative image generation.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- When Executed by Another Workflow (ExecuteWorkflowTrigger)  
- Get File from GitHub (GitHub node)  
- Download Initial Image (HTTP Request)  
- All Prompts (Set node)  
- Number of Steps (Set node)  
- Merge (Merge node)  
- Sticky Note, Sticky Note2, Sticky Note4, Sticky Note5, Sticky Note6, Sticky Note7, Sticky Note8 (Documentation notes)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point for manual execution  
  - Inputs: None  
  - Outputs: Triggers downstream nodes to start the workflow  
  - Failure cases: None typical, manual trigger only

- **When Executed by Another Workflow**  
  - Type: ExecuteWorkflowTrigger  
  - Role: Allows this workflow to be triggered and receive data from another workflow  
  - Parameters: Pass-through input source  
  - Inputs: External workflow inputs with image binary data and prompt string  
  - Outputs: Passes input data downstream  
  - Edge cases: Missing or malformed input data can cause downstream failures

- **Get File from GitHub**  
  - Type: GitHub node  
  - Role: Downloads the base mascot image from a specified GitHub repository and path  
  - Configuration: Owner = "teds-tech-talks", Repository = "n8n-community-leaderboard", File path = "_creators/eduard/mascot.png"  
  - Credentials: OAuth2 GitHub account  
  - Inputs: Trigger from manual execution  
  - Outputs: JSON containing download URL of the image  
  - Failures: Auth errors, file not found, rate limiting

- **Download Initial Image**  
  - Type: HTTP Request  
  - Role: Downloads the actual image file from the GitHub download URL  
  - Configuration: Response format set to file, output property "data0"  
  - Inputs: URL from previous node  
  - Outputs: Binary image data  
  - Failures: Network errors, invalid URL, timeouts

- **All Prompts**  
  - Type: Set node  
  - Role: Defines an array of 3 prompts describing different scenes or poses for the mascot character  
  - Parameters: Array assigned to field "Prompts"  
  - Inputs: Triggered after image download  
  - Outputs: JSON with "Prompts" array field  
  - Notes: Prompts preserve character style and features

- **Number of Steps**  
  - Type: Set node  
  - Role: Defines the number of iterations for the workflow, limited to maximum 5 (or length of prompts array, whichever is smaller)  
  - Parameters: Number assigned to field "Steps" using an expression `min(Prompts.length,5)`  
  - Inputs: From All Prompts  
  - Outputs: JSON with "Steps" field  
  - Edge cases: If prompts array is empty, Steps will be 0

- **Merge**  
  - Type: Merge node  
  - Role: Combines outputs from downloading image and setting steps/prompts into a single JSON stream  
  - Configuration: Combine mode, include unpaired data, combine by position  
  - Inputs: Download Initial Image, Number of Steps  
  - Outputs: Combined data for next processing  
  - Failures: Misaligned data, missing inputs

- **Sticky Notes (1,2,4,5,6,7,8)**  
  - Purpose: Provide documentation and visual guidance inside the workflow editor, including example images and prompt definitions.

---

#### 2.2 Iterative Image Generation with FLUX Kontext

**Overview:**  
This block sends the current image and prompt to the FLUX Kontext API to generate a new image. It iterates over the array of prompts, producing a sequence of images preserving character consistency.

**Nodes Involved:**  
- Current Step (Set)  
- FLUX Kontext (HTTP Request)  
- Wait 2 sec (Wait)  
- Check FLUX status (HTTP Request)  
- Is Ready? (If)  
- Get Image (HTTP Request)  
- Merge3 (Merge)  
- Run FLUX (Execute Workflow)  
- Sticky Note1 (Documentation)  
- Sticky Note4 (Documentation)

**Node Details:**

- **Current Step**  
  - Type: Set node  
  - Role: Selects the current prompt and binary image property names based on the iteration index ($runIndex)  
  - Parameters:  
    - "prompt" assigned from Prompts array at $runIndex  
    - "binaryin" and "binaryout" are dynamic property names for input/output image binaries (e.g., data0, data1)  
    - "currentstep" stores current iteration index  
  - Inputs: From If node controlling loop  
  - Outputs: JSON with prompt and binary keys for FLUX Kontext  
  - Edge cases: Index out of bounds if loop control fails

- **FLUX Kontext**  
  - Type: HTTP Request  
  - Role: Submits the current image and prompt to FLUX Kontext API for image generation  
  - Method: POST  
  - URL: https://api.bfl.ml/v1/flux-kontext-pro  
  - Authentication: HTTP Header Auth with bfl-FLUX credentials  
  - Body parameters:  
    - input_image: binary data from previous iteration or initial image  
    - prompt: current prompt string  
    - prompt_upsampling: false (hardcoded)  
    - output_format: png  
    - aspect_ratio: 1:1 (note: key has a leading space, could be a typo)  
  - Inputs: Binary image and prompt from Current Step  
  - Outputs: JSON containing job id and initial status  
  - Failures: API auth failure, invalid input, network issues, API rate limits, malformed body parameters

- **Wait 2 sec**  
  - Type: Wait node  
  - Role: Pauses workflow for 2 seconds to allow FLUX Kontext processing  
  - Inputs: From FLUX Kontext or status check node  
  - Outputs: Triggers status polling  
  - Failures: None typical, but long delays may slow execution

- **Check FLUX status**  
  - Type: HTTP Request  
  - Role: Queries FLUX API for job status using job id  
  - URL: https://api.bfl.ml/v1/get_result  
  - Method: GET with query parameter "id" from job id  
  - Authentication: HTTP Header Auth with bfl-FLUX credentials  
  - Outputs: JSON with updated job status and result when ready  
  - Failures: API errors, invalid job id, network errors

- **Is Ready?**  
  - Type: If node  
  - Role: Evaluates if the FLUX job status is "Ready" or the iteration count exceeds 5 to break loop  
  - Conditions:  
    - status == "Ready" OR $runIndex > 5  
  - Outputs:  
    - True: proceed to download the generated image  
    - False: wait and poll again  
  - Edge cases: Infinite loop if status never reaches "Ready", excessive retries

- **Get Image**  
  - Type: HTTP Request  
  - Role: Downloads the generated image file after job is ready  
  - URL: Taken from FLUX API result sample field  
  - Response: File binary with dynamic output property name  
  - Inputs: From Is Ready? True branch  
  - Failures: Network errors, invalid URL

- **Merge3**  
  - Type: Merge node  
  - Role: Combines current step data with previous step outputs for next iteration  
  - Configuration: Combine mode, by position  
  - Inputs: From Current Step iteration and Run FLUX execution  
  - Outputs: Combined data stream for loop continuation

- **Run FLUX**  
  - Type: Execute Workflow  
  - Role: Invokes this workflow recursively or another workflow that handles FLUX Kontext iterations  
  - Parameters: Workflow reference to self, no additional inputs mapped  
  - Edge cases: Recursive call depth, infinite loops

- **Sticky Note1 and Sticky Note4**  
  - Documentation notes explaining image processing and iterative prompt usage.

---

#### 2.3 Image Retrieval and Management

**Overview:**  
Handles conversion of binary image data for API input, and manages the flow of images between iterations.

**Nodes Involved:**  
- Image to Base64 (Extract from File)  
- Sticky Note2 (Documentation)

**Node Details:**

- **Image to Base64**  
  - Type: ExtractFromFile  
  - Role: Converts binary image data to a Base64 string property for use as API input  
  - Parameters:  
    - Operation: binaryToProperty  
    - Destination key and binary property name dynamically assigned based on incoming data (inherited from input workflow)  
  - Inputs: From external workflow trigger node  
  - Outputs: JSON with Base64 string representation of image  
  - Failures: Missing binary data, conversion errors

---

#### 2.4 Posting Images to Social Media via Upload Post

**Overview:**  
After completing all FLUX Kontext iterations, posts the generated images to social media through Upload Post, using the community node.

**Nodes Involved:**  
- Upload Post (Upload Post node)  
- Sticky Note5 (Documentation)

**Node Details:**

- **Upload Post**  
  - Type: Upload Post community node  
  - Role: Posts multiple images to social media platform(s)  
  - Parameters:  
    - user: "Ed" (hardcoded)  
    - title: Descriptive text about the upload  
    - photos: Dynamic array of image binary property names generated during iterations (e.g., data1, data2, ...)  
    - platform: ["x"] (platform specification, "x" likely placeholder)  
  - Credentials: Upload Post API credentials  
  - Inputs: From If node after loop completion  
  - Failures: Auth errors, platform API errors, missing image data

---

#### 2.5 Workflow Control and Data Merging

**Overview:**  
Manages the data flow and control logic between nodes, including loop continuation and branching.

**Nodes Involved:**  
- If (Loop control)  
- Merge (Data combination)  
- Merge3 (Iteration data combination)

**Node Details:**

- **If**  
  - Type: If node  
  - Role: Controls the iteration loop, continues if current $runIndex < Steps (number of prompts)  
  - Conditions: $runIndex < Steps  
  - Outputs:  
    - True: continue next iteration through Current Step and Run FLUX  
    - False: proceed to upload images  
  - Failures: Incorrect loop exit if Steps or $runIndex miscalculated

- **Merge** and **Merge3**  
  - Combine JSON data streams to maintain state and pass images/prompts between iterations

---

### 3. Summary Table

| Node Name               | Node Type                        | Functional Role                                    | Input Node(s)                      | Output Node(s)                         | Sticky Note                                                                                      |
|-------------------------|---------------------------------|--------------------------------------------------|----------------------------------|--------------------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                  | Manual start of workflow                          | None                             | Get File from GitHub, All Prompts    |                                                                                                |
| Get File from GitHub     | GitHub                          | Downloads initial mascot image metadata           | When clicking ‘Execute workflow’ | Download Initial Image                |                                                                                                |
| Download Initial Image   | HTTP Request                   | Downloads mascot image binary file                | Get File from GitHub             | Merge                               |                                                                                                |
| All Prompts             | Set                            | Defines array of prompts for iterations            | When clicking ‘Execute workflow’ | Number of Steps                     | ## Define prompts here Prepare an array of prompts that will be used one by one on the next steps. Update limit in the `Number of Steps` node if you need more than 5 iterations. |
| Number of Steps         | Set                            | Limits number of iterations based on prompts      | All Prompts                     | Merge                               |                                                                                                |
| Merge                   | Merge                          | Combines image and prompt data                     | Download Initial Image, Number of Steps | If                              |                                                                                                |
| If                      | If                             | Loop control based on current step vs. steps      | Merge                          | Current Step (True), Upload Post (False) |                                                                                                |
| Current Step            | Set                            | Picks prompt and binary property names per iteration | If (True)                      | Run FLUX, Merge3                     | ## Iterate over prompts - On each step a next prompt it taken from the original array - Outputs from the previous FLUX Kontext request are moved forward to the subsequent generation |
| Run FLUX                | Execute Workflow               | Executes FLUX Kontext image generation workflow   | Current Step                   | Merge3                             |                                                                                                |
| Merge3                  | Merge                          | Combines iteration data for next cycle             | Current Step, Run FLUX          | If                                |                                                                                                |
| FLUX Kontext            | HTTP Request                   | Sends image and prompt to FLUX Kontext API        | Image to Base64                | Wait 2 sec                        | # Image processing part                                                                         |
| Wait 2 sec              | Wait                           | Waits 2 seconds for FLUX job processing            | FLUX Kontext, Is Ready? (False) | Check FLUX status                  |                                                                                                |
| Check FLUX status       | HTTP Request                   | Polls FLUX API for job status                       | Wait 2 sec                    | Is Ready?                        |                                                                                                |
| Is Ready?               | If                             | Checks if FLUX job is ready or exceeded attempts   | Check FLUX status             | Get Image (True), Wait 2 sec (False) |                                                                                                |
| Get Image               | HTTP Request                   | Downloads generated image file                      | Is Ready? (True)              | Merge                            |                                                                                                |
| Image to Base64         | ExtractFromFile                | Converts binary image to Base64 string for FLUX    | When Executed by Another Workflow | FLUX Kontext                    | ## Load the initial image                                                                       |
| When Executed by Another Workflow | ExecuteWorkflowTrigger        | Allows external workflow to trigger this workflow  | External                       | Image to Base64                   |                                                                                                |
| Upload Post             | Upload Post                    | Posts generated images to social media             | If (False)                    | None                             | ## [Post several images via Upload Post](https://www.upload-post.com/?linkId=lp_144414&sourceId=post-now&tenantId=upload-post-app) [Click to create your own account](https://www.upload-post.com/?linkId=lp_144414&sourceId=post-now&tenantId=upload-post-app) |
| Sticky Notes (1-8)       | Sticky Note                    | Documentation and visual references                 | Various                       | None                             | See notes inline in nodes section                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Node: Manual Trigger (When clicking ‘Execute workflow’)  
   - Purpose: Start workflow manually. No parameters needed.

2. **Create GitHub Node (Get File from GitHub)**  
   - Operation: Get file  
   - Owner: "teds-tech-talks"  
   - Repository: "n8n-community-leaderboard"  
   - File Path: "_creators/eduard/mascot.png"  
   - Authentication: OAuth2 with GitHub credentials  
   - Connect manual trigger node to this node.

3. **Create HTTP Request Node (Download Initial Image)**  
   - Method: GET  
   - URL: Use expression `{{$json.download_url}}` from previous node  
   - Response Format: File  
   - Output Property: data0  
   - Connect GitHub node to this node.

4. **Create Set Node (All Prompts)**  
   - Add field: Prompts (Array)  
   - Value: Array of 3 prompt strings describing mascot in different scenarios  
   - Connect manual trigger node to this node as well.

5. **Create Set Node (Number of Steps)**  
   - Add field: Steps (Number)  
   - Value: Expression `={{ Math.min($json.Prompts.length,5) }}`  
   - Connect All Prompts node to this node.

6. **Create Merge Node (Merge)**  
   - Mode: Combine  
   - Options: Include unpaired, Combine by position  
   - Connect Download Initial Image (output 1) and Number of Steps (output 0) to Merge.

7. **Create If Node (Loop Control)**  
   - Condition: $runIndex < Steps (Number)  
   - Use Expression, version 2 condition  
   - Connect Merge node to If node.

8. **Create Set Node (Current Step)**  
   - Fields:  
     - prompt: `={{ $json.Prompts[$runIndex] }}`  
     - binaryin: `=data{{$runIndex}}`  
     - binaryout: `=data{{ Number($runIndex)+1 }}`  
     - currentstep: `={{ $runIndex }}`  
   - Include fields "Prompts" and "Steps"  
   - Connect If node True output to this node.

9. **Create Execute Workflow Node (Run FLUX)**  
   - Workflow ID: This workflow itself (for recursive iteration)  
   - Connect Current Step node output to Run FLUX.

10. **Create Merge Node (Merge3)**  
    - Mode: Combine  
    - Connect Current Step (output 1) and Run FLUX (output 0) to Merge3.

11. **Connect Merge3 output back to If node input for loop continuation.**

12. **Create HTTP Request Node (FLUX Kontext)**  
    - Method: POST  
    - URL: https://api.bfl.ml/v1/flux-kontext-pro  
    - Authentication: HTTP Header Auth with bfl-FLUX credentials  
    - Body Parameters:  
      - input_image: `={{ $json[$('When Executed by Another Workflow').first().json.binaryin] }}` (binary data from previous step)  
      - prompt: `={{ $('When Executed by Another Workflow').first().json.prompt }}`  
      - prompt_upsampling: false  
      - output_format: png  
      - aspect_ratio: 1:1 (note to check for leading spaces in key)  
    - Connect Image to Base64 node output to this node.

13. **Create Wait Node (Wait 2 sec)**  
    - Wait time: 2 seconds  
    - Connect FLUX Kontext output to Wait node.

14. **Create HTTP Request Node (Check FLUX status)**  
    - Method: GET  
    - URL: https://api.bfl.ml/v1/get_result  
    - Query Parameter: id = `={{ $json.id }}` from FLUX Kontext output  
    - Authentication: HTTP Header Auth with bfl-FLUX credentials  
    - Connect Wait 2 sec output to this node.

15. **Create If Node (Is Ready?)**  
    - Conditions:  
      - status == "Ready" OR $runIndex > 5  
    - Connect Check FLUX status output to this node.

16. **Create HTTP Request Node (Get Image)**  
    - Method: GET  
    - URL: `={{ $json.result.sample }}` (download URL of generated image)  
    - Response Format: File  
    - Output Property: dynamic, from input workflow's binaryout field  
    - Connect Is Ready? True output to this node.

17. **Create Extract From File Node (Image to Base64)**  
    - Operation: binaryToProperty  
    - Destination key and binary property name: use dynamic expressions from external workflow input fields  
    - Connect When Executed by Another Workflow node output to this node.

18. **Create ExecuteWorkflowTrigger Node (When Executed by Another Workflow)**  
    - Input Source: Passthrough  
    - Connect external trigger to this node.

19. **Create Upload Post Node (Upload Post)**  
    - User: "Ed"  
    - Title: "Testing n8n uploads with Upload Post community node hotfix and FLUX1. Kontext"  
    - Photos: `={{ Array.from({length: $json.Steps}, (_, i) => `data${i + 1}`).join(',') }}` (dynamic list of generated images)  
    - Platform: ["x"] (replace with actual platform identifiers)  
    - Credentials: Upload Post API credentials  
    - Connect If node False output (loop finished) to this node.

20. **Add Sticky Notes to document each functional block and example images as desired for clarity.**

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                                                      |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| The workflow uses FLUX Kontext API to generate images preserving character consistency across prompts. | https://api.bfl.ml/v1/flux-kontext-pro                                                                                             |
| Upload Post community node is used to post images to social media.                                  | https://www.upload-post.com/?linkId=lp_144414&sourceId=post-now&tenantId=upload-post-app                                            |
| Initial mascot image is sourced from n8n-community-leaderboard GitHub repository.                   | https://github.com/teds-tech-talks/n8n-community-leaderboard/blob/main/_creators/eduard/mascot.png                                  |
| The workflow limits iterations to a maximum of 5 steps to control execution time and API usage.    |                                                                                                |
| The workflow includes recursive calls to itself for iterative image generation.                     |                                                                                                |
| Potential API request failures include auth errors, rate limits, and network timeouts.             |                                                                                                |
| Pay attention to the "aspect_ratio" parameter in FLUX Kontext node; a leading space might cause issues. |                                                                                                |
| Sticky notes include example images from Twitter showcasing step progression.                       | https://pbs.twimg.com/media/Gs7SwU_XYAASZBn?format=jpg&name=medium (Step 1) and similar for Steps 2 and 3                            |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.