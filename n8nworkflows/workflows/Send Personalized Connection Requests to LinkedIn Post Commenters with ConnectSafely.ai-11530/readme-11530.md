Send Personalized Connection Requests to LinkedIn Post Commenters with ConnectSafely.ai

https://n8nworkflows.xyz/workflows/send-personalized-connection-requests-to-linkedin-post-commenters-with-connectsafely-ai-11530


# Send Personalized Connection Requests to LinkedIn Post Commenters with ConnectSafely.ai

### 1. Workflow Overview

This workflow automates sending personalized LinkedIn connection requests to people who comment on a specified LinkedIn post. It targets content creators, founders, sales professionals, and network builders who want to convert post engagement into meaningful professional connections while protecting their LinkedIn account from spam or rate-limit penalties.

The workflow is logically structured into the following blocks:

- **1.1 Input Reception & Data Collection**: Captures the LinkedIn post URL via a form and fetches all commenters on that post.
- **1.2 Commenters Splitting & Looping**: Splits the commenters list and processes them individually in batches.
- **1.3 Relationship Checking & Smart Filtering**: Checks if the user is already connected or has pending invitations with each commenter and filters accordingly.
- **1.4 Personalized Message Generation**: Creates unique, spin-text-based personalized messages for each eligible commenter.
- **1.5 Connection Request Sending & Rate Limiting**: Sends connection requests with generated messages, enforcing 1-2 hour delays between requests to avoid account suspension.
- **1.6 Logging & Reporting**: Logs both sent and skipped requests and produces a final summary report.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Data Collection

- **Overview:**  
  Collects the LinkedIn post URL from a user-submitted form and retrieves all comments on that post via the ConnectSafely.ai LinkedIn API.

- **Nodes Involved:**  
  - Form Trigger  
  - ConnectSafely LinkedIn (Get All Post Comments)  
  - Split Commenters

- **Node Details:**  

  - **Form Trigger**  
    - *Type & Role:* Form trigger node, entry point for workflow execution via a web form.  
    - *Configuration:* Expects a single required field "LinkedIn Post URL". The form title and description guide the user to input the post URL.  
    - *Key Expressions:* None beyond capturing input.  
    - *Connections:* Outputs to "ConnectSafely LinkedIn" node.  
    - *Edge Cases:* Invalid or malformed URLs can cause failures downstream. Missing input is prevented by form validation.  
    - *Sub-workflow:* None.

  - **ConnectSafely LinkedIn (Get All Post Comments)**  
    - *Type & Role:* ConnectSafely API node to fetch all comments on the provided LinkedIn post URL.  
    - *Configuration:* Uses the post URL from the form input; accountId is statically set; operation is "getAllPostComments".  
    - *Key Expressions:* `"postUrl": "={{ $json['LinkedIn Post URL'] }}"`.  
    - *Connections:* Output to "Split Commenters".  
    - *Edge Cases:* API errors, rate limits, or invalid post URLs may cause failures here.  
    - *Sub-workflow:* None.

  - **Split Commenters**  
    - *Type & Role:* Splits the array of comments into individual items for processing.  
    - *Configuration:* Splits on the "comments" field from the previous node's output.  
    - *Connections:* Outputs to "Loop Over Items".  
    - *Edge Cases:* Empty comments array leads to no further processing.  
    - *Sub-workflow:* None.

---

#### 2.2 Commenters Splitting & Looping

- **Overview:**  
  Iterates over each commenter individually in batches to handle subsequent checking and messaging steps efficiently.

- **Nodes Involved:**  
  - Loop Over Items

- **Node Details:**  

  - **Loop Over Items**  
    - *Type & Role:* Processes items in batches, allowing controlled iteration over all commenters.  
    - *Configuration:* Defaults to processing all items; batch size can be adjusted for rate-limiting or resource management.  
    - *Connections:* Two outputs: one to "Should Send?" and one to "ConnectSafely LinkedIn1" (relationship checking).  
    - *Edge Cases:* Large batch sizes may hit API or memory limits; empty inputs bypass further steps.  
    - *Sub-workflow:* None.

---

#### 2.3 Relationship Checking & Smart Filtering

- **Overview:**  
  Checks if the user is already connected or has a pending invitation with each commenter to prevent duplicate or spammy connection requests.

