tset test es tes

https://n8nworkflows.xyz/workflows/tset-test-es-tes-9373


# tset test es tes

### 1. Workflow Overview

This workflow, titled **"n8n Basics"**, is designed primarily to fetch inspirational quotes and programming jokes from public APIs and send them via Gmail either on-demand or on a schedule. It demonstrates basic n8n automation concepts such as HTTP requests, data transformation, scheduling triggers, and sending emails through Gmail. The workflow is structured into three main logical blocks:

- **1.1 Manual Quote Retrieval and Display:** Triggered manually, it fetches a random inspirational quote and formats the results for viewing.
- **1.2 Scheduled Quote Emailing:** Runs on a daily schedule, fetches a random inspirational quote, formats it, and sends it by Gmail.
- **1.3 Scheduled Programming Joke Retrieval and Mapping:** Runs on an annual schedule, fetches a programming joke, maps multiple fields for future use (e.g., email), but currently does not send it.

Supporting these blocks are multiple sticky notes with instructions and tips for setup and usage.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Quote Retrieval and Display

- **Overview:**  
  Allows the user to manually trigger the workflow to fetch an inspirational quote from the internet and display the quote with its author.

- **Nodes Involved:**  
  - `1. Click 'Execute workflow’` (Manual Trigger)  
  - `2. Get inspirational quote from the internet` (HTTP Request)  
  - `3. See the resutls` (Set)  
  - `Sticky Note` (Instructional note)

- **Node Details:**

  - **1. Click 'Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Initiates the workflow manually on demand.  
    - Configuration: No parameters; simply triggers the flow.  
    - Input: None  
    - Output: Triggers next node: `2. Get inspirational quote from the internet`  
    - Edge Cases: None; manual trigger depends on user.

  - **2. Get inspirational quote from the internet**  
    - Type: HTTP Request  
    - Role: Fetches a random inspirational quote.  
    - Configuration: URL set to `https://zenquotes.io/api/random`. No special headers or authentication.  
    - Input: Trigger from manual node.  
    - Output: JSON response with quote and author. Sent to `3. See the resutls`.  
    - Edge Cases: API downtime, invalid response, rate limiting.

  - **3. See the resutls**  
    - Type: Set  
    - Role: Extracts and formats the quote text and author from the API response.  
    - Configuration: Assigns two string fields — `quote` and `author` — from JSON properties `$json.q` and `$json.a` respectively.  
    - Input: Data from HTTP request node.  
    - Output: Processed data for viewing or further usage. No further node connected.  
    - Edge Cases: Missing fields in JSON; expression errors if API response format changes.

  - **Sticky Note**  
    - Purpose: Provides user instructions on how to execute and check results.  
    - Content Summary: Steps to run the workflow and verify the quote fetched.

#### 1.2 Scheduled Quote Emailing

- **Overview:**  
  Runs automatically daily at 7 AM, fetches a random quote, formats the data, and sends it via Gmail to a configured recipient.

- **Nodes Involved:**  
  - `Schedule me here` (Schedule Trigger)  
  - `Get quote` (HTTP Request)  
  - `Results` (Set)  
  - `Connect your Gmail` (Gmail node)  
  - `Sticky Note3` (Scheduling instructions)  
  - `Sticky Note15` (Empty note)  
  - `Sticky Note2` (Empty note)

