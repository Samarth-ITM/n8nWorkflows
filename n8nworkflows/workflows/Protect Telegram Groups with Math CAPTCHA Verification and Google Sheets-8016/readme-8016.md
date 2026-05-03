Protect Telegram Groups with Math CAPTCHA Verification and Google Sheets

https://n8nworkflows.xyz/workflows/protect-telegram-groups-with-math-captcha-verification-and-google-sheets-8016


# Protect Telegram Groups with Math CAPTCHA Verification and Google Sheets

### 1. Workflow Overview

This workflow implements a protective mechanism for Telegram groups by verifying new members using a math CAPTCHA challenge. It integrates Telegram messaging triggers, CAPTCHA generation, user interaction, answer validation, and member management (welcome or ban) actions. Additionally, it maintains records of user responses in Google Sheets for audit and tracking purposes.

The workflow is logically divided into the following blocks:

- **1.1 Telegram Event Handling:** Detect new Telegram group members and initialize processing.
- **1.2 CAPTCHA Generation and Delivery:** Create a math CAPTCHA question and send it to the new member.
- **1.3 User Response Management:** Capture and store user responses; retrieve pending CAPTCHA challenges.
- **1.4 CAPTCHA Verification:** Verify user answers, decide success or failure, and apply consequences.
- **1.5 Member Management and Cleanup:** Welcome successful members, ban those who fail, and clean up messages and stored data.
- **1.6 Configuration and Guides:** Set bot parameters and provide workflow documentation notes.

---

### 2. Block-by-Block Analysis

#### 2.1 Telegram Event Handling

**Overview:**  
This block listens for Telegram events, specifically new members joining the group, and routes the flow based on the event type.

**Nodes Involved:**  
- 📱 Telegram Trigger  
- ⚙️ Bot Configuration  
- 🔍 Check New Member

**Node Details:**

- **📱 Telegram Trigger**  
  - *Type:* Telegram Trigger node (listens to Telegram updates)  
  - *Configuration:* Default trigger on Telegram messages/events from the configured bot.  
  - *Input:* External Telegram events  
  - *Output:* Passes the event data downstream for processing.  
  - *Edge cases:* Possible delays or missed updates if webhook fails; ensure bot token and webhook setup are correct.  
  - *Version:* 1.2  

- **⚙️ Bot Configuration**  
  - *Type:* Set node (sets variables for the bot)  
  - *Configuration:* Defines parameters like bot token, group IDs, or CAPTCHA difficulty (not explicitly detailed).  
  - *Input:* From Telegram Trigger  
  - *Output:* To Check New Member node  
  - *Edge cases:* Misconfiguration can cause logic errors downstream.  
  - *Version:* 3.4  

- **🔍 Check New Member**  
  - *Type:* If node (conditional branching)  
  - *Configuration:* Checks if the Telegram event corresponds to a new member joining.  
  - *Input:* From Bot Configuration  
  - *Output:* Two branches — True (new member joined) leads to CAPTCHA generation; False leads to pending CAPTCHA check.  
  - *Expressions:* Evaluates Telegram update payload properties indicating a new chat member.  
  - *Edge cases:* Telegram API changes may affect detection; false negatives/positives possible if event structure varies.  
  - *Version:* 2.2  

---

#### 2.2 CAPTCHA Generation and Delivery

**Overview:**  
Generates a math CAPTCHA question for the new member and sends it via Telegram, then deletes the original join message to reduce clutter.

**Nodes Involved:**  
- 🎲 Generate CAPTCHA Question  
- ❓ Send CAPTCHA Question  
- 🗑️ Delete Join Message

**Node Details:**

- **🎲 Generate CAPTCHA Question**  
  - *Type:* Code node (custom JavaScript)  
  - *Configuration:* Generates a random math question (e.g., addition, subtraction) and stores the question and answer in workflow data.  
  - *Input:* True branch from Check New Member  
  - *Output:* Passes the question for sending via Telegram  
  - *Edge cases:* Code errors in question generation or format could break the flow.  
  - *Version:* 2  

