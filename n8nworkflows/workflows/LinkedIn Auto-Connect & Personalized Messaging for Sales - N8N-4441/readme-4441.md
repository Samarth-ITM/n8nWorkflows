LinkedIn Auto-Connect & Personalized Messaging for Sales - N8N

https://n8nworkflows.xyz/workflows/linkedin-auto-connect---personalized-messaging-for-sales---n8n-4441


# LinkedIn Auto-Connect & Personalized Messaging for Sales - N8N

### 1. Workflow Overview

This workflow automates LinkedIn sales outreach by managing connections and sending personalized messages using AI. It is targeted at sales professionals or marketers who want to scale their LinkedIn networking efficiently while maintaining personalization.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Initial Profile Scraping:** Receives user input for keywords, scrapes LinkedIn profiles accordingly.
- **1.2 Profile Management and Connection Status Checking:** Processes scraped profiles, checks connection status, and updates Google Sheets databases.
- **1.3 Connection Requests and Messaging:** Sends LinkedIn connection invitations and personalized messages using AI-generated content.
- **1.4 AI-Powered Message Generation:** Utilizes OpenAI models and a LangChain AI agent to craft personalized sales messages.
- **1.5 Scheduling and Batching:** Manages execution timing, batching of profiles, and wait periods to comply with limits and avoid spam.
- **1.6 Error Handling and Fallbacks:** Contains nodes for no-operation or fallback browser automation flows in case of failures.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception and Initial Profile Scraping

- **Overview:** This block initiates the workflow by receiving input keywords via a form, then scrapes LinkedIn profiles matching those keywords.
- **Nodes Involved:**  
  - Fill Out Keywords (Form Trigger)  
  - Profile Scraper (Browserflow)  
  - Split Out (Split Out)  
  - Add Profiles (Google Sheets)

- **Node Details:**

  - **Fill Out Keywords**  
    - Type: formTrigger  
    - Role: Receives keyword input from the user via a web form webhook.  
    - Configuration: Uses a webhook URL to await form submission.  
    - Inputs: External user input.  
    - Outputs: Triggers the Profile Scraper.  
    - Edge cases: Missing or malformed input can cause scraping to be ineffective.

  - **Profile Scraper**  
    - Type: browserflow  
    - Role: Scrapes LinkedIn profiles based on provided keywords.  
    - Configuration: Custom browser automation flow to extract profile data.  
    - Inputs: Keywords from Fill Out Keywords.  
    - Outputs: Data passed to Split Out for further processing.  
    - Edge cases: LinkedIn page layout changes, rate limits, or captchas could disrupt scraping.

  - **Split Out**  
    - Type: splitOut  
    - Role: Splits bulk scraped profile data into individual items for processing.  
    - Inputs: Profile Scraper output.  
    - Outputs: Sends individual profile data to Add Profiles node.  
    - Edge cases: Empty or malformed data arrays.

  - **Add Profiles**  
    - Type: googleSheets  
    - Role: Adds scraped profiles to a Google Sheets document for storage and tracking.  
    - Inputs: Individual profile items from Split Out.  
    - Outputs: Stored data, no further output.  
    - Edge cases: Google Sheets API quota or permission issues.

---

#### 1.2 Profile Management and Connection Status Checking

- **Overview:** This block checks if scraped profiles are already LinkedIn connections and updates status records accordingly, managing batches and limits.
- **Nodes Involved:**  
  - Schedule Trigger  
  - Get Profiles (Google Sheets)  
  - Check Connection (Browserflow)  
  - If connection (IF Node)  
  - Stat Update (Google Sheets)  
  - Wait (Wait)  
  - Loop Over Items (Split In Batches)  
  - Connection Checker (Browserflow)  
  - Run when profile is update (Google Sheets Trigger)  
  - Check if a person is a connection. (IF Node)  
  - Loop Over Items2 (Split In Batches)  
  - add browser flow if it gives error (NoOp)  
  - Update Invite (Google Sheets)  
  - Set Connection Limit (Set)  
  - Wait1 (Wait)