- **Node Details:**

  - **Schedule me here**  
    - Type: Schedule Trigger  
    - Role: Initiates workflow execution daily at 7:00 AM.  
    - Configuration: Trigger set to hour 7 daily; no other constraints.  
    - Input: None  
    - Output: Triggers `Get quote`.  
    - Edge Cases: Timezone mismatches, scheduler misconfiguration.

  - **Get quote**  
    - Type: HTTP Request  
    - Role: Fetches a random quote similar to the manual flow.  
    - Configuration: URL `https://zenquotes.io/api/random`.  
    - Input: Schedule trigger.  
    - Output: JSON quote data to `Results`.  
    - Edge Cases: Same as manual HTTP request node.

  - **Results**  
    - Type: Set  
    - Role: Extracts and formats the quote and author data.  
    - Configuration: Assigns `quote` and `author` fields from `$json.q` and `$json.a`.  
    - Input: HTTP response.  
    - Output: Data passed to Gmail node.  
    - Edge Cases: Missing or malformed data.

  - **Connect your Gmail**  
    - Type: Gmail node (SMTP via OAuth2)  
    - Role: Sends an email with the quote and author.  
    - Configuration:  
      - Subject: "Daily quote"  
      - Email type: Plain text  
      - Message body uses expressions to insert `author` and `quote` fields.  
      - Credentials: Gmail OAuth2 account configured (`Gmail account 2`).  
    - Input: Formatted quote data.  
    - Output: None (end node).  
    - Edge Cases: Authentication failure, quota limits, email delivery failure.

  - **Sticky Note3**  
    - Purpose: User instructions to set schedule, connect Gmail, and test workflow.  
    - Content Summary: Step-by-step guidance for scheduling and setup.

  - **Sticky Note15** and **Sticky Note2**  
    - Empty or no significant content; possibly placeholders or formatting aids.

#### 1.3 Scheduled Programming Joke Retrieval and Mapping

- **Overview:**  
  Runs annually at 6 AM, fetches a programming joke, maps the joke along with empty quote and author fields for future usage (e.g., email), but does not send it currently.

- **Nodes Involved:**  
  - `Schedule Trigger` (Schedule Trigger)  
  - `Quote` (HTTP Request)  
  - `Programming joke` (HTTP Request)  
  - `Map the fields` (Set)  
  - `Sticky Note4` (Instructional note)  
  - `Sticky Note5` (Empty note)  
  - `Sticky Note13` (Gmail add note)

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Runs workflow annually at 6 AM.  
    - Configuration: Trigger set to hour 6, months interval 12 (once a year).  
    - Input: None  
    - Output: Triggers `Quote`.  
    - Edge Cases: Timezone issues, scheduler misconfiguration.

  - **Quote**  
    - Type: HTTP Request  
    - Role: Fetches a random inspirational quote (same API as before).  
    - Configuration: URL `https://zenquotes.io/api/random`.  
    - Input: Schedule Trigger.  
    - Output: Triggers `Programming joke`.  
    - Edge Cases: API failure.

  - **Programming joke**  
    - Type: HTTP Request  
    - Role: Fetches a programming joke from JokeAPI.  
    - Configuration: URL is dynamically set to:  
      `https://v2.jokeapi.dev/joke/Programming?blacklistFlags=nsfw,religious,political,racist,sexist,explicit&type=single`  
    - Input: Data from `Quote` node.  
    - Output: To `Map the fields`.  
    - Edge Cases: API downtime, no jokes available, JSON format changes.

  - **Map the fields**  
    - Type: Set  
    - Role: Prepares data fields `author`, `quote`, and `joke` for future use.  
    - Configuration: Assigns empty strings for `author` and `quote` and assigns `joke` but expression values are currently empty (`"="`), indicating placeholders or incomplete mappings.  
    - Input: Data from programming joke node.  
    - Output: No further nodes connected.  
    - Edge Cases: Missing joke text, expression errors.

  - **Sticky Note4**  
    - Purpose: Explains how to execute workflow to get a joke and suggests mapping results to emails.  
    - Content: Instructions for workflow usage and extension.

  - **Sticky Note5** and **Sticky Note13**  
    - `Sticky Note5` is empty.  
    - `Sticky Note13` prompts user to add Gmail node (+ button), indicating this part is incomplete or for future enhancement.

---

### 3. Summary Table

