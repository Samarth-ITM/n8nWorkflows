Learn n8n Basics in 3 Easy Steps ✨

https://n8nworkflows.xyz/workflows/learn-n8n-basics-in-3-easy-steps---8527


# Learn n8n Basics in 3 Easy Steps ✨

### 1. Workflow Overview

This workflow, titled **"Learn n8n Basics in 3 Easy Steps ✨"**, is a beginner-friendly automation designed to demonstrate key n8n functionalities by fetching inspirational quotes and programming jokes from public APIs, processing their data, scheduling automated runs, and sending results via email. It targets users who want to learn the n8n basics by building a practical example that integrates web requests, data transformation, scheduling, and email dispatch.

The workflow is logically divided into three main blocks:

- **1.1 Manual Quote Retrieval Block:** Allows manual triggering to fetch a random inspirational quote and display the results immediately.
- **1.2 Scheduled Quote Delivery Block:** Automatically fetches a random inspirational quote daily at a specified time, processes the data, and sends it via Gmail.
- **1.3 Joke Retrieval and Mapping Block:** Fetches a programming joke, maps its data fields alongside the quote, and prepares it for further email integration.

Several sticky notes guide users interactively on running, scheduling, and extending the workflow.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Quote Retrieval Block

**Overview:**  
This block enables users to manually trigger the workflow, retrieve a random inspirational quote from an external API, and view the structured result.

**Nodes Involved:**  
- 1.1.1 "1. Click 'Execute workflow’" (Manual Trigger)  
- 1.1.2 "2. Get inspirational quote from the internet" (HTTP Request)  
- 1.1.3 "3. See the resutls" (Set)

**Node Details:**  

- **1.1.1 "1. Click 'Execute workflow’"**  
  - Type: Manual Trigger  
  - Role: Entry point for manual execution.  
  - Configuration: No parameters; triggers workflow on manual command.  
  - Connections: Outputs to "2. Get inspirational quote from the internet".  
  - Edge Cases: None significant; manual trigger is user-controlled.

- **1.1.2 "2. Get inspirational quote from the internet"**  
  - Type: HTTP Request  
  - Role: Fetches random inspirational quote JSON from https://zenquotes.io/api/random.  
  - Configuration: Simple GET request, no authentication or additional headers.  
  - Connections: Outputs to "3. See the resutls".  
  - Edge Cases: API unavailability, rate limiting, or malformed response. Should handle HTTP errors or empty data gracefully.

- **1.1.3 "3. See the resutls"**  
  - Type: Set  
  - Role: Extracts and formats the quote and author fields from the API response for easier consumption.  
  - Configuration: Assigns `quote` = `{{$json.q}}`, `author` = `{{$json.a}}`.  
  - Connections: Terminal node for this block.  
  - Edge Cases: Missing fields in API response may cause empty or undefined values.

---

#### 1.2 Scheduled Quote Delivery Block

**Overview:**  
This block schedules an automated daily task to fetch a random quote, process it, and send it via Gmail to a configured recipient.

**Nodes Involved:**  
- 1.2.1 "Schedule me here" (Schedule Trigger)  
- 1.2.2 "Get quote" (HTTP Request)  
- 1.2.3 "Results" (Set)  
- 1.2.4 "Connect your Gmail" (Gmail node)

**Node Details:**  

- **1.2.1 "Schedule me here"**  
  - Type: Schedule Trigger  
  - Role: Starts the workflow automatically every day at 07:00 AM.  
  - Configuration: Interval rule set to trigger daily at 7 AM.  
  - Connections: Outputs to "Get quote".  
  - Edge Cases: Timezone considerations; ensure server timezone matches expectation.

- **1.2.2 "Get quote"**  
  - Type: HTTP Request  
  - Role: Fetches random inspirational quote JSON from https://zenquotes.io/api/random, same as in manual block.  
  - Configuration: Simple GET request.  
  - Connections: Outputs to "Results".  
  - Edge Cases: Same as manual block; handle API errors.