- **Node Details:**

  - **Schedule Trigger**  
    - Type: scheduleTrigger  
    - Role: Periodically triggers the workflow to refresh connection checks.  
    - Configuration: Runs at configured intervals.  
    - Outputs: Starts with Get Profiles.

  - **Get Profiles**  
    - Type: googleSheets  
    - Role: Retrieves profiles from Google Sheets for processing.  
    - Inputs: Triggered by Schedule Trigger.  
    - Outputs: Feeds into Check Connection.

  - **Check Connection**  
    - Type: browserflow  
    - Role: Uses browser automation to verify if profiles are already connected on LinkedIn.  
    - Inputs: Profiles from Get Profiles.  
    - Outputs: Passes data to If connection1.

  - **If connection1**  
    - Type: IF  
    - Role: Determines next steps based on connection status.  
    - Inputs: Check Connection output.  
    - Outputs: If connected, triggers Wait2; otherwise, flows to other branches.

  - **Wait2**  
    - Type: wait  
    - Role: Introduces a delay to comply with rate limits or pacing.  
    - Inputs: If connection1 true branch.  
    - Outputs: Triggers Scrap New Connection Profile.

  - **Scrap New Connection Profile**  
    - Type: browserflow  
    - Role: Scrapes additional profile details for new connections.  
    - Inputs: Wait2 output.  
    - Outputs: Sends data to Organise Infomation.

  - **Organise Infomation**  
    - Type: set  
    - Role: Prepares and formats profile data for AI message generation.  
    - Inputs: Scrap New Connection Profile output.  
    - Outputs: Feeds into AI Agent.

  - **AI Agent**  
    - Type: langchain.agent  
    - Role: Orchestrates AI-based message generation.  
    - Inputs: Profile data from Organise Infomation.  
    - Outputs: Sends personalized messages to LinkedIn.

  - **Run when profile is update**  
    - Type: googleSheetsTrigger  
    - Role: Watches for profile updates in Google Sheets to re-check connection status.  
    - Outputs: Triggers Check if a person is a connection.

  - **Check if a person is a connection.**  
    - Type: IF  
    - Role: Decides if the person is connected.  
    - Outputs: True branch triggers Loop Over Items.

  - **Loop Over Items**  
    - Type: splitInBatches  
    - Role: Processes profiles in batches for connection checking and updates.  
    - Outputs: Feeds If connection and Connection Checker.

  - **If connection**  
    - Type: IF  
    - Role: Branches processing based on connection status.  
    - Outputs: Updates Stat Update for connected, or Wait for non-connected.

  - **Stat Update**  
    - Type: googleSheets  
    - Role: Updates statistics or logs in Google Sheets.  
    - Inputs: If connection true branch.  
    - Outputs: None.

  - **Wait**  
    - Type: wait  
    - Role: Delay for pacing non-connected profiles.  
    - Inputs: If connection false branch.  
    - Outputs: Triggers Loop Over Items1 for connection invitation.

  - **Loop Over Items1**  
    - Type: splitInBatches  
    - Role: Batching profiles for sending invites and emails.  
    - Outputs: Triggers Gmail and Invite Sender nodes.

  - **Invite Sender**  
    - Type: browserflow  
    - Role: Automates sending LinkedIn connection invitations.  
    - Inputs: Loop Over Items1.  
    - Outputs: Triggers Update Invite.

  - **Update Invite**  
    - Type: googleSheets  
    - Role: Updates invitation status in Google Sheets.  
    - Outputs: Triggers Set Connection Limit.

  - **Set Connection Limit**  
    - Type: set  
    - Role: Sets control variables or limits for connections.  
    - Outputs: Triggers Wait1.

  - **Wait1**  
    - Type: wait  
    - Role: Wait to respect limits and pacing.  
    - Outputs: Loops back to Loop Over Items1.

  - **Connection Checker**  
    - Type: browserflow  
    - Role: Checks connection status for profiles.  
    - Inputs: Loop Over Items false branch.  
    - Outputs: Feeds Loop Over Items.

  - **Loop Over Items2**  
    - Type: splitInBatches  
    - Role: Fallback batch processing.  
    - Outputs: Triggers NoOp node.

  - **add browser flow if it gives error**  
    - Type: NoOp  
    - Role: Placeholder/fallback node for error handling or alternate flows.  
    - Inputs: Loop Over Items2.

