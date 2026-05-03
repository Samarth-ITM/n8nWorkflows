Automated Replies to X Threads with Airtop Browser Automation

https://n8nworkflows.xyz/workflows/automated-replies-to-x-threads-with-airtop-browser-automation-4054


# Automated Replies to X Threads with Airtop Browser Automation

### 1. Workflow Overview

This workflow automates posting replies to specific X (formerly Twitter) threads by using browser automation via Airtop. It is designed to help users engage with X posts at scale by programmatically opening a browser session, navigating to a target thread URL, typing a reply message, and submitting it automatically.

**Target Use Case:**  
- Marketing teams or social media managers who want to respond to X posts quickly and consistently without manual intervention.  
- Automating customer engagement or interaction pipelines integrated with X monitoring systems.

**Logical Blocks:**

- **1.1 Input Reception:** Receives input parameters either via a form submission webhook or from another workflow.  
- **1.2 Parameter Preparation:** Maps inputs into usable workflow variables.  
- **1.3 Airtop Session Management:** Creates and later terminates a browser session connected to the specified Airtop profile.  
- **1.4 Browser Navigation:** Opens the target X thread URL in the Airtop browser session.  
- **1.5 Reply Composition and Submission:** Types the reply text, takes a screenshot for verification, and clicks the reply button.  
- **1.6 Timing Control:** Waits for page load and UI readiness before typing the reply.  
- **1.7 Post-Action Cleanup:** Terminates the browser session to free resources.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

**Overview:**  
Captures input parameters to trigger the workflow. It supports two entry points: a webhook form submission and a trigger from another workflow.

**Nodes Involved:**  
- On form submission  
- When Executed by Another Workflow

**Node Details:**

- **On form submission**  
  - Type: Form Trigger (Webhook)  
  - Role: Listens for HTTP POST with form data: `airtop_profile`, `thread_url`, `reply_text`  
  - Configuration: Each field is required; form titled "Reply to Thread"  
  - Inputs: External HTTP POST  
  - Outputs: Workflow data  
  - Edge Cases: Missing or invalid form fields; webhook downtime or latency.

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Entry point when called programmatically from another workflow  
  - Configuration: Accepts same three inputs as form trigger  
  - Inputs: Workflow call  
  - Outputs: Workflow data  
  - Edge Cases: Missing parameter values; calling workflow errors; version compatibility if caller changes.

---

#### 2.2 Parameter Preparation

**Overview:**  
Copies and prepares input parameters into a standardized internal format for downstream nodes.

**Nodes Involved:**  
- Parameters

**Node Details:**

- **Parameters**  
  - Type: Set node  
  - Role: Extracts and assigns input parameters (`airtop_profile`, `thread_url`, `reply_text`) to workflow variables  
  - Configuration: Assignments use expressions referencing previous node JSON input  
  - Inputs: Either from "On form submission" or "When Executed by Another Workflow" nodes  
  - Outputs: Structured parameters for session creation and navigation  
  - Edge Cases: Expression evaluation errors if expected keys are missing.

---

#### 2.3 Airtop Session Management

**Overview:**  
Starts a browser session using the Airtop API with the specified profile and terminates the session after the workflow finishes.

**Nodes Involved:**  
- Session  
- Terminate session

**Node Details:**

- **Session**  
  - Type: Airtop node  
  - Role: Creates a browser session with the `airtop_profile` from parameters  
  - Configuration: Uses `profileName` set dynamically via expression to the input parameter  
  - Credentials: Airtop API Key (configured with organization credential)  
  - Inputs: From Parameters node  
  - Outputs: Session data including `sessionId`  
  - Edge Cases: Authentication errors, invalid profile names, API downtime, session creation failure.

- **Terminate session**  
  - Type: Airtop node  
  - Role: Terminates the active Airtop browser session to free resources  
  - Configuration: Operation set to "terminate"  
  - Credentials: Same Airtop API Key as Session node  
  - Inputs: From Session node (parallel path)  
  - Outputs: Confirmation of termination  
  - Edge Cases: Session already terminated, API errors.

---

#### 2.4 Browser Navigation

**Overview:**  
Opens the specified X thread URL in the active Airtop browser session.

**Nodes Involved:**  
- Window

**Node Details:**

- **Window**  
  - Type: Airtop node  
  - Role: Opens a browser window/tab with the `thread_url`  
  - Configuration:  
    - URL: Dynamically set to `thread_url` parameter  
    - Resource: "window"  
    - `sessionId`: from Session node output  
    - Waits until page load is complete (`waitUntil: complete`)  
    - Screen resolution fixed at 1300x100 (narrow viewport)  
    - `getLiveView` enabled for live rendering preview  
    - Disable resize set  
  - Credentials: Airtop API Key  
  - Inputs: Session node output  
  - Outputs: Window data including `windowId`  
  - Edge Cases: Invalid or unreachable URL, page load timeouts, session expiration.

