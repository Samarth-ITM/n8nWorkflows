Generate and Quality Control Images with DALL-E 2 and Human Review Loop

https://n8nworkflows.xyz/workflows/generate-and-quality-control-images-with-dall-e-2-and-human-review-loop-7307


# Generate and Quality Control Images with DALL-E 2 and Human Review Loop

---

### 1. Workflow Overview

This n8n workflow automates the generation and quality control of images using OpenAI’s DALL-E 2 model combined with a human review process powered by GotoHuman. Its primary use case is to create images from textual prompts, have them reviewed by human evaluators, and if rejected, regenerate the images for a second round of review. The workflow ensures high-quality image outputs through an iterative AI-human feedback loop.

**Logical Blocks:**

- **1.1 Input & Initialization**: Manual trigger and prompt definition for image generation.
- **1.2 Batch Processing Setup**: Handling possible multiple prompts or items in batches.
- **1.3 Initial Image Generation**: Creating images from prompts with DALL-E 2.
- **1.4 Initial Human Review**: Sending generated images to GotoHuman for evaluation.
- **1.5 Conditional Logic for Rejected Images**: Checking review results to decide if regeneration is needed.
- **1.6 Second Image Generation**: Regenerating images if initially rejected, using feedback prompt.
- **1.7 Second Human Review**: Final human approval after regeneration.
- **1.8 Support & Documentation**: Sticky notes providing guidance and contact info.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Initialization

- **Overview:**  
  Starts the workflow manually and sets the initial prompt and image name for generation.

- **Nodes Involved:**  
  - Start Workflow  
  - Set Image Prompt

- **Node Details:**  
  - **Start Workflow**  
    - Type: Manual Trigger  
    - Role: Entry point to manually start the workflow  
    - Configuration: No parameters needed  
    - Inputs: None  
    - Outputs: Connects to "Set Image Prompt"  
    - Edge Cases: None  
    - Version: 1

  - **Set Image Prompt**  
    - Type: Set  
    - Role: Define the image generation prompt and image name variables for downstream use  
    - Configuration: Sets two string fields:  
      - `Prompt`: "Make an image of an attractive person standing in new york city"  
      - `Name`: "woman-nyc"  
    - Inputs: From "Start Workflow"  
    - Outputs: Connects to "Loop Over Items1"  
    - Edge Cases: If prompt or name is empty or malformed, image generation may fail or be irrelevant  
    - Version: 3.4

---

#### 2.2 Batch Processing Setup

- **Overview:**  
  Prepares to process multiple items or prompts in batches, enabling scalability and parallelism.

- **Nodes Involved:**  
  - Loop Over Items1

- **Node Details:**  
  - **Loop Over Items1**  
    - Type: SplitInBatches  
    - Role: Processes input data in batches; useful if multiple prompts exist (though current setup uses one prompt)  
    - Configuration: Default, no reset, processes items sequentially per batch  
    - Inputs: From "Set Image Prompt"  
    - Outputs: Connects to "Initial Image Generation"  
    - Edge Cases: Empty input array leads to no processing; batch size not defined explicitly (defaults apply)  
    - Version: 3

---

#### 2.3 Initial Image Generation

- **Overview:**  
  Generates images using OpenAI’s DALL-E 2 based on the initial prompt.

- **Nodes Involved:**  
  - Initial Image Generation

- **Node Details:**  
  - **Initial Image Generation**  
    - Type: OpenAI (LangChain integration)  
    - Role: Calls OpenAI API to create images from text prompt  
    - Configuration:  
      - Model: "dall-e-2"  
      - Prompt: Uses expression `{{$json.Prompt}}` from Set Image Prompt node  
      - Options: Requests returned image URLs  
    - Credentials: OpenAI API key ("OpenAi account")  
    - Inputs: From "Loop Over Items1"  
    - Outputs: Connects to "Initial Review"  
    - Edge Cases: API errors (rate limits, invalid keys), malformed prompt, network issues  
    - Version: 1.8

---

#### 2.4 Initial Human Review

- **Overview:**  
  Sends generated images and prompt data to GotoHuman platform for human quality control.

- **Nodes Involved:**  
  - Initial Review

- **Node Details:**  
  - **Initial Review**  
    - Type: GotoHuman Review Node  
    - Role: Creates a review task with image and prompt for human reviewers  
    - Configuration:  
      - Maps image URL as an array of objects with URL and label "Generated Image"  
      - Includes prompt text from "Set Image Prompt" node  
      - Uses review template ID: "3473LaRDbdf03sd6uzYG" (configured in GotoHuman dashboard)  
    - Credentials: GotoHuman API ("gotoHuman account")  
    - Inputs: From "Initial Image Generation"  
    - Outputs: Connects to "If rejected" conditional node  
    - Edge Cases: API auth errors, template misconfiguration, webhook failures  
    - Version: 1  
    - Webhook: Waits for human review callback via webhook ID

