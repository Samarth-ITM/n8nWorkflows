Daily Habit RPG: Gamified Gmail Reminders with Quest System and Progress Tracking

https://n8nworkflows.xyz/workflows/daily-habit-rpg--gamified-gmail-reminders-with-quest-system-and-progress-tracking-10688


# Daily Habit RPG: Gamified Gmail Reminders with Quest System and Progress Tracking

### 1. Workflow Overview

This workflow, titled **Daily Habit RPG: Gamified Gmail Reminders with Quest System and Progress Tracking**, transforms daily habit tracking into an engaging RPG-style adventure by sending users a personalized, gamified email every morning. The workflow targets productivity enthusiasts, habit consistency seekers, and lovers of gamification who want to make their routines motivating and fun.

**Core logical blocks:**

- **1.1 Input Reception & Trigger:** A scheduled trigger node fires every day at 6:00 AM to start the process.
- **1.2 Player Stats Initialization:** Randomly (demo mode) or in production from a database, player stats like level, streak, XP, and coins are initialized.
- **1.3 Quest Generation:** Based on the current day, daily quests are generated with RPG-style rewards and difficulty.
- **1.4 Motivational Quote Retrieval:** A motivational quote is fetched from an external API, with fallback handling.
- **1.5 Game Stats Processing:** Player achievements, rank, XP progress, and motivational messages are computed.
- **1.6 Email Template Creation:** A rich HTML email template is dynamically generated showing player stats, quests, achievements, and motivational content.
- **1.7 Email Sending:** The composed email is sent via Gmail using OAuth2 credentials.
- **1.8 Logging:** Execution stats are logged for tracking and potential storage.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Trigger

- **Overview:** This block initiates the workflow automatically every 24 hours at 6:00 AM.
- **Nodes Involved:**  
  - Daily Morning Trigger  
  - Step 1 (sticky note)  
  - 📋 Workflow Overview (sticky note)

- **Node Details:**

  - **Daily Morning Trigger**  
    - *Type:* Schedule Trigger  
    - *Role:* Starts workflow every 24 hours (default 6:00 AM)  
    - *Configuration:* Interval set to 24 hours; customizable trigger hour via node parameters  
    - *Inputs:* None (start node)  
    - *Outputs:* Initializes flow to next node  
    - *Edge Cases:* Misconfiguration might cause incorrect timing; node must be active to trigger  
    - *Sticky Notes:* Step 1 explains how to customize trigger time

  - **Step 1 (sticky note)**  
    - *Role:* Instructional note describing the trigger setup and customization

  - **📋 Workflow Overview (sticky note)**  
    - *Role:* High-level explanation and usage instructions for the entire workflow

#### 1.2 Player Stats Initialization

- **Overview:** Simulates or loads player stats such as level, streak, class, and coins; provides date context for subsequent logic.
- **Nodes Involved:**  
  - Initialize Player Stats (Code)  
  - Step 2 (sticky note)

- **Node Details:**

  - **Initialize Player Stats**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Generates demo player stats with random values and derives player class based on level; injects date info  
    - *Key Variables:* `level`, `streak`, `experience`, `coins`, `playerClass`, `date`, `weekday`, etc.  
    - *Inputs:* Trigger from schedule node  
    - *Outputs:* JSON object with player stats and date info passed downstream  
    - *Edge Cases:* Random values for demo; in production, failure to load real stats would require error handling or fallback  
    - *Version Requirements:* Uses n8n Code node version 2 for modern JS support

  - **Step 2 (sticky note)**  
    - *Role:* Explains that stats are demo-generated and suggests connecting to a real database for production

#### 1.3 Quest Generation

- **Overview:** Builds a list of daily quests with associated RPG rewards based on the current weekday; includes random bonus quest logic.
- **Nodes Involved:**  
  - Generate Daily Quests (Code)  
  - Step 3 (sticky note)

