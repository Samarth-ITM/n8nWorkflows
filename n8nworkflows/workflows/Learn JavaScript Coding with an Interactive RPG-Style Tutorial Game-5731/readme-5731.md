Learn JavaScript Coding with an Interactive RPG-Style Tutorial Game

https://n8nworkflows.xyz/workflows/learn-javascript-coding-with-an-interactive-rpg-style-tutorial-game-5731


# Learn JavaScript Coding with an Interactive RPG-Style Tutorial Game

### 1. Workflow Overview

This n8n workflow implements an **interactive RPG-style coding tutorial game** designed to teach JavaScript coding and n8n workflow concepts through a progressive series of challenges. The game simulates an adventure where players complete coding tasks, earn experience points (XP), collect power-ups, and unlock achievements. The workflow includes three main levels culminating in a final boss battle and a game over summary screen.

The logic is grouped into these blocks:

- **1.1 Input Reception and Initialization:** Triggering the game start and setting up the initial player profile and game state.
- **1.2 Level 1: Data Warrior Challenge:** A JavaScript coding challenge focused on data deduplication.
- **1.3 Level 2: API Ninja Challenge:** A challenge involving data transformation and validation of API responses.
- **1.4 Final Boss: Automation Master Challenge:** An advanced challenge simulating a complete workflow system with task prioritization and error handling.
- **1.5 Game Over Summary:** Displaying final results, achievements, motivational messages, and coaching/consulting offers.
- **1.6 Informational Sticky Notes:** Provide game instructions, level descriptions, and feature overviews.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

- **Overview:** Handles manual trigger to start the game and initializes the player profile and game state.
- **Nodes Involved:**  
  - 🎯 Start Game  
  - Initialize Game  

- **Node Details:**

  - **🎯 Start Game**  
    - Type: Manual Trigger  
    - Role: Entry point for the workflow; starts the game on manual trigger.  
    - Configuration: No parameters needed; user manually triggers this node.  
    - Inputs: None (start node)  
    - Outputs: Connects to "Initialize Game" node.  
    - Failures: None expected; manual trigger reliability depends on user action.  
  
  - **Initialize Game**  
    - Type: Code Node (JavaScript)  
    - Role: Creates initial player profile and game state with default stats, levels, and challenges.  
    - Configuration: Defines a player object (name, level, XP, health, inventory, achievements), game state (current level, total levels, challenge descriptions), and game introduction text.  
    - Key variables: `player`, `gameState`, `gameIntro` returned in output JSON.  
    - Inputs: Receives trigger from "🎯 Start Game" node.  
    - Outputs: Passes initialized game data to Level 1 node.  
    - Edge Cases: None critical; static initialization.  
    - Version: Uses Code Node version 2 syntax.  

---

#### 2.2 Level 1: Data Warrior Challenge

- **Overview:** Implements the first coding challenge where the player must deduplicate a corrupted user database using JavaScript array filtering.
- **Nodes Involved:**  
  - 🎲 Level 1: Data Warrior  
  - Level 1 Info (sticky note)

- **Node Details:**

  - **🎲 Level 1: Data Warrior**  
    - Type: Code Node (JavaScript)  
    - Role: Performs data deduplication on a mock corrupted user list and updates player stats and rewards accordingly.  
    - Configuration: Contains embedded JS code that filters duplicates by email, checks success, updates XP (+100), level, inventory (adds "Data Shield 🛡️"), and achievements ("Data Warrior").  
    - Key expressions: Uses `.filter()` and `.findIndex()` for deduplication; conditional logic for success.  
    - Inputs: Receives game state and player data from "Initialize Game".  
    - Outputs: Challenge result JSON including updated player state and next level indicator.  
    - Edge Cases: If no duplicates removed, player fails challenge and remains at level 1.  
    - Version: Code Node version 2.  
    - Sticky Note: Level 1 Info node describes challenge, skills learned, and rewards.  

---

#### 2.3 Level 2: API Ninja Challenge

- **Overview:** The second challenge focuses on transforming and validating API responses, requiring data cleaning and validation logic.
- **Nodes Involved:**  
  - ⚔️ Level 2: API Ninja  
  - Level 2 Info (sticky note)

