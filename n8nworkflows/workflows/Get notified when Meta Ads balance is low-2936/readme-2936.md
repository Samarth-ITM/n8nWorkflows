Get notified when Meta Ads balance is low

https://n8nworkflows.xyz/workflows/get-notified-when-meta-ads-balance-is-low-2936


# Get notified when Meta Ads balance is low

### 1. Workflow Overview

This workflow automates notifications when a Facebook (Meta) Ads account balance falls below a specified threshold. It targets marketing professionals who want to avoid manual balance checks by receiving timely alerts via Telegram or other messaging platforms.

The workflow includes two alternative methods to fetch the available balance, accommodating different billing setups in Facebook Ads accounts. After retrieving the balance, it evaluates whether the amount is below a preset limit (400 units in this case) and sends a notification message if so.

Logical blocks:

- **1.1 Scheduled Trigger:** Periodically initiates the balance check every 12 hours.
- **1.2 Balance Retrieval Methods:** Two parallel methods (Method 1 and Method 2) using Facebook Graph API nodes to fetch balance data depending on account billing setup.
- **1.3 Data Processing:** Nodes that format and prepare the balance data for evaluation.
- **1.4 Threshold Evaluation:** Conditional check to determine if the balance is below the threshold.
- **1.5 Notification Dispatch:** Sends a Telegram message alert if the balance is low.
- **1.6 No Operation:** Placeholder node for cases when balance is above threshold (no action taken).
- **1.7 Sticky Notes:** Documentation and guidance nodes for user reference.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  Initiates the workflow automatically every 12 hours to check the Facebook Ads balance.

- **Nodes Involved:**  
  - Every 12h

- **Node Details:**  
  - **Every 12h**  
    - Type: Schedule Trigger  
    - Configuration: Default schedule set to trigger every 12 hours (no custom cron expression shown, uses default 12h interval)  
    - Inputs: None (trigger node)  
    - Outputs: Connects to "Method 1" node to start balance retrieval  
    - Edge Cases: If n8n instance is down or paused, scheduled trigger will not fire; no retry mechanism here  
    - Version: 1.2  

#### 1.2 Balance Retrieval Methods

- **Overview:**  
  Two alternative Facebook Graph API nodes fetch the available balance using different API endpoints or parameters, accommodating different billing setups.

- **Nodes Involved:**  
  - Method 1  
  - Method 2

- **Node Details:**  
  - **Method 1**  
    - Type: Facebook Graph API  
    - Configuration: Calls Facebook API to retrieve balance data (specific endpoint or parameters not shown)  
    - Inputs: Triggered by "Every 12h" node  
    - Outputs: Connects to "Edit Fields" node for data processing  
    - Retries: Up to 5 attempts with 5 seconds wait between tries  
    - Edge Cases: API authentication errors, rate limits, network timeouts, invalid tokens  
    - Version: 1  

  - **Method 2**  
    - Type: Facebook Graph API  
    - Configuration: Alternative API call for balance retrieval  
    - Inputs: Not directly triggered in current connections (likely an alternative path for manual switching)  
    - Outputs: Connects to "Edit Fields1" node for data processing  
    - Retries: Same as Method 1  
    - Edge Cases: Same as Method 1  
    - Version: 1  

#### 1.3 Data Processing

- **Overview:**  
  These nodes prepare and format the balance data retrieved from Facebook API calls for evaluation.

- **Nodes Involved:**  
  - Edit Fields  
  - Edit Fields1

- **Node Details:**  
  - **Edit Fields**  
    - Type: Set  
    - Configuration: Likely extracts and formats balance value from Method 1 response (exact fields not shown)  
    - Inputs: From "Method 1"  
    - Outputs: Connects to "Lower than 400 ?" conditional node  
    - Edge Cases: Expression errors if expected data fields are missing or malformed  
    - Version: 3.4  

  - **Edit Fields1**  
    - Type: Set  
    - Configuration: Similar to "Edit Fields" but for Method 2 response data  
    - Inputs: From "Method 2"  
    - Outputs: Not connected in current workflow (manual switching needed)  
    - Edge Cases: Same as "Edit Fields"  
    - Version: 3.4  

#### 1.4 Threshold Evaluation

- **Overview:**  
  Evaluates if the processed balance is below the threshold (400 units) to decide whether to send a notification.

- **Nodes Involved:**  
  - Lower than 400 ?

- **Node Details:**  
  - **Lower than 400 ?**  
    - Type: If (conditional)  
    - Configuration: Checks if balance value < 400  
    - Inputs: From "Edit Fields"  
    - Outputs:  
      - True branch: "Send message" node  
      - False branch: "No Operation, do nothing" node  
    - Edge Cases: Missing or non-numeric balance values could cause evaluation errors  
    - Version: 2.2  

#### 1.5 Notification Dispatch

- **Overview:**  
  Sends a Telegram message alerting the user that the Facebook Ads balance is low.

- **Nodes Involved:**  
  - Send message

- **Node Details:**  
  - **Send message**  
    - Type: Telegram  
    - Configuration: Sends a predefined message to a Telegram chat (chat ID and message content configured in parameters)  
    - Inputs: From "Lower than 400 ?" (true branch)  
    - Outputs: None (end node)  
    - Credentials: Requires Telegram Bot credentials configured in n8n  
    - Edge Cases: Telegram API errors, invalid chat ID, network issues  
    - Version: 1.2  

#### 1.6 No Operation

- **Overview:**  
  Placeholder node that performs no action when balance is above threshold.

- **Nodes Involved:**  
  - No Operation, do nothing