- **1.2.3 "Results"**  
  - Type: Set  
  - Role: Formats the quote and author fields from the API response, identical to the manual block.  
  - Configuration: Assigns `quote` = `{{$json.q}}`, `author` = `{{$json.a}}`.  
  - Connections: Outputs to "Connect your Gmail".  
  - Edge Cases: Missing or malformed data from API.

- **1.2.4 "Connect your Gmail"**  
  - Type: Gmail node (Send email)  
  - Role: Sends an email containing the quote and author to the configured recipient.  
  - Configuration:  
    - Subject: "Daily quote"  
    - Email body: Plain text with template:  
      ```
      Hey,

      Here's a quote by {{ $json.author }}:

      "{{ $json.quote }}"
      ```  
    - Credentials: OAuth2 Gmail account configured (note: user must connect manually).  
  - Connections: Terminal node for this block.  
  - Edge Cases: Authentication failures, rate limiting on Gmail API, invalid recipient addresses.

---

#### 1.3 Joke Retrieval and Mapping Block

**Overview:**  
This block fetches a programming joke from a public API, combines it with the quote data, and maps all fields for future use, such as adding jokes to emails.

**Nodes Involved:**  
- 1.3.1 "Schedule Trigger" (Schedule Trigger)  
- 1.3.2 "Quote" (HTTP Request)  
- 1.3.3 "Programming joke" (HTTP Request)  
- 1.3.4 "Map the fields" (Set)

**Node Details:**  

- **1.3.1 "Schedule Trigger"**  
  - Type: Schedule Trigger  
  - Role: Triggers the workflow on a schedule (currently set to run once every 12 months at 6 AM, which seems unusual and likely requires adjustment).  
  - Configuration: Interval set with a 12-month interval at hour 6.  
  - Connections: Outputs to "Quote".  
  - Edge Cases: Scheduling frequency likely misconfigured or placeholder; adjust for practical use.

- **1.3.2 "Quote"**  
  - Type: HTTP Request  
  - Role: Fetches a random quote from https://zenquotes.io/api/random.  
  - Configuration: Simple GET request.  
  - Connections: Outputs to "Programming joke".  
  - Edge Cases: Same as previous quote nodes.

- **1.3.3 "Programming joke"**  
  - Type: HTTP Request  
  - Role: Retrieves a programming joke from https://v2.jokeapi.dev/joke/Programming with safe content filters.  
  - Configuration: URL includes parameters to blacklist nsfw, religious, political, racist, sexist, explicit content, and requests single-part jokes only.  
  - Connections: Outputs to "Map the fields".  
  - Edge Cases: API downtime, empty or unexpected joke format.

- **1.3.4 "Map the fields"**  
  - Type: Set  
  - Role: Intended to map `author`, `quote`, and `joke` fields for downstream use.  
  - Configuration: All three fields assigned with empty expressions `=`, indicating placeholders that require proper field mapping expressions to extract data from previous nodes.  
  - Connections: Terminal node in this block.  
  - Edge Cases: Incorrect or missing mapping expressions will lead to empty data fields.

---

### 3. Summary Table