- **Edge Cases:**  
  - API rate limits or Google Sheets quota exceeded.  
  - LinkedIn UI changes breaking browser automation.  
  - Connection status misreads due to UI or network errors.  
  - Infinite loops or batch processing stuck due to data errors.

---

#### 1.3 Connection Requests and Personalized Messaging

- **Overview:** This block sends personalized messages to new connections via LinkedIn using AI-generated content.
- **Nodes Involved:**  
  - AI Agent (Langchain agent)  
  - OpenAI Chat Model  
  - Organise Infomation (Set)  
  - Send Personalised Message (Browserflow)  

- **Node Details:**

  - **Organise Infomation**  
    - Type: set  
    - Role: Formats and organizes data for AI input.  
    - Inputs: Scraped profile data.  
    - Outputs: Feeds AI Agent.

  - **AI Agent**  
    - Type: langchain.agent  
    - Role: Controls AI language model interaction for personalized message generation.  
    - Inputs: Organized profile data.  
    - Outputs: Feeds Send Personalised Message node.

  - **OpenAI Chat Model**  
    - Type: lmChatOpenAi  
    - Role: OpenAI GPT model providing conversational AI for message crafting.  
    - Inputs: Used internally by AI Agent.  
    - Outputs: AI Agent receives responses.

  - **Send Personalised Message**  
    - Type: browserflow  
    - Role: Automates sending personalized LinkedIn messages based on AI output.  
    - Inputs: AI Agent output.  
    - Outputs: None further.

- **Edge Cases:**  
  - OpenAI API quota or authentication failures.  
  - AI generating irrelevant or inappropriate content.  
  - LinkedIn message sending failures (rate limits, captchas).

---

#### 1.4 Scheduling and Batching

- **Overview:** Controls timing, batch sizes, and pacing to ensure compliance with LinkedIn limits and smooth execution.
- **Nodes Involved:**  
  - Schedule Trigger  
  - Loop Over Items, Loop Over Items1, Loop Over Items2 (all SplitInBatches)  
  - Wait, Wait1, Wait2 (Wait nodes)  
  - Set Connection Limit (Set node)

- **Node Details:**

  - **Schedule Trigger**  
    - Triggers periodic runs.

  - **SplitInBatches Nodes**  
    - Process profiles in manageable batches.

  - **Wait Nodes**  
    - Introduce delays to avoid API/rate limit issues.

  - **Set Connection Limit**  
    - Sets variables controlling batch sizes or maximum connections.

- **Edge Cases:**  
  - Batches with zero items causing no executions.  
  - Wait times too short or too long causing rate limit problems or inefficiency.

---

#### 1.5 Error Handling and Fallbacks

- **Overview:** Provides fallback paths and placeholders for error handling or alternative flows.
- **Nodes Involved:**  
  - add browser flow if it gives error (NoOp)  
  - Loop Over Items2

- **Node Details:**

  - **add browser flow if it gives error**  
    - Type: NoOp  
    - Role: Placeholder node to add browser automation flows if errors occur.  
    - Inputs: Loop Over Items2 output.

  - **Loop Over Items2**  
    - Batch processing that leads to the fallback node.

- **Edge Cases:**  
  - Triggered only when certain errors occur; no active processing by default.

---

### 3. Summary Table

