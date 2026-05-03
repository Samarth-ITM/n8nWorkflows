Daily Trello Task Deadline Digests to Slack

https://n8nworkflows.xyz/workflows/daily-trello-task-deadline-digests-to-slack-7613


# Daily Trello Task Deadline Digests to Slack

---

### 1. Workflow Overview

This n8n workflow automates daily task deadline reminders by extracting due tasks from a specified Trello board and sending a digest message to Slack. It is designed for teams or individuals who want to stay updated on task deadlines within Slack without manually checking Trello.

**Target Use Cases:**  
- Project management daily reminders  
- Administrative task tracking  
- Team notifications on due tasks

**Logical Blocks:**  
- **1.1 Input Trigger:** Manual execution trigger to start the workflow  
- **1.2 Trello Data Retrieval:** Fetch board, lists, and cards data from Trello  
- **1.3 Data Transformation & Filtering:** Map Trello card fields and filter tasks due on the current day  
- **1.4 Slack Notification:** Send a formatted message to a specified Slack user about due tasks  

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger

**Overview:**  
Starts the workflow manually when the user clicks “Execute workflow” in n8n.

**Nodes Involved:**  
- When clicking ‘Execute workflow’

**Node Details:**  
- **Type:** Manual Trigger  
- **Role:** Initiates workflow execution on demand  
- **Configuration:** Default manual trigger, no parameters needed  
- **Input/Output:** No input; outputs a trigger event to downstream nodes  
- **Edge Cases:** None specific; workflow will not run unless manually triggered  
- **Version Requirements:** n8n v1.x+ standard node  

---

#### 1.2 Trello Data Retrieval

**Overview:**  
Retrieves the Trello board by URL, fetches all lists on that board, then retrieves all cards from each list.

**Nodes Involved:**  
- Get Board1  
- Get Lists1  
- Get Cards1  

**Node Details:**  

- **Get Board1:**  
  - Type: Trello node  
  - Role: Fetches Trello board metadata by URL  
  - Configuration: Board URL set to `https://trello.com/b/DCpuJbnd/administrative-tasks`  
  - Credentials: Uses Trello API key and token stored in n8n credentials  
  - Input: Trigger from manual node  
  - Output: Board JSON including board id and name  
  - Edge Cases: Invalid URL, expired credentials, API rate limits  

- **Get Lists1:**  
  - Type: Trello node  
  - Role: Fetches all lists on the board using board ID from Get Board1  
  - Configuration: Board ID dynamically set as `={{ $json.id }}` from previous node  
  - Credentials: Same Trello credentials as above  
  - Input: Output from Get Board1  
  - Output: Array of lists with their IDs and names  
  - Edge Cases: Board with no lists, API timeouts  

- **Get Cards1:**  
  - Type: Trello node  
  - Role: Fetches all cards for each list using list ID from Get Lists1  
  - Configuration: List ID dynamically set as `={{ $json.id }}`  
  - Credentials: Same Trello credentials  
  - Input: Output from Get Lists1  
  - Output: Cards JSON with details per card  
  - Edge Cases: Lists with no cards, API errors  

---

#### 1.3 Data Transformation & Filtering

**Overview:**  
Maps relevant Trello card fields into a simplified structure, merges with today's date, then filters tasks to include only those due today.

**Nodes Involved:**  
- Map Fields1  
- Today's Date  
- Merge  
- Filter  

**Node Details:**  

- **Map Fields1:**  
  - Type: Set node  
  - Role: Extracts and renames fields from Trello card JSON for easier processing  
  - Configuration: Creates fields:  
    - Board Name (from Get Board1)  
    - List Name (from Get Lists1)  
    - Task Name (card name)  
    - Task Description (card description)  
    - Due Date (card due date badge, trimmed to YYYY-MM-DD)  
    - url (card URL)  
  - Key Expressions: Uses expressions like `={{ $('Get Board1').item.json.name }}` and substring for due date  
  - Input: Cards JSON from Get Cards1  
  - Output: Structured task objects  
  - Edge Cases: Missing due date badges, missing descriptions, malformed dates  

- **Today's Date:**  
  - Type: Code node (JavaScript)  
  - Role: Generates current date in ISO format (YYYY-MM-DD) for filtering  
  - Configuration: Returns JSON with `badges.today` set to today’s date string  
  - Output: Single JSON item with current date  
  - Edge Cases: Timezone differences could affect 'today' definition  