- **❓ Send CAPTCHA Question**  
  - *Type:* Telegram node (sends a message)  
  - *Configuration:* Sends the generated CAPTCHA question to the new member’s chat.  
  - *Input:* From Generate CAPTCHA Question  
  - *Output:* Triggers deletion of join message.  
  - *Edge cases:* Telegram rate limits or message failures; user may not receive the CAPTCHA.  
  - *Version:* 1.2  

- **🗑️ Delete Join Message**  
  - *Type:* Telegram node (deletes message)  
  - *Configuration:* Deletes the original system message that notifies about the new member joining, keeping chat clean.  
  - *Input:* From Send CAPTCHA Question  
  - *Output:* Proceeds to store user answer.  
  - *Edge cases:* Message might already be deleted or insufficient permissions cause failure.  
  - *Version:* 1.2  

---

#### 2.3 User Response Management

**Overview:**  
Manages user answers by storing their responses in Google Sheets and retrieving any pending CAPTCHA challenges for verification.

**Nodes Involved:**  
- 💾 Store User Answer  
- 📋 Find Pending CAPTCHA

**Node Details:**

- **💾 Store User Answer**  
  - *Type:* Google Sheets node (append or update)  
  - *Configuration:* Stores user answer data in a designated Google Sheet for tracking.  
  - *Input:* From Delete Join Message (after sending CAPTCHA)  
  - *Output:* Not explicitly connected downstream in the provided workflow snippet but likely for record keeping.  
  - *Credentials:* Requires Google Sheets OAuth2 credentials.  
  - *Edge cases:* Google Sheets API limits, auth token expiry, or sheet permission errors.  
  - *Version:* 4.7  

- **📋 Find Pending CAPTCHA**  
  - *Type:* Google Sheets node (read/search)  
  - *Configuration:* Looks up if there is a pending CAPTCHA answer from a user to verify.  
  - *Input:* False branch from Check New Member (i.e., not a new join event but possibly an answer message)  
  - *Output:* Leads to deleting CAPTCHA question message on success.  
  - *Edge cases:* Missed or incorrect data lookup can cause verification to fail or delay.  
  - *Version:* 4.7  

---

#### 2.4 CAPTCHA Verification

**Overview:**  
Verifies the user’s answer against the stored CAPTCHA answer and routes the flow accordingly—success leads to welcoming the user, failure leads to banning.

**Nodes Involved:**  
- ✅ Verify Answer  
- 🎉 Welcome New Member  
- Create URL for banrequest  
- 🚫 Ban User (Failed CAPTCHA)

**Node Details:**

- **✅ Verify Answer**  
  - *Type:* If node (conditional branch)  
  - *Configuration:* Compares the user’s provided answer with the correct CAPTCHA answer.  
  - *Input:* From Delete CAPTCHA Question (Success) node  
  - *Output:* True branch to Welcome New Member; False branch to create ban URL for request.  
  - *Edge cases:* Expression errors or data format mismatches can cause false negatives.  
  - *Version:* 2.2  

- **🎉 Welcome New Member**  
  - *Type:* Telegram node (send message)  
  - *Configuration:* Sends a welcome message to the user upon successful CAPTCHA verification.  
  - *Input:* True branch from Verify Answer  
  - *Output:* Proceeds to delete user message.  
  - *Edge cases:* Message send failure or rate limiting.  
  - *Version:* 1.2  

- **Create URL for banrequest**  
  - *Type:* Set node (builds URL)  
  - *Configuration:* Constructs the URL for banning the user based on Telegram API parameters.  
  - *Input:* False branch from Verify Answer  
  - *Output:* To Ban User node.  
  - *Edge cases:* Incorrect URL formatting or missing parameters leads to failed ban.  
  - *Version:* 3.4  

