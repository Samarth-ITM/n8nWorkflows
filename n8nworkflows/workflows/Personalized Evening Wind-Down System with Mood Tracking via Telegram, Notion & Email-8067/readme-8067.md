Personalized Evening Wind-Down System with Mood Tracking via Telegram, Notion & Email

https://n8nworkflows.xyz/workflows/personalized-evening-wind-down-system-with-mood-tracking-via-telegram--notion---email-8067


# Personalized Evening Wind-Down System with Mood Tracking via Telegram, Notion & Email

### 1. Workflow Overview

This workflow, titled **The Quiet Evening Ritual — Wind‑Down Automation**, implements a personalized evening wind-down system designed for wellness-focused users such as moms and creators. Its primary use case is to facilitate a gentle, ritualized transition from daytime activities to rest through mood tracking and personalized content delivery using Telegram, Notion, and Email.

The workflow triggers daily at 9 PM to prompt the user via Telegram to reflect on their day by selecting their mood ("Tired" or "Great"). Based on the mood selected:

- If **Tired**: it sends a 5-minute meditation link and optionally provides an AI-generated reflection prompt.
- If **Great**: it celebrates the positive mood, encouraging gratitude.

In all cases, the mood and optional reflections are logged into a Notion database named **Evening Reflection Log**, and a personalized goodnight email with the next day's affirmation is sent.

**Logical Blocks:**

- **1.1 Scheduling & Configuration Setup**: Defines the daily trigger and user-specific configurations.
- **1.2 User Interaction via Telegram**: Sends mood prompt, receives mood tap, and parses response.
- **1.3 Mood-Based Conditional Processing**:
  - **1.3.1 Tired Path**: Sends meditation, optionally generates AI reflection, logs to Notion, sends email.
  - **1.3.2 Great Path**: Sends celebration message, logs to Notion, sends email.
- **1.4 Affirmation Selection**: Picks a personalized affirmation for goodnight emails.
- **1.5 Data Logging & Email Sending**: Logs data to Notion and sends a goodnight email with affirmation.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Scheduling & Configuration Setup

**Overview:**  
This block handles the daily scheduling of the workflow and sets user-specific configuration parameters needed throughout the workflow, such as email addresses, Notion database IDs, and feature toggles.

**Nodes Involved:**  
- Cron: 9PM Daily  
- Set: User Config (Cron Path)  
- Set: User Config (Trigger Path)

**Node Details:**

- **Cron: 9PM Daily**  
  - *Type:* Cron trigger  
  - *Role:* Initiates the workflow daily at 9:00 PM server time  
  - *Configuration:* Set to trigger once daily at 21:00  
  - *Input:* None (external trigger)  
  - *Output:* Triggers Set: User Config (Cron Path)  
  - *Edge Cases:* Cron time zone misconfiguration may cause unexpected trigger times.

- **Set: User Config (Cron Path)**  
  - *Type:* Set node  
  - *Role:* Defines user-specific settings for the scheduled path (e.g., Telegram chat ID, Notion DB ID, email, meditation links, toggles)  
  - *Configuration:* Static key-value pairs including email address, Notion database ID, meditation link URL, GPT enabled flag, etc.  
  - *Input:* From Cron node  
  - *Output:* Triggers Telegram: Ask Mood  
  - *Edge Cases:* Incorrect config values lead to failures in downstream nodes.

- **Set: User Config (Trigger Path)**  
  - *Type:* Set node  
  - *Role:* Defines user config for the Telegram-triggered path (after user taps mood button)  
  - *Configuration:* Similar static config as above, ensuring consistent parameters for triggered flow  
  - *Input:* From Telegram Trigger: Button Tap  
  - *Output:* Triggers Set: Mood Parse  
  - *Edge Cases:* Same as above

---

#### 1.2 User Interaction via Telegram

**Overview:**  
This block manages interaction with the user on Telegram, including sending the mood prompt and receiving the selected mood via button tap.

**Nodes Involved:**  
- Telegram: Ask Mood  
- Telegram Trigger: Button Tap  
- Set: Mood Parse

**Node Details:**

