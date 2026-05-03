Send Linkedin Connection Request with Airtop

https://n8nworkflows.xyz/workflows/send-linkedin-connection-request-with-airtop-4253


# Send Linkedin Connection Request with Airtop

---

### 1. Workflow Overview

This workflow automates sending LinkedIn connection requests using an Airtop browser profile authenticated with LinkedIn. It supports two trigger methods: via a web form submission or execution from another workflow. The core logic verifies if the target LinkedIn user is already a first-degree connection or if a connection request is pending, to avoid redundant requests. If the user is not connected, the workflow sends a connection request either with or without a personalized message based on input.

The workflow is organized into the following logical blocks:

- **1.1 Input Reception:** Handles triggering via form submission or external workflow call and normalizes input parameters.
- **1.2 Airtop Session & Navigation:** Starts an Airtop session, opens the target LinkedIn profile in a browser window, and interacts with UI elements.
- **1.3 Connection Status Check:** Extracts and interprets the connection status of the target user.
- **1.4 Connection Request Handling:** Depending on the connection status and presence of a message, sends a connection request with or without a personalized note.
- **1.5 Session Termination:** Ends the Airtop browser session cleanly.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block receives input parameters either from a web form submission or from another workflow execution trigger. It then unifies these inputs into consistent variable names for downstream use.

**Nodes Involved:**  
- On form submission  
- When Executed by Another Workflow  
- Unify Params  

**Node Details:**

- **On form submission**  
  - Type: Form Trigger  
  - Role: Starts the workflow on a web form submission with fields for LinkedIn URL, Airtop Profile, and an optional message.  
  - Configuration:  
    - Form titled "LinkedIn connection requests" with required fields "LinkedIn URL" and "Airtop Profile (connected to Linkedin)" and an optional textarea "Message".  
    - Form description includes instructions and a link to create an Airtop Profile: https://portal.airtop.ai/browser-profiles  
  - Inputs: HTTP webhook from form submission  
  - Outputs: JSON with form fields  
  - Edge cases: Missing required fields will block submission; malformed URLs may cause errors downstream.

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Allows external workflows to trigger this workflow with parameters "airtop_profile", "linked_url", and "message".  
  - Inputs: Workflow input parameters  
  - Outputs: JSON payload with input parameters  
  - Edge cases: Missing parameters or incorrect parameter names can cause downstream errors.

- **Unify Params**  
  - Type: Set  
  - Role: Normalizes input parameters regardless of the trigger source into unified JSON keys: linkedin_url, airtop_profile, message.  
  - Configuration: Uses expressions to select from either incoming keys or form field names (e.g., `$json.linkedin_url || $json['LinkedIn URL']`)  
  - Inputs: From either form submission or execute workflow trigger  
  - Outputs: Single JSON object with unified keys  
  - Edge cases: If neither source provides expected keys, values may be empty strings causing errors later.

---

#### 1.2 Airtop Session & Navigation

**Overview:**  
This block creates an Airtop browser session using the provided Airtop profile, opens the LinkedIn profile URL in a new browser window, and performs initial UI interactions to reach the connection area.

**Nodes Involved:**  
- Create a Session  
- Create a window  
- Click More button  

**Node Details:**

- **Create a Session**  
  - Type: Airtop  
  - Role: Initiates an Airtop browser session authenticating with the specified Airtop Profile.  
  - Configuration: Uses profileName from unified input (`$json.airtop_profile`), saves profile on termination (true)  
  - Inputs: Unified parameters output  
  - Outputs: Session details including sessionId  
  - Edge cases: Invalid or non-authenticated Airtop profile will cause session creation failure; network issues may cause timeouts.

- **Create a window**  
  - Type: Airtop  
  - Role: Opens the LinkedIn profile URL in a new browser window within the Airtop session.  
  - Configuration: URL from unified params (`$('Unify Params').item.json.linkedin_url`), requests live view enabled  
  - Inputs: Session output from previous node  
  - Outputs: Window details including windowId  
  - Edge cases: Invalid URL or LinkedIn page not loading properly may cause failures.