---

#### 2.5 Conditional Logic for Rejected Images

- **Overview:**  
  Evaluates human review results to determine if the image was rejected and needs regeneration.

- **Nodes Involved:**  
  - If rejected

- **Node Details:**  
  - **If rejected**  
    - Type: If (conditional)  
    - Role: Checks if the review response equals "rejected"  
    - Configuration:  
      - Condition: `$json.response == "rejected"` (strict string equality)  
    - Inputs: From "Initial Review"  
    - Outputs:  
      - True: Connects to "Second Image Generation" for regeneration  
      - False: No connection (workflow ends or can be extended)  
    - Edge Cases: Unexpected or missing response fields; case sensitivity may cause logic errors  
    - Version: 2.2

---

#### 2.6 Second Image Generation

- **Overview:**  
  Generates a new image based on prompt feedback from the initial review, enabling iterative refinement.

- **Nodes Involved:**  
  - Second Image Generation

- **Node Details:**  
  - **Second Image Generation**  
    - Type: OpenAI (LangChain integration)  
    - Role: Calls OpenAI's DALL-E 2 again with updated prompt from review feedback  
    - Configuration:  
      - Model: "dall-e-2"  
      - Prompt: Uses prompt string from `$('Initial Review').item.json.responseValues.Prompt.value` (reviewed prompt)  
      - Options: Requests image URLs  
    - Credentials: Same OpenAI API account as initial generation  
    - Inputs: From "If rejected" node (true branch)  
    - Outputs: Connects to "Second Review"  
    - Edge Cases: If review feedback is missing or malformed, generation may fail or produce irrelevant images  
    - Version: 1.8

---

#### 2.7 Second Human Review

- **Overview:**  
  Sends regenerated images for final human approval to ensure quality before acceptance.

- **Nodes Involved:**  
  - Second Review

- **Node Details:**  
  - **Second Review**  
    - Type: GotoHuman Review Node  
    - Role: Creates a second review task using the same review template and fields as initial review  
    - Configuration:  
      - Maps regenerated image URLs as array labeled "Generated Image"  
      - Uses original prompt from "Set Image Prompt" node  
      - Review Template ID: same as initial review ("3473LaRDbdf03sd6uzYG")  
    - Credentials: Same GotoHuman API account  
    - Inputs: From "Second Image Generation"  
    - Outputs: None (workflow ends here)  
    - Edge Cases: Same as initial review  
    - Version: 1  
    - Webhook: Waits for human review callback

---

#### 2.8 Support & Documentation

- **Overview:**  
  Provides embedded guidance and contact info as sticky notes within the workflow editor.

- **Nodes Involved:**  
  - Sticky Note2  
  - Sticky Note3  
  - Sticky Note4

- **Node Details:**  
  - **Sticky Note2**  
    - Content:  
      "**Need more help?**  \nWebsite: https://ynteractive.com  \nEmail: robert@ynteractive.com"  
    - Role: Quick support contact info  
    - Position: Visual aid only; no workflow logic

  - **Sticky Note3**  
    - Content:  
      Step-by-step implementation guide including prerequisites, credential setup instructions, review template configuration, and node-specific configuration details.  
    - Role: Detailed setup instructions for users importing this workflow

  - **Sticky Note4**  
    - Content:  
      Detailed explanations for key nodes in the workflow, configuration summaries, and testing instructions.  
    - Role: Operational guidance and customization tips

---

### 3. Summary Table

| Node Name             | Node Type                  | Functional Role                         | Input Node(s)        | Output Node(s)           | Sticky Note                                                                                     |
|-----------------------|----------------------------|---------------------------------------|----------------------|--------------------------|------------------------------------------------------------------------------------------------|
| Start Workflow        | Manual Trigger             | Entry point to start workflow          | -                    | Set Image Prompt          |                                                                                                |
| Set Image Prompt      | Set                        | Defines initial prompt and image name  | Start Workflow       | Loop Over Items1          | See Sticky Note3 for detailed setup instructions                                              |
| Loop Over Items1      | SplitInBatches             | Processes inputs in batches             | Set Image Prompt     | Initial Image Generation  | See Sticky Note4 for configuration summary                                                    |
| Initial Image Generation | OpenAI (LangChain)       | Generates images with DALL-E 2          | Loop Over Items1     | Initial Review            | See Sticky Note4 for configuration summary                                                    |
| Initial Review        | GotoHuman Review           | Sends images for initial human review  | Initial Image Generation | If rejected             | See Sticky Note4 for configuration summary                                                    |
| If rejected           | If                         | Checks if image was rejected            | Initial Review       | Second Image Generation (if true) | See Sticky Note4 for configuration summary                                         |
| Second Image Generation | OpenAI (LangChain)        | Regenerates images based on review feedback | If rejected (true) | Second Review             | See Sticky Note4 for configuration summary                                                    |
| Second Review         | GotoHuman Review           | Sends regenerated images for final approval | Second Image Generation | -                    | See Sticky Note4 for configuration summary                                                    |
| Sticky Note2          | Sticky Note                | Provides support contact info           | -                    | -                        | "**Need more help?**  Website: https://ynteractive.com  Email: robert@ynteractive.com"          |
| Sticky Note3          | Sticky Note                | Setup and implementation instructions  | -                    | -                        | Step-by-step implementation guide, prerequisites, and credential setup instructions           |
| Sticky Note4          | Sticky Note                | Node configuration explanations        | -                    | -                        | Detailed node descriptions and testing instructions                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new n8n workflow.**