| Node Name                            | Node Type           | Functional Role                         | Input Node(s)                          | Output Node(s)                 | Sticky Note                                                                                      |
|------------------------------------|---------------------|---------------------------------------|--------------------------------------|-------------------------------|-------------------------------------------------------------------------------------------------|
| Sticky Note3                       | Sticky Note         | Guidance on scheduling and Gmail setup| None                                 | None                          | ## ... now let's schedule it ⏰ 1. Double-click “Schedule” to set when it runs 2. Connect your Gmail and set the recepient 3. To test, click “Execute workflow” on the "Schedule" node |
| Sticky Note2                       | Sticky Note         | Empty note                           | None                                 | None                          |                                                                                                 |
| Schedule me here                  | Schedule Trigger     | Triggers daily quote retrieval at 7 AM| None                                 | Get quote                     | Double-click to set!                                                                            |
| Results                           | Set                 | Formats quote and author fields       | Get quote                            | Connect your Gmail             |                                                                                                 |
| Get quote                        | HTTP Request        | Fetches random inspirational quote   | Schedule me here                    | Results                       |                                                                                                 |
| Connect your Gmail               | Gmail                | Sends quote email                    | Results                             | None                          | Double-click here to connect!                                                                   |
| Sticky Note15                    | Sticky Note         | Empty note                           | None                                 | None                          |                                                                                                 |
| Sticky Note                     | Sticky Note         | Introductory instructions for manual workflow| None                         | None                          | ## 🚀🚀 Start here 🚀🚀 1. Click “Execute workflow” to run it 2. This gets a random quote from the internet 3. Double-click the last node to check what quote you got |
| 1. Click 'Execute workflow’      | Manual Trigger       | Manual trigger for quote retrieval   | None                                 | 2. Get inspirational quote from the internet |                                                                                                 |
| 2. Get inspirational quote from the internet | HTTP Request | Fetches random inspirational quote   | 1. Click 'Execute workflow’          | 3. See the resutls            |                                                                                                 |
| 3. See the resutls               | Set                 | Formats quote and author fields       | 2. Get inspirational quote from the internet | None                      | Double-click to open!                                                                            |
| Sticky Note6                    | Sticky Note         | Empty note                           | None                                 | None                          |                                                                                                 |
| Sticky Note4                   | Sticky Note         | Guidance on adding a joke to emails   | None                                 | None                          | ## ❓ How about a joke... 1. Execute this workflow 2. Check out the new "Programming Joke" 3. Try mapping the new results and add them to your email |
| Sticky Note5                   | Sticky Note         | Empty note                           | None                                 | None                          |                                                                                                 |
| Quote                          | HTTP Request        | Fetches random inspirational quote   | Schedule Trigger                   | Programming joke              |                                                                                                 |
| Schedule Trigger               | Schedule Trigger     | Triggers quote and joke retrieval (yearly at 6 AM) | None                          | Quote                         |                                                                                                 |
| Map the fields                | Set                 | Maps fields for author, quote, and joke | Programming joke                   | None                          | Double-click to set!                                                                            |
| Programming joke              | HTTP Request        | Fetches programming joke              | Quote                            | Map the fields                | Check out after running!                                                                        |
| Sticky Note1                  | Sticky Note         | YouTube link                         | None                                 | None                          | @[youtube](dKTcAfBfFLU)                                                                         |
| Sticky Note8                  | Sticky Note         | YouTube link                         | None                                 | None                          | @[youtube](LdE0KnhRtsY)                                                                         |
| Sticky Note7                  | Sticky Note         | YouTube link                         | None                                 | None                          | @[youtube](5CBUXMO_L2Y)                                                                         |
| Sticky Note9                  | Sticky Note         | Step 1 title                        | None                                 | None                          | # Step 1                                                                                       |
| Sticky Note10                 | Sticky Note         | Step 2 title                        | None                                 | None                          | # Step 2                                                                                       |
| Sticky Note11                 | Sticky Note         | Step 3 title                        | None                                 | None                          | # Step 3                                                                                       |
| Sticky Note13                 | Sticky Note         | Reminder to add Gmail credentials    | None                                 | None                          | ### Click + to add Gmail                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Manual Trigger Block**  
   1.1 Add a **Manual Trigger** node named `"1. Click 'Execute workflow’"`. No parameters needed.  
   1.2 Add an **HTTP Request** node named `"2. Get inspirational quote from the internet"`. Configure:  
       - HTTP Method: GET  
       - URL: `https://zenquotes.io/api/random`  
   1.3 Connect the Manual Trigger node output to the HTTP Request node input.  
   1.4 Add a **Set** node named `"3. See the resutls"`. Configure to assign two string fields:  
       - `quote` = `{{$json.q}}`  
       - `author` = `{{$json.a}}`  
   1.5 Connect the HTTP Request node output to the Set node input.

