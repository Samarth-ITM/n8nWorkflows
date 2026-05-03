Send Standardized Alert Emails via Gmail with Custom Subject and Content

https://n8nworkflows.xyz/workflows/send-standardized-alert-emails-via-gmail-with-custom-subject-and-content-6189


# Send Standardized Alert Emails via Gmail with Custom Subject and Content

### 1. Workflow Overview

This workflow, named "❗ Send Email Alert," is designed to standardize the process of sending alert emails via Gmail with a customizable subject line and message content. It is primarily intended to be triggered programmatically by other workflows, passing in parameters to define the email's subject and body content. The workflow abstracts email sending into a reusable helper process, ensuring consistent formatting and easy identification of alert emails in the recipient’s inbox.

**Logical Blocks:**

- **1.1 Input Reception:** Accepts input parameters (`subject` and `lines`) from an external triggering workflow.
- **1.2 Email Composition and Sending:** Formats the input into a standard alert email and sends it through a Gmail OAuth2 authenticated account.
- **1.3 Documentation and User Guidance:** Included sticky notes provide usage instructions and warnings about email filtering configuration.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block accepts input parameters from other workflows, serving as the entry point for the alert email data.

**Nodes Involved:**  
- When Executed by Another Workflow

**Node Details:**

- **Node Name:** When Executed by Another Workflow  
- **Type:** Execute Workflow Trigger (n8n-native node)  
- **Role:** Receives input parameters when triggered by another workflow instance.  
- **Configuration:**  
  - Declares expected inputs: `"subject"` (string) and `"lines"` (array of strings).  
- **Expressions/Variables:**  
  - Inputs are passed downstream as `$json.subject` and `$json.lines`.  
- **Connections:**  
  - Output connected to the "Send a message" node.  
- **Version Requirements:**  
  - Uses typeVersion 1.1, compatible with n8n recent versions supporting Execute Workflow Trigger nodes.  
- **Potential Failures:**  
  - Missing or malformed input parameters could cause downstream message formatting issues.  
  - Trigger may fail if invoked improperly or with incompatible data types.  
- **Sub-workflow:**  
  - This node is designed to be triggered by other workflows, acting as a sub-workflow entry point.

---

#### 2.2 Email Composition and Sending

**Overview:**  
This block formats the input data into a standardized alert email subject and message body, then sends the email through Gmail using OAuth2 authentication.

**Nodes Involved:**  
- Send a message  
- Sticky Note (warning about subject changes)

**Node Details:**

- **Node Name:** Send a message  
- **Type:** Gmail node (n8n-nodes-base.gmail)  
- **Role:** Sends an email with the specified subject and content via Gmail.  
- **Configuration:**  
  - `sendTo`: Hardcoded recipient email address to be replaced by the user (`YOUREMAILHERE@gmail.com`).  
  - `subject`: Uses expression to prepend `"❗ n8n Alert: "` to the input subject:  
    ```plaintext
    =❗ n8n Alert: {{ $json.subject }}
    ```  
  - `message`: Joins the input array `lines` into a single string separated by newlines:  
    ```plaintext
    ={{ $json.lines.join("\n") }}
    ```  
  - `emailType`: Set to `"text"` (plain text email).  
  - `options.appendAttribution`: Disabled to avoid n8n attribution text in the email body.  
- **Expressions/Variables:**  
  - `${json.subject}` for dynamic subject content.  
  - `${json.lines}` array joined for message body.  
- **Connections:**  
  - Input from "When Executed by Another Workflow" node.  
- **Credentials:**  
  - Uses Gmail OAuth2 credentials, which need to be configured with appropriate access scopes.  
- **Version Requirements:**  
  - Uses typeVersion 2.1, requiring n8n versions supporting Gmail OAuth2 node with these options.  
- **Potential Failures:**  
  - Authentication failures due to invalid or expired OAuth2 credentials.  
  - Email sending errors if recipient address is invalid or Gmail API limits are exceeded.  
  - Expression errors if input data is malformed or missing.  
- **Sticky Note (related):**  
  - Warns users that changing the subject prefix requires updating any Gmail filters relying on that prefix.

---

#### 2.3 Documentation and User Guidance

**Overview:**  
This block provides context, instructions, and warnings to users to ensure proper understanding and usage of the workflow.

**Nodes Involved:**  
- Sticky Note1 (usage and setup instructions)  
- Sticky Note (subject change warning)

**Node Details:**

- **Sticky Note1:**  
  - Contains detailed instructions about the workflow’s purpose: sending standardized alert emails with prefixed subjects for easy Gmail filtering.  
  - Explains input parameters and recommends Gmail inbox configuration (starring messages rather than marking important).  
  - Positioned separately to serve as a reference for workflow users.  
- **Sticky Note:**  
  - Positioned near the email sending node, advising that changing the subject prefix requires updating Gmail filters accordingly.  