| Node Name                    | Node Type                 | Functional Role                            | Input Node(s)              | Output Node(s)                            | Sticky Note             |
|-----------------------------|---------------------------|--------------------------------------------|----------------------------|--------------------------------------------|-------------------------|
| Fill Out Keywords            | formTrigger               | Receives keyword input                      | -                          | Profile Scraper                            |                         |
| Profile Scraper              | browserflow               | Scrapes LinkedIn profiles                   | Fill Out Keywords           | Split Out                                 |                         |
| Split Out                   | splitOut                  | Splits bulk data into individual items     | Profile Scraper             | Add Profiles                              |                         |
| Add Profiles                | googleSheets              | Adds profiles to Google Sheets              | Split Out                  | -                                          |                         |
| Schedule Trigger            | scheduleTrigger           | Periodic trigger for refresh                | -                          | Get Profiles                              |                         |
| Get Profiles                | googleSheets              | Retrieves profiles from Google Sheets       | Schedule Trigger           | Check Connection                          |                         |
| Check Connection            | browserflow               | Checks LinkedIn connection status           | Get Profiles               | If connection1                            |                         |
| If connection1              | IF                        | Branches by connection status                | Check Connection           | Wait2 (true), other branches (false)     |                         |
| Wait2                      | wait                      | Wait/delay for pacing                        | If connection1             | Scrap New Connection Profile              |                         |
| Scrap New Connection Profile | browserflow               | Scrapes detailed profile info                | Wait2                      | Organise Infomation                       |                         |
| Organise Infomation         | set                       | Prepares data for AI                         | Scrap New Connection Profile | AI Agent                                  |                         |
| AI Agent                   | langchain.agent            | AI message generation orchestration          | Organise Infomation        | Send Personalised Message                 |                         |
| OpenAI Chat Model          | lmChatOpenAi               | OpenAI GPT model used by AI Agent            | -                          | AI Agent (ai_languageModel input)         |                         |
| Send Personalised Message   | browserflow               | Sends personalized LinkedIn messages         | AI Agent                   | -                                          |                         |
| Run when profile is update | googleSheetsTrigger        | Detects updates in Google Sheets             | -                          | Check if a person is a connection.        |                         |
| Check if a person is a connection. | IF                 | Checks connection status                      | Run when profile is update | Loop Over Items                           |                         |
| Loop Over Items            | splitInBatches             | Processes profiles in batches                 | Check if a person is a connection. | If connection, Connection Checker       |                         |
| If connection              | IF                        | Branches based on connection                  | Loop Over Items            | Stat Update (true), Wait (false)          |                         |
| Stat Update                | googleSheets              | Updates statistics in Google Sheets           | If connection              | -                                          |                         |
| Wait                       | wait                      | Delay for pacing                              | If connection              | Loop Over Items1                          |                         |
| Loop Over Items1           | splitInBatches             | Batches for invite sending and emails         | Wait                       | Gmail, Invite Sender                      |                         |
| Gmail                      | gmail                     | Sends emails                                  | Loop Over Items1           | -                                          |                         |
| Invite Sender              | browserflow               | Sends LinkedIn connection invitations          | Loop Over Items1           | Update Invite                            |                         |
| Update Invite              | googleSheets              | Updates invite status in Google Sheets          | Invite Sender              | Set Connection Limit                     |                         |
| Set Connection Limit       | set                       | Sets limits and control variables               | Update Invite              | Wait1                                    |                         |
| Wait1                      | wait                      | Delay to respect limits                          | Set Connection Limit       | Loop Over Items1                          |                         |
| Connection Checker         | browserflow               | Checks if connection exists                      | Loop Over Items false branch | Loop Over Items                          |                         |
| Loop Over Items2           | splitInBatches             | Fallback batch processing                        | add browser flow if it gives error | -                                      |                         |
| add browser flow if it gives error | noOp                | Placeholder fallback node for error handling    | Loop Over Items2           | Loop Over Items2                          |                         |
| Split Out                  | splitOut                  | Splits data into individual profiles            | Profile Scraper            | Add Profiles                              |                         |
| Check if a person is a connection. | IF                 | Determines connection status                      | Run when profile is update | Loop Over Items                           |                         |
| Schedule Trigger           | scheduleTrigger           | Periodic workflow trigger                         | -                         | Get Profiles                             |                         |
| Sticky Note                | stickyNote                | -                                               | -                         | -                                        |                         |
| Sticky Note1               | stickyNote                | -                                               | -                         | -                                        |                         |
| Sticky Note2               | stickyNote                | -                                               | -                         | -                                        |                         |
| Sticky Note3               | stickyNote                | -                                               | -                         | -                                        |                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node**  
   - Type: formTrigger  
   - Configure webhook to receive keywords input from users.

2. **Create a Browserflow node "Profile Scraper"**  
   - Use browser automation to scrape LinkedIn profiles based on keywords from form trigger.  
   - Output scraped profile data.

3. **Create a Split Out node**  
   - Splits the scraped profiles into individual items.

