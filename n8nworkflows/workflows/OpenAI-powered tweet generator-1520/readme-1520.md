OpenAI-powered tweet generator

https://n8nworkflows.xyz/workflows/openai-powered-tweet-generator-1520


# OpenAI-powered tweet generator

### 1. Workflow Overview

This workflow automates the generation and storage of tweets using OpenAI's text generation capabilities, targeted for review and potential publication. It is designed to create tweet content that includes specific hashtags randomly selected from a predefined list. The generated tweets are then stored in an Airtable base for easy management and review.

The workflow is logically divided into the following blocks:

- **1.1 Input Trigger:** Manual execution trigger to start the tweet generation process.
- **1.2 Hashtag Selection:** Randomly selects a hashtag from a predefined list.
- **1.3 Tweet Generation:** Sends a prompt with the selected hashtag to OpenAI to generate a tweet.
- **1.4 Data Formatting:** Structures the generated tweet and hashtag for storage.
- **1.5 Storage:** Appends the generated tweet data into Airtable for review.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger

- **Overview:**  
  Initiates the workflow manually by user action, allowing controlled tweet generation.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**  
  - **Type:** Manual Trigger  
  - **Role:** Starts the workflow on user command.  
  - **Configuration:** Default manual trigger settings without parameters.  
  - **Expressions/Variables:** None.  
  - **Input/Output:** No input; outputs an empty item to start the flow.  
  - **Version Requirements:** Compatible with all n8n versions supporting manual triggers.  
  - **Edge Cases:** None; user must trigger manually, so no automatic failures expected.  
  - **Sub-workflow:** None.

---

#### 1.2 Hashtag Selection

- **Overview:**  
  Randomly selects one hashtag from a static list to include in the tweet prompt.

- **Nodes Involved:**  
  - FunctionItem

- **Node Details:**  
  - **Type:** Function Item  
  - **Role:** Executes JavaScript code on each input item (only one in this case).  
  - **Configuration:**  
    - Defines an array of hashtags: `["#techtwitter", "#n8n"]`  
    - Selects a random hashtag using `Math.random()`  
    - Assigns selected hashtag to `item.hashtag`.  
  - **Expressions/Variables:** Output JSON contains `{ hashtag: "#chosenHashtag" }`.  
  - **Input/Output:** Receives empty input from manual trigger; outputs item with `hashtag` property.  
  - **Version Requirements:** Standard JavaScript supported in all recent n8n versions.  
  - **Edge Cases:**  
    - Empty or malformed hashtag list would cause errors.  
    - Random selection always valid here due to static list.  
  - **Sub-workflow:** None.

---

#### 1.3 Tweet Generation

- **Overview:**  
  Calls OpenAI's API to generate a tweet based on the selected hashtag, enforcing length and style constraints.

- **Nodes Involved:**  
  - HTTP Request

- **Node Details:**  
  - **Type:** HTTP Request  
  - **Role:** Sends POST request to OpenAI's text generation endpoint (`text-davinci-001`).  
  - **Configuration:**  
    - URL: `https://api.openai.com/v1/engines/text-davinci-001/completions`  
    - Method: POST  
    - Authentication: Header-based with OpenAI API key  
    - JSON Body Parameters:  
      - `prompt`: `"Generate a tweet, with under 100 characters, about and including the hashtag {{$node["FunctionItem"].json["hashtag"]}}:"`  
      - `temperature`: 0.7 (controls creativity)  
      - `max_tokens`: 64 (limits response length)  
      - `top_p`: 1  
      - `frequency_penalty`: 0  
      - `presence_penalty`: 0  
  - **Expressions/Variables:** Uses expression to inject the selected hashtag from the previous node.  
  - **Input/Output:** Input is item with hashtag; output JSON includes OpenAI response in `choices[0].text`.  
  - **Version Requirements:** Requires HTTP Request node with support for header auth and JSON parameters.  
  - **Edge Cases:**  
    - API key issues (invalid, missing, expired) causing authentication errors.  
    - Network timeouts or API rate limits.  
    - Unexpected or empty response from OpenAI.  
    - Expression failures if hashtag property missing.  
  - **Sub-workflow:** None.

---

#### 1.4 Data Formatting

- **Overview:**  
  Extracts relevant data (hashtag and generated tweet) into a simplified structure for Airtable insertion.

- **Nodes Involved:**  
  - Set