- **Node Details:**

  - **Generate Daily Quests**  
    - *Type:* Code node  
    - *Role:* Uses the player's weekday to select quests from predefined lists: daily, weekend, Monday; adds random bonus quests 20% of the time  
    - *Key Variables:* `todaysQuests`, `questDatabase` (with quests defined by name, difficulty, XP, coins, emoji), total XP and coins  
    - *Inputs:* Player stats JSON from previous node  
    - *Outputs:* Extended JSON with quests and total reward sums  
    - *Edge Cases:* If weekday data missing, quests default to daily only; random bonus quest is probabilistic  
    - *Customization:* Quest database can be edited to add/remove habits

  - **Step 3 (sticky note)**  
    - *Role:* Guides user on customizing quests and explains daily, weekend, Monday, and bonus quests

#### 1.4 Motivational Quote Retrieval

- **Overview:** Retrieves a motivational quote from a free external API to include in the email; continues gracefully on failure.
- **Nodes Involved:**  
  - Fetch Daily Quote (HTTP Request)  
  - Step 4 (sticky note)

- **Node Details:**

  - **Fetch Daily Quote**  
    - *Type:* HTTP Request  
    - *Role:* Fetches JSON from `https://api.quotable.io/random` without authentication  
    - *Configuration:* Response format JSON; `continueOnFail` set to true to avoid workflow failure if API is down  
    - *Inputs:* Quest data from previous node  
    - *Outputs:* Quote response JSON or fallback handled downstream  
    - *Edge Cases:* Network errors, rate limiting, or API downtime; graceful fallback implemented

  - **Step 4 (sticky note)**  
    - *Role:* Explains purpose and fallback mechanism for quote retrieval

#### 1.5 Game Stats Processing

- **Overview:** Consolidates player stats, achievements, ranks, and motivational messages; prepares data for email generation.
- **Nodes Involved:**  
  - Process Game Stats (Code)  
  - Step 5 (sticky note)

- **Node Details:**

  - **Process Game Stats**  
    - *Type:* Code node  
    - *Role:*  
      - Processes quote data with fallback default  
      - Checks streak and level for achievements (7-day, 30-day streaks, level 10)  
      - Assigns player rank and color based on level tiers (D to SSS)  
      - Calculates XP progress percentage and HP percentage  
      - Selects daily motivational message based on weekday  
    - *Key Variables:* `achievements[]`, `rankInfo`, `xpProgress`, `hpPercent`, `motivation`  
    - *Inputs:* Player/quest data and quote API response (two inputs)  
    - *Outputs:* Enriched player data with achievements and motivation  
    - *Edge Cases:* Missing or malformed input JSON; no achievements if conditions unmet

  - **Step 5 (sticky note)**  
    - *Role:* Summarizes achievement and rank calculation logic

#### 1.6 Email Template Creation

- **Overview:** Generates a polished, responsive HTML email and a plain text alternative, embedding all player and quest data with styling.
- **Nodes Involved:**  
  - Create Email Template (Code)  
  - Step 6 (sticky note)

- **Node Details:**

  - **Create Email Template**  
    - *Type:* Code node  
    - *Role:*  
      - Builds a full HTML email with CSS styling featuring gradient backgrounds, progress bars, badges, and quest lists  
      - Includes player stats, XP progress bar, streak, coins, rank badge, achievements banner, motivational message, and daily quests with icons and difficulty colors  
      - Also creates a plain text version for email clients that do not support HTML  
      - Returns email subject, HTML body, text body, and playerData for logging  
    - *Inputs:* Enriched player data from previous node  
    - *Outputs:* JSON with email content and metadata  
    - *Edge Cases:* Large quest lists may affect email size; ensures HTML is self-contained for email compatibility  
    - *Customization:* CSS styles and HTML structure can be edited for branding or personalization

  - **Step 6 (sticky note)**  
    - *Role:* Highlights the email template features and design

#### 1.7 Email Sending

- **Overview:** Sends the generated email via Gmail API using OAuth2 authentication.
- **Nodes Involved:**  
  - Send Quest Email (Gmail node)  
  - Step 7 - IMPORTANT (sticky note)