- **🚫 Ban User (Failed CAPTCHA)**  
  - *Type:* HTTP Request node (Telegram API call)  
  - *Configuration:* Sends a ban request to Telegram using constructed URL, banning the user who failed CAPTCHA.  
  - *Input:* From Create URL for banrequest  
  - *Output:* Deletes user message after banning.  
  - *Edge cases:* API errors, insufficient permissions, or network timeouts.  
  - *Version:* 4.2  

---

#### 2.5 Member Management and Cleanup

**Overview:**  
Deletes messages related to the CAPTCHA process and cleans stored user data after successful verification or banning.

**Nodes Involved:**  
- 🗑️ Delete User Message  
- 🗑️ Clean User Data (Success)  
- 🗑️ Delete CAPTCHA Question (Success)

**Node Details:**

- **🗑️ Delete User Message**  
  - *Type:* Telegram node (delete message)  
  - *Configuration:* Deletes the user’s answer message to keep the chat clean.  
  - *Input:* From Welcome New Member or Ban User node  
  - *Output:* Leads to clean user data after success path.  
  - *Edge cases:* Permission issues or message already deleted.  
  - *Version:* 1.2  

- **🗑️ Clean User Data (Success)**  
  - *Type:* Google Sheets node (delete or clear row)  
  - *Configuration:* Removes the user’s CAPTCHA data from the Google Sheet after successful verification.  
  - *Input:* From Delete User Message  
  - *Output:* End of successful path cleanup.  
  - *Edge cases:* Sheet access errors or race condition if multiple writes occur.  
  - *Version:* 4.7  

- **🗑️ Delete CAPTCHA Question (Success)**  
  - *Type:* Telegram node (delete message)  
  - *Configuration:* Deletes the CAPTCHA question message after user answered correctly.  
  - *Input:* From Find Pending CAPTCHA  
  - *Output:* Leads to Verify Answer.  
  - *Edge cases:* Message may not exist or deletion permission missing.  
  - *Version:* 1.2  

---

#### 2.6 Configuration and Guides

**Overview:**  
Includes several sticky notes that document the workflow steps, configuration instructions, and usage guides.

**Nodes Involved:**  
- 📋 Template Description  
- 📝 Step 1 Guide  
- 📝 Step 2 Guide  
- 📝 Step 3 Guide  
- 📝 Step 4 Guide  
- ✅ Success Path Guide  
- 🚫 Failure Path Guide  
- 🧹 Cleanup Guide  
- ⚙️ Config Guide  
- 📊 Sheets Guide

**Node Details:**

- **Sticky Notes**  
  - *Type:* Sticky Note node  
  - *Configuration:* Contain textual instructions and explanations for the workflow usage and setup.  
  - *Input/Output:* None (documentation only)  
  - *Edge cases:* None  

---

### 3. Summary Table