- **Click More button**  
  - Type: Airtop Interaction  
  - Role: Simulates a click on the "More" button on the LinkedIn page to reveal connection status options.  
  - Configuration: Targets element described as "More button"  
  - Inputs: Window output from "Create a window"  
  - Outputs: Interaction result for next extraction  
  - Edge cases: UI changes on LinkedIn may cause element not found errors; session inactivity may expire.

---

#### 1.3 Connection Status Check

**Overview:**  
This block extracts the connection status of the target LinkedIn user by querying page elements and deciding if the user is a connection, pending connection, or not connected.

**Nodes Involved:**  
- Check connection status  
- Is a connection?  

**Node Details:**

- **Check connection status**  
  - Type: Airtop Extraction Query  
  - Role: Uses a prompt to analyze the LinkedIn profile page's top box and returns one of three strings: "Connect", "Pending", or "1st degree".  
  - Configuration: Prompt instructs detection of "Connect" button, "Pending" message, or "1st degree" indicator near the name, with no other outputs accepted.  
  - Inputs: Interaction output from clicking "More button"  
  - Outputs: JSON data with modelResponse indicating connection status  
  - Edge cases: UI changes or network lag may cause incorrect extraction; unexpected page layouts may cause parsing failures.

- **Is a connection?**  
  - Type: Switch  
  - Role: Branches workflow based on connection status string containing "Connect" (not connected) or not (already connected/pending).  
  - Configuration: Checks if `data.modelResponse` contains "Connect" or not.  
  - Inputs: Output of status check  
  - Outputs:  
    - True branch leads to sending connection request  
    - False branch leads to terminating session (already connected)  
  - Edge cases: Unexpected or empty status strings may cause wrong branching.

---

#### 1.4 Connection Request Handling

**Overview:**  
This block sends the actual LinkedIn connection request using Airtop interactions, either with a personalized message or without, depending on input.

**Nodes Involved:**  
- Click on connect  
- Switch1  
- Send connection w/out message  
- Click add a note  
- Type the message  
- Send connection w/ message  

**Node Details:**

- **Click on connect**  
  - Type: Airtop Interaction  
  - Role: Simulates clicking the "Connect" button on LinkedIn.  
  - Configuration: Uses windowId and sessionId from prior nodes, targets element "Connect"  
  - Inputs: From "Is a connection?" true branch  
  - Outputs: Interaction context for message sending  
  - Edge cases: UI changes or disabled buttons may cause failure.

- **Switch1**  
  - Type: Switch  
  - Role: Determines if a message is provided to decide sending request with or without note.  
  - Configuration: Checks if `$('Unify Params').item.json.message` is empty or not.  
  - Inputs: Output from "Click on connect"  
  - Outputs:  
    - Empty message branch leads to "Send connection w/out message"  
    - Non-empty message branch leads to "Click add a note"  
  - Edge cases: Null or whitespace-only messages could be misclassified.

- **Send connection w/out message**  
  - Type: Airtop Interaction  
  - Role: Clicks "Send without a note" to send the connection request without a message.  
  - Configuration: Uses session/window from "Click on connect" outputs, targets element "Send without a note"  
  - Outputs: Triggers session termination  
  - Edge cases: Button may be missing or disabled; network lag may cause double clicks.

- **Click add a note**  
  - Type: Airtop Interaction  
  - Role: Clicks "Add a note" button to open the message input field before sending a connection request.  
  - Configuration: Uses session/window from "Click on connect" outputs, targets element "Add a note"  
  - Outputs: Leads to typing the message  
  - Edge cases: UI changes may alter element descriptor.

- **Type the message**  
  - Type: Airtop Interaction (type operation)  
  - Role: Types the personalized message into the note field.  
  - Configuration: Text from unified params message, uses session/window from preceding node  
  - Outputs: Leads to sending connection with message  
  - Edge cases: Special characters in message may cause typing issues; excessive length may cause truncation.

