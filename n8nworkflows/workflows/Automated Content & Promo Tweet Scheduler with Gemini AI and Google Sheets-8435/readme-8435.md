Automated Content & Promo Tweet Scheduler with Gemini AI and Google Sheets

https://n8nworkflows.xyz/workflows/automated-content---promo-tweet-scheduler-with-gemini-ai-and-google-sheets-8435


# Automated Content & Promo Tweet Scheduler with Gemini AI and Google Sheets

### 1. Workflow Overview

This workflow automates the generation and scheduling of Twitter content by leveraging Google Sheets as a data source and the Gemini AI model for tweet creation. It targets users who want to maintain an active Twitter presence with a mix of content-driven and promotional tweets, ensuring uniqueness and adherence to defined content strategies.

The workflow consists of the following logical blocks:

- **1.1 Trigger and Timing Control:** Manages periodic execution and introduces a randomized delay to avoid posting at fixed times.
- **1.2 Content Type Selection:** Randomly selects between promotional tweets and content strategy templates, ensuring varied tweet types.
- **1.3 Tweet Generation:** Uses AI to create unique tweets based on selected templates or promotional content, checking against past tweets to avoid duplication.
- **1.4 Database Interaction:** Reads from and writes to Google Sheets to check past tweets and log newly generated tweets for future reference.
- **1.5 Tweet Posting:** Sends the generated tweet to Twitter via OAuth2 authentication.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Timing Control

- **Overview:** This block triggers the workflow every two hours and optionally delays execution by a random number of minutes (disabled by default) to stagger tweet posting times.
- **Nodes Involved:** `Schedule Trigger`, `Time randomizer` (disabled), `Code`
  
**Node Details:**

- **Schedule Trigger**
  - Type: Schedule Trigger
  - Configuration: Interval set to trigger every 2 hours.
  - Inputs: None (start node)
  - Outputs: Connects to `Time randomizer`.
  - Failure Modes: Scheduling misconfiguration or internal n8n scheduler failures.

- **Time randomizer** (disabled)
  - Type: Code
  - Configuration: Generates a random delay between 0 and 120 minutes (2 hours) by setting a timeout before proceeding.
  - Inputs: from `Schedule Trigger`
  - Outputs: connects to `Code` node.
  - Failure Modes: Execution timeout or JavaScript errors.
  - Disabled, so no effect in the current workflow.

- **Code**
  - Type: Code (JavaScript)
  - Configuration:
    - Defines a list of 10 tweet content templates.
    - Implements logic to randomly select a "tweet type": 80% chance for a content template, 20% for a promotional ad.
    - Prevents immediate repetition of the same tweet type using a global variable.
  - Inputs: from `Time randomizer`
  - Outputs: provides `{tweet: <type>, ads: boolean}` to `If` node.
  - Edge Cases: Global variable persistence might fail if the workflow is restarted; randomness could cause rare repeats.
  - Failure Modes: Expression errors or global variable issues.

---

#### 2.2 Content Type Routing

- **Overview:** Routes the workflow based on whether the selected tweet type is promotional or content-template based.
- **Nodes Involved:** `If`
  
**Node Details:**

- **If**
  - Type: If (Conditional)
  - Configuration: Checks if the `ads` boolean from `Code` node is true.
  - Inputs: from `Code`
  - Outputs: 
    - True branch → `Promotional Tweet maker`
    - False branch → `Tweet maker`
  - Failure Modes: Expression evaluation errors.

---

#### 2.3 Tweet Generation

- **Overview:** Generates tweets using AI models based on the selected content type, ensuring uniqueness by consulting the tweet logs.
- **Nodes Involved:** `Tweet maker`, `Promotional Tweet maker`, `Google Gemini Chat Model`

**Node Details:**

- **Tweet maker**
  - Type: Langchain Agent (AI content generator)
  - Configuration:
    - Input: Template type from `Code` node.
    - System instructions: Generate a short, punchy tweet using the input template.
    - Reads past tweets from Google Sheets to avoid duplicates.
    - Writes the new tweet to the past tweets column for logging.
    - Output: The generated tweet text.
  - Inputs: from `If` (false branch), and AI language model node.
  - Outputs: connects to `Tweet` node.
  - Edge Cases: AI generation failure, API limits, duplicate detection logic failure.
  - Requires: Google Sheets OAuth2 credentials, Gemini AI credentials.

- **Promotional Tweet maker**
  - Type: Langchain Agent
  - Configuration:
    - Input: Promotional ad data fetched from Google Sheets.
    - Instructions: Generate a unique, punchy promotional tweet.
    - Checks for duplicates using the database.
    - Updates last posted date in Google Sheets.
    - Output: The generated promotional tweet text.
  - Inputs: from `If` (true branch), reads from promo Google Sheets.
  - Outputs: connects to `Tweet` node.
  - Edge Cases: AI failures, sheet access errors, duplicate detection failures.
  - Requires: Google Sheets OAuth2, Gemini AI credentials.