- **Node Details:**  
  - **No Operation, do nothing**  
    - Type: NoOp (no operation)  
    - Configuration: No parameters  
    - Inputs: From "Lower than 400 ?" (false branch)  
    - Outputs: None  
    - Purpose: Explicitly ends workflow branch without side effects  
    - Version: 1  

#### 1.7 Sticky Notes

- **Overview:**  
  Provide user guidance, instructions, and documentation within the workflow canvas.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2  
  - Sticky Note3  
  - Sticky Note4  
  - Sticky Note5  
  - Sticky Note6  
  - Sticky Note7

- **Node Details:**  
  - All are Sticky Note nodes with empty content in the JSON provided, presumably placeholders or for user-added notes.  
  - Positioned near relevant nodes for contextual help.  
  - No inputs or outputs.  
  - Version: 1  

---

### 3. Summary Table

| Node Name              | Node Type             | Functional Role                | Input Node(s)       | Output Node(s)          | Sticky Note |
|------------------------|-----------------------|-------------------------------|---------------------|-------------------------|-------------|
| Every 12h              | Schedule Trigger      | Periodic workflow trigger     | None                | Method 1                |             |
| Method 1               | Facebook Graph API    | Retrieve balance (method 1)   | Every 12h           | Edit Fields             |             |
| Edit Fields            | Set                   | Format/process balance data   | Method 1            | Lower than 400 ?        |             |
| Lower than 400 ?       | If                    | Check if balance < 400        | Edit Fields         | Send message, No Operation, do nothing |             |
| Send message           | Telegram               | Send low balance notification | Lower than 400 ? (true) | None                  |             |
| No Operation, do nothing | NoOp                 | End branch if balance OK      | Lower than 400 ? (false) | None                  |             |
| Method 2               | Facebook Graph API    | Retrieve balance (method 2)   | None (not connected) | Edit Fields1            |             |
| Edit Fields1           | Set                   | Format/process balance data   | Method 2            | None (not connected)    |             |
| Sticky Note            | Sticky Note           | User guidance/documentation   | None                | None                    |             |
| Sticky Note1           | Sticky Note           | User guidance/documentation   | None                | None                    |             |
| Sticky Note2           | Sticky Note           | User guidance/documentation   | None                | None                    |             |
| Sticky Note3           | Sticky Note           | User guidance/documentation   | None                | None                    |             |
| Sticky Note4           | Sticky Note           | User guidance/documentation   | None                | None                    |             |
| Sticky Note5           | Sticky Note           | User guidance/documentation   | None                | None                    |             |
| Sticky Note6           | Sticky Note           | User guidance/documentation   | None                | None                    |             |
| Sticky Note7           | Sticky Note           | User guidance/documentation   | None                | None                    |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**
   - Name: `Every 12h`
   - Type: Schedule Trigger
   - Set to trigger every 12 hours (default interval)
   - No credentials needed

3. **Add a Facebook Graph API node for Method 1:**
   - Name: `Method 1`
   - Type: Facebook Graph API
   - Configure credentials with Facebook OAuth2 or token with Ads read permissions
   - Set API endpoint to retrieve Ads account balance (specific endpoint depends on Facebook API version, e.g., `/act_<AD_ACCOUNT_ID>/billing_balance`)
   - Set retry options: max 5 tries, 5 seconds wait between tries
   - Connect `Every 12h` node output to this node input

4. **Add a Set node to process Method 1 data:**
   - Name: `Edit Fields`
   - Type: Set
   - Configure to extract the balance value from the API response JSON (e.g., set a field `balance` with expression referencing the correct JSON path)
   - Connect `Method 1` output to this node input

5. **Add an If node to evaluate balance threshold:**
   - Name: `Lower than 400 ?`
   - Type: If
   - Condition: Check if `balance` field < 400
   - Connect `Edit Fields` output to this node input

6. **Add a Telegram node to send notification:**
   - Name: `Send message`
   - Type: Telegram
   - Configure Telegram Bot credentials (Bot token)
   - Set chat ID and message text (e.g., "Alert: Your Facebook Ads balance is below 400.")
   - Connect `Lower than 400 ?` node's true output to this node input

7. **Add a No Operation node:**
   - Name: `No Operation, do nothing`
   - Type: NoOp
   - Connect `Lower than 400 ?` node's false output to this node input

8. **(Optional) Add a second Facebook Graph API node for Method 2:**
   - Name: `Method 2`
   - Type: Facebook Graph API
   - Configure similarly to Method 1 but with alternative API endpoint or parameters
   - Add a Set node named `Edit Fields1` to process Method 2 data
   - Connect `Method 2` output to `Edit Fields1`
   - This branch can be connected to the workflow as an alternative to Method 1 depending on user choice

9. **Add Sticky Note nodes as needed for documentation and instructions.**

10. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow supports replacing the Telegram node with other messaging platforms like WhatsApp, Slack, Discord, Twilio, or Email. | User customization note from workflow description                                                |
| Creator's other templates are available at: https://n8n.io/creators/solomon/                      | External resource for additional workflow templates                                              |
| Two methods are provided to retrieve Facebook Ads balance to accommodate different billing setups | Workflow description and screenshot references                                                  |
| Ensure Facebook API credentials have the necessary permissions to read Ads account billing data  | Important for avoiding authentication and permission errors                                     |
| Telegram Bot credentials must be configured in n8n for message sending                            | Credential setup requirement                                                                    |

---

This documentation provides a complete, structured reference to understand, reproduce, and modify the "Get notified when Meta Ads balance is low" workflow in n8n. It highlights node roles, configurations, connections, and potential failure points to facilitate robust usage and customization.