- **Telegram: Ask Mood**  
  - *Type:* Telegram node (send message)  
  - *Role:* Sends a Telegram message at 9 PM asking: “🌙 Time to wind down, sweet soul. How was your day?” with two buttons: "Tired" and "Great"  
  - *Configuration:* Message text with inline keyboard buttons for mood selection  
  - *Input:* From Set: User Config (Cron Path)  
  - *Output:* None (message sent)  
  - *Edge Cases:* Telegram API rate limits, bot not added to chat, or invalid chat ID cause send failures.

- **Telegram Trigger: Button Tap**  
  - *Type:* Telegram Trigger node  
  - *Role:* Listens for user button tap responses from the mood prompt  
  - *Configuration:* Listens to callback queries in the private chat with the bot  
  - *Input:* External event (Telegram callback)  
  - *Output:* Triggers Set: User Config (Trigger Path)  
  - *Edge Cases:* Timeout if user does not respond; malformed callback data.

- **Set: Mood Parse**  
  - *Type:* Set node  
  - *Role:* Parses the user’s mood selection from the trigger data into a clear variable for conditional checks  
  - *Configuration:* Extracts mood value from Telegram callback payload, sets a variable like `mood` with "Tired" or "Great"  
  - *Input:* From Set: User Config (Trigger Path)  
  - *Output:* Triggers IF: Tired?  
  - *Edge Cases:* Unexpected or missing mood values lead to misrouting.

---

#### 1.3 Mood-Based Conditional Processing

##### 1.3.1 Tired Path

**Overview:**  
If the user selects "Tired," this branch sends a meditation link, optionally generates an AI reflection prompt via OpenAI GPT, logs data to Notion, and sends a goodnight email.

**Nodes Involved:**  
- IF: Tired? (true branch)  
- Telegram: Send Meditation  
- GPT: Reflection Prompt (optional)  
- IF: GPT Enabled?  
- Set: Notion Entry (Tired)  
- Notion: Log Entry (Tired)  
- Email: Goodnight (Tired)  
- Code: Pick Affirmation

**Node Details:**

- **IF: Tired?**  
  - *Type:* If node  
  - *Role:* Checks if parsed mood equals "Tired"  
  - *Input:* From Set: Mood Parse  
  - *Output:* True → Telegram: Send Meditation; False → Telegram: Celebrate Win  
  - *Edge Cases:* Case sensitivity or missing values may cause wrong path.

- **Telegram: Send Meditation**  
  - *Type:* Telegram node (send message)  
  - *Role:* Sends a message with a 5-minute meditation link to help wind down  
  - *Configuration:* Message text + meditation URL from user config  
  - *Input:* From IF: Tired? (true)  
  - *Output:* Triggers both GPT: Reflection Prompt (optional) and Code: Pick Affirmation  
  - *Edge Cases:* Telegram API failures; invalid meditation link URL.

- **GPT: Reflection Prompt (optional)**  
  - *Type:* OpenAI GPT node  
  - *Role:* Generates a personalized reflection prompt based on the user's mood and input (enabled if `gpt_enabled` is true)  
  - *Configuration:* Model, prompt template, temperature; uses input text from Telegram or config  
  - *Input:* From Telegram: Send Meditation  
  - *Output:* Triggers IF: GPT Enabled?  
  - *Edge Cases:* API quota limits, network errors, or disabled GPT flag.

- **IF: GPT Enabled?**  
  - *Type:* If node  
  - *Role:* Checks if GPT feature is enabled (`gpt_enabled` flag)  
  - *Input:* From GPT: Reflection Prompt (optional)  
  - *Output:* Both true and false paths lead to Set: Notion Entry (Tired) (to ensure logging regardless)  
  - *Edge Cases:* Misconfiguration of flag may skip AI step improperly.

- **Set: Notion Entry (Tired)**  
  - *Type:* Set node  
  - *Role:* Prepares the data payload for Notion database entry based on mood "Tired," including reflection text and date  
  - *Configuration:* Maps properties like Name, Mood, Reflection, Date, Affirmation from prior nodes and config  
  - *Input:* From IF: GPT Enabled? (both paths)  
  - *Output:* Triggers Notion: Log Entry (Tired)  
  - *Edge Cases:* Missing required properties cause Notion API failures.

