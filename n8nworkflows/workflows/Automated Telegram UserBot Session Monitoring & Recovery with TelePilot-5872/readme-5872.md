Automated Telegram UserBot Session Monitoring & Recovery with TelePilot

https://n8nworkflows.xyz/workflows/automated-telegram-userbot-session-monitoring---recovery-with-telepilot-5872


# Automated Telegram UserBot Session Monitoring & Recovery with TelePilot

### 1. Workflow Overview

This workflow automates monitoring and recovery of Telegram UserBot sessions using the TelePilot integration in n8n. Its primary goal is to maintain active Telegram sessions by detecting disconnected states and triggering automatic reconnection or manual control commands via chat. It supports session lifecycle commands sent through Telegram chat messages, enabling flexible user interaction with the UserBot session management.

The workflow is logically divided into three main blocks:

**1.1 Input Reception**  
Receives Telegram chat messages to trigger manual session control commands (e.g., start, stop, clear, status).

**1.2 Automatic Session Monitoring**  
A scheduled trigger periodically checks the Telegram UserBot session status and processes whether the session is active or disconnected.

**1.3 Session Control and Recovery Actions**  
Based on session status or manual commands, this block issues start or stop session commands via TelePilot nodes, handling automatic reconnection or session termination.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
This block listens for incoming Telegram chat messages and routes them to manual control logic for processing UserBot session commands.

- **Nodes Involved:**  
  - When chat message received  
  - Manual control  
  - Sticky Note (commands explanation)

- **Node Details:**  

| Node Name               | Details                                                                                          |
|-------------------------|-------------------------------------------------------------------------------------------------|
| When chat message received | Type: LangChain Chat Trigger node specialized for Telegram chat input. <br> Configuration: Listens for chat messages using a webhook. No filters applied to capture all messages. <br> Inputs: External Telegram chat messages. <br> Outputs: Passes chat message data to Manual control node. <br> Potential failures: Webhook misconfiguration, Telegram API downtime. |
| Manual control           | Type: TelePilot login resource node <br> Configuration: Executes login-related commands based on incoming chat messages (e.g., /start, /stop). Uses configured TelePilot API credentials for authentication. <br> Inputs: Receives chat messages from "When chat message received". <br> Outputs: Performs session commands accordingly. <br> Failure modes: Authentication errors, command misinterpretation, invalid session states. |
| Sticky Note             | Type: Informational sticky note. <br> Content lists supported commands: /start, /stop, /clear, /cred, /stat. <br> Purpose: Provides user guidance inside the n8n editor. |

---

#### 2.2 Automatic Session Monitoring

- **Overview:**  
This block automatically checks the UserBot session status daily at 8 AM. It determines if the session is active or disconnected, triggering appropriate follow-up actions.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get Session Status (Set node)  
  - Automatic control (TelePilot login resource)  
  - Pass on Closed Status (Filter)  
  - Send Closed Status message (Telegram node, disabled)  
  - Stop Session (Set node)  
  - Check Session Connection (Filter)  
  - Send Session Connection message (Telegram node, disabled)  
  - Sticky Note2

- **Node Details:**  

| Node Name                | Details                                                                                          |
|--------------------------|-------------------------------------------------------------------------------------------------|
| Schedule Trigger          | Type: Schedule Trigger <br> Configuration: Runs once daily at 8:00 AM. <br> Inputs: Trigger node, no inputs. <br> Outputs: Fires the workflow to check session status. <br> Failure modes: Scheduler misconfiguration, time zone issues. |
| Get Session Status        | Type: Set node <br> Configuration: Sets fixed variables - sessionId (hardcoded), action ("sendMessage"), and chatInput ("/stat"). These prepare a request to check the Telegram session status. <br> Inputs: Triggered by Schedule Trigger. <br> Outputs: Passes parameters to Automatic control node. <br> Failure modes: Hardcoded sessionId may cause issues if session changes. |
| Automatic control         | Type: TelePilot login resource node <br> Configuration: Executes the "sendMessage" action with "/stat" command to check session status. Uses TelePilot API credentials. <br> Inputs: From Get Session Status. <br> Outputs: Returns session status JSON data. <br> Failures: Authentication failure, Telegram API errors, network timeouts. |
| Pass on Closed Status     | Type: Filter node <br> Configuration: Checks if the session's authState is not "authorizationStateReady". This means the session is closed or disconnected. <br> Inputs: Receives session status data from Automatic control. <br> Outputs: If closed, routes to Send Closed Status message and Stop Session nodes. <br> Failures: JSON path errors if data structure changes. |
| Send Closed Status message| Type: Telegram node (disabled) <br> Configuration: Sends a notification message about the UserBot error including the full JSON status. <br> Inputs: From Pass on Closed Status. <br> Outputs: None. <br> Failures: Telegram API issues, disabled node means no current use. |
| Stop Session             | Type: Set node <br> Configuration: Sets variables to send the "/stop" command to terminate the session. Uses same sessionId hardcoded as before. <br> Inputs: From Pass on Closed Status. <br> Outputs: Triggers Stop auth node. <br> Failure modes: SessionId mismatch, command failure. |
| Check Session Connection  | Type: Filter node <br> Configuration: Checks if authState equals "authorizationStateReady", meaning session is active. <br> Inputs: From Start auth node (after session start). <br> Outputs: Routes to Send Session Connection message node. <br> Failures: JSON structure changes, false positives. |
| Send Session Connection message | Type: Telegram node (disabled) <br> Configuration: Sends notification that the UserBot session is active with JSON details. <br> Inputs: From Check Session Connection. <br> Outputs: None. <br> Failures: Telegram API issues, disabled node not in use currently. |
| Sticky Note2             | Type: Informational sticky note. <br> Content highlights “Automatic Session Check” block purpose. |