- **Merge:**  
  - Type: Merge node  
  - Role: Combines today's date and mapped tasks into one data stream  
  - Configuration: Mode “combine” with “combine all” option  
  - Input: Two inputs — today's date and mapped tasks  
  - Output: Combined JSON array for filtering  
  - Edge Cases: Inputs not synchronized or empty  

- **Filter:**  
  - Type: Filter node  
  - Role: Filters tasks where `badges.today` equals the task's due date  
  - Configuration: Condition: `{{$json.badges.today}} == {{$json['Due Date']}}`  
  - Input: Combined data from Merge  
  - Output: Only tasks due today pass through  
  - Edge Cases: Tasks without due date, time zone issues, empty input  

---

#### 1.4 Slack Notification

**Overview:**  
Sends a Slack direct message to a specific user with a link to each task due today.

**Nodes Involved:**  
- Send a message  

**Node Details:**  
- Type: Slack node (OAuth2)  
- Role: Delivers message to Slack user  
- Configuration:  
  - Text: `"Task Due Today: {{ $json.url }}"` — includes task URL dynamically  
  - Recipient: Slack user selected by user ID `U09ADJPB7QA` (can be changed to a channel or other user)  
  - Authentication: OAuth2 via Slack credentials stored in n8n  
- Input: Filtered tasks from Filter node  
- Output: Slack API response  
- Edge Cases: Invalid Slack token, user ID not found, message rate limits, Slack API downtime  

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role                 | Input Node(s)                     | Output Node(s)                  | Sticky Note                                                                                                                                                                                                                                                                                                                                                       |
|----------------------------|---------------------|--------------------------------|----------------------------------|--------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger      | Workflow entry point            | —                                | Get Board1, Today's Date        |                                                                                                                                                                                                                                                                                                                                                                  |
| Get Board1                 | Trello              | Fetch Trello board data         | When clicking ‘Execute workflow’ | Get Lists1                     | ### 1️⃣ Connect Trello (Developer API) 1. Get your **API key**: https://trello.com/app-key  2. Generate a **token** (from the same page → **Token**) 3. In n8n → **Credentials → New → Trello API**, paste **API Key** and **Token**, save.                                                                                                                      |
| Get Lists1                 | Trello              | Fetch all lists on board        | Get Board1                      | Get Cards1                     | Same as Get Board1                                                                                                                                                                                                                                                                                                                                                |
| Get Cards1                 | Trello              | Fetch all cards on each list    | Get Lists1                     | Map Fields1                    | Same as Get Board1                                                                                                                                                                                                                                                                                                                                                |
| Map Fields1                | Set                 | Map and rename Trello card fields | Get Cards1                     | Merge                         |                                                                                                                                                                                                                                                                                                                                                                  |
| Today's Date               | Code                | Generate current date string    | When clicking ‘Execute workflow’ | Merge                         |                                                                                                                                                                                                                                                                                                                                                                  |
| Merge                     | Merge               | Combine today's date and tasks  | Map Fields1, Today's Date       | Filter                        |                                                                                                                                                                                                                                                                                                                                                                  |
| Filter                    | Filter              | Keep only tasks due today       | Merge                         | Send a message                |                                                                                                                                                                                                                                                                                                                                                                  |
| Send a message             | Slack               | Send Slack message with task URL | Filter                        | —                             | ### 2️⃣ Connect Slack 1. Go to [Slack API Apps](https://api.slack.com/apps) and create a new app.  2. Add **OAuth & Permissions** → include scopes like: `chat:write`, `users:read`  3. Install app and copy **Bot User OAuth Token**  4. In n8n → **Credentials → New → Slack OAuth2 API**, paste token and save  5. In Slack node, select credential and recipient user/channel. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow on demand  
   - No parameters  

