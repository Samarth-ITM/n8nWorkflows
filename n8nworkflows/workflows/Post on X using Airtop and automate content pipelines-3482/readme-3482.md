Post on X using Airtop and automate content pipelines

https://n8nworkflows.xyz/workflows/post-on-x-using-airtop-and-automate-content-pipelines-3482


# Post on X using Airtop and automate content pipelines

### 1. Workflow Overview

This workflow automates posting content on X (formerly Twitter) by leveraging Airtop’s browser automation capabilities within n8n. It is designed to streamline and schedule social media posts, integrating manual inputs or AI-generated content, and reliably publishing them on X without manual intervention.

**Logical Blocks:**

- **1.1 Input Reception:** Receives post content and Airtop profile information either via a web form submission or from another workflow trigger.
- **1.2 Parameter Preparation:** Normalizes and prepares input data for downstream processing.
- **1.3 Airtop Session Management:** Establishes a session with Airtop using a specified profile signed into X.
- **1.4 Browser Automation for Posting:** Opens X in a controlled browser window, types the post content into the tweet box, and clicks the Post button.
- **1.5 Session Termination:** Ends the Airtop session cleanly after posting.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block captures the input data required for posting on X. It supports two entry points: a web form submission and execution triggered by another workflow.

**Nodes Involved:**  
- On form submission  
- When Executed by Another Workflow

**Node Details:**

- **On form submission**  
  - Type: Form Trigger  
  - Role: Captures user input from a web form with fields for Airtop profile name and post text.  
  - Configuration:  
    - Form title: "Post on X"  
    - Fields: "Airtop profile name" (required), "Text to post" (required)  
    - Button label: "Post on X"  
    - Response: Confirmation message "✅ Your post has been published!"  
    - Webhook ID: Unique webhook for form submissions  
  - Inputs: HTTP request from form submission  
  - Outputs: JSON containing form data  
  - Edge cases: Missing required fields, invalid profile names, or malformed post text may cause downstream failures.

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Allows this workflow to be triggered programmatically by other workflows, passing the same input parameters.  
  - Configuration: Defines expected inputs "airtop_profile" and "post_text".  
  - Inputs: Trigger from external workflow  
  - Outputs: JSON with input parameters  
  - Edge cases: Missing or invalid inputs from calling workflows.

---

#### 2.2 Parameter Preparation

**Overview:**  
This block standardizes input data from either trigger source, ensuring consistent variable names for downstream nodes.

**Nodes Involved:**  
- Parameters

**Node Details:**

- **Parameters**  
  - Type: Set  
  - Role: Maps incoming JSON fields from either form or workflow trigger to standardized variables `airtop_profile` and `post_text`.  
  - Configuration:  
    - Assigns `airtop_profile` from either `"Airtop profile name"` or `airtop_profile` fields in input JSON.  
    - Assigns `post_text` from either `"Text to post"` or `post_text` fields.  
  - Inputs: JSON from either trigger node  
  - Outputs: JSON with normalized fields  
  - Edge cases: If both input fields are missing, variables will be empty, potentially causing errors downstream.

---

#### 2.3 Airtop Session Management

**Overview:**  
This block initiates an Airtop session using the specified profile to enable automated browser interactions on X.

**Nodes Involved:**  
- Create session

**Node Details:**