- **Send connection w/ message**  
  - Type: Airtop Interaction  
  - Role: Clicks "Send" button to submit connection request with the typed note.  
  - Configuration: Uses session/window info, targets element "Send"  
  - Outputs: Triggers session termination  
  - Edge cases: Button may be disabled if message field validation fails.

---

#### 1.5 Session Termination

**Overview:**  
This block cleanly terminates the Airtop browser session once all actions are complete or if the user is already a connection.

**Nodes Involved:**  
- Terminate session  
- Terminate session1  

**Node Details:**

- **Terminate session**  
  - Type: Airtop  
  - Role: Terminates the Airtop session using sessionId, typically when the user is already connected or pending.  
  - Configuration: Requires sessionId from upstream node  
  - Inputs: Session info from "Is a connection?" false branch  
  - Outputs: None  
  - Edge cases: Session may have expired or already terminated causing no-op.

- **Terminate session1**  
  - Type: Airtop  
  - Role: Terminates Airtop session after sending a connection request (with or without message).  
  - Configuration: No sessionId parameter (assumed to terminate current session)  
  - Inputs: From connection request sending nodes  
  - Outputs: None  
  - Edge cases: Same as above.

---

### 3. Summary Table

| Node Name                  | Node Type                   | Functional Role                               | Input Node(s)                     | Output Node(s)                        | Sticky Note                                                                                       |
|----------------------------|-----------------------------|-----------------------------------------------|----------------------------------|-------------------------------------|-------------------------------------------------------------------------------------------------|
| On form submission          | Form Trigger                | Trigger workflow via web form submission       | None                             | Unify Params                        | ## Input Parameters Run this workflow using a form or another workflow                           |
| When Executed by Another Workflow | Execute Workflow Trigger | Trigger workflow from another workflow          | None                             | Unify Params                        | ## Input Parameters Run this workflow using a form or another workflow                           |
| Unify Params               | Set                         | Normalize input parameters                      | On form submission, When Executed by Another Workflow | Create a Session                   |                                                                                                 |
| Create a Session            | Airtop                      | Start Airtop browser session with profile      | Unify Params                    | Create a window                    |                                                                                                 |
| Create a window             | Airtop                      | Open LinkedIn profile URL in Airtop window     | Create a Session                | Click More button                  |                                                                                                 |
| Click More button           | Airtop Interaction          | Click "More" button on LinkedIn profile page   | Create a window                 | Check connection status            |                                                                                                 |
| Check connection status     | Airtop Extraction Query     | Extract connection status from LinkedIn page   | Click More button               | Is a connection?                   | ## LinkedIn Connection Checker Verifies whether a specified individual is a first-degree connection on LinkedIn |
| Is a connection?            | Switch                      | Branch based on connection status               | Check connection status         | Click on connect, Terminate session |                                                                                                 |
| Click on connect            | Airtop Interaction          | Click "Connect" button if not connected         | Is a connection? (true branch) | Switch1                          |                                                                                                 |
| Switch1                    | Switch                      | Check if message is provided                     | Click on connect               | Send connection w/out message, Click add a note |                                                                                                 |
| Send connection w/out message | Airtop Interaction         | Send request without a message                    | Switch1 (empty message branch) | Terminate session1                | ## Send request without a note If a message is not provided, the request is send without a note |
| Click add a note            | Airtop Interaction          | Click "Add a note" to add personalized message  | Switch1 (non-empty message branch) | Type the message                |                                                                                                 |
| Type the message            | Airtop Interaction (type)   | Type personalized message into note field        | Click add a note               | Send connection w/ message         |                                                                                                 |
| Send connection w/ message  | Airtop Interaction          | Send request with a personalized message         | Type the message               | Terminate session1                | ## Send request with a note If a message is provided, the request is send with a note           |
| Terminate session           | Airtop                      | End Airtop session when already connected/pending | Is a connection? (false branch) | None                             | ## End Airtop session The user is already a connection                                          |
| Terminate session1          | Airtop                      | End Airtop session after sending connection request | Send connection w/ message, Send connection w/out message | None                  | ## End Airtop session                                                                           |
| Sticky Note                 | Sticky Note                 | Documentation and comments                       | None                          | None                              | ## Input Parameters Run this workflow using a form or another workflow                           |
| Sticky Note1                | Sticky Note                 | Documentation                                   | None                          | None                              | ## LinkedIn Connection Checker Verifies whether a specified individual is a first-degree connection on LinkedIn |
| Sticky Note2                | Sticky Note                 | Documentation                                   | None                          | None                              | ## Send connection request                                                                      |
| Sticky Note3                | Sticky Note                 | Documentation                                   | None                          | None                              | ## End Airtop session The user is already a connection                                          |
| Sticky Note4                | Sticky Note                 | Documentation                                   | None                          | None                              | ## End Airtop session                                                                           |
| Sticky Note5                | Sticky Note                 | Documentation                                   | None                          | None                              | ## Send request without a note If a message is not provided, the request is send without a note |
| Sticky Note6                | Sticky Note                 | Documentation                                   | None                          | None                              | ## Send request with a note If a message is provided, the request is send with a note           |
| Sticky Note7                | Sticky Note                 | README with detailed project overview           | None                          | None                              | README: Automating LinkedIn Connection Requests... [full text as in node]                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Input Triggers**  
   - Add a **Form Trigger** node named "On form submission".  
     - Configure form title: "LinkedIn connection requests".  
     - Add required fields: "LinkedIn URL" (text), "Airtop Profile (connected to Linkedin)" (text).  
     - Add optional field: "Message" (textarea).  
     - Add form description with HTML including a link to Airtop profiles: https://portal.airtop.ai/browser-profiles.  
   - Add an **Execute Workflow Trigger** node named "When Executed by Another Workflow".  
     - Define workflow inputs: airtop_profile, linked_url, message.