- **Nodes Involved:**  
  - ConnectSafely LinkedIn1 (Check Relationship)  
  - Should Send?  
  - Log Skipped

- **Node Details:**  

  - **ConnectSafely LinkedIn1 (Check Relationship)**  
    - *Type & Role:* ConnectSafely API node to check connection status with each commenter.  
    - *Configuration:* Uses the commenter's public LinkedIn identifier (`profileId`) dynamically from current item JSON.  
    - *Key Expressions:* `"profileId": "={{ $json.publicIdentifier }}"`.  
    - *Connections:* Output to "Should Send?".  
    - *Edge Cases:* API failures or rate limiting. Invalid or missing profile IDs can cause errors.  
    - *Sub-workflow:* None.

  - **Should Send?**  
    - *Type & Role:* Conditional node to decide if a connection request should be sent.  
    - *Configuration:* Checks two boolean conditions:  
      - `connected` is false (not yet connected)  
      - `invitationSent` is false (no pending invitation)  
    - *Connections:*  
      - True branch to "Generate Message" (send request)  
      - False branch to "Log Skipped" (skip request)  
    - *Edge Cases:* Missing attributes or incorrect data types can cause condition failures.  
    - *Sub-workflow:* None.

  - **Log Skipped**  
    - *Type & Role:* Code node that logs skipped commenters with reasons (already connected or pending invitation).  
    - *Configuration:* Extracts relationship data and commenter info for reporting.  
    - *Connections:* Output merges into "Merge Results".  
    - *Edge Cases:* Missing data fields or null values should be handled gracefully.  
    - *Sub-workflow:* None.

---

#### 2.4 Personalized Message Generation

- **Overview:**  
  Constructs personalized connection request messages using spin text templates to ensure uniqueness and LinkedIn message length compliance.

- **Nodes Involved:**  
  - Generate Message

- **Node Details:**  

  - **Generate Message**  
    - *Type & Role:* Code node generating a custom message per commenter with randomized spin text.  
    - *Configuration:*  
      - Spin text contains multiple interchangeable phrases for greetings, appreciation, and closing.  
      - Extracts commenter first name; defaults to "there" if absent.  
      - Limits final message length to 300 characters to comply with LinkedIn limits.  
    - *Key Expressions:* Uses `$('Loop Over Items').item.json` to access current commenter data.  
    - *Connections:* Output to "ConnectSafely LinkedIn2" (send request).  
    - *Edge Cases:* Missing or malformed commenter data can reduce personalization quality.  
    - *Sub-workflow:* None.

---

#### 2.5 Connection Request Sending & Rate Limiting

- **Overview:**  
  Sends connection requests with the generated personalized message, enforcing a wait time between sends to avoid LinkedIn account restrictions.

- **Nodes Involved:**  
  - ConnectSafely LinkedIn2 (Send Connection Request)  
  - Wait 1-2 Hours  
  - Log Success

- **Node Details:**  

  - **ConnectSafely LinkedIn2 (Send Connection Request)**  
    - *Type & Role:* ConnectSafely API node to send a LinkedIn connection request.  
    - *Configuration:*  
      - Sends to `profileId` and `profileUrn` derived from the "Should Send?" node outputs.  
      - Includes the personalized message from "Generate Message".  
    - *Key Expressions:*  
      - `"profileId": "={{ $('Should Send?').item.json.accountId }}"`  
      - `"profileUrn": "={{ $('Should Send?').item.json.profileUrn }}"`  
      - `"customMessage": "={{ $json.generatedMessage }}"`  
    - *Connections:* Output to "Wait 1-2 Hours".  
    - *Edge Cases:* API failures, invalid profile IDs, or message length issues may cause failure.  
    - *Sub-workflow:* None.

  - **Wait 1-2 Hours**  
    - *Type & Role:* Wait node to pause workflow execution to comply with LinkedIn rate limits.  
    - *Configuration:* Waits for 1 hour by default (adjustable depending on account age).  
    - *Connections:* Output to "Log Success".  
    - *Edge Cases:* Long wait times may cause workflow timeouts depending on n8n setup.  
    - *Sub-workflow:* None.

  - **Log Success**  
    - *Type & Role:* Code node to log successful connection requests with relevant data.  
    - *Configuration:* Captures commenter name, profile ID, sent message, timestamp, and original comment text.  
    - *Connections:* Output merges into "Merge Results".  
    - *Edge Cases:* Missing data fields should be handled gracefully.  
    - *Sub-workflow:* None.