- **Google Gemini Chat Model**
  - Type: AI Language Model
  - Configuration: Used as the AI engine for both `Tweet maker` and `Promotional Tweet maker`.
  - Inputs: from both agent nodes.
  - Outputs: back to both agent nodes.
  - Requires: Google Palm API credentials.

---

#### 2.4 Database Interaction

- **Overview:** Reads past tweets and promo data from Google Sheets and logs newly generated tweets for tracking and uniqueness checks.
- **Nodes Involved:** `read database`, `log database`, `read database1`, `log database1`

**Node Details:**

- **read database**
  - Type: Google Sheets Tool (Read)
  - Configuration: Reads the main posts sheet (gid=0) for past tweets.
  - Inputs: from AI `Tweet maker` (as a database tool).
  - Outputs: data to `Tweet maker` for duplicate checks.
  - Failure Modes: OAuth errors, rate limits, invalid sheet ID.

- **log database**
  - Type: Google Sheets Tool (Append)
  - Configuration: Appends new tweets with current date to posts sheet.
  - Inputs: from `Tweet maker` (AI output).
  - Outputs: none forward.
  - Failure Modes: OAuth errors, write permission issues.

- **read database1**
  - Type: Google Sheets Tool (Read)
  - Configuration: Reads the promotional content sheet (gid=814034323).
  - Inputs: from `Promotional Tweet maker` (database tool).
  - Outputs: promo data to `Promotional Tweet maker`.
  - Failure Modes: Same as `read database`.

- **log database1**
  - Type: Google Sheets Tool (Update)
  - Configuration: Updates promo sheet with last posted date for the used promo item.
  - Inputs: from `Promotional Tweet maker`.
  - Outputs: none forward.
  - Failure Modes: OAuth or write errors.

---

#### 2.5 Tweet Posting

- **Overview:** Prepares and sends the generated tweet to Twitter.
- **Nodes Involved:** `Tweet`, `Creates the tweet`, `Creates the tweet` is the Twitter node.

**Node Details:**

- **Tweet**
  - Type: Set
  - Configuration: Assigns the AI-generated tweet text to a variable `Tweet`.
  - Inputs: from both tweet generators.
  - Outputs: to Twitter node.
  - Failure Modes: None significant (data mapping).

- **Creates the tweet**
  - Type: Twitter node (OAuth2)
  - Configuration: Posts the tweet using the OAuth2 credentials linked to the Twitter account.
  - Inputs: from `Tweet`
  - Outputs: none forward.
  - Failure Modes: Twitter API rate limits, OAuth token expiration, network errors.

---

### 3. Summary Table

| Node Name             | Node Type                          | Functional Role                          | Input Node(s)                | Output Node(s)              | Sticky Note                                                  |
|-----------------------|----------------------------------|----------------------------------------|-----------------------------|-----------------------------|--------------------------------------------------------------|
| Schedule Trigger      | Schedule Trigger                 | Initiates workflow every 2 hours       | None                        | Time randomizer             | TRIGGER                                                      |
| Time randomizer       | Code (disabled)                  | Random delay before next steps         | Schedule Trigger             | Code                        | Generates a random number between 0–120 and waits for that many minutes, allowing tweets to be posted at pseudo-random times throughout the day instead of fixed scheduled times |
| Code                  | Code                            | Selects tweet type (content or promo)  | Time randomizer              | If                          | Randomly selects one of the 10 content strategy templates or one of the 4 promotional ads, with an 80% chance of picking a template and a 20% chance of picking a promo |
| If                    | If                             | Routes to promo or content tweet maker | Code                        | Promotional Tweet maker, Tweet maker | Checks whether the randomly selected item is a content strategy template or a promotional ad, then routes it accordingly |
| Tweet maker           | Langchain Agent                 | Generates content tweets                | If (false)                  | Tweet                       | Generates a unique tweet while checking the database to avoid repetition, and logs the newly created tweet into the database |
| Promotional Tweet maker| Langchain Agent                 | Generates promotional tweets            | If (true)                   | Tweet                       | Generates a promotional tweet based on the provided ad template, ensuring it follows the ad’s structure and messaging |
| Google Gemini Chat Model | AI Language Model               | Provides AI processing for tweet makers| Tweet maker, Promotional Tweet maker | Tweet maker, Promotional Tweet maker |                                                              |
| read database         | Google Sheets Tool (Read)       | Reads past tweets from main sheet      | Tweet maker (as AI tool)    | Tweet maker                 |                                                              |
| log database          | Google Sheets Tool (Append)     | Logs new tweets to main sheet           | Tweet maker (AI tool)       | None                       |                                                              |
| read database1        | Google Sheets Tool (Read)       | Reads promotional data from promo sheet| Promotional Tweet maker (AI tool) | Promotional Tweet maker      |                                                              |
| log database1         | Google Sheets Tool (Update)     | Updates promo sheet with last posted   | Promotional Tweet maker (AI tool) | None                       |                                                              |
| Tweet                 | Set                            | Prepares tweet text for posting         | Tweet maker, Promotional Tweet maker | Creates the tweet          |                                                              |
| Creates the tweet     | Twitter                        | Posts the tweet to Twitter               | Tweet                       | None                       |                                                              |
| Sticky Note           | Sticky Note                    | Visual annotation                       | None                        | None                       | TRIGGER                                                      |
| Sticky Note1          | Sticky Note                    | Visual annotation                       | None                        | None                       | Triggers at 8am,12pm,6pm                                     |
| Sticky Note2          | Sticky Note                    | Visual annotation                       | None                        | None                       | Explains random delay node                                   |
| Sticky Note3          | Sticky Note                    | Visual annotation                       | None                        | None                       | Explains tweet type selection logic                          |
| Sticky Note4          | Sticky Note                    | Visual annotation                       | None                        | None                       | Explains routing logic by tweet type                         |
| Sticky Note5          | Sticky Note                    | Visual annotation                       | None                        | None                       | Describes promotional tweet generation                       |
| Sticky Note6          | Sticky Note                    | Visual annotation                       | None                        | None                       | Describes content tweet generation and logging              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create `Schedule Trigger` node:**
   - Type: Schedule Trigger
   - Set interval to every 2 hours.
   - Position: start node.