2. **Normalize Input Parameters**  
   - Add a **Set** node named "Unify Params".  
     - Set three variables using expressions:  
       - linkedin_url = `{{$json.linkedin_url || $json["LinkedIn URL"]}}`  
       - airtop_profile = `{{$json.airtop_profile || $json["Airtop Profile (connected to Linkedin)"]}}`  
       - message = `{{$json.message || $json["Message"]}}`  
   - Connect both trigger nodes ("On form submission" and "When Executed by Another Workflow") to "Unify Params".

3. **Start Airtop Session**  
   - Add an **Airtop** node named "Create a Session".  
     - Operation: Start session.  
     - Set `profileName` to `{{$json.airtop_profile}}`.  
     - Enable "Save Profile On Termination".  
     - Configure Airtop credentials with your Airtop API key.

4. **Open LinkedIn Profile in Browser Window**  
   - Add an **Airtop** node named "Create a window".  
     - Operation: Create window.  
     - URL: `{{$('Unify Params').item.json.linkedin_url}}`.  
     - Enable "Get Live View".  
   - Connect "Create a Session" output to "Create a window".

5. **Click "More" Button on LinkedIn Profile**  
   - Add an **Airtop** node named "Click More button".  
     - Operation: Interaction.  
     - Element description: "More button".  
   - Connect "Create a window" to "Click More button".

6. **Extract Connection Status**  
   - Add an **Airtop** node named "Check connection status".  
     - Operation: Extraction query.  
     - Prompt:  
       ```
       This is a LinkedIn profile page. Please perform the following actions based on the presence of specific elements in the top profile box:

       - Reply "Connect" if a "Connect" button is present.
       - Reply "Pending" if a "Pending" message is present.
       - Reply "1st degree" if this person is a first-degree connection, indicated by "1st" near the name.

       Do not reply with anything else.

       For example:
       - If the "Connect" button is present, the output should be: "Connect"
       - If the "Pending" message is present, the output should be: "Pending"
       - If the person is a first-degree connection, the output should be: "1st degree"
       ```
     - Resource: Extraction  
   - Connect "Click More button" to "Check connection status".