- **Potential Issues:**  
  - Users ignoring these notes may misconfigure email filters or fail to fill in the recipient email address, leading to missed alerts.

---

### 3. Summary Table

| Node Name                   | Node Type                        | Functional Role                 | Input Node(s)                  | Output Node(s)            | Sticky Note                                                                                      |
|-----------------------------|---------------------------------|--------------------------------|-------------------------------|---------------------------|------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger (n8n-nodes-base.executeWorkflowTrigger) | Input Reception / Sub-workflow Entry | —                             | Send a message            |                                                                                                |
| Send a message              | Gmail (n8n-nodes-base.gmail)    | Email Composition and Sending  | When Executed by Another Workflow | —                         | ⚠️ If you change the subject, you should change the email filter that selects these emails!     |
| Sticky Note                 | Sticky Note (n8n-nodes-base.stickyNote) | User Guidance / Warning        | —                             | —                         | ⚠️ If you change the subject, you should change the email filter that selects these emails!     |
| Sticky Note1                | Sticky Note (n8n-nodes-base.stickyNote) | User Guidance / Documentation  | —                             | —                         | This is a basic helper workflow to abstract the process of sending an alert email through Gmail.<br>It takes in two parameters:<br>- Subject<br>- Lines (as an array of lines)<br>You'll also need to fill in your email.<br>Notably, all emails it sends have `❗ n8n Alert: ` prefixed to the subject line, which makes them easy to identify and highlight in an email inbox.<br>In Gmail, star these messages and use Priority Inbox to push starred messages to the top.<br>_It's important to star the message rather than Mark As Important, because Gmail refuses to mark emails sent by automation as important._ |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named "❗ Send Email Alert".

2. **Add the Execute Workflow Trigger node:**
   - Node Type: `Execute Workflow Trigger` (n8n-nodes-base.executeWorkflowTrigger)
   - Name it: `When Executed by Another Workflow`
   - Configure Inputs:  
     - Add two inputs:  
       - `subject` (string)  
       - `lines` (array)
   - Position: [0, 0]

3. **Add the Gmail node:**
   - Node Type: `Gmail` (n8n-nodes-base.gmail)
   - Name it: `Send a message`
   - Position: [208, 0]
   - Configure parameters:  
     - `Send To`: Replace `"YOUREMAILHERE@gmail.com"` with your actual recipient email address.  
     - `Subject`: Use expression mode and enter:  
       ```
       =❗ n8n Alert: {{ $json.subject }}
       ```
     - `Message`: Use expression mode and enter:  
       ```
       ={{ $json.lines.join("\n") }}
       ```
     - `Email Type`: Set to `text` (plain text)  
     - Options: Disable `appendAttribution` to prevent adding n8n branding in the email body.  
   - Credentials:  
     - Set up Gmail OAuth2 credentials with required scopes to send emails. Assign them to this node.

4. **Connect the nodes:**
   - Connect the output of `When Executed by Another Workflow` to the input of `Send a message`.

5. **Add Sticky Notes for guidance:**
   - Add a sticky note near the `Send a message` node with content:  
     ```
     ⚠️ If you change the subject, you should change the email filter that selects these emails!
     ```
   - Add a larger sticky note somewhere visible with the following content:  
     ```
     This is a basic helper workflow to abstract the process of sending an alert email through Gmail.

     It takes in two parameters:
     - Subject
     - Lines (as an array of lines)

     You'll also need to fill in your email.

     Notably, all emails it sends have `❗ n8n Alert: ` prefixed to the subject line, which makes them easy to identify and highlight in an email inbox.

     In Gmail, star these messages and use Priority Inbox to push starred messages to the top.

     _It's important to star the message rather than Mark As Important, because Gmail refuses to mark emails sent by automation as important._
     ```

6. **Set workflow settings:**
   - Ensure the execution order is set to "v1" to maintain expected execution behavior.
   - Save and activate the workflow.

7. **Testing:**
   - Trigger this workflow from another workflow or manually with input data:  
     - Example input:  
       ```json
       {
         "subject": "Subject",
         "lines": ["This is a line", "", "This is another line"]
       }
       ```
   - Confirm that the email is received with the correct subject prefix and message body format.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                           | Context or Link                                                                                         |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| The workflow prefixes all alert emails with `❗ n8n Alert:` to enable Gmail filtering and easy identification. Gmail filters can be configured to star such messages, and Priority Inbox can be used to prioritize starred messages. Star the messages instead of marking them as important. | Workflow email filtering recommendation                                                               |
| Gmail OAuth2 credentials must be correctly configured with appropriate scopes to send emails via the Gmail node. Follow n8n documentation for setting up Gmail OAuth2 credentials.                                                                     | [n8n Gmail OAuth2 Setup](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/)     |
| This workflow is designed as a helper sub-workflow to be invoked by other workflows to send alert emails uniformly.                                                                                                                                  | Workflow usage pattern                                                                                  |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.