- **Notion: Log Entry (Tired)**  
  - *Type:* Notion node (create entry)  
  - *Role:* Creates a new record in the Notion database **Evening Reflection Log** with the prepared data  
  - *Configuration:* Uses Notion integration credentials and DB ID from user config  
  - *Input:* From Set: Notion Entry (Tired)  
  - *Output:* Triggers Email: Goodnight (Tired)  
  - *Edge Cases:* Invalid DB ID, permission errors, or API rate limits.

- **Email: Goodnight (Tired)**  
  - *Type:* Email Send node  
  - *Role:* Sends a personalized goodnight email containing tomorrow’s affirmation and well wishes  
  - *Configuration:* Uses SMTP or Gmail OAuth credentials, recipient from user config, subject and body templates including affirmation  
  - *Input:* From Notion: Log Entry (Tired)  
  - *Output:* End of path  
  - *Edge Cases:* Email delivery failures, invalid recipient address.

- **Code: Pick Affirmation**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Selects a daily affirmation text from a predefined list or logic for the goodnight email  
  - *Configuration:* Contains affirmation array or logic, outputs one affirmation string  
  - *Input:* From Telegram: Send Meditation  
  - *Output:* Feeds into GPT: Reflection Prompt (optional) and downstream nodes  
  - *Edge Cases:* Empty array or code errors cause no affirmation selection.

---

##### 1.3.2 Great Path

**Overview:**  
If the user selects "Great," this branch sends a celebration message, logs the positive mood to Notion, and sends a goodnight email with affirmation.

**Nodes Involved:**  
- IF: Tired? (false branch)  
- Telegram: Celebrate Win  
- Code: Pick Affirmation (Great)  
- Set: Notion Entry (Great)  
- Notion: Log Entry (Great)  
- Email: Goodnight (Great)

**Node Details:**

- **Telegram: Celebrate Win**  
  - *Type:* Telegram node (send message)  
  - *Role:* Sends a congratulatory message celebrating the user’s positive mood  
  - *Configuration:* Simple text message with emojis and encouragement  
  - *Input:* From IF: Tired? (false)  
  - *Output:* Triggers Code: Pick Affirmation (Great)  
  - *Edge Cases:* Telegram API failures.

- **Code: Pick Affirmation (Great)**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Selects an affirmation similar to the tired path, possibly from the same or a different list  
  - *Configuration:* Similar to the previous affirmation picker  
  - *Input:* From Telegram: Celebrate Win  
  - *Output:* Triggers Set: Notion Entry (Great)  
  - *Edge Cases:* Same as above.

- **Set: Notion Entry (Great)**  
  - *Type:* Set node  
  - *Role:* Prepares Notion entry data for "Great" mood, logging the win and related info  
  - *Configuration:* Maps mood, date, and optional reflections, uses user config for DB ID  
  - *Input:* From Code: Pick Affirmation (Great)  
  - *Output:* Triggers Notion: Log Entry (Great)  
  - *Edge Cases:* Same as tired path.

- **Notion: Log Entry (Great)**  
  - *Type:* Notion node  
  - *Role:* Logs the "Great" mood entry into Notion database  
  - *Input:* From Set: Notion Entry (Great)  
  - *Output:* Triggers Email: Goodnight (Great)  
  - *Edge Cases:* Same as tired path.

- **Email: Goodnight (Great)**  
  - *Type:* Email Send node  
  - *Role:* Sends the goodnight email with affirmation for a positive closure  
  - *Input:* From Notion: Log Entry (Great)  
  - *Output:* End of workflow path  
  - *Edge Cases:* Same as tired path.

---

### 3. Summary Table