| Node Name                                | Node Type           | Functional Role                          | Input Node(s)                                  | Output Node(s)                | Sticky Note                                                                                      |
|-----------------------------------------|---------------------|----------------------------------------|-----------------------------------------------|------------------------------|------------------------------------------------------------------------------------------------|
| Sticky Note3                            | Sticky Note         | Scheduling setup instructions          | None                                          | None                         | ## ... now let's schedule it ⏰ 1. Double-click “Schedule” to set when it runs 2. Connect your Gmail and set the recepient 3. To test, click “Execute workflow” on the "Schedule" node |
| Sticky Note2                            | Sticky Note         | Empty / placeholder                    | None                                          | None                         |                                                                                                |
| Schedule me here                       | Schedule Trigger    | Triggers daily quote fetching          | None                                          | Get quote                    |                                                                                                |
| Results                                | Set                 | Formats quote and author fields        | Get quote                                      | Connect your Gmail           |                                                                                                |
| Get quote                             | HTTP Request        | Fetches random inspirational quote    | Schedule me here                              | Results                      |                                                                                                |
| Connect your Gmail                     | Gmail               | Sends email with quote                  | Results                                        | None                         | Double-click here to connect!                                                                   |
| Sticky Note15                         | Sticky Note         | Empty / placeholder                    | None                                          | None                         |                                                                                                |
| Sticky Note                           | Sticky Note         | Manual workflow instructions          | None                                          | None                         | ## 🚀🚀 Start here 🚀🚀 1. Click “Execute workflow” to run it 2. This gets a random quote from the internet 3. Double-click the last node to check what quote you got |
| 1. Click 'Execute workflow’            | Manual Trigger      | Manual trigger to start quote process | None                                          | 2. Get inspirational quote from the internet |                                                                                                |
| 2. Get inspirational quote from the internet | HTTP Request        | Fetches random quote                   | 1. Click 'Execute workflow’                     | 3. See the resutls           |                                                                                                |
| 3. See the resutls                     | Set                 | Formats quote and author fields        | 2. Get inspirational quote from the internet | None                         | Double-click to open!                                                                           |
| Sticky Note6                         | Sticky Note         | Empty / placeholder                    | None                                          | None                         |                                                                                                |
| Sticky Note4                         | Sticky Note         | Instructions for programming joke block | None                                          | None                         | ## ❓ How about a joke... 1. Execute this workflow 2. Check out the new "Programming Joke" 3. Try mapping the new results and add them to your email |
| Sticky Note5                         | Sticky Note         | Empty / placeholder                    | None                                          | None                         |                                                                                                |
| Quote                                | HTTP Request        | Fetches inspirational quote (annual) | Schedule Trigger                               | Programming joke             |                                                                                                |
| Schedule Trigger                     | Schedule Trigger    | Annual trigger at 6 AM                 | None                                          | Quote                        |                                                                                                |
| Map the fields                      | Set                 | Maps joke, quote, author fields        | Programming joke                               | None                         | Double-click to set!                                                                            |
| Programming joke                   | HTTP Request        | Fetches programming joke               | Quote                                          | Map the fields               | Check out after running!                                                                        |
| Sticky Note1                       | Sticky Note         | YouTube video reference                | None                                          | None                         | @[youtube](dKTcAfBfFLU)                                                                        |
| Sticky Note8                       | Sticky Note         | YouTube video reference                | None                                          | None                         | @[youtube](LdE0KnhRtsY)                                                                        |
| Sticky Note7                       | Sticky Note         | YouTube video reference                | None                                          | None                         | @[youtube](5CBUXMO_L2Y)                                                                        |
| Sticky Note9                       | Sticky Note         | Step 1 heading                         | None                                          | None                         | # Step 1                                                                                       |
| Sticky Note10                      | Sticky Note         | Step 2 heading                         | None                                          | None                         | # Step 2                                                                                       |
| Sticky Note11                      | Sticky Note         | Step 3 heading                         | None                                          | None                         | # Step 3                                                                                       |
| Sticky Note13                      | Sticky Note         | Gmail node add prompt                  | None                                          | None                         | ### Click + to add Gmail                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: `1. Click 'Execute workflow’`  
   - Type: Manual Trigger  
   - Purpose: To manually start the workflow.

