Create a Dynamic Telegram Bot Menu System with Multi-Level Navigation

https://n8nworkflows.xyz/workflows/create-a-dynamic-telegram-bot-menu-system-with-multi-level-navigation-8844


# Create a Dynamic Telegram Bot Menu System with Multi-Level Navigation

---

### 1. Workflow Overview

This workflow implements a **Dynamic Telegram Bot Menu System with Multi-Level Navigation** using n8n. It is designed to provide Telegram users with an interactive, hierarchical menu interface where they can navigate through multiple levels, trigger actions, and receive dynamic responses.

**Target Use Cases:**
- Bots requiring complex menu-driven navigation
- Multi-level menus with nested submenus and action buttons
- Dynamic responses based on user commands or callbacks
- Easily extensible with new menus and handlers without modifying core logic

**Logical Blocks:**

- **1.1 Input Reception:** Handles incoming Telegram updates (messages and callback queries).
- **1.2 Menu Configuration Loading:** Loads the static menu structure defining all menus and buttons.
- **1.3 Command Extraction:** Extracts the command and user information from Telegram updates.
- **1.4 Data Merging:** Combines menu config and command data for unified processing.
- **1.5 Command Processing:** Determines the target menu and whether the command requires special handling.
- **1.6 Routing Logic:** Routes commands that require business logic to dedicated handlers.
- **1.7 Business Logic Handlers:** Seven separate nodes handling specific command actions (e.g., rating, language change, analytics).
- **1.8 Response Building:** Constructs the final message text and keyboard to send back.
- **1.9 Telegram API Interaction:** Prepares and sends requests to Telegram API to update or send messages.
- **1.10 Auxiliary:** Sticky notes providing guides, troubleshooting, and examples.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Listens for Telegram updates such as messages and callback queries that trigger the bot.
- **Nodes Involved:** Telegram Trigger4
- **Node Details:**
  - Type: Telegram Trigger
  - Configuration: Listens to "message" and "callback_query" update types.
  - Credentials: Configured with Telegram API credentials for bot authorization.
  - Input: Telegram webhook updates.
  - Output: Raw Telegram update JSON.
  - Failure Cases: Invalid webhook URL, credential errors, Telegram API downtime.

#### 1.2 Menu Configuration Loading

- **Overview:** Defines the entire multi-level menu navigation structure as a pure data object with no business logic.
- **Nodes Involved:** Load Menu Config2
- **Node Details:**
  - Type: Function
  - Configuration: Defines a constant `MENU_CONFIG` object with all menu keys, display texts (supporting HTML), and inline keyboard button arrays.
  - Key Expressions: Static JavaScript object with detailed comments explaining menu addition, formatting, and navigation rules.
  - Inputs: Telegram update JSON from trigger.
  - Outputs: Original data augmented with `menuConfig`.
  - Edge Cases: Syntax errors in menu config, missing back navigation buttons.
  - Notes: This is the single source of truth for all menus.

#### 1.3 Command Extraction

- **Overview:** Parses incoming update to extract the user command, chat and user IDs, and whether the update is a callback.
- **Nodes Involved:** Extract Command4
- **Node Details:**
  - Type: Function
  - Logic:
    - Detects if update is a message or callback query.
    - For messages, detects commands like /start, help, settings, etc.
    - For callback queries, extracts callback data as the command.
    - Extracts user info (first name), chat ID, message ID if callback.
  - Outputs: JSON with keys: `command`, `userName`, `userId`, `chatId`, `messageId`, `isCallback`, `callbackQueryId`.
  - Edge Cases: Missing text, malformed callback data, anonymous users.

#### 1.4 Data Merging

- **Overview:** Combines the menu configuration data with extracted command/user data into a single JSON item.
- **Nodes Involved:** Merge Config & Command2
- **Node Details:**
  - Type: Merge (combine by position)
  - Inputs: Outputs from Load Menu Config2 and Extract Command4
  - Outputs: Merged JSON object containing both menuConfig and command info.
  - Edge Cases: Mismatched item counts, null values.

#### 1.5 Command Processing

- **Overview:** Determines target menu object for the command and checks if the command requires a special business logic action.
- **Nodes Involved:** Process Command2
- **Node Details:**
  - Type: Function
  - Logic:
    - Looks up command key in `menuConfig`.
    - If not found, uses `default` menu.
    - Defines an array `actionCommands` listing commands that require business logic handlers.
    - Sets boolean `requiresAction`.
  - Outputs: JSON including `menu` and `requiresAction`.
  - Edge Cases: Unknown commands, empty menu config.

#### 1.6 Routing Logic

