Send Email if server has upgradable packages

https://n8nworkflows.xyz/workflows/send-email-if-server-has-upgradable-packages-2925


# Send Email if server has upgradable packages

### 1. Workflow Overview

This workflow automates the daily monitoring of upgradable packages on an Ubuntu server and sends an email notification if updates are available. It is designed for system administrators or DevOps engineers who want to maintain server security and performance by staying informed about package upgrades without manual intervention.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger**: Initiates the workflow execution daily.
- **1.2 Package Upgrade Check via SSH**: Connects to the Ubuntu server over SSH and runs a command to list upgradable packages.
- **1.3 Data Formatting**: Converts the raw command output into a clean HTML list for email readability.
- **1.4 Conditional Check**: Determines if any packages are upgradable and decides whether to send an email.
- **1.5 Email Notification**: Sends an email alert with the list of upgradable packages if any are found.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the workflow automatically once every day to perform the upgrade check without manual initiation.

- **Nodes Involved:**  
  - Run workflow every day

- **Node Details:**  
  - **Run workflow every day**  
    - Type: Schedule Trigger  
    - Configuration: Set to trigger at a daily interval (default time unspecified, runs once every 24 hours)  
    - Inputs: None (trigger node)  
    - Outputs: Connects to "List upgradable packages" node  
    - Edge Cases: Misconfiguration of schedule interval could cause no runs or multiple runs per day  
    - Version: 1.2  
    - Notes: Ensures automation and timeliness of the update checks

#### 1.2 Package Upgrade Check via SSH

- **Overview:**  
  Connects securely to the Ubuntu server via SSH and executes the command `apt list --upgradable` to retrieve a list of packages that can be upgraded.

- **Nodes Involved:**  
  - List upgradable packages

- **Node Details:**  
  - **List upgradable packages**  
    - Type: SSH  
    - Configuration: Runs the shell command `apt list --upgradable` on the remote server  
    - Credentials: Uses SSH Password authentication (credential named "SSH Password account")  
    - Inputs: Triggered by schedule node  
    - Outputs: Raw command output passed to "Format as HTML list" node  
    - Edge Cases:  
      - SSH connection failures (wrong credentials, network issues)  
      - Command execution errors (e.g., server not Ubuntu, apt not installed)  
      - Empty output if no packages are upgradable  
    - Version: 1  
    - Notes: Command output includes package names and versions in plain text

#### 1.3 Data Formatting

- **Overview:**  
  Transforms the raw text output from the SSH command into a structured HTML unordered list for better readability in the notification email.

- **Nodes Involved:**  
  - Format as HTML list

- **Node Details:**  
  - **Format as HTML list**  
    - Type: Code (JavaScript)  
    - Configuration: Custom JS code that:  
      - Splits the command output by new lines  
      - Filters out empty lines and the header line "Listing..."  
      - Wraps each package line in `<li>` tags  
      - Wraps the entire list in `<ul>` tags  
      - Returns an object with property `htmlList` containing the HTML string  
    - Inputs: Receives `stdout` string from SSH node  
    - Outputs: Passes formatted HTML string to "Check if there are updates" node  
    - Edge Cases:  
      - Unexpected command output format could break parsing  
      - Empty or missing stdout property  
    - Version: 2

#### 1.4 Conditional Check

- **Overview:**  
  Checks if the formatted HTML list is empty (`<ul></ul>`), indicating no upgradable packages. Only if packages exist does it proceed to send an email.

- **Nodes Involved:**  
  - Check if there are updates

- **Node Details:**  
  - **Check if there are updates**  
    - Type: If (conditional)  
    - Configuration:  
      - Condition: `$json.htmlList` not equal to `<ul></ul>`  
      - Case sensitive, strict type validation  
    - Inputs: Receives formatted HTML list  
    - Outputs:  
      - True branch: Connects to "Send Email through SMTP" node  
      - False branch: Ends workflow (no email sent)  
    - Edge Cases:  
      - If formatting changes, condition might fail to detect empty lists  
      - Expression evaluation errors if `htmlList` missing  
    - Version: 2.2

#### 1.5 Email Notification

- **Overview:**  
  Sends an email notification with the list of upgradable packages formatted in HTML, alerting the recipient to perform server updates.

- **Nodes Involved:**  
  - Send Email through SMTP

- **Node Details:**  
  - **Send Email through SMTP**  
    - Type: Email Send  
    - Configuration:  
      - Subject: "Server needs updates"  
      - To: Configurable recipient email (default placeholder: change.me@example.com)  
      - From: Configurable sender email (default placeholder: change.me@example.com)  
      - Body: HTML content including the `htmlList` variable injected into the message  
      - Options: Default SMTP options  
    - Credentials: SMTP credentials (credential named "SMTP account")  
    - Inputs: Triggered only if updates exist  
    - Outputs: None (terminal node)  
    - Edge Cases:  
      - SMTP authentication failures  
      - Invalid email addresses  
      - Network issues preventing email delivery  
    - Version: 2.1  
    - Notes: Requires user to update email addresses before use