---

#### 2.3 Session Control and Recovery Actions

- **Overview:**  
Handles starting and stopping Telegram sessions via TelePilot commands, enabling session recovery or termination based on manual commands or automated monitoring results.

- **Nodes Involved:**  
  - Stop Session (Set node)  
  - Stop auth (TelePilot login resource)  
  - Start Session (Set node)  
  - Start auth (TelePilot login resource)  
  - Sticky Note1

- **Node Details:**  

| Node Name      | Details                                                                                          |
|----------------|-------------------------------------------------------------------------------------------------|
| Stop Session   | Type: Set node <br> Configuration: Prepares a "/stop" message command with sessionId hardcoded, instructing to stop the session. <br> Inputs: From Pass on Closed Status filter. <br> Outputs: Connects to Stop auth node to execute session stop. <br> Failures: Hardcoded sessionId risks. |
| Stop auth      | Type: TelePilot login resource node <br> Configuration: Executes session termination using the TelePilot API and configured credentials. <br> Inputs: From Stop Session set node. <br> Outputs: Triggers Start Session node to attempt session restart. <br> Failures: Authentication failure, network issues. |
| Start Session  | Type: Set node <br> Configuration: Prepares a "/start" message command with sessionId hardcoded for starting a session. <br> Inputs: From Stop auth node. <br> Outputs: Triggers Start auth node. <br> Failures: SessionId hardcoding. |
| Start auth     | Type: TelePilot login resource node <br> Configuration: Executes session start command via TelePilot API with credentials. <br> Inputs: From Start Session node. <br> Outputs: Triggers Check Session Connection to verify session status. <br> Failures: Authentication errors, rate limiting. |
| Sticky Note1   | Type: Informational sticky note. <br> Content simply states "# Automatic Reconnect" to mark this block's purpose. |

---

### 3. Summary Table

| Node Name                  | Node Type                      | Functional Role                      | Input Node(s)                  | Output Node(s)                      | Sticky Note                                                                                         |
|----------------------------|--------------------------------|------------------------------------|-------------------------------|-----------------------------------|---------------------------------------------------------------------------------------------------|
| When chat message received | LangChain Chat Trigger          | Receives Telegram chat messages    | External webhook               | Manual control                    |                                                                                                   |
| Manual control             | TelePilot login resource        | Processes manual session commands  | When chat message received     | None (executes commands)          |                                                                                                   |
| Sticky Note               | Sticky Note                    | Lists supported chat commands      | None                          | None                             | # Supported Commands: /start, /stop, /clear, /cred, /stat                                        |
| Schedule Trigger          | Schedule Trigger                | Triggers daily automatic check     | None                          | Get Session Status               |                                                                                                   |
| Get Session Status        | Set                            | Prepares session status check inputs| Schedule Trigger              | Automatic control                |                                                                                                   |
| Automatic control         | TelePilot login resource        | Sends "/stat" command to get status| Get Session Status            | Pass on Closed Status            |                                                                                                   |
| Pass on Closed Status     | Filter                         | Detects disconnected session       | Automatic control             | Send Closed Status message, Stop Session |                                                                                                   |
| Send Closed Status message| Telegram                       | Sends error notification (disabled)| Pass on Closed Status         | None                             |                                                                                                   |
| Stop Session             | Set                            | Prepares "/stop" command           | Pass on Closed Status         | Stop auth                       |                                                                                                   |
| Stop auth                | TelePilot login resource        | Executes session stop              | Stop Session                 | Start Session                   |                                                                                                   |
| Start Session            | Set                            | Prepares "/start" command          | Stop auth                   | Start auth                     |                                                                                                   |
| Start auth               | TelePilot login resource        | Executes session start             | Start Session               | Check Session Connection         |                                                                                                   |
| Check Session Connection | Filter                         | Detects active session             | Start auth                   | Send Session Connection message  |                                                                                                   |
| Send Session Connection message | Telegram                | Sends active session notification (disabled)| Check Session Connection    | None                             |                                                                                                   |
| Sticky Note2             | Sticky Note                    | Indicates Automatic Session Check  | None                          | None                             |                                                                                                   |
| Sticky Note1             | Sticky Note                    | Marks Automatic Reconnect block    | None                          | None                             |                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "When chat message received" node**  
   - Type: LangChain Chat Trigger (@n8n/n8n-nodes-langchain.chatTrigger)  
   - Parameters: Default, no filters (listen to all messages)  
   - Position: Left side (e.g., x=0, y=640)  
   - Configure webhook permissions and URL as required.