- **Create session**  
  - Type: Airtop node  
  - Role: Starts an Airtop session with the profile named in `airtop_profile`.  
  - Configuration:  
    - Profile name: Dynamic expression `={{ $json.airtop_profile }}`  
    - Timeout: 5 minutes  
    - Save profile on termination: true (preserves session state)  
  - Credentials: Airtop API key required  
  - Inputs: JSON with `airtop_profile`  
  - Outputs: Session context for subsequent Airtop nodes  
  - Edge cases:  
    - Invalid or unsigned Airtop profile causes authentication failure.  
    - Network or API errors may cause session creation to fail.  
  - Sticky Note: Emphasizes using an Airtop Profile signed into x.com to ensure smooth operation. [Airtop Profile Guide](https://docs.airtop.ai/guides/how-to/saving-a-profile)

---

#### 2.4 Browser Automation for Posting

**Overview:**  
This block automates the browser actions to open X, enter the post text, and submit the post.

**Nodes Involved:**  
- Create window  
- Type text  
- Click on Post

**Node Details:**

- **Create window**  
  - Type: Airtop node  
  - Role: Opens a new browser window pointed to https://x.com/ within the Airtop session.  
  - Configuration:  
    - URL: "https://x.com/"  
    - Resource: "window"  
  - Inputs: Session context from "Create session"  
  - Outputs: Window context for interaction nodes  
  - Edge cases: Network issues or page load failures.

- **Type text**  
  - Type: Airtop node  
  - Role: Types the post content into the "What's happening?" text box on X.  
  - Configuration:  
    - Resource: "interaction"  
    - Operation: "type"  
    - Text: Dynamic expression `={{ $('Parameters').item.json.post_text }}`  
    - Press Enter key: true (to simulate submission or confirm input)  
    - Element description: `"What's happening?" text box on top`  
  - Inputs: Window context from "Create window"  
  - Outputs: Interaction context for clicking  
  - Edge cases:  
    - Element not found due to UI changes on X.  
    - Empty or invalid text input.

- **Click on Post**  
  - Type: Airtop node  
  - Role: Clicks the Post button to publish the tweet.  
  - Configuration:  
    - Resource: "interaction"  
    - Additional fields: Visual scope set to "viewport" to limit interaction area  
    - Element description: "Click on the Post button"  
  - Inputs: Interaction context from "Type text"  
  - Outputs: Confirmation of click action  
  - Edge cases:  
    - Button not clickable due to UI changes or network delays.  
    - Post failure if content violates X policies.

---

#### 2.5 Session Termination

**Overview:**  
Ends the Airtop session to free resources and maintain clean state.

**Nodes Involved:**  
- End session

**Node Details:**

- **End session**  
  - Type: Airtop node  
  - Role: Terminates the active Airtop session gracefully.  
  - Configuration:  
    - Operation: "terminate"  
  - Inputs: Output from "Click on Post"  
  - Outputs: Session closed confirmation  
  - Edge cases: Session may already be closed or timeout errors.

---

### 3. Summary Table

| Node Name                     | Node Type                  | Functional Role                          | Input Node(s)               | Output Node(s)            | Sticky Note                                                                                           |
|-------------------------------|----------------------------|----------------------------------------|-----------------------------|---------------------------|-----------------------------------------------------------------------------------------------------|
| On form submission             | Form Trigger               | Receives post data via web form        | —                           | Parameters                |                                                                                                     |
| When Executed by Another Workflow | Execute Workflow Trigger  | Receives post data from other workflows| —                           | Parameters                |                                                                                                     |
| Parameters                    | Set                        | Normalizes input parameters             | On form submission, When Executed by Another Workflow | Create session            |                                                                                                     |
| Create session                | Airtop                     | Starts Airtop session with profile      | Parameters                  | Create window             | To make sure everything works smoothly, use an [Airtop Profile](https://docs.airtop.ai/guides/how-to/saving-a-profile) signed into x.com |
| Create window                 | Airtop                     | Opens X website in Airtop browser window| Create session              | Type text                 |                                                                                                     |
| Type text                    | Airtop                     | Types post content into X's tweet box  | Create window               | Click on Post             |                                                                                                     |
| Click on Post                | Airtop                     | Clicks the Post button to publish tweet| Type text                   | End session               |                                                                                                     |
| End session                  | Airtop                     | Terminates Airtop session               | Click on Post               | —                         |                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**

   - Add a **Form Trigger** node named "On form submission":
     - Set form title: "Post on X"
     - Add two required fields:
       - "Airtop profile name" (text input, placeholder: e.g. my-x-profile)
       - "Text to post" (text input, placeholder: e.g. This X post was made with Airtop and n8n)
     - Button label: "Post on X"
     - Response message on submission: "✅ Your post has been published!"
     - Save webhook ID generated automatically.

   - Add an **Execute Workflow Trigger** node named "When Executed by Another Workflow":
     - Define expected inputs: "airtop_profile" and "post_text".

2. **Add Parameter Normalization Node:**

   - Add a **Set** node named "Parameters":
     - Add two string fields with expressions:
       - `airtop_profile` = `={{ $json["Airtop profile name"] || $json.airtop_profile }}`
       - `post_text` = `={{ $json["Text to post"] || $json.post_text }}`
     - Connect both trigger nodes ("On form submission" and "When Executed by Another Workflow") to this node.

3. **Add Airtop Session Node:**

   - Add an **Airtop** node named "Create session":
     - Operation: Start session
     - Profile name: `={{ $json.airtop_profile }}`
     - Timeout: 5 minutes
     - Enable "Save profile on termination"
     - Set Airtop API credentials (requires Airtop API key)
     - Connect "Parameters" node output to this node.

4. **Add Airtop Window Node:**

   - Add an **Airtop** node named "Create window":
     - Operation: Open window
     - URL: https://x.com/
     - Resource: "window"
     - Use same Airtop API credentials
     - Connect "Create session" output to this node.

5. **Add Airtop Interaction Node to Type Text:**

   - Add an **Airtop** node named "Type text":
     - Operation: Interaction - type
     - Text: `={{ $('Parameters').item.json.post_text }}`
     - Press Enter key: true
     - Element description: `"What's happening?" text box on top`
     - Use Airtop credentials
     - Connect "Create window" output to this node.

6. **Add Airtop Interaction Node to Click Post:**

   - Add an **Airtop** node named "Click on Post":
     - Operation: Interaction - click
     - Visual scope: viewport
     - Element description: "Click on the Post button"
     - Use Airtop credentials
     - Connect "Type text" output to this node.

7. **Add Airtop Session Termination Node:**

   - Add an **Airtop** node named "End session":
     - Operation: Terminate session
     - Use Airtop credentials
     - Connect "Click on Post" output to this node.

8. **Activate Workflow:**

   - Ensure all nodes are connected as above.
   - Save and activate the workflow.
   - Test by submitting the form or triggering from another workflow with appropriate inputs.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Use an Airtop Profile signed into x.com for the "Create session" node to ensure authentication and smooth operation. | [Airtop Profile Guide](https://docs.airtop.ai/guides/how-to/saving-a-profile)                     |
| Airtop API key is required and must be configured in credentials for all Airtop nodes.                               | Airtop portal: https://portal.airtop.ai/?utm_campaign=n8n                                        |
| This workflow supports both manual form submissions and programmatic triggers from other workflows.                 | Enables flexible integration for social media automation                                        |
| Posting on X via Airtop automates browser interaction, so UI changes on X may require workflow updates.              | Monitor for UI updates on https://x.com/                                                        |
| Recommended to schedule or trigger this workflow at desired intervals externally for regular posting automation.     | Use n8n scheduling or external triggers                                                         |

---

This document provides a detailed, structured reference to understand, reproduce, and maintain the "Post on X" automation workflow using Airtop and n8n. It addresses all nodes, configurations, and potential edge cases to ensure reliable operation and extensibility.