- **Overview:** Routes commands flagged as requiring action to one of seven dedicated business logic handlers based on command patterns.
- **Nodes Involved:** Needs Action?2, Action Router2
- **Node Details:**
  - Needs Action?2:
    - Type: If
    - Checks `requiresAction` boolean.
    - Routes true to Action Router2, false to Build Response2.
  - Action Router2:
    - Type: Switch
    - Routes based on command prefix or exact matches:
      - Outputs:
        0: rating commands (rate_1 to rate_5)
        1: language commands (lang_en, lang_es)
        2: analytics/report
        3: stats/achievements
        4: bug_report/feature_request
        5: notif/profile
        6: default fallback
  - Edge Cases: Unrecognized commands route to default.

#### 1.7 Business Logic Handlers

- **Overview:** Handle specific command actions, generate dynamic data, and prepare custom response content.
- **Nodes Involved:**
  - Rating Handler2
  - Language Handler2
  - Analytics Handler2
  - Statistics Handler3
  - Feedback Handler3
  - Settings Handler3
  - Default Handler2
- **Node Details:**

| Handler Node        | Role & Logic Summary                                                                                  | Outputs                        | Edge Cases / Failures                   |
|---------------------|----------------------------------------------------------------------------------------------------|-------------------------------|---------------------------------------|
| Rating Handler2     | Parses rating (1-5), returns custom thank-you message.                                              | JSON with `customData.ratingMessage` and `actionExecuted` | Invalid rating number, parsing errors |
| Language Handler2   | Extracts language code, returns confirmation.                                                      | JSON with `actionExecuted`     | Unknown language codes                 |
| Analytics Handler2  | Generates fake analytics or report data, returns formatted strings.                                 | JSON with analytics/report data and `actionExecuted` | Random data generation issues          |
| Statistics Handler3 | Generates fake stats or achievements data for user.                                               | JSON with stats/achievements and `actionExecuted` | Missing command key                    |
| Feedback Handler3   | Creates ticket ID for bug or feature request, returns ticket info.                                 | JSON with ticket info and `actionExecuted` | Timestamp errors                      |
| Settings Handler3   | Loads notification or profile settings, returns status strings.                                   | JSON with settings data and `actionExecuted` | Not matching commands                  |
| Default Handler2    | Handles commands with no specific actions, empty customData.                                      | JSON with empty customData and `actionExecuted` | None                                |

- Each handler outputs to Build Response2.

#### 1.8 Response Building

- **Overview:** Constructs the final text message replacing placeholders with dynamic user or command data and prepares keyboard layout.
- **Nodes Involved:** Build Response2
- **Node Details:**
  - Type: Function
  - Logic:
    - Replaces placeholders `{userName}`, `{userId}`, and any keys from `customData` in menu text.
    - Removes any leftover placeholders.
    - Passes final `responseText` and `keyboard` for sending.
  - Edge Cases: Missing placeholders, malformed strings.

#### 1.9 Telegram API Interaction

- **Overview:** Prepares and sends the Telegram API request to either send a new message or edit an existing message for callback queries.
- **Nodes Involved:** Prepare Telegram2, Set Bot Token4, Is Callback?4, Send to Telegram4, Answer Callback4
- **Node Details:**
  - Prepare Telegram2:
    - Determines API method: `sendMessage` or `editMessageText`.
    - Builds request body including chat ID, text, parse mode, and inline keyboard.
  - Set Bot Token4:
    - Sets bot token string for API calls (placeholder to be replaced by user).
  - Is Callback?4:
    - Checks if update is callback query.
    - If true: sends message edit and answers callback query with a confirmation.
    - If false: sends new message.
  - Send to Telegram4 and Answer Callback4:
    - HTTP Request nodes invoking Telegram API.
    - Send JSON with method and payload.
  - Edge Cases: Invalid token, Telegram API errors, network failures.

#### 1.10 Auxiliary Nodes (Sticky Notes)

- **Overview:** Provide extensive documentation, examples, troubleshooting, architecture overview, and guides inside the workflow.
- **Nodes Involved:** Multiple sticky notes, e.g., 📚 REAL EXAMPLES, 🚀 QUICK START GUIDE, 📚 WORKFLOW ARCHITECTURE, etc.
- **Node Details:**
  - Contain instructions on adding menus, handlers, troubleshooting tips.
  - Include code snippets for quick modifications.
  - Help users understand workflow design and extend it.

---

### 3. Summary Table