7. **Decide Next Step Based on Connection Status**  
   - Add a **Switch** node named "Is a connection?".  
     - Rule 1: If `{{$json.data.modelResponse}}` contains "Connect" → user is not connected.  
     - Rule 2: Else → user is connected or pending.  
   - Connect "Check connection status" to "Is a connection?".

8. **If Connected or Pending: Terminate Session**  
   - Add an **Airtop** node named "Terminate session".  
     - Operation: Terminate session.  
     - Use `sessionId` from "Is a connection?" node.  
   - Connect "Is a connection?" (false branch) to "Terminate session".

9. **If Not Connected: Click "Connect" Button**  
   - Add an **Airtop** node named "Click on connect".  
     - Operation: Interaction.  
     - Element description: "Connect".  
     - Use `windowId` and `sessionId` from previous nodes.  
   - Connect "Is a connection?" (true branch) to "Click on connect".

10. **Check if Message is Provided**  
    - Add a **Switch** node named "Switch1".  
      - Rule 1: If `{{$('Unify Params').item.json.message}}` is empty string → no message.  
      - Rule 2: If message is not empty → message provided.  
    - Connect "Click on connect" to "Switch1".

11. **Send Connection Request Without Message**  
    - Add an **Airtop** node named "Send connection w/out message".  
      - Operation: Interaction.  
      - Element description: "Send without a note".  
      - Use `windowId` and `sessionId` from "Click on connect".  
    - Connect "Switch1" (empty message branch) to this node.

12. **Send Connection Request With Message**  
    - Add an **Airtop** node named "Click add a note".  
      - Operation: Interaction.  
      - Element description: "Add a note".  
      - Use `windowId` and `sessionId` from "Click on connect".  
    - Connect "Switch1" (non-empty message branch) to "Click add a note".

    - Add an **Airtop** node named "Type the message".  
      - Operation: Interaction (type).  
      - Text: `{{$('Unify Params').item.json.message}}`.  
      - Use `windowId` and `sessionId` from "Click add a note".  
    - Connect "Click add a note" to "Type the message".

    - Add an **Airtop** node named "Send connection w/ message".  
      - Operation: Interaction.  
      - Element description: "Send".  
      - Use `windowId` and `sessionId` from "Type the message".  
    - Connect "Type the message" to "Send connection w/ message".

13. **Terminate Airtop Session After Sending Request**  
    - Add an **Airtop** node named "Terminate session1".  
      - Operation: Terminate session.  
      - No explicit sessionId parameter needed (terminates current session).  
    - Connect "Send connection w/out message" and "Send connection w/ message" to "Terminate session1".

14. **Credentials Setup**  
    - Configure Airtop credentials in all Airtop nodes requiring API access using a valid Airtop API key.  
    - Ensure the Airtop Profile specified is authenticated with LinkedIn.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Automating LinkedIn Connection Requests enables streamlined outreach by checking existing connections before sending requests, optionally including personalized messages. Requires Airtop API key and a LinkedIn-authenticated Airtop Profile.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | https://portal.airtop.ai/api-keys and https://portal.airtop.ai/browser-profiles                      |
| Pair this automation with People Enrichment or LinkedIn Profile Finder tools to generate URLs before sending requests, or integrate with CRM systems to log connection attempts and responses.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | https://www.airtop.ai/blog/find-linkedin-profile-from-email                                        |
| Airtop interactions rely on LinkedIn page UI elements; UI changes by LinkedIn may require workflow updates. Network latency or Airtop session expiry may cause failures.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |                                                                                                    |
| The form trigger includes a description with HTML links and instructions for users to create and authenticate Airtop Profiles.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | https://portal.airtop.ai/browser-profiles                                                          |
| Sticky notes within the workflow provide detailed documentation on each block, including a comprehensive README describing use case, setup, and operational details.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Internal workflow documentation                                                                     |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---