- **Node Details:**

  - **Send Quest Email**  
    - *Type:* Gmail node  
    - *Role:* Sends email to configured recipient with HTML and plain text content  
    - *Configuration:*  
      - `sendTo` email address must be updated by user  
      - Subject and message dynamically set from previous node’s output expressions  
      - OAuth2 credentials required and must be configured in n8n credentials  
    - *Inputs:* Email content JSON from template node  
    - *Outputs:* Email sending result passed downstream  
    - *Edge Cases:* Authentication failure, invalid recipient, Gmail API quota limits  
    - *Sticky Notes:* Step 7 describes credential setup and testing steps

  - **Step 7 - IMPORTANT (sticky note)**  
    - *Role:* Guides user to configure Gmail OAuth2 credentials and recipient email before activation

#### 1.8 Logging

- **Overview:** Logs the execution outcome and key stats for monitoring or analytics.
- **Nodes Involved:**  
  - Log Execution Stats (Code)  
  - Step 8 (sticky note)

- **Node Details:**

  - **Log Execution Stats**  
    - *Type:* Code node  
    - *Role:*  
      - Extracts email send results and player data  
      - Creates a stats object including execution time, status, player level, quests sent, total XP and coins, next execution time, and a status message  
      - Outputs stats for console logging or further integration (e.g., database, Google Sheets)  
    - *Inputs:* Email sending result JSON  
    - *Outputs:* JSON stats object logged to console  
    - *Edge Cases:* Missing player data fallback to defaults

  - **Step 8 (sticky note)**  
    - *Role:* Recommends optional integration for storing execution logs externally

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                                  | Input Node(s)             | Output Node(s)            | Sticky Note                                                                                                    |
|-----------------------|---------------------|-------------------------------------------------|---------------------------|---------------------------|---------------------------------------------------------------------------------------------------------------|
| 📋 Workflow Overview  | Sticky Note         | Provides high-level workflow description        | None                      | None                      | Describes workflow purpose, setup, and customization                                                          |
| Daily Morning Trigger  | Schedule Trigger    | Triggers workflow once daily at 6:00 AM         | None                      | Initialize Player Stats    | Step 1 explains customization of trigger time                                                                 |
| Step 1                | Sticky Note         | Explains scheduling and trigger customization   | None                      | None                      |                                                                                                               |
| Initialize Player Stats| Code                | Creates demo/random player stats and date info  | Daily Morning Trigger     | Generate Daily Quests     | Step 2 explains demo mode and production recommendations                                                      |
| Step 2                | Sticky Note         | Explains player stats generation                 | None                      | None                      |                                                                                                               |
| Generate Daily Quests  | Code                | Generates quests based on weekday and randomness| Initialize Player Stats   | Fetch Daily Quote          | Step 3 describes quest customization and logic                                                                |
| Step 3                | Sticky Note         | Guides quest generation customization            | None                      | None                      |                                                                                                               |
| Fetch Daily Quote      | HTTP Request        | Fetches motivational quote from API              | Generate Daily Quests     | Process Game Stats        | Step 4 explains quote retrieval and fallback                                                                   |
| Step 4                | Sticky Note         | Describes quote API and fallback mechanism       | None                      | None                      |                                                                                                               |
| Process Game Stats     | Code                | Calculates achievements, rank, progress, motivation | Fetch Daily Quote         | Create Email Template     | Step 5 summarizes achievements and rank calculations                                                          |
| Step 5                | Sticky Note         | Explains achievements and ranking logic          | None                      | None                      |                                                                                                               |
| Create Email Template  | Code                | Builds HTML and text email with stats and quests | Process Game Stats        | Send Quest Email          | Step 6 highlights email template design and features                                                          |
| Step 6                | Sticky Note         | Describes email template elements                 | None                      | None                      |                                                                                                               |
| Send Quest Email       | Gmail                | Sends the gamified quest email                     | Create Email Template     | Log Execution Stats       | Step 7 details Gmail OAuth2 credential setup and email testing                                                 |
| Step 7 - IMPORTANT    | Sticky Note         | Guides Gmail credential and recipient setup       | None                      | None                      |                                                                                                               |
| Log Execution Stats    | Code                | Logs execution and player stats for monitoring    | Send Quest Email          | None                      | Step 8 recommends optional persistent logging                                                                  |
| Step 8                | Sticky Note         | Explains logging and optional storage             | None                      | None                      |                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Daily Morning Trigger" node:**
   - Type: Schedule Trigger
   - Set interval to 24 hours
   - Set trigger hour to 6:00 AM (customizable)
   - Connect this as the workflow start node