---

#### 2.6 Logging & Reporting

- **Overview:**  
  Combines logs from skipped and successful requests and generates a comprehensive summary report.

- **Nodes Involved:**  
  - Merge Results  
  - Generate Summary

- **Node Details:**  

  - **Merge Results**  
    - *Type & Role:* Combines outputs from both "Log Success" and "Log Skipped" nodes into a single data stream.  
    - *Configuration:* Merge mode: combine all items.  
    - *Connections:* Output to "Generate Summary".  
    - *Edge Cases:* Unequal or empty inputs could affect report completeness.  
    - *Sub-workflow:* None.

  - **Generate Summary**  
    - *Type & Role:* Code node that creates a detailed summary of total commenters processed, requests sent, skips, reasons for skips, and success rate.  
    - *Configuration:* Iterates over merged inputs, counts successes and skips by reason, and formats a summary JSON object.  
    - *Connections:* None (end of workflow).  
    - *Edge Cases:* No commenters or only skipped may lead to zero success rate.  
    - *Sub-workflow:* None.

---

### 3. Summary Table

| Node Name                | Node Type                       | Functional Role                            | Input Node(s)                | Output Node(s)                  | Sticky Note                                                                                                                                                                                                                                             |
|--------------------------|--------------------------------|--------------------------------------------|-----------------------------|-------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Form Trigger             | n8n-nodes-base.formTrigger      | Entry point: captures LinkedIn post URL    | -                           | ConnectSafely LinkedIn          | ## Data Collection Capture the post URL and fetch all commenters from that LinkedIn post.                                                                                                                                                                |
| ConnectSafely LinkedIn    | n8n-nodes-connectsafely-ai.connectSafelyLinkedIn | Fetch all comments on LinkedIn post        | Form Trigger                | Split Commenters               | ## Data Collection Capture the post URL and fetch all commenters from that LinkedIn post.                                                                                                                                                                |
| Split Commenters          | n8n-nodes-base.splitOut         | Splits commenters array into individual items | ConnectSafely LinkedIn       | Loop Over Items                | ## Process Loop Handle each commenter one by one. The loop continues until all commenters are processed, then moves to final reporting.                                                                                                                 |
| Loop Over Items           | n8n-nodes-base.splitInBatches  | Loops over commenters in batches            | Split Commenters             | Should Send?, ConnectSafely LinkedIn1 | ## Process Loop Handle each commenter one by one. The loop continues until all commenters are processed, then moves to final reporting.                                                                                                                 |
| ConnectSafely LinkedIn1   | n8n-nodes-connectsafely-ai.connectSafelyLinkedIn | Checks relationship status with commenter  | Loop Over Items              | Should Send?                  | ## Smart Filtering Check relationship status and skip anyone you're already connected with or have a pending invitation to.                                                                                                                              |
| Should Send?              | n8n-nodes-base.if              | Conditional: decide whether to send request | Loop Over Items, ConnectSafely LinkedIn1 | Generate Message, Log Skipped | ## Smart Filtering Check relationship status and skip anyone you're already connected with or have a pending invitation to. <br> ## Tracking Log skipped connections (already connected or pending) to avoid duplicate requests.                       |
| Generate Message          | n8n-nodes-base.code            | Creates personalized LinkedIn connection message | Should Send?                | ConnectSafely LinkedIn2        | ## Send Requests Generate personalized messages and send connection requests with rate limiting to protect your account.                                                                                                                                  |
| ConnectSafely LinkedIn2   | n8n-nodes-connectsafely-ai.connectSafelyLinkedIn | Sends LinkedIn connection request           | Generate Message             | Wait 1-2 Hours                | ## Send Requests Generate personalized messages and send connection requests with rate limiting to protect your account.                                                                                                                                  |
| Wait 1-2 Hours            | n8n-nodes-base.wait            | Pauses workflow to respect LinkedIn rate limits | ConnectSafely LinkedIn2      | Log Success                   | ## Send Requests Generate personalized messages and send connection requests with rate limiting to protect your account.                                                                                                                                  |
| Log Success               | n8n-nodes-base.code            | Logs successful connection requests         | Wait 1-2 Hours              | Merge Results                 | ## Tracking Log skipped connections (already connected or pending) to avoid duplicate requests. <br> ## Final Report Combine all results and generate a summary showing total commenters, requests sent, and skip reasons.                                  |
| Log Skipped               | n8n-nodes-base.code            | Logs skipped connection requests with reasons | Should Send? (false branch) | Merge Results                 | ## Tracking Log skipped connections (already connected or pending) to avoid duplicate requests.                                                                                                                                                           |
| Merge Results             | n8n-nodes-base.merge           | Combines logs from successes and skips      | Log Success, Log Skipped     | Generate Summary              | ## Final Report Combine all results and generate a summary showing total commenters, requests sent, and skip reasons.                                                                                                                                      |
| Generate Summary          | n8n-nodes-base.code            | Final report generation                      | Merge Results               | -                             | ## Final Report Combine all results and generate a summary showing total commenters, requests sent, and skip reasons.                                                                                                                                      |
| Sticky Note               | n8n-nodes-base.stickyNote      | Documentation and explanation                | -                           | -                             | ## Auto-Connect with LinkedIn Post Commenters ... [Detailed explanation present in notes above]                                                                                                                                                          |
| Sticky Note1              | n8n-nodes-base.stickyNote      | Data collection explanation                   | -                           | -                             | ## Data Collection Capture the post URL and fetch all commenters from that LinkedIn post.                                                                                                                                                                |
| Sticky Note2              | n8n-nodes-base.stickyNote      | Process loop explanation                      | -                           | -                             | ## Process Loop Handle each commenter one by one. The loop continues until all commenters are processed, then moves to final reporting.                                                                                                                 |
| Sticky Note3              | n8n-nodes-base.stickyNote      | Smart filtering explanation                    | -                           | -                             | ## Smart Filtering Check relationship status and skip anyone you're already connected with or have a pending invitation to.                                                                                                                              |
| Sticky Note4              | n8n-nodes-base.stickyNote      | Send requests explanation                      | -                           | -                             | ## Send Requests Generate personalized messages and send connection requests with rate limiting to protect your account.                                                                                                                                  |
| Sticky Note5              | n8n-nodes-base.stickyNote      | Tracking explanation                           | -                           | -                             | ## Tracking Log skipped connections (already connected or pending) to avoid duplicate requests.                                                                                                                                                           |
| Sticky Note6              | n8n-nodes-base.stickyNote      | Final report explanation                       | -                           | -                             | ## Final Report Combine all results and generate a summary showing total commenters, requests sent, and skip reasons.                                                                                                                                      |
| Sticky Note7              | n8n-nodes-base.stickyNote      | (Empty)                                       | -                           | -                             |                                                                                                                                                                                                                                                         |
| Sticky Note8              | n8n-nodes-base.stickyNote      | (Empty)                                       | -                           | -                             |                                                                                                                                                                                                                                                         |
| Sticky Note9              | n8n-nodes-base.stickyNote      | (Empty)                                       | -                           | -                             |                                                                                                                                                                                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Add a Form Trigger Node**  
   - Set path to `auto-connect-commenters`.  
   - Configure form fields: one required field labeled "LinkedIn Post URL".  
   - Form title: "🔗 LinkedIn Post Engagement Automation".  
   - Form description: "Enter your LinkedIn post URL below to automatically send personalized connection requests to commenters".

