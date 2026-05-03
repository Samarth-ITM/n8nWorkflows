Private Secret Santa Assignment & Notification System with Gmail & Verification

https://n8nworkflows.xyz/workflows/private-secret-santa-assignment---notification-system-with-gmail---verification-9609


# Private Secret Santa Assignment & Notification System with Gmail & Verification

### 1. Workflow Overview

This workflow implements a **Private Secret Santa Assignment & Notification System** using Gmail for communication and verification. The main purpose is to randomly assign Secret Santa pairings among a predefined list of participants and notify each participant privately via email about their assigned recipient. The workflow also deletes sent messages from the Gmail Sent folder to maintain privacy and compiles a summary log of all assignments.

The workflow is organized into the following logical blocks:

- **1.1 Input and Initialization:** Define Secret Santa participants and trigger the workflow manually.
- **1.2 Random Pairing Generation:** Generate random Secret Santa assignments ensuring no self-assignments or duplicates.
- **1.3 Notification Loop:** Iterate over each assignment, send personalized emails, delete sent messages to preserve secrecy, and log assignment info.
- **1.4 Summary Compilation and Notification:** Aggregate all assignments into a compact format and send a final summary email.
- **1.5 Utility and Logging:** Transform email addresses into numeric IDs for concise reporting.

---

### 2. Block-by-Block Analysis

#### 2.1 Input and Initialization

**Overview:**  
This block sets up the participants as name-email pairs and provides a manual trigger to start the workflow.

**Nodes Involved:**  
- Run  
- Emails and name  
- Sticky Note (explanatory)

**Node Details:**  

- **Run**  
  - Type: Manual Trigger  
  - Role: Starts the workflow manually in n8n.  
  - Configuration: No parameters; acts as entry point.  
  - Inputs: None  
  - Outputs: Connects to "Emails and name"  
  - Failure Modes: None (manual trigger)  
  - Sub-workflows: None  

- **Emails and name**  
  - Type: Set  
  - Role: Defines participants by assigning name → email pairs as JSON keys and values.  
  - Configuration: Manually set fields like `Jesus` = `example@gmail.com`, `John` = `example2@gmail.com`, `Jan` = `example3@gmail.com`.  
  - Inputs: From "Run"  
  - Outputs: Connects to "Random"  
  - Edge Cases: Must have at least 2 valid name-email pairs; otherwise downstream errors occur.  
  - Notes: Participants must be valid emails; duplicates will be filtered downstream.  

- **Sticky Note** (positioned near initial nodes)  
  - Purpose: Describes the function of the manual trigger and the participant definition.

---

#### 2.2 Random Pairing Generation

**Overview:**  
Generates random Secret Santa assignments ensuring no participant is assigned to themselves and no duplicate assignments exist.

**Nodes Involved:**  
- Random (Code)  
- loop mails (SplitInBatches)  
- Sticky Note (attached to this block)

**Node Details:**  

- **Random**  
  - Type: Code (JavaScript)  
  - Role: Reads input JSON of name-email pairs, filters valid emails, removes duplicates, and produces deranged pairings (no self-assignment).  
  - Configuration: Custom JavaScript implementing:  
    - Validation of emails with regex  
    - Creation of participant arrays from JSON keys and values  
    - Shuffle and derangement algorithm with retries  
    - Output: array of items `{emisor, nombreEmisor, receptor, nombreReceptor}`  
  - Inputs: From "Emails and name"  
  - Outputs: Connects to "loop mails"  
  - Edge Cases:  
    - Throws error if fewer than 2 valid participants  
    - Throws error if no valid derangement after 20 attempts  
    - Ignores invalid emails or duplicates silently  
  - Failure Modes: Code exceptions, invalid input JSON structure  

- **loop mails**  
  - Type: SplitInBatches  
  - Role: Iterates over each Secret Santa assignment item one-by-one to handle sequential processing.  
  - Configuration: Default batch size = 1  
  - Inputs: From "Random"  
  - Outputs: Two connections: to "Name to INT" and "Send a message" for parallel processing  
  - Notes: Supports sequential processing per assignment to maintain clarity and control.  

- **Sticky Note**  
  - Describes the random pairing logic and batch iteration purpose.

---

#### 2.3 Notification Loop

**Overview:**  
For each assignment, an email is sent to the giver notifying their recipient, the sent message is deleted to maintain secrecy, and a log string is created to track the assignment.

**Nodes Involved:**  
- Send a message (Gmail)  
- Delete a message (Gmail)  
- Who? (Set)  
- Sticky Note (attached to nodes involved)

**Node Details:**  