| Node Name                      | Node Type           | Functional Role                     | Input Node(s)                     | Output Node(s)                                          | Sticky Note                           |
|-------------------------------|---------------------|-----------------------------------|----------------------------------|---------------------------------------------------------|-------------------------------------|
| 📱 Telegram Trigger            | Telegram Trigger    | Listens to Telegram events         | External Telegram updates         | ⚙️ Bot Configuration                                    |                                     |
| ⚙️ Bot Configuration           | Set                 | Sets bot parameters and variables  | 📱 Telegram Trigger              | 🔍 Check New Member                                     | ⚙️ Config Guide                     |
| 🔍 Check New Member            | If                  | Checks if event is new member join | ⚙️ Bot Configuration             | 🎲 Generate CAPTCHA Question (true), 📋 Find Pending CAPTCHA (false) | 📝 Step 1 Guide                      |
| 🎲 Generate CAPTCHA Question   | Code                 | Generates math CAPTCHA question    | 🔍 Check New Member (true)        | ❓ Send CAPTCHA Question                                | 📝 Step 2 Guide                    |
| ❓ Send CAPTCHA Question        | Telegram             | Sends CAPTCHA question to user     | 🎲 Generate CAPTCHA Question      | 🗑️ Delete Join Message                                  | 📝 Step 3 Guide                    |
| 🗑️ Delete Join Message         | Telegram             | Deletes join message               | ❓ Send CAPTCHA Question          | 💾 Store User Answer                                   | 🧹 Cleanup Guide                   |
| 💾 Store User Answer           | Google Sheets        | Stores user answer in spreadsheet  | 🗑️ Delete Join Message           | —                                                       | 📊 Sheets Guide                   |
| 📋 Find Pending CAPTCHA        | Google Sheets        | Finds pending CAPTCHA to verify   | 🔍 Check New Member (false)       | 🗑️ Delete CAPTCHA Question (Success)                   | 📊 Sheets Guide                   |
| 🗑️ Delete CAPTCHA Question     | Telegram             | Deletes CAPTCHA question message   | 📋 Find Pending CAPTCHA           | ✅ Verify Answer                                        | 🧹 Cleanup Guide                   |
| ✅ Verify Answer               | If                   | Compares user answer with correct  | 🗑️ Delete CAPTCHA Question       | 🎉 Welcome New Member (true), Create URL for banrequest (false) | ✅ Success Path Guide, 🚫 Failure Path Guide |
| 🎉 Welcome New Member          | Telegram             | Sends welcome message              | ✅ Verify Answer (true)            | 🗑️ Delete User Message                                 | ✅ Success Path Guide             |
| Create URL for banrequest      | Set                  | Builds ban request URL             | ✅ Verify Answer (false)           | 🚫 Ban User (Failed CAPTCHA)                            | 🚫 Failure Path Guide             |
| 🚫 Ban User (Failed CAPTCHA)   | HTTP Request         | Bans user via Telegram API         | Create URL for banrequest          | 🗑️ Delete User Message                                 | 🚫 Failure Path Guide             |
| 🗑️ Delete User Message         | Telegram             | Deletes user’s message             | 🎉 Welcome New Member, 🚫 Ban User | 🗑️ Clean User Data (Success)                           | 🧹 Cleanup Guide                   |
| 🗑️ Clean User Data (Success)  | Google Sheets        | Cleans user data after success     | 🗑️ Delete User Message            | —                                                       | 📊 Sheets Guide                   |
| 📋 Template Description        | Sticky Note          | Workflow description and overview  | None                            | None                                                    |                                     |
| 📝 Step 1 Guide                | Sticky Note          | Documentation step 1               | None                            | None                                                    |                                     |
| 📝 Step 2 Guide                | Sticky Note          | Documentation step 2               | None                            | None                                                    |                                     |
| 📝 Step 3 Guide                | Sticky Note          | Documentation step 3               | None                            | None                                                    |                                     |
| 📝 Step 4 Guide                | Sticky Note          | Documentation step 4               | None                            | None                                                    |                                     |
| ✅ Success Path Guide          | Sticky Note          | Documentation success path         | None                            | None                                                    |                                     |
| 🚫 Failure Path Guide          | Sticky Note          | Documentation failure path         | None                            | None                                                    |                                     |
| 🧹 Cleanup Guide               | Sticky Note          | Documentation cleanup steps        | None                            | None                                                    |                                     |
| ⚙️ Config Guide               | Sticky Note          | Bot configuration instructions     | None                            | None                                                    |                                     |
| 📊 Sheets Guide               | Sticky Note          | Google Sheets integration guide    | None                            | None                                                    |                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Node Type: Telegram Trigger  
   - Configure with your Telegram bot credentials (bot token).  
   - Set to listen for all messages and updates.  

2. **Create Set Node for Bot Configuration**  
   - Node Type: Set  
   - Define variables such as bot token, group chat ID, CAPTCHA difficulty.  
   - Connect Telegram Trigger node output to this node.  