2. **Add ConnectSafely LinkedIn Node (Get All Post Comments)**  
   - Credential: ConnectSafely.AI API credentials (must be configured first).  
   - Operation: `getAllPostComments`.  
   - Parameter: `postUrl` set dynamically from the form input: `={{ $json['LinkedIn Post URL'] }}`.  
   - Connect Form Trigger output to this node.

3. **Add Split Out Node**  
   - Field to split out: `comments` (the array of comments from previous node).  
   - Connect output of ConnectSafely LinkedIn (getAllPostComments) to this node.

4. **Add Split In Batches Node**  
   - Default batch size or configure as needed (e.g., 1 for strict sequential processing).  
   - Connect output of Split Commenters node to this node.

5. **Add ConnectSafely LinkedIn Node (Check Relationship)**  
   - Operation: `checkRelationship`.  
   - Parameter: `profileId` set dynamically as `={{ $json.publicIdentifier }}` from current commenter.  
   - Connect one output of Loop Over Items node to this node.

6. **Add If Node (Should Send?)**  
   - Conditions (AND):  
     - `connected` equals `false`  
     - `invitationSent` equals `false`  
   - Connect outputs of Loop Over Items and ConnectSafely LinkedIn1 (check relationship) so that this node evaluates each commenter with relationship data.