- **Node Details:**

  - **⚔️ Level 2: API Ninja**  
    - Type: Code Node (JavaScript)  
    - Role: Processes a mock API response with inconsistent data, normalizes fields, validates entries (name, email, age), and updates player stats and rewards (200 XP, "Speed Boost ⚡" power-up).  
    - Configuration: Uses `.map()` for transformation and validation logic; accumulates errors in user objects.  
    - Key variables: `transformedUsers`, `validUsers`, `invalidUsers`, success threshold (≥3 valid users).  
    - Inputs: Receives updated player data from Level 1 node.  
    - Outputs: JSON with challenge results, updated player, and next level indicator.  
    - Edge Cases: Validation failure if users do not meet criteria; challenge failure keeps player at level 2.  
    - Version: Code Node version 2.  
    - Sticky Note: Level 2 Info node describes challenge details and rewards.  

---

#### 2.4 Final Boss: Automation Master Challenge

- **Overview:** The final, expert-level challenge simulates processing a complex workflow system with tasks prioritized and error handling, testing advanced n8n skills.
- **Nodes Involved:**  
  - 🏆 Final Boss: Automation Master  
  - Boss Info (sticky note)

- **Node Details:**

  - **🏆 Final Boss: Automation Master**  
    - Type: Code Node (JavaScript)  
    - Role: Sorts tasks by priority, processes each with simulated completion and error handling, computes workflow summary and efficiency, and updates player rewards (500 XP, "Master Badge 🏅") and final title.  
    - Configuration: Implements sorting (`priorityOrder`), try-catch error handling per task, aggregation of results and errors, success criteria (100% completion, ≥80% efficiency).  
    - Key variables: `workflowResult`, `summary`, `isSuccess`, `finalScore`, `finalTitle`.  
    - Inputs: Receives data from Level 2 node.  
    - Outputs: Challenge results with updated player and game completion flag, and prompt to play again.  
    - Edge Cases: Errors during task processing captured; overall failure if criteria unmet.  
    - Version: Code Node version 2.  
    - Sticky Note: Boss Info node details challenge and rewards.  

---

#### 2.5 Game Over Summary

- **Overview:** Presents a comprehensive game summary including achievements, performance metrics, motivational messages, and coaching/consulting contact info.
- **Nodes Involved:**  
  - 🎊 Game Over Screen  
  - Game Header (sticky note)  
  - Game Features (sticky note)

- **Node Details:**

  - **🎊 Game Over Screen**  
    - Type: Code Node (JavaScript)  
    - Role: Generates final player summary, achievement showcase, performance report, special rewards, and motivational messages based on completion and XP. Provides contact info for coaching and consulting.  
    - Configuration: Aggregates player data, computes completion rate, lists skills learned and next steps, and conditionally adds special rewards.  
    - Key variables: `achievementShowcase`, `performanceReport`, `motivationalMessage`, `specialRewards`, `gameSummary`.  
    - Inputs: Receives final game and player state from Final Boss node.  
    - Outputs: JSON with complete game summary and replay prompt.  
    - Edge Cases: Handles incomplete games gracefully with encouraging messages.  
    - Version: Code Node version 2.  
    - Sticky Notes:  
      - Game Header provides game title, rules, coaching and consulting email links.  
      - Game Features outlines game mechanics, rewards, learning outcomes, and fun elements.  

---

### 3. Summary Table

| Node Name                            | Node Type           | Functional Role                      | Input Node(s)            | Output Node(s)                    | Sticky Note                                                                                                 |
|------------------------------------|---------------------|-----------------------------------|--------------------------|---------------------------------|-------------------------------------------------------------------------------------------------------------|
| 🎯 Start Game                      | Manual Trigger      | Entry point to start the game     | None                     | Initialize Game                 |                                                                                                             |
| Initialize Game                   | Code Node           | Initializes player and game state | 🎯 Start Game             | 🎲 Level 1: Data Warrior         |                                                                                                             |
| 🎲 Level 1: Data Warrior          | Code Node           | Level 1 coding challenge           | Initialize Game           | ⚔️ Level 2: API Ninja           | Level 1 Info: Challenge description, skills, rewards.                                                      |
| ⚔️ Level 2: API Ninja             | Code Node           | Level 2 coding challenge           | 🎲 Level 1: Data Warrior   | 🏆 Final Boss: Automation Master | Level 2 Info: Challenge details and rewards.                                                              |
| 🏆 Final Boss: Automation Master  | Code Node           | Final challenge with workflow logic| ⚔️ Level 2: API Ninja      | 🎊 Game Over Screen             | Boss Info: Final boss challenge description and rewards.                                                   |
| 🎊 Game Over Screen               | Code Node           | Displays final game summary        | 🏆 Final Boss: Automation Master | None                          | Game Header: Game rules, author, coaching and consulting emails; Game Features: Game mechanics and rewards. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node:**  
   - Name: 🎯 Start Game  
   - Purpose: Entry point to start the game manually. No parameters needed.