- **Node Details:**  
  - **Type:** Set  
  - **Role:** Creates a new item with only the desired output properties.  
  - **Configuration:**  
    - Defines two string fields:  
      - `Hashtag` set from `{{$node["FunctionItem"].json["hashtag"]}}`  
      - `Content` set from `{{$node["HTTP Request"].json["choices"][0]["text"]}}`  
    - `keepOnlySet` option enabled to discard other data.  
  - **Expressions/Variables:** Uses expressions to map values from prior nodes.  
  - **Input/Output:** Input is OpenAI response item; output is item with clean, formatted fields.  
  - **Version Requirements:** Standard Set node features.  
  - **Edge Cases:**  
    - Missing or malformed OpenAI response causing empty `Content`.  
    - Missing hashtag property (unlikely here).  
  - **Sub-workflow:** None.

---

#### 1.5 Storage

- **Overview:**  
  Appends the formatted tweet and hashtag into an Airtable base for further review or publication.

- **Nodes Involved:**  
  - Airtable

- **Node Details:**  
  - **Type:** Airtable  
  - **Role:** Inserts new record into Airtable table `main`.  
  - **Configuration:**  
    - Operation: Append  
    - Table: `main`  
    - Application ID: `appOaG8kEA8FAABOr` (specific Airtable base)  
  - **Credentials:** Requires Airtable API key credential configured in n8n.  
  - **Input/Output:** Inputs formatted item with `Hashtag` and `Content`; outputs Airtable response on success.  
  - **Version Requirements:** Requires Airtable node version supporting append operation and credentials.  
  - **Edge Cases:**  
    - Invalid or missing Airtable API credentials.  
    - Airtable API rate limits or network failures.  
    - Table or base misconfiguration causing insertion failure.  
  - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name            | Node Type           | Functional Role               | Input Node(s)          | Output Node(s)       | Sticky Note                          |
|----------------------|---------------------|------------------------------|-----------------------|----------------------|------------------------------------|
| On clicking 'execute' | Manual Trigger      | Starts workflow manually      | —                     | FunctionItem         |                                    |
| FunctionItem          | Function Item       | Random hashtag selection      | On clicking 'execute'  | HTTP Request         |                                    |
| HTTP Request         | HTTP Request        | Calls OpenAI to generate tweet| FunctionItem           | Set                  |                                    |
| Set                  | Set                 | Formats data for Airtable     | HTTP Request           | Airtable             |                                    |
| Airtable             | Airtable            | Stores generated tweet        | Set                    | —                    |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node**  
   - Name: `On clicking 'execute'`  
   - No special parameters needed.

3. **Add a Function Item node**  
   - Name: `FunctionItem`  
   - Paste the following JavaScript code:  
     ```javascript
     // hashtag list
     const Hashtags = [
       "#techtwitter",
       "#n8n"
     ];

     // random output function
     const randomHashtag = Hashtags[Math.floor(Math.random() * Hashtags.length)];
     item.hashtag = randomHashtag;
     return item;
     ```
   - Connect `On clicking 'execute'` → `FunctionItem`.

4. **Add an HTTP Request node**  
   - Name: `HTTP Request`  
   - Set:  
     - HTTP Method: POST  
     - URL: `https://api.openai.com/v1/engines/text-davinci-001/completions`  
     - Authentication: Header Auth  
       - Configure credentials with your OpenAI API key.  
     - Enable `JSON/RAW Parameters`  
     - Body Parameters (JSON):  
       ```json
       {
         "prompt": "Generate a tweet, with under 100 characters, about and including the hashtag {{$node[\"FunctionItem\"].json[\"hashtag\"]}}:",
         "temperature": 0.7,
         "max_tokens": 64,
         "top_p": 1,
         "frequency_penalty": 0,
         "presence_penalty": 0
       }
       ```
   - Connect `FunctionItem` → `HTTP Request`.

5. **Add a Set node**  
   - Name: `Set`  
   - Enable "Keep Only Set"  
   - Add two string fields:  
     - `Hashtag` with value `={{$node["FunctionItem"].json["hashtag"]}}`  
     - `Content` with value `={{$node["HTTP Request"].json["choices"][0]["text"]}}`  
   - Connect `HTTP Request` → `Set`.

6. **Add an Airtable node**  
   - Name: `Airtable`  
   - Operation: Append  
   - Table: `main` (adjust as per your Airtable base)  
   - Connect your Airtable API credentials.  
   - Connect `Set` → `Airtable`.

7. **Save and test the workflow by clicking execute on the manual trigger.**

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                       |
|-----------------------------------------------------------------------------------------------|-------------------------------------|
| The workflow image `tweetgeneratorbot.jpg` illustrates the workflow setup and node layout.   | Attached file in the original workflow description. |

---

This document fully describes the entire OpenAI-powered tweet generator workflow, enabling reproduction, modification, and troubleshooting by advanced users and automation agents.