4. **Create a Google Sheets node "Add Profiles"**  
   - Use Google Sheets credentials with edit access.  
   - Append each profile to a spreadsheet for tracking.

5. **Create a Schedule Trigger node**  
   - Configure to run periodically (e.g., daily or hourly) to refresh connection status.

6. **Create a Google Sheets node "Get Profiles"**  
   - Reads profiles from the Google Sheet.

7. **Create a Browserflow node "Check Connection"**  
   - Automates LinkedIn UI to check if each profile is already connected.

8. **Create an IF node "If connection1"**  
   - Checks boolean from "Check Connection" to branch flow.

9. **Create a Wait node "Wait2"**  
   - Delay to pace requests before scraping new connection profiles.

10. **Create a Browserflow node "Scrap New Connection Profile"**  
    - Scrapes detailed info on new connections.

11. **Create a Set node "Organise Infomation"**  
    - Prepare data for AI message generation (e.g., format names, job titles).

12. **Create a Langchain AI Agent node**  
    - Uses OpenAI Chat Model to generate personalized messages.  
    - Connect OpenAI credentials (API key).  
    - Configure prompts/templates for sales messaging.

13. **Create an OpenAI Chat Model node**  
    - Use GPT-3.5 or GPT-4 as backend for AI Agent.

14. **Create a Browserflow node "Send Personalised Message"**  
    - Automates sending the AI-generated messages on LinkedIn.

15. **Create a Google Sheets Trigger node "Run when profile is update"**  
    - Watches for changes in the profile sheet to re-check connection status.

16. **Create an IF node "Check if a person is a connection."**  
    - Branches based on updated connection status.

17. **Create a SplitInBatches node "Loop Over Items"**  
    - Processes profiles in batches.

18. **Create an IF node "If connection"**  
    - Branches true to update stats, false to wait and send invites.

19. **Create a Google Sheets node "Stat Update"**  
    - Updates connection statistics.

20. **Create a Wait node "Wait"**  
    - Delay for pacing before sending invites.

21. **Create a SplitInBatches node "Loop Over Items1"**  
    - Batches profiles for invites and emails.

22. **Create a Gmail node (optional)**  
    - Sends emails if configured and connected.

23. **Create a Browserflow node "Invite Sender"**  
    - Sends LinkedIn invitations.

24. **Create a Google Sheets node "Update Invite"**  
    - Logs invite status.

25. **Create a Set node "Set Connection Limit"**  
    - Sets variables controlling limits on invites.

26. **Create a Wait node "Wait1"**  
    - Final pacing delay.

27. **Create a Browserflow node "Connection Checker"**  
    - Re-checks connection status in batch loops.

28. **Create a SplitInBatches node "Loop Over Items2"**  
    - Batch processing fallback.

29. **Create a NoOp node "add browser flow if it gives error"**  
    - Placeholder for error fallback automation.

30. **Connect all nodes according to the logical flow described in section 2: link triggers, batches, decisions, and outputs properly.**

31. **Ensure all credentials are configured:**  
    - Google Sheets OAuth2 or API Key with spreadsheet access.  
    - OpenAI API key for Chat model.  
    - Gmail OAuth2 if using Gmail node.  
    - Browserflow configured to automate LinkedIn sessions (authenticated).

32. **Test each block individually before full integration to catch errors early.**

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                |
|-----------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| The workflow requires valid LinkedIn browser session/cookies for browserflow nodes to function.     | LinkedIn login required for Browserflow nodes.                |
| OpenAI API key must have access to GPT-3.5 or GPT-4 for message generation.                          | https://platform.openai.com/account/api-keys                   |
| Google Sheets API access requires spreadsheet sharing with the service account or OAuth credentials. | https://developers.google.com/sheets/api/quickstart/python     |
| To avoid LinkedIn rate limits, pacing with Wait nodes and batch sizes is critical.                   | Follow LinkedIn automation best practices to prevent bans.    |
| For troubleshooting browserflow nodes, refer to n8n documentation on browser automation.             | https://docs.n8n.io/nodes/builtin/browserflow/                 |

---

*Disclaimer: The text above is exclusively derived from an automated workflow created using n8n, a workflow automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly available.*