2. **Create Trello Credentials**  
   - In n8n Credentials, add new Trello API credentials  
   - Enter API Key and Token obtained from [Trello App Key](https://trello.com/app-key)  
   - Save credentials  

3. **Add Trello Node “Get Board1”**  
   - Resource: Board  
   - Operation: Get  
   - Board URL: `https://trello.com/b/DCpuJbnd/administrative-tasks` (replace with your board)  
   - Credentials: Select Trello credentials created  
   - Connect Manual Trigger → Get Board1  

4. **Add Trello Node “Get Lists1”**  
   - Resource: List  
   - Operation: Get All  
   - Board ID: Set to `={{ $json.id }}` from Get Board1 output  
   - Credentials: Same Trello credentials  
   - Connect Get Board1 → Get Lists1  

5. **Add Trello Node “Get Cards1”**  
   - Resource: List  
   - Operation: Get Cards  
   - List ID: Set to `={{ $json.id }}` from Get Lists1 output  
   - Credentials: Same Trello credentials  
   - Connect Get Lists1 → Get Cards1  

6. **Add Set Node “Map Fields1”**  
   - Purpose: Map relevant fields for each card  
   - Create fields:  
     - Board Name: `={{ $('Get Board1').item.json.name }}`  
     - List Name: `={{ $('Get Lists1').item.json.name }}`  
     - Task Name: `={{ $json.name }}`  
     - Task Description: `={{ $json.desc }}`  
     - Due Date: `={{ $json.badges.due.trim().substring(0, 10) }}`  
     - url: `={{ $json.url }}`  
   - Connect Get Cards1 → Map Fields1  

7. **Add Code Node “Today's Date”**  
   - Purpose: Generate current date string  
   - Code (JavaScript):  
     ```javascript
     return [{
       json: {
         badges: {
           today: new Date().toISOString().split('T')[0]
         }
       }
     }];
     ```  
   - Connect Manual Trigger → Today's Date (parallel to Get Board1)  

8. **Add Merge Node “Merge”**  
   - Mode: Combine  
   - Combine By: Combine All  
   - Connect Today's Date → Merge (input 1)  
   - Connect Map Fields1 → Merge (input 2)  

9. **Add Filter Node “Filter”**  
   - Condition:  
     - Left Value: `={{ $json.badges.today }}`  
     - Operator: Equals  
     - Right Value: `={{ $json['Due Date'] }}`  
   - Connect Merge → Filter  

10. **Create Slack OAuth2 Credentials**  
    - Go to [Slack API Apps](https://api.slack.com/apps), create app  
    - Add OAuth & Permissions scopes: `chat:write` and optionally `users:read`  
    - Install app and copy Bot User OAuth Token  
    - In n8n Credentials, add new Slack OAuth2 API credentials and paste token  

11. **Add Slack Node “Send a message”**  
    - Authentication: OAuth2 (select Slack credentials)  
    - Text: `Task Due Today: {{ $json.url }}`  
    - Recipient: Select User or Channel by ID (e.g., user ID `U09ADJPB7QA`)  
    - Connect Filter → Send a message  

12. **Save and Test**  
    - Execute the workflow manually  
    - Confirm Trello data retrieval and Slack message sending  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| ⚙️ Setup Instructions for Trello and Slack API connections, including how to create credentials in n8n.                                                                                                                                                                                                                                                                  | See Sticky Note1 and Sticky Note46 in the workflow JSON for detailed steps                       |
| Trello API Key and Token are mandatory for accessing Trello data; ensure they have proper permissions.                                                                                                                                                                                                                                                                   | https://trello.com/app-key                                                                        |
| Slack Bot OAuth Token requires `chat:write` permission to send messages; ensure the bot is installed in the target workspace.                                                                                                                                                                                                                                          | https://api.slack.com/apps                                                                       |
| Workflow author and support contact: Robert Breen, [LinkedIn](https://www.linkedin.com/in/robert-breen-29429625/), [ynteractive.com](https://ynteractive.com), email: robert@ynteractive.com                                                                                                                                                                            | Contact for customization and assistance                                                        |
| The workflow assumes tasks have due dates set in Trello and uses the due date badge for filtering. Tasks without due dates will be excluded.                                                                                                                                                                                                                              | Important for filtering logic                                                                    |
| Timezone considerations: The date generated in the code node uses UTC (ISO string). Adjust if your Trello due dates or Slack notifications need local timezones.                                                                                                                                                                                                       | May require customization in “Today's Date” node                                               |
| Slack user ID (`U09ADJPB7QA`) used in the Slack node must be replaced with the intended recipient’s Slack user or channel ID.                                                                                                                                                                                                                                          | User/channel selection in Slack node configuration                                              |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---