3. **Create If Node to Check for New Member**  
   - Node Type: If  
   - Condition: Check if the Telegram update payload indicates a new chat member (e.g., `message.new_chat_members` exists).  
   - Connect Bot Configuration output to this node.  
   - True branch for new members; False branch for other messages.  

4. **Create Code Node to Generate CAPTCHA Question**  
   - Node Type: Code  
   - JavaScript to generate a simple math question (e.g., random addition/subtraction) and store question & correct answer in output.  
   - Connect True branch of Check New Member node here.  

5. **Create Telegram Node to Send CAPTCHA Question**  
   - Node Type: Telegram  
   - Configure to send message to the new member’s chat with the generated CAPTCHA question.  
   - Connect from Code node.  

6. **Create Telegram Node to Delete Join Message**  
   - Node Type: Telegram  
   - Configure to delete the original join notification message in the group chat.  
   - Connect from Send CAPTCHA Question node.  

7. **Create Google Sheets Node to Store User Answer**  
   - Node Type: Google Sheets (Append)  
   - Configure with Google Sheets OAuth2 credentials.  
   - Set the target spreadsheet and worksheet to store user ID, CAPTCHA question, correct answer, timestamp, etc.  
   - Connect from Delete Join Message node.  

8. **Create Google Sheets Node to Find Pending CAPTCHA**  
   - Node Type: Google Sheets (Lookup)  
   - Configure to search for pending CAPTCHA entries matching the user ID and message.  
   - Connect False branch of Check New Member node here.  

9. **Create Telegram Node to Delete CAPTCHA Question (on success)**  
   - Node Type: Telegram  
   - Configure to delete the CAPTCHA question message after user answers correctly.  
   - Connect from Find Pending CAPTCHA node.  

10. **Create If Node to Verify User Answer**  
    - Node Type: If  
    - Condition: Compare stored correct answer with user’s submitted answer from the message.  
    - Connect from Delete CAPTCHA Question node.  
    - True branch: correct answer; False branch: incorrect answer.  

11. **Create Telegram Node to Welcome New Member**  
    - Node Type: Telegram  
    - Configure message welcoming the user to the group.  
    - Connect from True branch of Verify Answer node.  

12. **Create Telegram Node to Delete User Message**  
    - Node Type: Telegram  
    - Configure to delete the user’s CAPTCHA answer message to keep chat clean.  
    - Connect from Welcome New Member node and also from Ban User node (see next step).  

13. **Create Set Node to Create Ban Request URL**  
    - Node Type: Set  
    - Construct URL for Telegram API to ban the user, including necessary parameters (chat ID, user ID).  
    - Connect from False branch of Verify Answer node.  

14. **Create HTTP Request Node to Ban User**  
    - Node Type: HTTP Request  
    - Configure to call Telegram’s ban chat member API using the constructed URL.  
    - Use bot token for auth header.  
    - Connect from Create URL for banrequest node.  

15. **Create Google Sheets Node to Clean User Data (Success)**  
    - Node Type: Google Sheets (Delete)  
    - Configure to remove the user’s CAPTCHA record from the spreadsheet after success.  
    - Connect from Delete User Message node (success path).  

16. **Add Sticky Notes**  
    - Add documentation notes at relevant positions for guidance on each step, configuration, and cleanup.  

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                              |
|-------------------------------------------------------------------------------------------------|----------------------------------------------|
| This workflow uses Google Sheets for persistent storage of CAPTCHA challenges and user answers. | Requires Google Sheets OAuth2 credential setup. |
| Telegram Bot Token must have admin rights to delete messages and ban users.                      | Telegram Bot API documentation.               |
| CAPTCHA questions are simple math problems to ensure human verification.                         | Customizable in the code node.                 |
| Rate limits on Telegram and Google Sheets APIs may affect workflow execution timing.            | See Telegram and Google API rate limit docs.  |
| Workflow includes detailed sticky notes for step-by-step guidance and configuration tips.       | Found within the workflow editor.              |

---

*Disclaimer:* The text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.