| Node Name                    | Node Type             | Functional Role                          | Input Node(s)               | Output Node(s)                  | Sticky Note                            |
|------------------------------|-----------------------|----------------------------------------|-----------------------------|--------------------------------|---------------------------------------|
| Note: Overview               | Sticky Note           | Documentation overview                  |                             |                                |                                       |
| Note: Setup                  | Sticky Note           | Setup instructions                      |                             |                                |                                       |
| Note: Compliance             | Sticky Note           | Compliance notes                        |                             |                                |                                       |
| Cron: 9PM Daily              | Cron                  | Daily 9 PM trigger                      |                             | Set: User Config (Cron Path)    |                                       |
| Set: User Config (Cron Path)| Set                   | User config for scheduled path         | Cron: 9PM Daily             | Telegram: Ask Mood              |                                       |
| Telegram: Ask Mood           | Telegram              | Sends mood prompt message               | Set: User Config (Cron Path)|                                | Telegram bot must be added to private chat |
| Telegram Trigger: Button Tap | Telegram Trigger      | Listens for mood button tap             |                             | Set: User Config (Trigger Path)| Telegram bot must be added to private chat |
| Set: User Config (Trigger Path)| Set                | User config for triggered path          | Telegram Trigger: Button Tap| Set: Mood Parse                |                                       |
| Set: Mood Parse             | Set                   | Extracts mood from Telegram response    | Set: User Config (Trigger Path)| IF: Tired?                  |                                       |
| IF: Tired?                  | If                    | Routes flow based on mood "Tired"       | Set: Mood Parse             | Telegram: Send Meditation (true), Telegram: Celebrate Win (false)|                                       |
| Telegram: Send Meditation   | Telegram              | Sends meditation link message           | IF: Tired? (true)           | GPT: Reflection Prompt, Code: Pick Affirmation | Meditation link must be valid URL    |
| GPT: Reflection Prompt (optional)| OpenAI GPT        | Optional AI-generated reflection prompt | Telegram: Send Meditation   | IF: GPT Enabled?               | Requires OpenAI credentials if enabled |
| IF: GPT Enabled?            | If                    | Checks if GPT feature is enabled        | GPT: Reflection Prompt      | Set: Notion Entry (Tired)      |                                       |
| Set: Notion Entry (Tired)   | Set                   | Prepares Notion data for "Tired" mood  | IF: GPT Enabled?            | Notion: Log Entry (Tired)      |                                       |
| Notion: Log Entry (Tired)   | Notion                | Logs "Tired" mood entry to Notion DB    | Set: Notion Entry (Tired)   | Email: Goodnight (Tired)       | Requires Notion integration & DB access |
| Email: Goodnight (Tired)    | Email Send            | Sends goodnight email with affirmation  | Notion: Log Entry (Tired)   |                                | Requires Email credentials (SMTP or OAuth) |
| Telegram: Celebrate Win     | Telegram              | Sends celebration message for "Great"  | IF: Tired? (false)          | Code: Pick Affirmation (Great) |                                       |
| Code: Pick Affirmation (Great)| Code                | Picks affirmation text for "Great" mood| Telegram: Celebrate Win      | Set: Notion Entry (Great)      |                                       |
| Set: Notion Entry (Great)   | Set                   | Prepares Notion data for "Great" mood  | Code: Pick Affirmation (Great)| Notion: Log Entry (Great)     |                                       |
| Notion: Log Entry (Great)   | Notion                | Logs "Great" mood entry to Notion DB    | Set: Notion Entry (Great)   | Email: Goodnight (Great)       |                                       |
| Email: Goodnight (Great)    | Email Send            | Sends goodnight email with affirmation  | Notion: Log Entry (Great)   |                                |                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Credentials:**
   - Telegram Bot via BotFather; ensure bot is added to your private chat.
   - Notion Integration with access to a database named **Evening Reflection Log**.
   - Email credentials (Gmail OAuth2 or SMTP).

2. **Set up Notion Database:**
   - Create **Evening Reflection Log** with properties:
     - Name (Title)
     - Mood (Select)
     - Reflection (Text)
     - Top Win (Text)
     - Date (Date)
     - Affirmation (Text)

3. **Create Cron Node "Cron: 9PM Daily":**
   - Set to trigger every day at 21:00.

4. **Create Set Node "Set: User Config (Cron Path)":**
   - Add key-value pairs for:
     - `telegramChatId` (Telegram chat ID)
     - `notionDbId` (Notion database ID)
     - `emailRecipient` (your email address)
     - `meditationLink` (URL to 5-minute meditation)
     - `gpt_enabled` (boolean true/false)
     - Any other toggles or links needed.

5. **Connect Cron to Set: User Config (Cron Path).**

6. **Create Telegram Node "Telegram: Ask Mood":**
   - Message: “🌙 Time to wind down, sweet soul. How was your day?”
   - Inline buttons: "Tired" and "Great" with callback data set accordingly.
   - Use Telegram credentials.
   - Connect Set: User Config (Cron Path) → Telegram: Ask Mood.

