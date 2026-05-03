Create an image procedurally using Bannerbear

https://n8nworkflows.xyz/workflows/create-an-image-procedurally-using-bannerbear-544


# Create an image procedurally using Bannerbear

### 1. Workflow Overview

This workflow demonstrates how to create an image procedurally using the Bannerbear API within n8n. It serves as a companion example for the Bannerbear node documentation, showcasing a simple trigger-to-image generation flow. The workflow consists of two main logical blocks:

- **1.1 Input Reception:** Manual triggering of the workflow to initiate the process.
- **1.2 Image Generation via Bannerbear:** Uses the Bannerbear node to generate an image based on a predefined template and specified modifications, then waits for the image to be fully processed.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block provides the starting point for the workflow. The process is initiated manually by the user, allowing controlled execution.

- **Nodes Involved:**  
  - On clicking 'execute' (Manual Trigger)

- **Node Details:**

  - **Node Name:** On clicking 'execute'  
  - **Type:** Manual Trigger  
  - **Technical Role:** Initiates the workflow manually when the user clicks the execute button.  
  - **Configuration:** No parameters are required; it simply waits for user interaction.  
  - **Key Expressions/Variables:** None.  
  - **Input/Output Connections:** No input; output connects to the Bannerbear node.  
  - **Version Requirements:** n8n version supporting manual trigger node (standard since early versions).  
  - **Potential Failures:** Minimal failure risk; user must trigger the node manually.  
  - **Sub-workflow:** None.

#### 1.2 Image Generation via Bannerbear

- **Overview:**  
  This block generates an image using Bannerbear's API based on a specified template and modifications. It waits for the image generation to complete before returning the result.

- **Nodes Involved:**  
  - Bannerbear

- **Node Details:**

  - **Node Name:** Bannerbear  
  - **Type:** Bannerbear node (API integration)  
  - **Technical Role:** Sends a request to Bannerbear to create an image using a template with specified dynamic modifications.  
  - **Configuration:**  
    - **Template ID:** `8BK3vWZJ7Wl5Jzk1aX` (a Bannerbear template identifier).  
    - **Modifications:**  
      - `message` field set to "this is some text" with text color `#3097BC` and background color `#28A96F`.  
    - **Additional Fields:** `waitForImage` enabled to have the node wait until image processing completes and the URL is available.  
  - **Credential:** Uses `bannerbear_creds` (an API key credential configured in n8n).  
  - **Key Expressions/Variables:** Static text and color values are hardcoded; no dynamic expressions.  
  - **Input/Output Connections:** Receives input from the Manual Trigger node; outputs the generated image data including the image URL.  
  - **Version Requirements:** Requires n8n version that supports Bannerbear node with `waitForImage` feature (introduced in recent versions).  
  - **Potential Failures:**  
    - API authentication errors if credentials are invalid.  
    - Network or timeout errors during image generation.  
    - Template ID or modification validation errors if invalid or mismatched.  
  - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name          | Node Type           | Functional Role               | Input Node(s)       | Output Node(s) | Sticky Note                          |
|--------------------|---------------------|------------------------------|---------------------|----------------|------------------------------------|
| On clicking 'execute' | Manual Trigger      | Initiates workflow execution | -                   | Bannerbear     |                                    |
| Bannerbear         | Bannerbear API Node  | Generates image procedurally | On clicking 'execute' | -              | Companion workflow for Bannerbear node docs |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a new node of type **Manual Trigger**.  
   - No configuration needed. This node will initiate the workflow manually.

2. **Create Bannerbear Node**  
   - Add a new node of type **Bannerbear**.  
   - Configure the following parameters:  
     - **Template ID:** Enter `8BK3vWZJ7Wl5Jzk1aX` (ensure this template exists in your Bannerbear account).  
     - **Modifications:**  
       - Add a modification with the name `message`.  
       - Set the text to `this is some text`.  
       - Set text color to `#3097BC`.  
       - Set background color to `#28A96F`.  
     - **Additional Fields:** Enable the option to **Wait for Image** so the node waits until the image is ready before continuing.  
   - Under **Credentials**, select or create Bannerbear API credentials (`bannerbear_creds`) with a valid Bannerbear API key.

3. **Connect Nodes**  
   - Connect the output of the Manual Trigger node to the input of the Bannerbear node.

4. **Save and Execute**  
   - Save the workflow.  
   - To test, trigger the workflow manually by clicking the execute button on the Manual Trigger node.  
   - The Bannerbear node will generate an image using the specified template and modifications and output the image data including the generated image URL.

---

### 5. General Notes & Resources

| Note Content                                                                            | Context or Link                                  |
|----------------------------------------------------------------------------------------|-------------------------------------------------|
| This workflow is a companion example for the Bannerbear node documentation in n8n.     | Related documentation in n8n’s official docs.   |
| Ensure your Bannerbear account has the specified template `8BK3vWZJ7Wl5Jzk1aX` created. | Bannerbear dashboard and template management.   |
| Waiting for image generation completion improves reliability over asynchronous calls.  | Bannerbear API feature: `waitForImage` option.  |
| For advanced use, you can parameterize the message text and colors dynamically.        | n8n workflow enhancement suggestion.             |

---

This concludes the comprehensive reference for the "Create an image procedurally using Bannerbear" workflow.