2. **Create "Initialize Player Stats" node:**
   - Type: Code
   - Paste JS code to generate demo player stats (level, streak, coins, class) and date info
   - Connect output of "Daily Morning Trigger" to this node

3. **Create "Generate Daily Quests" node:**
   - Type: Code
   - Insert JS code defining quest database (daily, weekend, Monday quests) and logic to select quests based on current weekday
   - Add random bonus quest with 20% chance
   - Connect output of "Initialize Player Stats" to this node

4. **Create "Fetch Daily Quote" node:**
   - Type: HTTP Request
   - URL: `https://api.quotable.io/random`
   - Response format JSON
   - Set "Continue On Fail" to true
   - Connect output of "Generate Daily Quests" to this node

5. **Create "Process Game Stats" node:**
   - Type: Code
   - Use JS code to process quote data, calculate achievements (7-day, 30-day streaks, level milestones), assign rank and colors, calculate XP progress and HP %, and set daily motivation based on weekday
   - Connect outputs of both "Generate Daily Quests" (first input) and "Fetch Daily Quote" (second input) to this node

6. **Create "Create Email Template" node:**
   - Type: Code
   - Paste provided JS code that builds a styled HTML email and plain text alternative email content embedding player stats, quests, achievements, and motivation
   - Connect output of "Process Game Stats" to this node

7. **Create "Send Quest Email" node:**
   - Type: Gmail node
   - Authenticate with Gmail OAuth2 credentials (create new credentials if needed)
   - Set "Send To" address to your recipient email
   - Set Subject expression: `={{ $json.subject }}`
   - Set Message expression: `={{ $json.htmlBody }}`
   - Connect output of "Create Email Template" to this node

8. **Create "Log Execution Stats" node:**
   - Type: Code
   - Paste JS code that logs execution time, player level, quests sent, total XP and coins, next execution time, and message
   - Connect output of "Send Quest Email" to this node

9. **Add Sticky Notes for each step:**
   - Create sticky notes as per original content describing each step and setup instructions for clarity and maintainability

10. **Activate workflow:**
    - Test workflow manually first to verify all steps run correctly
    - Adjust Gmail OAuth2 credentials and recipient email as needed
    - Set workflow to active for daily automation

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow created with ❤️ for the n8n community                                                    | High-level workflow overview sticky note                                                        |
| No API key required for motivational quote API                                                   | Quote API: https://api.quotable.io/random                                                       |
| Gmail node requires OAuth2 credential setup with Google account                                  | See Step 7 sticky note for setup details                                                        |
| Customize habits by editing `questDatabase` object in "Generate Daily Quests" code node          | Allows adding/removing/modifying quests                                                          |
| Email template uses modern CSS with gradients, progress bars, and responsive design              | Editable in "Create Email Template" code node                                                   |
| Logging can be extended by connecting "Log Execution Stats" output to database or Google Sheets  | Optional enhancement for tracking user progress over time                                       |
| Motivational messages and achievements can be customized in "Process Game Stats" code node       | Customize texts and add more achievements or ranks if desired                                   |

---

This completes the structured, detailed reference document for the **Daily Habit RPG: Gamified Gmail Reminders with Quest System and Progress Tracking** workflow, enabling comprehension, reproduction, and modification.