| Node Name             | Node Type              | Functional Role                          | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                      |
|-----------------------|------------------------|----------------------------------------|-------------------------------|-------------------------------|------------------------------------------------------------------------------------------------|
| Telegram Trigger4      | Telegram Trigger       | Receives Telegram updates              |                               | Load Menu Config2, Extract Command4 |                                                                                                |
| Load Menu Config2      | Function               | Loads static menu structure             | Telegram Trigger4              | Merge Config & Command2        | See "📋 CHECK THE CODE!" sticky note for menu code details                                      |
| Extract Command4       | Function               | Extracts command and user info          | Telegram Trigger4              | Merge Config & Command2        |                                                                                                |
| Merge Config & Command2| Merge                  | Combines menu config and command data   | Load Menu Config2, Extract Command4 | Process Command2              |                                                                                                |
| Process Command2       | Function               | Determines menu and action requirement  | Merge Config & Command2        | Needs Action?2                 |                                                                                                |
| Needs Action?2         | If                     | Checks if command requires action       | Process Command2               | Action Router2, Build Response2|                                                                                                |
| Action Router2         | Switch                 | Routes action commands to handlers      | Needs Action?2                 | Rating Handler2, Language Handler2, Analytics Handler2, Statistics Handler3, Feedback Handler3, Settings Handler3, Default Handler2 |                                                                                                |
| Rating Handler2        | Function               | Processes rating commands                | Action Router2 (out 0)         | Build Response2               |                                                                                                |
| Language Handler2      | Function               | Processes language change commands       | Action Router2 (out 1)         | Build Response2               |                                                                                                |
| Analytics Handler2     | Function               | Processes analytics/report commands      | Action Router2 (out 2)         | Build Response2               |                                                                                                |
| Statistics Handler3    | Function               | Processes stats and achievements commands| Action Router2 (out 3)         | Build Response2               |                                                                                                |
| Feedback Handler3      | Function               | Processes feedback commands (tickets)    | Action Router2 (out 4)         | Build Response2               |                                                                                                |
| Settings Handler3      | Function               | Processes settings commands              | Action Router2 (out 5)         | Build Response2               |                                                                                                |
| Default Handler2       | Function               | Handles unknown or default commands      | Action Router2 (out 6)         | Build Response2               |                                                                                                |
| Build Response2       | Function               | Builds final message text and keyboard   | Needs Action?2, Handlers      | Prepare Telegram2             |                                                                                                |
| Prepare Telegram2     | Function               | Prepares Telegram API request             | Build Response2               | Set Bot Token4                |                                                                                                |
| Set Bot Token4        | Set                    | Sets bot token for API calls              | Prepare Telegram2             | Is Callback?4                | Replace placeholder with your bot token                                                        |
| Is Callback?4         | If                     | Checks if update is callback query        | Set Bot Token4                | Send to Telegram4, Answer Callback4 |                                                                                                |
| Send to Telegram4     | HTTP Request           | Sends message or edits message            | Is Callback?4                 |                               |                                                                                                |
| Answer Callback4      | HTTP Request           | Sends callback query answer confirmation  | Is Callback?4                 |                               |                                                                                                |
| 📚 REAL EXAMPLES       | StickyNote             | Provides example menus and usage tips     |                               |                               | Contains real menu examples for quick copy/paste                                               |
| 🚀 QUICK START GUIDE1  | StickyNote             | Quick start instructions for setup        |                               |                               | Provides bot token setup, webhook, and testing instructions                                   |
| 📚 WORKFLOW ARCHITECTURE1 | StickyNote          | Explains workflow design and data flow    |                               |                               | Details modular design with menu config, router, handlers                                     |
| 💡 ADDING HANDLERS1    | StickyNote             | Guide to add custom handlers               |                               |                               | Step-by-step instructions for handler creation and router update                              |
| 📋 MENU CODE1          | StickyNote             | Menu config code comments overview         |                               |                               | Encourages reading the detailed comments inside Load Menu Config node                         |
| ⚙️ MENU SETUP GUIDE1    | StickyNote             | Complete guide on adding menus and actions |                               |                               | Stepwise instructions on extending menus and commands                                        |
| 🐛 TROUBLESHOOTING & TIPS1 | StickyNote          | Common issues and production tips          |                               |                               | Covers errors, debugging, and best practices                                                 |
| 🔀 Router Logic2       | StickyNote             | Describes Action Router outputs            |                               |                               | Lists all handler outputs and their roles                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**
   - Type: Telegram Trigger
   - Parameters: Listen to updates "message" and "callback_query"
   - Credentials: Set Telegram API credentials (bot token via n8n credentials)
   - Position: Start of workflow

2. **Create Load Menu Config node**
   - Type: Function
   - Paste the entire MENU_CONFIG JavaScript object defining all menus.
   - Add detailed comments explaining menu keys, text, and keyboard arrays.
   - Input: Connect from Telegram Trigger (main output)
   - Output: Pass original data with added `menuConfig`

3. **Create Extract Command node**
   - Type: Function
   - Implement logic to parse command from message text or callback data
   - Extract user info: userName, userId, chatId, messageId, isCallback, callbackQueryId
   - Input: Connect from Telegram Trigger (main output)
   - Output: JSON with extracted command and metadata

4. **Create Merge node**
   - Type: Merge (combine by position)
   - Inputs: Outputs from Load Menu Config and Extract Command
   - Output: Combined JSON with `menuConfig` and command data