- **Send a message**  
  - Type: Gmail (OAuth2)  
  - Role: Sends an HTML email to the giver (`emisor`) with their assigned recipient's name.  
  - Configuration:  
    - Recipient: `={{ $json.emisor }}` (giver email)  
    - Subject: `Secret Santa`  
    - Message: HTML formatted message showing `<giver name> → <receiver name>`  
    - Options: Attribution disabled  
    - Credentials: Gmail OAuth2 account  
    - Retry: Enabled with 5s delay between tries  
  - Inputs: From "loop mails"  
  - Outputs: Connects to "Delete a message"  
  - Edge Cases: Auth errors, rate limits, invalid emails, network timeouts.  
  - Failure Modes: Email sending failure triggers retry.  

- **Delete a message**  
  - Type: Gmail (OAuth2)  
  - Role: Deletes the sent message from the Sent folder using the message ID from the previous node to preserve secrecy.  
  - Configuration:  
    - Message ID: `={{ $json.id }}` (from "Send a message")  
    - Operation: Delete  
    - Credentials: Same Gmail OAuth2 account as sending  
    - Retry: Enabled with 5s delay  
  - Inputs: From "Send a message"  
  - Outputs: Connects to "Who?"  
  - Edge Cases: Message may not exist if sending failed; network issues.  
  - Failure Modes: Delete failure triggers retry.  

- **Who?**  
  - Type: Set  
  - Role: Creates a text field `info` with the format `"giver_email envió a receiver_email"` for logging.  
  - Configuration: Sets `info` with current loop item JSON fields `emisor` and `receptor`.  
  - Inputs: From "Delete a message"  
  - Outputs: Connects back to "loop mails" (completes iteration)  
  - Edge Cases: JSON fields missing or malformed cause incomplete logging.  

- **Sticky Note**  
  - Explains the email sending, message deletion for privacy, and info logging.

---

#### 2.4 Summary Compilation and Notification

**Overview:**  
After all individual emails are sent and logged, this block aggregates all assignment info lines, converts emails to numeric IDs for compactness, and sends a summary email to a fixed recipient.

**Nodes Involved:**  
- Name to INT (Code)  
- Send a message results (Gmail)  
- Sticky Note (attached to these nodes)

**Node Details:**  

- **Name to INT**  
  - Type: Code (JavaScript)  
  - Role: Processes all `info` lines, extracts two emails per line, assigns unique numeric IDs to each email, and outputs a single HTML string summarizing all assignments as `"<id1> envió a <id2>"`.  
  - Configuration:  
    - Uses regex to extract emails from the `info` strings  
    - Builds a map of emails to incremental IDs  
    - Joins all lines with `<br>` for HTML formatting  
  - Inputs: From "loop mails" (consolidated batch)  
  - Outputs: Connects to "Send a message results"  
  - Edge Cases: No valid `info` lines cause error; malformed lines skipped silently.  

- **Send a message results**  
  - Type: Gmail (OAuth2)  
  - Role: Sends a final summary email to a fixed recipient (hardcoded as `youremail@gmail.com`) with subject "Amic invisible" containing the aggregated assignment info.  
  - Configuration:  
    - Recipient: `youremail@gmail.com` (replace with actual email)  
    - Subject: "Amic invisible"  
    - Message body includes greeting and the HTML-formatted summary from previous node.  
    - Retry enabled with 5s delay  
    - Credentials: Same Gmail OAuth2 account  
  - Inputs: From "Name to INT"  
  - Outputs: None (end of workflow)  
  - Edge Cases: Auth issues, invalid destination email, network errors.  

- **Sticky Note**  
  - Explains the conversion of emails to numeric IDs and sending the final summary.

---

#### 2.5 Utility and Logging

**Overview:**  
This block mainly includes nodes that support logging and tracking of assignments with numeric simplification for easy review.

**Nodes Involved:**  
- Name to INT  
- Who?  

**Details:**  
Covered above as part of main blocks; they help produce a clean, numeric summary of the Secret Santa assignments for audit or record keeping.

---

### 3. Summary Table