2. **Create "Manual control" node**  
   - Type: TelePilot login resource node (@telepilotco/n8n-nodes-telepilot.telePilot)  
   - Resource: login  
   - Credentials: Configure with TelePilot API credentials for the Telegram account (e.g., "Lesnikov Telegram account")  
   - Connect input from "When chat message received" node.  
   - This node processes commands like /start, /stop, etc.

3. **Create a Sticky Note for Supported Commands**  
   - Content:  
     ```markdown
     # Supported Commands  
     Following commands can be used in Chat:  
     /start - start login via Phone Number and code (MFA supported)  
     /stop - terminates current ClientSession  
     /clear - deletes local tdlib database, requires re-login  
     /cred - show Telegram Credential info  
     /stat - print all open Telegram sessions  
     ```  
   - Position near manual control nodes.

4. **Create "Schedule Trigger" node**  
   - Type: Schedule Trigger (n8n-nodes-base.scheduleTrigger)  
   - Parameters: Interval trigger set to daily at 8:00 AM (triggerAtHour=8)  
   - Position: e.g., x=0, y=224

5. **Create "Get Session Status" node**  
   - Type: Set (n8n-nodes-base.set)  
   - Parameters: Set variables:  
     - sessionId: "67a853b868cd4f06a0113163ec8b2458" (hardcoded session ID, replace as needed)  
     - action: "sendMessage"  
     - chatInput: "/stat"  (to check status)  
   - Connect input from Schedule Trigger node.

6. **Create "Automatic control" node**  
   - Type: TelePilot login resource node  
   - Resource: login  
   - Credentials: same TelePilot API credentials  
   - Connect input from "Get Session Status" node.

7. **Create "Pass on Closed Status" node**  
   - Type: Filter (n8n-nodes-base.filter)  
   - Condition: Check if `authState` not equals "authorizationStateReady"  
   - Connect input from "Automatic control" node.

8. **Create "Send Closed Status message" node** (optional, disabled)  
   - Type: Telegram node (n8n-nodes-base.telegram)  
   - Parameters: Message text with JSON status details for error reporting  
   - Credentials: Telegram bot credentials (e.g., "LesnikovCoreBot")  
   - Node should be disabled if not used.  
   - Connect input from "Pass on Closed Status" node.

9. **Create "Stop Session" node**  
   - Type: Set  
   - Parameters: Set variables:  
     - sessionId: same hardcoded ID as before  
     - action: "sendMessage"  
     - chatInput: "/stop"  
   - Connect input from "Pass on Closed Status" node.

10. **Create "Stop auth" node**  
    - Type: TelePilot login resource node  
    - Resource: login  
    - Credentials: TelePilot API credentials  
    - Connect input from "Stop Session" node.

11. **Create "Start Session" node**  
    - Type: Set  
    - Parameters: Set variables:  
      - sessionId: same hardcoded ID  
      - action: "sendMessage"  
      - chatInput: "/start"  
    - Connect input from "Stop auth" node.

12. **Create "Start auth" node**  
    - Type: TelePilot login resource node  
    - Resource: login  
    - Credentials: TelePilot API credentials  
    - Connect input from "Start Session" node.

13. **Create "Check Session Connection" node**  
    - Type: Filter  
    - Condition: `authState` equals "authorizationStateReady"  
    - Connect input from "Start auth" node.

14. **Create "Send Session Connection message" node** (optional, disabled)  
    - Type: Telegram node  
    - Parameters: Message text with JSON session info for successful connection  
    - Credentials: Telegram bot credentials  
    - Node disabled if not in use.  
    - Connect input from "Check Session Connection" node.

15. **Create Sticky Notes**  
    - One for "Automatic Session Check" near schedule trigger block.  
    - One for "Automatic Reconnect" near session control block.

16. **Link Manual Control Node Output**  
    - Connect "When chat message received" node output to "Manual control" node.

17. **Ensure Credentials Setup**  
    - TelePilot API credentials configured with valid API keys and Telegram account info.  
    - Telegram API credentials configured for sending messages (error or info) if enabled.

---

### 5. General Notes & Resources

| Note Content                                                                                                              | Context or Link                                                  |
|---------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Hardcoded sessionId ("67a853b868cd4f06a0113163ec8b2458") is critical; update it to match your actual Telegram session.     | Session management sensitive to sessionId configuration.        |
| Telegram notification nodes are currently disabled; enable them to receive real-time alerts about session status changes. | Useful for operational monitoring and alerting.                 |
| The workflow uses TelePilot integration from @telepilotco, which requires a valid API key and account setup.              | TelePilot Node Docs: https://docs.telepilot.co/                  |
| Command descriptions are available inside the workflow for user guidance via sticky notes.                                | Helps operators understand supported commands quickly.          |
| Scheduled workflow trigger runs once daily at 8:00 AM to ensure session health checks.                                     | Adjust timing as needed to fit operational requirements.        |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, a workflow automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.