2. **Add "Start Workflow" node (Manual Trigger):**  
   - No configuration needed.  
   - This is the entry point to start the process manually.

3. **Add "Set Image Prompt" node (Set):**  
   - Add two string fields:  
     - `Prompt`: "Make an image of an attractive person standing in new york city"  
     - `Name`: "woman-nyc"  
   - Connect "Start Workflow" output to this node’s input.

4. **Add "Loop Over Items1" node (SplitInBatches):**  
   - No special configuration; default batch size applies.  
   - Connect "Set Image Prompt" output to this node’s input.

5. **Add "Initial Image Generation" node (OpenAI LangChain):**  
   - Set Resource to "Image".  
   - Set Model to "dall-e-2".  
   - Set Prompt to expression: `{{$json.Prompt}}`.  
   - Enable option to return image URLs.  
   - Set credentials to your OpenAI API credential ("OpenAi account").  
   - Connect "Loop Over Items1" output to this node’s input.

6. **Add "Initial Review" node (GotoHuman Review):**  
   - Set Review Template ID to "3473LaRDbdf03sd6uzYG" (preconfigured in GotoHuman).  
   - Map fields as follows:  
     - `Image`: Use expression to JSON.stringify an array of image URL objects:  
       ```  
       JSON.stringify([{url: $json.url, label: 'Generated Image'}])  
       ```  
     - `Prompt`: Pull from "Set Image Prompt" node using expression: `{{ $('Set Image Prompt').item.json.Prompt }}`  
   - Set credentials to your GotoHuman API credential ("gotoHuman account").  
   - Connect "Initial Image Generation" output to this node’s input.  
   - Ensure webhook ID is set for review callback.

7. **Add "If rejected" node (If):**  
   - Add condition:  
     - Check if `$json.response` equals "rejected" (strict string comparison).  
   - Connect "Initial Review" output to this node’s input.

8. **Add "Second Image Generation" node (OpenAI LangChain):**  
   - Same configuration as "Initial Image Generation" node, except:  
     - Prompt is set to expression: `{{ $('Initial Review').item.json.responseValues.Prompt.value }}` — updated prompt from review feedback.  
   - Connect "If rejected" TRUE output to this node’s input.

9. **Add "Second Review" node (GotoHuman Review):**  
   - Same configuration as "Initial Review" node.  
   - Map regenerated image URLs similarly.  
   - Use same review template ID.  
   - Connect "Second Image Generation" output to this node’s input.  
   - Configure webhook for final review callback.

10. **(Optional) Add Sticky Notes for documentation:**  
    - Add notes with support info, step-by-step instructions, and node summaries as inline documentation for users.

11. **Verify all credentials:**  
    - OpenAI API credential setup with your API key.  
    - GotoHuman API credential with your API key.

12. **Test the entire workflow:**  
    - Execute manually from "Start Workflow".  
    - Confirm images are generated, sent for review, and conditional regeneration works when images are rejected.

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                          |
|-----------------------------------------------------------------------------------------------------------------------------|-----------------------------------------|
| **Need more help?** Visit https://ynteractive.com or email robert@ynteractive.com for support.                              | Support contact info (Sticky Note2)     |
| Detailed step-by-step setup guide including prerequisites, credential creation, and workflow import instructions.           | Sticky Note3 content                     |
| Node configuration summary and testing instructions to customize prompts, reviews, and notifications.                       | Sticky Note4 content                     |
| GotoHuman review template ID: `3473LaRDbdf03sd6uzYG` — must be created and configured in GotoHuman dashboard prior to use.  | Integration detail                      |
| OpenAI DALL-E 2 model requires appropriate API access and quota from OpenAI platform.                                        | API access requirement                   |
| Human review webhooks require n8n instance to be publicly accessible or tunneled for callbacks from GotoHuman.               | Operational requirement                  |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.

---