2. **Create `Time randomizer` node (optional):**
   - Type: Code
   - JavaScript code to generate a random delay between 0 and 120 minutes, then `setTimeout` for that duration.
   - Connect `Schedule Trigger` output to this node.
   - (Note: This node is disabled by default; enable if random posting times are desired.)

3. **Create `Code` node:**
   - Type: Code
   - JavaScript to select randomly between 10 content templates and promotional ads with 80/20 probability.
   - Prevent immediate repeats using a global variable.
   - Connect previous node (Time randomizer or Schedule Trigger if randomizer disabled) output to this node.

4. **Create `If` node:**
   - Type: If
   - Condition: check if `ads` field from `Code` node is true.
   - Connect `Code` node output to this node.

5. **Create `Tweet maker` node:**
   - Type: Langchain Agent
   - Set input text to `={{ $json.tweet }}` from `Code` node.
   - System message: instruct AI to generate short, punchy tweets based on templates.
   - Enable reading from the main tweets Google Sheet for duplicate checking.
   - Enable logging new tweets in the sheet.
   - Connect `If` node false output to this node.

6. **Create `Promotional Tweet maker` node:**
   - Type: Langchain Agent
   - Input: prompt instructing to pick promotional content from promo Google Sheet.
   - System message: generate unique, punchy promotional tweets.
   - Enable reading from promo Google Sheet and logging last posted date.
   - Connect `If` node true output to this node.

7. **Create `Google Gemini Chat Model` node:**
   - Type: AI Language Model (Gemini)
   - Configure with Google Palm API credentials.
   - Connect both `Tweet maker` and `Promotional Tweet maker` nodes to this node as their AI language model.

8. **Create `read database` node:**
   - Type: Google Sheets Tool (Read)
   - Configure to read the main posts sheet (gid=0).
   - Set credentials with Google Sheets OAuth2.
   - Connect as a database tool input to `Tweet maker`.

9. **Create `log database` node:**
   - Type: Google Sheets Tool (Append)
   - Configure to append new tweets with current date to main posts sheet.
   - Use Google Sheets OAuth2 credentials.
   - Connect as a database tool output from `Tweet maker`.

10. **Create `read database1` node:**
    - Type: Google Sheets Tool (Read)
    - Configure to read the promo sheet (gid=814034323).
    - Use Google Sheets OAuth2 credentials.
    - Connect as a database tool input to `Promotional Tweet maker`.

11. **Create `log database1` node:**
    - Type: Google Sheets Tool (Update)
    - Configure to update the `last_posted` column for the used promo in promo sheet.
    - Use Google Sheets OAuth2 credentials.
    - Connect as a database tool output from `Promotional Tweet maker`.

12. **Create `Tweet` node:**
    - Type: Set
    - Assign the generated tweet text under the variable `Tweet`.
    - Connect output from both tweet makers to this node.

13. **Create `Creates the tweet` node:**
    - Type: Twitter node (OAuth2)
    - Configure OAuth2 credentials for Twitter account (X account).
    - Set text to `={{ $json.Tweet }}`.
    - Connect output from `Tweet` node.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The workflow triggers three times per day at 8am, 12pm, and 6pm, but the actual Schedule Trigger runs every 2 hours. | Sticky Note1                                                                                         |
| Randomized delay node is disabled but can be enabled to post tweets at pseudo-random times within 2-hour windows | Sticky Note2                                                                                         |
| Tweet selection logic ensures 80% content strategy tweets and 20% promotional tweets for balanced engagement  | Sticky Note3                                                                                         |
| The `If` node routes tweets properly to either content or promo generation blocks                             | Sticky Note4                                                                                         |
| Promotional tweets are generated uniquely with constraints based on promotional data in Google Sheets        | Sticky Note5                                                                                         |
| Content tweets are generated uniquely and logged to avoid duplicates                                         | Sticky Note6                                                                                         |

---

**Disclaimer:** The content processed by this workflow is fully compliant with all applicable policies and contains no illegal or offensive material. Data processed is legal and publicly available.