7. **Add Code Node (Generate Message)**  
   - Insert JavaScript code to generate a spin-text personalized message:  
     - Use a template with interchangeable phrases.  
     - Extract commenter's first name or default to "there".  
     - Enforce LinkedIn's 300 char message limit.  
   - Connect True output of Should Send? node to this node.

8. **Add ConnectSafely LinkedIn Node (Send Connection Request)**  
   - Operation: `sendConnectionRequest`.  
   - Parameters:  
     - `profileId`: from `$('Should Send?').item.json.accountId`  
     - `profileUrn`: from `$('Should Send?').item.json.profileUrn`  
     - `customMessage`: from generated message JSON.  
   - Connect output of Generate Message to this node.

9. **Add Wait Node**  
   - Unit: Hours  
   - Amount: 1 (adjustable for account age; 1-2 hours recommended)  
   - Connect output of ConnectSafely LinkedIn2 (send connection request) to Wait node.

10. **Add Code Node (Log Success)**  
    - Capture and log: commenter name, profile ID, message sent, timestamp, and comment text.  
    - Connect output of Wait node to this node.

11. **Add Code Node (Log Skipped)**  
    - Capture skipped commenters with reasons ("Already connected" or "Pending invitation").  
    - Connect False output of Should Send? node to this node.

12. **Add Merge Node**  
    - Mode: Combine  
    - Connect outputs of Log Success and Log Skipped nodes to this Merge node.

13. **Add Code Node (Generate Summary)**  
    - Aggregate all processed commenters: total, sent, skipped, reasons for skip, success rate.  
    - Connect output of Merge Results node to this node.

14. **Credentials Setup:**  
    - Configure ConnectSafely.AI API credentials in n8n for all ConnectSafely LinkedIn nodes.

15. **Optional:**  
    - Add sticky notes in the workflow for documentation as per the original.  
    - Adjust batch size or wait times to match account safety requirements.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                           | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Auto-Connect with LinkedIn Post Commenters: Automatically sends personalized connection requests to people who comment on your LinkedIn posts. Designed for authentic engagement growth with built-in rate limiting and smart filtering.                                                                                                                                                                  | See Sticky Note content in the workflow for detailed explanation.                               |
| Important: Start with small tests (5-10 commenters) and never exceed 30-40 requests per day to prevent LinkedIn account restrictions.                                                                                                                                                                                                                                                                | Workflow usage best practice.                                                                    |
| Setup required: Install the community node `n8n-nodes-connectsafely-ai`, add your ConnectSafely.AI API credentials, customize message templates with your name, and adjust wait times for your account age (new accounts require longer delays).                                                                                                                                                        | Setup instructions from sticky notes.                                                          |
| Spin text method creates varied messages to avoid repetitive connection requests, increasing acceptance likelihood and reducing spam detection.                                                                                                                                                                                                                                                       | Refer to code in "Generate Message" node.                                                       |
| ConnectSafely.AI API handles LinkedIn interactions securely; ensure credentials have sufficient permissions and are valid. Monitor API usage to avoid hitting limits.                                                                                                                                                                                                                                   | ConnectSafely.AI platform documentation.                                                       |
| Video overview and detailed instructions are recommended for users new to n8n or ConnectSafely.AI; check official ConnectSafely.AI or n8n community resources for tutorials and best practices.                                                                                                                                                                                                       | External resources (not included in workflow).                                                 |

---

**Disclaimer:**  
The provided content is derived exclusively from an automated workflow created using n8n, an integration and automation tool. This process strictly complies with prevailing content policies and contains no illegal, offensive, or protected material. All data handled are legal and public.