| Node Name             | Node Type          | Functional Role                                     | Input Node(s)        | Output Node(s)                 | Sticky Note                                                                                                     |
|-----------------------|--------------------|----------------------------------------------------|----------------------|-------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Run                   | Manual Trigger     | Starts workflow manually                            | None                 | Emails and name               | Run (Manual Trigger) starts the workflow manually.                                                             |
| Emails and name       | Set                | Defines participants as name → email pairs          | Run                  | Random                       | Defines Secret Santa participants as name → email pairs.                                                      |
| Random                | Code                | Generates random Secret Santa pairings               | Emails and name       | loop mails                   | Generates pairings without self-assignment or duplicates.                                                      |
| loop mails            | SplitInBatches      | Iterates over each pairing item                       | Random                | Name to INT, Send a message  | Iterates each pairing one-by-one for processing.                                                               |
| Send a message        | Gmail               | Sends personalized Secret Santa email                | loop mails            | Delete a message             | Sends email to giver about their assigned recipient, retries on failure.                                       |
| Delete a message      | Gmail               | Deletes sent email message for privacy               | Send a message        | Who?                        | Deletes sent message from Sent folder to maintain secrecy.                                                     |
| Who?                  | Set                 | Creates log string "giver_email envió a receiver_email" | Delete a message      | loop mails                  | Logs each assignment for summary.                                                                               |
| Name to INT           | Code                | Converts emails to numeric IDs and summarizes assignments | loop mails            | Send a message results       | Processes all info lines to build a compact numeric summary.                                                   |
| Send a message results | Gmail               | Sends final summary email to fixed recipient          | Name to INT           | None                        | Sends summary email with all Secret Santa pairings.                                                            |
| Sticky Note           | Sticky Note         | Various explanations and instructions                 | Various               | Various                     | Multiple notes explaining node functions and workflow usage.                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: `Run`  
   - Purpose: Manual start of the workflow. No parameters needed.

2. **Create a Set Node for Participants**  
   - Name: `Emails and name`  
   - Add fields with participant names as keys and their emails as string values, e.g.:  
     - Jesus = example@gmail.com  
     - John = example2@gmail.com  
     - Jan = example3@gmail.com  
   - Connect `Run` output to this node.

3. **Create a Code Node for Random Assignments**  
   - Name: `Random`  
   - Paste JavaScript code that:  
     - Reads input JSON keys and values as name-email pairs  
     - Validates emails and removes duplicates  
     - Generates deranged assignments (no one assigned to self)  
     - Throws errors for invalid or insufficient participants  
     - Outputs items with `{emisor, nombreEmisor, receptor, nombreReceptor}`  
   - Connect `Emails and name` output to this node.

4. **Add a SplitInBatches Node**  
   - Name: `loop mails`  
   - Configure default batch size = 1 (iterate items one by one)  
   - Connect `Random` output to this node.

5. **Create a Gmail Send Node for Notifications**  
   - Name: `Send a message`  
   - Credentials: Configure Gmail OAuth2 credentials.  
   - Parameters:  
     - Send To: `={{ $json.emisor }}`  
     - Subject: `Secret Santa`  
     - Message: HTML formatted message showing `{{ $json.nombreEmisor }} → {{ $json.nombreReceptor }}`  
     - Disable Attribution  
   - Connect `loop mails` output to this node.

6. **Create a Gmail Delete Node**  
   - Name: `Delete a message`  
   - Credentials: Same Gmail OAuth2 credentials as sending node.  
   - Parameters:  
     - Operation: Delete  
     - Message ID: `={{ $json.id }}` (from sent message)  
   - Connect `Send a message` output to this node.

7. **Create a Set Node for Logging**  
   - Name: `Who?`  
   - Parameters:  
     - Set field `info` to: `={{ $('loop mails').item.json.emisor }} envió a {{ $('loop mails').item.json.receptor }}`  
   - Connect `Delete a message` output to this node.

8. **Connect `Who?` back to `loop mails`**  
   - This closes the iteration loop for batch processing.

9. **Create a Code Node to Convert Emails to Numeric IDs**  
   - Name: `Name to INT`  
   - Paste JavaScript code that:  
     - Takes all `info` lines from input items  
     - Extracts emails and assigns numeric IDs  
     - Compiles a single HTML string like `1 envió a 2<br>2 envió a 3`  
     - Throws error if no valid lines found  
   - Connect `loop mails` output to this node (ensure it receives all items).

10. **Create a Final Gmail Send Node for Summary**  
    - Name: `Send a message results`  
    - Credentials: Same Gmail OAuth2 credentials.  
    - Parameters:  
      - Send To: your email address (e.g., `youremail@gmail.com`)  
      - Subject: `Amic invisible`  
      - Message Body: Includes greeting and `{{ $json.info }}` from previous node  
    - Connect `Name to INT` output to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                             | Context or Link                               |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------|
| Thanks for using this workflow 🙌 If you enjoyed it and want to discover more creative and original automations, visit my n8n Creator profile: https://n8n.io/creators/oxsr11/ There you’ll find unique, AI-powered, and real-world inspired workflows designed to make automation smarter and more fun. Also visit https://github.com/OXSR/ for more resources. | Branding and project credits                   |
| Non-Commercial Use and Modification License: Permission granted to use, copy, modify, and redistribute for free with attribution. Commercial use prohibited. Software provided “as is” without warranty. Copyright © [2025/6] - Oseguir.                                                                                                                                    | License terms included as sticky note          |

---

**Disclaimer:**  
The provided text and workflow come exclusively from an automated n8n workflow. It complies strictly with applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly available.