---

#### 2.5 Timing Control

**Overview:**  
Waits a fixed duration after the page loads to ensure UI elements are interactive before typing.

**Nodes Involved:**  
- Wait 8 secs

**Node Details:**

- **Wait 8 secs**  
  - Type: Wait node  
  - Role: Pauses execution for 8 seconds  
  - Configuration: Fixed duration of 8 seconds  
  - Inputs: From Window node  
  - Outputs: Passes data forward after wait  
  - Edge Cases: Excessive or insufficient wait time (could cause typing too early or delay).

---

#### 2.6 Reply Composition and Submission

**Overview:**  
Types the reply text into the reply input field, takes a screenshot for auditing, and clicks the reply button.

**Nodes Involved:**  
- Type response  
- Post-action screenshot  
- Click Reply button

**Node Details:**

- **Type response**  
  - Type: Airtop node (interaction)  
  - Role: Types the reply text into the "Post your reply" input field near the "Reply" button  
  - Configuration:  
    - `text`: dynamically from `reply_text` parameter  
    - `resource`: "interaction"  
    - `operation`: "type"  
    - `sessionId`: from Session node output  
    - `windowId`: from Window node output  
    - `elementDescription`: Describes the input field visually and contextually to the Airtop automation engine ("The input field labeled 'Post your reply' located directly next to the 'Reply' button")  
    - Visual scope limited to "scan" to locate element  
  - Credentials: Airtop API Key  
  - Inputs: From Wait 8 secs node  
  - Outputs: Confirmation of typing action  
  - Edge Cases: Element not found (UI changes), typing failure, session expiration.

- **Post-action screenshot**  
  - Type: Airtop node (window operation)  
  - Role: Takes a screenshot of the window after typing to verify the reply is typed correctly  
  - Configuration:  
    - `resource`: "window"  
    - `operation`: "takeScreenshot"  
    - `windowId`: from Window node output  
    - `sessionId`: from Session node output  
  - Credentials: Airtop API Key  
  - Inputs: From Type response node  
  - Outputs: Screenshot data  
  - Edge Cases: Screenshot failure, permission issues.

- **Click Reply button**  
  - Type: Airtop node (interaction)  
  - Role: Clicks the gray rounded "Reply" button located directly below the main tweet, submitting the reply  
  - Configuration:  
    - `resource`: "interaction"  
    - `operation`: "click" (implied)  
    - `sessionId`: from Session node output  
    - `windowId`: from Window node output  
    - `elementDescription`: Visual description used to locate the button  
    - Visual scope set to "page" for broader element search  
  - Credentials: Airtop API Key  
  - Inputs: From Post-action screenshot node  
  - Outputs: Confirmation of click action  
  - Edge Cases: Button not found, click failure, UI changes, session timeout.

---

#### 2.7 Post-Action Cleanup

**Overview:**  
Ensures the browser session is terminated after navigation and interaction complete to avoid resource leakage.

**Nodes Involved:**  
- Terminate session (already covered in 2.3)

---

### 3. Summary Table