2. **Create a Code Node for game initialization:**  
   - Name: Initialize Game  
   - Paste JavaScript code that creates `player` profile, `gameState` with challenges, and `gameIntro` message.  
   - Output JSON includes player, gameState, and message.  
   - Connect 🎯 Start Game → Initialize Game.

3. **Create Code Node for Level 1 challenge:**  
   - Name: 🎲 Level 1: Data Warrior  
   - Paste JavaScript code that:  
     - Deduplicates a mock corrupted user database by email.  
     - Updates player XP (+100), level, inventory with "Data Shield 🛡️", and achievements.  
     - Sets nextLevel to 2 on success; otherwise 1.  
   - Connect Initialize Game → 🎲 Level 1: Data Warrior.

4. **Add Sticky Note for Level 1:**  
   - Name: Level 1 Info  
   - Content describing challenge, skills learned, and rewards.  
   - Position near Level 1 node for reference.

5. **Create Code Node for Level 2 challenge:**  
   - Name: ⚔️ Level 2: API Ninja  
   - Paste JavaScript code that:  
     - Transforms and validates API user data (name trimming, email lowercase, age parsing).  
     - Validates fields with error collection.  
     - Updates player XP (+200), inventory with "Speed Boost ⚡", and achievements.  
     - Sets nextLevel to 3 on success; otherwise 2.  
   - Connect 🎲 Level 1: Data Warrior → ⚔️ Level 2: API Ninja.

6. **Add Sticky Note for Level 2:**  
   - Name: Level 2 Info  
   - Content describing challenge, skills, and rewards.  
   - Position near Level 2 node.

7. **Create Code Node for Final Boss challenge:**  
   - Name: 🏆 Final Boss: Automation Master  
   - Paste JavaScript code that:  
     - Processes a predefined set of tasks sorted by priority.  
     - Simulates processing with error handling and timing.  
     - Computes summary and efficiency, checks success criteria (100% completion, ≥80% efficiency).  
     - Updates player XP (+500), inventory with "Master Badge 🏅", achievements, and title.  
     - Sets gameComplete flag and next steps.  
   - Connect ⚔️ Level 2: API Ninja → 🏆 Final Boss: Automation Master.

8. **Add Sticky Note for Final Boss:**  
   - Name: Boss Info  
   - Content describing final challenge and rewards.  
   - Position near Final Boss node.

9. **Create Code Node for Game Over screen:**  
   - Name: 🎊 Game Over Screen  
   - Paste JavaScript code that:  
     - Builds final achievement showcase and performance report.  
     - Generates motivational messages based on completion.  
     - Lists skills learned, next steps, and special rewards.  
     - Includes coaching and consulting contact info with email links.  
   - Connect 🏆 Final Boss: Automation Master → 🎊 Game Over Screen.

10. **Add Sticky Notes for game overview and features:**  
    - Name: Game Header  
    - Content includes game title, author, game rules, coaching and consulting email links.  
    - Position near workflow start.  

    - Name: Game Features  
    - Content lists game mechanics, rewards, learning outcomes, and fun elements.  
    - Position prominently for user orientation.

11. **Set execution order:** Confirm nodes connect as:  
    🎯 Start Game → Initialize Game → 🎲 Level 1 → ⚔️ Level 2 → 🏆 Final Boss → 🎊 Game Over Screen.

12. **Credentials:** None required since all logic is internal JavaScript and manual trigger.

13. **Version:** Use Code Node version 2 for all JavaScript nodes to ensure compatibility with modern JS features.

---

### 5. General Notes & Resources

| Note Content                                                                                                            | Context or Link                                      |
|-------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| The game uses RPG-style progression and achievement badges to motivate learning JavaScript and n8n workflow concepts. | Game design rationale                               |
| Coaching and consulting offers with direct mailto links: david@daexai.com                                                | Coaching/consulting contact emails                   |
| Game rules emphasize practical coding challenges with rewards and power-ups to enhance engagement.                      | Game Header sticky note                              |
| Features include progressive difficulty, real n8n scenarios, and performance tracking for effective learning.           | Game Features sticky note                            |
| The workflow is not active by default; manual start is required to begin the tutorial game.                              | Workflow settings                                    |

---

**Disclaimer:** This documentation is based exclusively on the provided n8n workflow JSON. It complies with all content policies and contains no illegal or offensive material. All data processed is legal and publicly accessible.