5. **Create Process Command node**
   - Type: Function
   - Lookup menu by command key in `menuConfig`, fallback to `default`
   - Define `actionCommands` array listing commands that require handlers
   - Set `requiresAction` boolean
   - Input: Connect from Merge node
   - Output: JSON with `menu` and `requiresAction`

6. **Create Needs Action? node**
   - Type: If
   - Condition: `requiresAction` is true
   - Input: Connect from Process Command node
   - True Output: Connect to Action Router
   - False Output: Connect directly to Build Response node

7. **Create Action Router node**
   - Type: Switch
   - Outputs: 7
   - Expression routing:
     - Output 0: commands starting with `rate_`
     - Output 1: commands starting with `lang_`
     - Output 2: commands `analytics` or `report`
     - Output 3: commands `stats` or `achievements`
     - Output 4: commands `bug_report` or `feature_request`
     - Output 5: commands `notif` or `profile`
     - Output 6: default fallback for others
   - Input: True output from Needs Action? node

8. **Create Business Logic Handlers (7 nodes)**
   - Each node is a Function node processing commands routed to it:
     - Rating Handler: parse rating, return rating message
     - Language Handler: process language change
     - Analytics Handler: generate analytics or report data
     - Statistics Handler: generate user stats or achievements
     - Feedback Handler: create support ticket info
     - Settings Handler: load notification or profile settings
     - Default Handler: no action, empty customData
   - Connect each output of Action Router to corresponding handler
   - Each handler outputs to Build Response node

9. **Create Build Response node**
   - Type: Function
   - Replaces placeholders in menu text with user and custom data
   - Returns final `responseText` and `keyboard`
   - Inputs: From Needs Action? node (false output) and all handlers

10. **Create Prepare Telegram node**
    - Type: Function
    - Determines Telegram API method (`sendMessage` or `editMessageText`)
    - Constructs request body with chat_id, text, parse_mode HTML, inline_keyboard
    - Includes message_id if callback
    - Input: From Build Response node

11. **Create Set Bot Token node**
    - Type: Set
    - Assign string variable `botToken` with your actual Telegram bot token (replace placeholder)
    - Input: From Prepare Telegram node

12. **Create Is Callback? node**
    - Type: If
    - Checks if `isCallback` is true
    - Input: From Set Bot Token node
    - True output: Connect to Send to Telegram node and Answer Callback node
    - False output: Connect to Send to Telegram node only

13. **Create Send to Telegram node**
    - Type: HTTP Request
    - Method: POST
    - URL: `https://api.telegram.org/bot{{ $json.botToken }}/{{ $json.apiMethod }}`
    - JSON Body: `requestBody`
    - Input: From Is Callback? node (both outputs)

14. **Create Answer Callback node**
    - Type: HTTP Request
    - Method: POST
    - URL: `https://api.telegram.org/bot{{ $json.botToken }}/answerCallbackQuery`
    - JSON Body: JSON with callback_query_id and confirmation text "✅"
    - Input: From Is Callback? node (true output)

15. **Connect all nodes as per described flow:**
    - Telegram Trigger → Load Menu Config + Extract Command (parallel)
    - Merge → Process Command → Needs Action?
    - Needs Action? True → Action Router → Handlers → Build Response
    - Needs Action? False → Build Response
    - Build Response → Prepare Telegram → Set Bot Token → Is Callback?
    - Is Callback? True → Send to Telegram + Answer Callback
    - Is Callback? False → Send to Telegram

16. **Add Sticky Notes for Documentation**
    - Add sticky notes to provide user guides, architecture explanations, troubleshooting info, and real examples as per original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                 |
|-------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| The workflow architecture separates navigation (menu config), routing, and business logic handlers for modularity and clarity. | Sticky Note "📚 WORKFLOW ARCHITECTURE1"                         |
| Extensive guides to add new menus and actions, including code snippets and common mistakes to avoid.                          | Sticky Note "⚙️ MENU SETUP GUIDE1"                              |
| Troubleshooting tips covering common issues like webhook errors, token problems, and button callback_data mismatches.         | Sticky Note "🐛 TROUBLESHOOTING & TIPS1"                        |
| Instructions for adding custom handlers with detailed steps to extend routing and handler logic.                              | Sticky Note "💡 ADDING HANDLERS1"                               |
| Real menu examples illustrating multi-level menus, action buttons, and placeholders for dynamic text replacement.             | Sticky Note "📚 REAL EXAMPLES"                                  |
| Quick start guide to set bot token, webhook URL, and testing instructions.                                                    | Sticky Note "🚀 QUICK START GUIDE1"                             |

---

**Disclaimer:** This document describes an n8n workflow automating a Telegram bot menu system. All data handled are legal and public. The workflow complies with applicable content policies and contains no illegal or offensive elements.

---