| Node Name                   | Node Type                      | Functional Role                               | Input Node(s)                     | Output Node(s)                   | Sticky Note                                                                                                                                    |
|-----------------------------|--------------------------------|-----------------------------------------------|----------------------------------|---------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger (n8n) | Entry point via workflow call                   | None                             | Parameters                      |                                                                                                                                                 |
| On form submission          | Form Trigger (Webhook)          | Entry point via form submission                 | None                             | Parameters                      |                                                                                                                                                 |
| Parameters                 | Set                            | Extracts and sets input parameters              | When Executed by Another Workflow, On form submission | Session                        |                                                                                                                                                 |
| Session                   | Airtop                         | Creates Airtop browser session                   | Parameters                      | Window, Terminate session       |                                                                                                                                                 |
| Window                    | Airtop                         | Opens X thread URL in Airtop browser             | Session                        | Wait 8 secs, Terminate session  |                                                                                                                                                 |
| Wait 8 secs               | Wait                           | Waits 8 seconds for page readiness               | Window                         | Type response                  |                                                                                                                                                 |
| Type response             | Airtop                         | Types reply text into reply input field          | Wait 8 secs                    | Post-action screenshot         |                                                                                                                                                 |
| Post-action screenshot    | Airtop                         | Takes screenshot after typing reply               | Type response                  | Click Reply button             |                                                                                                                                                 |
| Click Reply button         | Airtop                         | Clicks “Reply” button to submit reply             | Post-action screenshot         | (none)                        |                                                                                                                                                 |
| Terminate session          | Airtop                         | Terminates Airtop browser session                 | Session, Window (parallel)      | (none)                        |                                                                                                                                                 |
| Sticky Note                | Sticky Note                   | Documentation note                               | None                          | None                          | README with detailed use case, setup instructions, and links: [Airtop Profiles](https://portal.airtop.ai/browser-profiles), [Example X post](https://x.com/thepatwalls/status/1921932138401726866), [Airtop API Keys](https://portal.airtop.ai/api-keys), [Automation info](https://www.airtop.ai/automations/post-on-x-n8n) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the input triggers:**

   - Add a **Form Trigger** node named "On form submission"  
     - Configure form fields: `airtop_profile` (required), `thread_url` (required), `reply_text` (required)  
     - Enable webhook, set form title "Reply to Thread"

   - Add an **Execute Workflow Trigger** node named "When Executed by Another Workflow"  
     - Configure workflow inputs: `airtop_profile`, `thread_url`, `reply_text`

2. **Create a Set node named "Parameters":**

   - Add assignments for three string variables:  
     - `airtop_profile` = `{{$json.airtop_profile}}`  
     - `thread_url` = `{{$json.thread_url}}`  
     - `reply_text` = `{{$json.reply_text}}`

   - Connect both "On form submission" and "When Executed by Another Workflow" nodes to this node.

3. **Add an Airtop node named "Session":**

   - Set operation to create a session (default)  
   - Profile Name: `={{ $json.airtop_profile }}` (use expression referencing Parameters node output)  
   - Assign Airtop API credentials (create credential with Airtop API key from https://portal.airtop.ai/api-keys)  
   - Connect "Parameters" node output to this node.

4. **Add an Airtop node named "Window":**

   - Set resource to "window"  
   - URL: `={{ $json.thread_url }}` (expression from Parameters)  
   - Session ID: `={{ $json.sessionId }}` (from Session node output)  
   - Additional fields: wait until page load complete  
   - Screen resolution: 1300x100  
   - Enable live view and disable resize  
   - Connect "Session" node output to this node.

5. **Add a Wait node named "Wait 8 secs":**

   - Set wait duration to 8 seconds  
   - Connect "Window" node output to this node.

6. **Add an Airtop node named "Type response":**

   - Resource: Interaction  
   - Operation: Type  
   - Text: `={{ $json.reply_text }}` (expression)  
   - Session ID: from Session node output  
   - Window ID: from Window node output  
   - Additional fields: Visual scope "scan"  
   - Element description: "The input field labeled \"Post your reply\" located directly next to the \"Reply\" button"  
   - Connect "Wait 8 secs" node output to this node.

7. **Add an Airtop node named "Post-action screenshot":**

   - Resource: Window  
   - Operation: TakeScreenshot  
   - Session ID and Window ID: from respective node outputs  
   - Connect "Type response" node output to this node.

8. **Add an Airtop node named "Click Reply button":**

   - Resource: Interaction  
   - Operation: Click (default for interaction nodes)  
   - Session ID and Window ID: from respective node outputs  
   - Additional fields: Visual scope "page"  
   - Element description: "Gray rounded button \"Reply\" located directly below the main tweet"  
   - Connect "Post-action screenshot" node output to this node.

9. **Add an Airtop node named "Terminate session":**

   - Operation: Terminate  
   - Session ID: from Session node output  
   - Connect "Session" node output also to this node (parallel branch) and optionally to "Window" node to ensure session termination after navigation.

10. **Add a Sticky Note node (optional):**

    - Add the README content as documentation inside the workflow for user reference.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                    | Context or Link                                                                                             |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| This automation requires an Airtop API Key and an Airtop profile connected to X (one-time login).                                                                                                                                                               | https://portal.airtop.ai/api-keys ; https://portal.airtop.ai/browser-profiles                              |
| Example thread URL format for `thread_url`: https://x.com/thepatwalls/status/1921932138401726866                                                                                                                                                                | Example link in Sticky Note                                                                                  |
| Combine this workflow with an X monitoring automation to build a full engagement pipeline.                                                                                                                                                                       | Mentioned in README sticky note                                                                             |
| Extend this automation to other platforms like LinkedIn or Reddit by adapting the Airtop interaction steps to those sites.                                                                                                                                       | Mentioned in README sticky note                                                                             |
| Detailed automation guide and branding information available at Airtop website.                                                                                                                                                                                  | https://www.airtop.ai/automations/post-on-x-n8n                                                            |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, a workflow automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected material. All processed data is legal and public.