---

### 3. Summary Table

| Node Name               | Node Type          | Functional Role              | Input Node(s)            | Output Node(s)           | Sticky Note                                                                                  |
|-------------------------|--------------------|-----------------------------|--------------------------|--------------------------|----------------------------------------------------------------------------------------------|
| Run workflow every day   | Schedule Trigger   | Initiates daily execution    | None                     | List upgradable packages |                                                                                              |
| List upgradable packages | SSH                | Executes upgrade check cmd   | Run workflow every day    | Format as HTML list       | apt list --upgradable                                                                        |
| Format as HTML list      | Code (JavaScript)  | Formats raw output to HTML   | List upgradable packages  | Check if there are updates|                                                                                              |
| Check if there are updates| If                 | Checks if updates exist      | Format as HTML list       | Send Email through SMTP   |                                                                                              |
| Send Email through SMTP  | Email Send         | Sends notification email    | Check if there are updates| None                     | Update From and To email addresses in this node to receive notifications                      |
| Sticky Note              | Sticky Note        | Workflow description        | None                     | None                     | ## VPS upgrade notify \nThis workflow will everyday check if server has upgradable packages and inform you by email if there is. |
| Sticky Note1             | Sticky Note        | Email address reminder      | None                     | None                     | ## Update email addresses\nUpdate From and To email addresses in this node to receive notifications |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Name: `Run workflow every day`  
   - Type: Schedule Trigger  
   - Set the trigger interval to run once every day (default daily interval)  

2. **Create an SSH node**  
   - Name: `List upgradable packages`  
   - Type: SSH  
   - Configure the command to run: `apt list --upgradable`  
   - Set SSH credentials using password authentication (create or select credential named "SSH Password account")  
   - Connect the output of `Run workflow every day` to this node's input  

3. **Create a Code node**  
   - Name: `Format as HTML list`  
   - Type: Code (JavaScript)  
   - Paste the following JS code to format the stdout:  
     ```javascript
     function formatStdoutAsHtmlList(stdoutData) {
         const htmlListItems = stdoutData.split('\n').map((line) => {
             if (line.trim() && line !== "Listing...") {
                 return `<li>${line.trim()}</li>`;
             }
         }).filter(item => item);
         const htmlList = `<ul>${htmlListItems.join('')}</ul>`;
         return { "htmlList": htmlList };
     }
     return formatStdoutAsHtmlList($input.first().json.stdout);
     ```  
   - Connect the output of `List upgradable packages` to this node's input  

4. **Create an If node**  
   - Name: `Check if there are updates`  
   - Type: If  
   - Set condition: Check if expression `{{$json.htmlList}}` is **not equal** to `<ul></ul>` (case sensitive, strict)  
   - Connect the output of `Format as HTML list` to this node's input  

5. **Create an Email Send node**  
   - Name: `Send Email through SMTP`  
   - Type: Email Send  
   - Configure:  
     - Subject: `Server needs updates`  
     - To Email: Set your recipient email address  
     - From Email: Set your sender email address  
     - Email Body (HTML):  
       ```
       The following packages can be updated on your server:

       {{ $json.htmlList }}

       Please login and perform upgrade.
       ```  
   - Set SMTP credentials (create or select credential named "SMTP account")  
   - Connect the **true** output of `Check if there are updates` to this node's input  

6. **Add Sticky Notes (optional but recommended)**  
   - Add a sticky note near the start explaining the workflow purpose:  
     ```
     ## VPS upgrade notify 
     This workflow will everyday check if server has upgradable packages and inform you by email if there is.
     ```  
   - Add a sticky note near the email node reminding to update email addresses:  
     ```
     ## Update email addresses
     Update From and To email addresses in this node to receive notifications
     ```  

7. **Save and activate the workflow**  
   - Ensure all credentials are valid and tested  
   - Activate the workflow to run daily  

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow requires valid SSH credentials with permission to run `apt list --upgradable`.    | SSH Credential setup                                                                             |
| SMTP credentials must allow sending emails from the configured sender address.                   | SMTP Credential setup                                                                            |
| The command output format is expected to be standard Ubuntu `apt` output; changes may break parsing. | Command output parsing                                                                           |
| Reminder: Update the "To" and "From" email addresses in the email node before use.               | Sticky note in workflow                                                                          |
| For more advanced monitoring, consider integrating with alerting platforms or dashboards.       | External enhancement suggestion                                                                 |

---

This documentation provides a complete, detailed reference for understanding, reproducing, and maintaining the "Send Email if server has upgradable packages" workflow in n8n.