7. **Create Telegram Trigger Node "Telegram Trigger: Button Tap":**
   - Listens for button taps in the private chat.
   - Create Set Node "Set: User Config (Trigger Path)" with same config keys as cron path.
   - Connect Telegram Trigger → Set: User Config (Trigger Path).

8. **Create Set Node "Set: Mood Parse":**
   - Extract the mood from the Telegram callback data.
   - Store in a variable `mood`.
   - Connect Set: User Config (Trigger Path) → Set: Mood Parse.

9. **Create If Node "IF: Tired?":**
   - Condition: `mood` equals "Tired".
   - Connect Set: Mood Parse → IF: Tired?.

10. **Tired Path:**

    a. Telegram Node "Telegram: Send Meditation":
       - Send meditation message with meditation link.
       - Connect IF: Tired? (true) → Telegram: Send Meditation.

    b. Code Node "Code: Pick Affirmation":
       - JavaScript code to select a daily affirmation string from an array.
       - Connect Telegram: Send Meditation → Code: Pick Affirmation.

    c. OpenAI Node "GPT: Reflection Prompt (optional)":
       - Use OpenAI credentials.
       - Prompt for reflection based on mood; enabled only if `gpt_enabled` is true.
       - Connect Telegram: Send Meditation → GPT: Reflection Prompt.

    d. If Node "IF: GPT Enabled?":
       - Check `gpt_enabled` boolean.
       - Connect GPT: Reflection Prompt → IF: GPT Enabled?.

    e. Set Node "Set: Notion Entry (Tired)":
       - Map data: Name, Mood = "Tired", Reflection (from GPT or empty), Date (current), Affirmation (from Code node).
       - Connect both outputs from IF: GPT Enabled? → Set: Notion Entry (Tired).

    f. Notion Node "Notion: Log Entry (Tired)":
       - Create new page in Notion DB with data from Set node.
       - Connect Set: Notion Entry (Tired) → Notion: Log Entry (Tired).

    g. Email Node "Email: Goodnight (Tired)":
       - Email to user with affirmation and goodnight message.
       - Connect Notion: Log Entry (Tired) → Email: Goodnight (Tired).

11. **Great Path:**

    a. Telegram Node "Telegram: Celebrate Win":
       - Sends congratulatory message.
       - Connect IF: Tired? (false) → Telegram: Celebrate Win.

    b. Code Node "Code: Pick Affirmation (Great)":
       - Same or similar function as above.
       - Connect Telegram: Celebrate Win → Code: Pick Affirmation (Great).

    c. Set Node "Set: Notion Entry (Great)":
       - Prepare data for Notion with Mood = "Great".
       - Connect Code: Pick Affirmation (Great) → Set: Notion Entry (Great).

    d. Notion Node "Notion: Log Entry (Great)":
       - Logs "Great" mood entry.
       - Connect Set: Notion Entry (Great) → Notion: Log Entry (Great).

    e. Email Node "Email: Goodnight (Great)":
       - Sends goodnight email with affirmation.
       - Connect Notion: Log Entry (Great) → Email: Goodnight (Great).

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                                       |
|------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| This workflow is designed for a warm, ritualized transition from work to rest, enhancing wellness and mindfulness.| Wellness & Lifestyle automation                                                                                       |
| Requires Telegram Bot created via BotFather and added to a private chat for correct operation.                    | Telegram bot setup guide                                                                                              |
| Notion database must have exact properties as named for correct logging.                                          | Notion database schema: Evening Reflection Log                                                                        |
| GPT feature is optional; enable by setting `gpt_enabled` to true and adding OpenAI credentials in n8n.            | OpenAI API integration                                                                                                |
| Email node supports Gmail OAuth or SMTP; ensure credentials are correctly configured for email delivery.          | Email credential setup                                                                                                |
| Meditation link can be customized by user in config Set nodes.                                                    | Customize meditation content                                                                                          |
| The affirmation picker is implemented in JavaScript Code nodes, allowing easy customization of affirmations.     | Customizable affirmations via Code node                                                                               |
| Sticky notes are used in the workflow to provide setup instructions and compliance confirmations.                 | See workflow sticky notes for detailed guidance                                                                       |

---

**Disclaimer:**  
The provided content is exclusively derived from an n8n automated workflow. All data processed is legal and public, strictly respecting content policies. No illegal or protected elements are present.