2. **Create the Scheduled Quote Delivery Block**  
   2.1 Add a **Schedule Trigger** node named `"Schedule me here"`. Configure to trigger daily at 07:00 AM (adjust timezone as needed).  
   2.2 Add an **HTTP Request** node named `"Get quote"` with the same configuration as in 1.2.  
   2.3 Connect `"Schedule me here"` output to `"Get quote"` input.  
   2.4 Add a **Set** node named `"Results"` with the same field assignments as in 1.4.  
   2.5 Connect `"Get quote"` output to `"Results"` input.  
   2.6 Add a **Gmail** node named `"Connect your Gmail"`. Configure:  
       - Connect your Gmail account via OAuth2 credentials.  
       - Set email subject to `"Daily quote"`.  
       - Set email body (plain text) to:  
         ```
         Hey,

         Here's a quote by {{ $json.author }}:

         "{{ $json.quote }}"
         ```  
   2.7 Connect `"Results"` output to `"Connect your Gmail"` input.

3. **Create the Joke Retrieval and Mapping Block**  
   3.1 Add a **Schedule Trigger** node named `"Schedule Trigger"`. Initially set to run yearly at 6 AM (adjust as needed).  
   3.2 Add an **HTTP Request** node named `"Quote"` with GET to `https://zenquotes.io/api/random`.  
   3.3 Connect `"Schedule Trigger"` output to `"Quote"` input.  
   3.4 Add an **HTTP Request** node named `"Programming joke"`. Configure:  
       - HTTP Method: GET  
       - URL: `https://v2.jokeapi.dev/joke/Programming?blacklistFlags=nsfw,religious,political,racist,sexist,explicit&type=single`  
   3.5 Connect `"Quote"` output to `"Programming joke"` input.  
   3.6 Add a **Set** node named `"Map the fields"`. Configure to assign the following string fields (to be updated with correct expressions):  
       - `author` = `=your expression here`  
       - `quote` = `=your expression here`  
       - `joke` = `=your expression here`  
   3.7 Connect `"Programming joke"` output to `"Map the fields"` input.

4. **Add Sticky Notes** as per the original workflow for guidance and video links:  
   - Add sticky notes with instructional content on scheduling, workflow execution, and adding Gmail credentials.  
   - Add YouTube video links in sticky notes for additional learning resources.

5. **Credentials Setup**  
   - For the Gmail node, configure OAuth2 credentials by connecting your Gmail account securely inside n8n.  
   - No credentials are needed for the HTTP requests as they use public APIs.

6. **Final Testing and Validation**  
   - Manually execute the manual trigger block to verify quote retrieval and formatting.  
   - Adjust schedule triggers and test scheduled runs.  
   - Verify Gmail node sends emails correctly with dynamic quote content.  
   - Fix any mapping expressions in `"Map the fields"` node to extract correct values from prior nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                         |
|-----------------------------------------------------------------------------------------------------------------|---------------------------------------|
| @[youtube](dKTcAfBfFLU)                                                                                        | YouTube video linked in Sticky Note1  |
| @[youtube](LdE0KnhRtsY)                                                                                        | YouTube video linked in Sticky Note8  |
| @[youtube](5CBUXMO_L2Y)                                                                                        | YouTube video linked in Sticky Note7  |
| ## ... now let's schedule it ⏰ 1. Double-click “Schedule” to set when it runs 2. Connect your Gmail and set the recepient 3. To test, click “Execute workflow” on the "Schedule" node | Guidance in Sticky Note3                |
| ## 🚀🚀 Start here 🚀🚀 1. Click “Execute workflow” to run it 2. This gets a random quote from the internet 3. Double-click the last node to check what quote you got | Guidance in Sticky Note                 |
| ## ❓ How about a joke... 1. Execute this workflow 2. Check out the new "Programming Joke" 3. Try mapping the new results and add them to your email | Guidance in Sticky Note4                |
| ### Click + to add Gmail                                                                                         | Reminder in Sticky Note13              |

---

**Disclaimer:** The provided content is exclusively generated from the n8n workflow JSON shared. It adheres strictly to content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.