2. **Create HTTP Request Node for Quote:**  
   - Name: `2. Get inspirational quote from the internet`  
   - Type: HTTP Request  
   - Parameters:  
     - URL: `https://zenquotes.io/api/random`  
     - Method: GET (default)  
   - Connect output of Manual Trigger to this node.

3. **Create Set Node to Format Quote:**  
   - Name: `3. See the resutls`  
   - Type: Set  
   - Parameters:  
     - Assign `quote` = `{{$json.q}}`  
     - Assign `author` = `{{$json.a}}`  
   - Connect output of HTTP Request node to this node.

4. **Create Schedule Trigger Node:**  
   - Name: `Schedule me here`  
   - Type: Schedule Trigger  
   - Parameters:  
     - Set to trigger daily at 7:00 AM (hour=7)  
   - No input connection (trigger node).

5. **Create HTTP Request Node for Scheduled Quote:**  
   - Name: `Get quote`  
   - Type: HTTP Request  
   - Parameters:  
     - URL: `https://zenquotes.io/api/random`  
   - Connect output of `Schedule me here` to this node.

6. **Create Set Node for Scheduled Quote:**  
   - Name: `Results`  
   - Type: Set  
   - Parameters:  
     - Assign `quote` = `{{$json.q}}`  
     - Assign `author` = `{{$json.a}}`  
   - Connect output of `Get quote` to this node.

7. **Create Gmail Node to Send Email:**  
   - Name: `Connect your Gmail`  
   - Type: Gmail  
   - Parameters:  
     - Subject: `Daily quote`  
     - Email Type: Text  
     - Message:  
       ```
       Hey,

       Here's a quote by {{ $json.author }}:

       "{{ $json.quote }}"
       ```  
   - Credentials: Connect your Gmail account using OAuth2 credentials setup in n8n (e.g., "Gmail account 2").  
   - Connect output of `Results` to this node.

8. **Create Annual Schedule Trigger:**  
   - Name: `Schedule Trigger`  
   - Type: Schedule Trigger  
   - Parameters:  
     - Set to trigger at 6:00 AM, every 12 months (annual)  
   - No input connection.

9. **Create HTTP Request Node for Annual Quote:**  
   - Name: `Quote`  
   - Type: HTTP Request  
   - Parameters:  
     - URL: `https://zenquotes.io/api/random`  
   - Connect output of `Schedule Trigger` to this node.

10. **Create HTTP Request Node for Programming Joke:**  
    - Name: `Programming joke`  
    - Type: HTTP Request  
    - Parameters:  
      - URL (set expression):  
        `https://v2.jokeapi.dev/joke/Programming?blacklistFlags=nsfw,religious,political,racist,sexist,explicit&type=single`  
    - Connect output of `Quote` to this node.

11. **Create Set Node to Map Fields:**  
    - Name: `Map the fields`  
    - Type: Set  
    - Parameters:  
      - Assign fields for future use:  
        - `author` = `""` (empty string)  
        - `quote` = `""` (empty string)  
        - `joke` = `{{$json.joke}}` (or appropriate field if JSON structure differs)  
    - Connect output of `Programming joke` to this node.

12. **Add Sticky Notes:**  
    - Add notes with instructions for scheduling, Gmail connection, manual execution, and recommendations for extending the workflow.  
    - Add YouTube video resource links as sticky notes at convenient positions.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                         |
|----------------------------------------------------------------------------------------------|---------------------------------------|
| Video reference for learning n8n basics                                                     | @[youtube](dKTcAfBfFLU)                |
| Additional tutorial video                                                                    | @[youtube](LdE0KnhRtsY)                |
| Another instructional video                                                                  | @[youtube](5CBUXMO_L2Y)                |
| Workflow instructions for scheduling and Gmail setup                                        | Sticky Note3 content in workflow       |
| Encouragement to map programming joke results and add email sending functionality           | Sticky Note4 content                    |
| Reminder to add Gmail credentials and node for full email functionality                      | Sticky Note13 content                   |

---

**Disclaimer**: The provided content is derived solely from an n8n automated workflow. It complies fully with content policies and does not include any illegal or protected material. All data utilized comes from legal, public sources.