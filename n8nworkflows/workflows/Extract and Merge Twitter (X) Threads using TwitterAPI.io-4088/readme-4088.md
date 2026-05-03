Extract and Merge Twitter (X) Threads using TwitterAPI.io

https://n8nworkflows.xyz/workflows/extract-and-merge-twitter--x--threads-using-twitterapi-io-4088


# Extract and Merge Twitter (X) Threads using TwitterAPI.io

### 1. Workflow Overview

This workflow, titled **"Extract and Merge Twitter (X) Threads using TwitterAPI.io"**, is designed to automate the extraction of Twitter threads or single tweets from a given Twitter URL. It is ideal for users who want to programmatically retrieve complete Twitter threads—consisting of the initial tweet plus all its replies forming the thread—and reconstruct them in order for further processing or analysis.

The workflow logically divides into the following blocks:

- **1.1 Input Reception:** Receives the Twitter link either manually or triggered by another workflow.
- **1.2 Tweet and Thread Identification:** Extracts tweet ID and username, detects if the link is for a single tweet or a thread, and fetches the first tweet.
- **1.3 Thread Extraction:** Retrieves replies connected to the first tweet to assemble the full thread.
- **1.4 Merging and Filtering:** Combines the first tweet with replies, orders them correctly, filters out empty or irrelevant results, and outputs the final merged thread.
- **1.5 Completion:** Handles the final output or no-operation scenario to gracefully end the workflow.

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

**Overview:**  
This block handles the triggering of the workflow, allowing it to be started manually for testing or executed programmatically via another workflow.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- When Executed by Another Workflow (Execute Workflow Trigger)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: `Manual Trigger`  
  - Role: Allows manual initiation, useful for testing with input such as a Twitter URL.  
  - Configuration: No parameters; expects input JSON with `tweet_url`.  
  - Inputs: None  
  - Outputs: Connects to “Extract Tweet ID and Username”  
  - Edge Cases: No input or malformed URL may cause downstream errors.

- **When Executed by Another Workflow**  
  - Type: `Execute Workflow Trigger`  
  - Role: Allows external workflows to trigger this workflow programmatically and pass input data.  
  - Configuration: No parameters; expects input data with Twitter URL.  
  - Inputs: None  
  - Outputs: Connects to “Extract Tweet ID and Username”  
  - Edge Cases: Missing or invalid input data; integration failures if called improperly.

---

#### 2.2 Tweet and Thread Identification

**Overview:**  
Extracts the Tweet ID and username from the provided Twitter URL, then fetches the first tweet data to determine if the tweet is standalone or part of a thread. Extracts conversation and author IDs for further thread fetching.

**Nodes Involved:**  
- Extract Tweet ID and Username (Function)  
- Get first tweet (HTTP Request)  
- Extract Conversation and Author ID (Function)

**Node Details:**

- **Extract Tweet ID and Username**  
  - Type: `Function` (JavaScript logic)  
  - Role: Parses input Twitter URL to extract the Tweet ID and username needed for API calls.  
  - Key Expressions: Uses regex or string functions to parse URL parameters.  
  - Inputs: Receives input from triggers containing `tweet_url`  
  - Outputs: Passes extracted `tweet_id` and `username` to “Get first tweet” and “Merge all tweet infos”  
  - Edge Cases: Malformed URLs, missing tweet ID, or unexpected link formats may cause failure.

- **Get first tweet**  
  - Type: `HTTP Request`  
  - Role: Calls twitterapi.io to retrieve data about the first tweet using extracted tweet ID.  
  - Configuration: API endpoint configured with credentials (Twitter API via twitterapi.io).  
  - Inputs: Receives tweet ID from function node.  
  - Outputs: Provides tweet data to “Extract Conversation and Author ID” and “Merge first tweet and others”.  
  - Edge Cases: API rate limits, authentication failure, network errors, invalid tweet ID.

- **Extract Conversation and Author ID**  
  - Type: `Function`  
  - Role: Extracts the conversation ID (thread identifier) and author ID from the first tweet data for fetching replies.  
  - Inputs: Receives tweet JSON data from “Get first tweet”.  
  - Outputs: Passes conversation and author IDs to “Merge all tweet infos”.  
  - Edge Cases: Unexpected data structure or missing fields.

---

#### 2.3 Thread Extraction

**Overview:**  
Fetches all replies connected to the first tweet based on the conversation ID and author ID, aggregating all tweets that form the thread.

**Nodes Involved:**  
- Merge all tweet infos (Merge node)  
- Get Tweet Replies (HTTP Request)  
- Fetch tweets which are connected to first tweet (Code node)

**Node Details:**

- **Merge all tweet infos**  
  - Type: `Merge`  
  - Role: Combines outputs from “Extract Tweet ID and Username” and “Extract Conversation and Author ID” to prepare for fetching replies.  
  - Inputs: Receives data from both previous function nodes.  
  - Outputs: Passes merged data to “Get Tweet Replies”.  
  - Edge Cases: Mismatched or missing data could affect downstream nodes.

- **Get Tweet Replies**  
  - Type: `HTTP Request`  
  - Role: Calls twitterapi.io API to fetch replies linked to the conversation ID and author ID.  
  - Configuration: Proper API endpoint and authentication with twitterapi.io credentials.  
  - Inputs: Receives merged info with conversation and author IDs.  
  - Outputs: Passes replies data to “Fetch tweets which are connected to first tweet”.  
  - On Error: Configured to continue regular output to avoid complete failure on API error.  
  - Edge Cases: API rate limits, network issues, large thread size, partial data.

- **Fetch tweets which are connected to first tweet**  
  - Type: `Code` (JavaScript)  
  - Role: Processes replies data to filter and identify only those tweets that are part of the thread connected to the first tweet.  
  - Inputs: Receives replies JSON from API.  
  - Outputs: Passes filtered tweets to “Merge first tweet and others”.  
  - Edge Cases: Logic errors in filtering, empty replies, unexpected data formats.

---

#### 2.4 Merging and Filtering

**Overview:**  
Combines the first tweet and all subsequent thread tweets, orders them correctly, and filters out any empty or irrelevant results to reconstruct the full thread.

**Nodes Involved:**  
- Merge first tweet and others (Merge node)  
- Filter empty ones (Filter node)

**Node Details:**

- **Merge first tweet and others**  
  - Type: `Merge`  
  - Role: Combines the first tweet and the filtered thread tweets into a single dataset for output.  
  - Inputs: Receives first tweet data and connected replies.  
  - Outputs: Passes combined data to “Filter empty ones”.  
  - Edge Cases: Missing input from either branch may cause incomplete output.

- **Filter empty ones**  
  - Type: `Filter`  
  - Role: Filters out any empty or invalid tweets from the merged list to ensure clean output.  
  - Inputs: Receives merged tweets.  
  - Outputs: Passes filtered tweets to “No Operation, do nothing”.  
  - Edge Cases: Overly aggressive filtering could remove valid tweets; insufficient filtering might leave empty or malformed entries.

---

#### 2.5 Completion

**Overview:**  
Final node to gracefully end the workflow or pass the output for further use.

**Nodes Involved:**  
- No Operation, do nothing (NoOp node)

**Node Details:**

- **No Operation, do nothing**  
  - Type: `NoOp` (No Operation)  
  - Role: Final placeholder node representing the end of the workflow execution.  
  - Inputs: Receives filtered tweet thread data.  
  - Outputs: None (end node)  
  - Edge Cases: None; purely structural.

---

### 3. Summary Table

| Node Name                         | Node Type               | Functional Role                                | Input Node(s)                    | Output Node(s)                      | Sticky Note |
|----------------------------------|-------------------------|-----------------------------------------------|---------------------------------|-----------------------------------|-------------|
| When clicking ‘Test workflow’     | Manual Trigger          | Manual start of workflow                       | None                            | Extract Tweet ID and Username      |             |
| When Executed by Another Workflow | Execute Workflow Trigger| Start triggered by other workflows            | None                            | Extract Tweet ID and Username      |             |
| Extract Tweet ID and Username     | Function                | Parses Tweet ID and username from URL         | When clicking ‘Test workflow’, When Executed by Another Workflow | Get first tweet, Merge all tweet infos |             |
| Get first tweet                  | HTTP Request            | Fetches first tweet data via TwitterAPI.io    | Extract Tweet ID and Username   | Extract Conversation and Author ID, Merge first tweet and others |             |
| Extract Conversation and Author ID| Function                | Extracts conversation and author IDs          | Get first tweet                 | Merge all tweet infos              |             |
| Merge all tweet infos             | Merge                   | Combines tweet and conversation data          | Extract Tweet ID and Username, Extract Conversation and Author ID | Get Tweet Replies                 |             |
| Get Tweet Replies                 | HTTP Request            | Fetches replies forming the thread             | Merge all tweet infos           | Fetch tweets which are connected to first tweet |             |
| Fetch tweets which are connected to first tweet | Code (JavaScript)    | Filters replies to those connected to first tweet | Get Tweet Replies             | Merge first tweet and others       |             |
| Merge first tweet and others      | Merge                   | Combines first tweet with fetched replies      | Get first tweet, Fetch tweets which are connected to first tweet | Filter empty ones                 |             |
| Filter empty ones                 | Filter                  | Removes empty or invalid tweets                 | Merge first tweet and others    | No Operation, do nothing           |             |
| No Operation, do nothing          | NoOp                    | Marks workflow end                              | Filter empty ones               | None                              |             |
| Sticky Note                      | Sticky Note             | Various notes on workflow canvas                | None                          | None                              |             |
| Sticky Note1                     | Sticky Note             | Various notes on workflow canvas                | None                          | None                              |             |
| Sticky Note2                     | Sticky Note             | Various notes on workflow canvas                | None                          | None                              |             |
| Sticky Note3                     | Sticky Note             | Various notes on workflow canvas                | None                          | None                              |             |
| Sticky Note4                     | Sticky Note             | Various notes on workflow canvas                | None                          | None                              |             |
| Sticky Note5                     | Sticky Note             | Various notes on workflow canvas                | None                          | None                              |             |
| Sticky Note6                     | Sticky Note             | Various notes on workflow canvas                | None                          | None                              |             |

---

### 4. Reproducing the Workflow from Scratch

To recreate the **Twitter Thread Fetcher** workflow in n8n, follow these steps:

1. **Create Trigger Nodes:**

   - Add a **Manual Trigger** node named `When clicking ‘Test workflow’`. No special parameters needed.
   - Add an **Execute Workflow Trigger** node named `When Executed by Another Workflow`. No special parameters needed.

2. **Extract Tweet ID and Username:**

   - Add a **Function** node named `Extract Tweet ID and Username`.
   - Code should parse input JSON property `tweet_url` to extract the Tweet ID and username using regex or string methods.
   - Connect both trigger nodes’ outputs to this node’s input.

3. **Fetch First Tweet:**

   - Add an **HTTP Request** node named `Get first tweet`.
   - Configure to call twitterapi.io API endpoint for fetching a tweet by ID.
   - Use credentials for twitterapi.io with your Twitter API keys.
   - URL should include the Tweet ID extracted in the previous node.
   - Connect `Extract Tweet ID and Username` output to this node.

4. **Extract Conversation and Author ID:**

   - Add a **Function** node named `Extract Conversation and Author ID`.
   - The node extracts `conversation_id` and `author_id` from the JSON response of the first tweet.
   - Connect `Get first tweet` output to this node.

5. **Merge Tweet Info:**

   - Add a **Merge** node named `Merge all tweet infos`.
   - Configure to combine data from `Extract Tweet ID and Username` and `Extract Conversation and Author ID`.
   - Connect the outputs of both function nodes to this merge.

6. **Fetch Tweet Replies:**

   - Add an **HTTP Request** node named `Get Tweet Replies`.
   - Configure to call twitterapi.io API endpoint to fetch replies for the conversation ID and author ID.
   - Use the merged data from the previous step as input.
   - Set error handling to "Continue on Fail" to handle API errors gracefully.

7. **Filter Connected Replies:**

   - Add a **Code** node named `Fetch tweets which are connected to first tweet`.
   - Implement JavaScript code to filter only tweets that are part of the thread connected to the first tweet (e.g., replies directly in the conversation).
   - Connect `Get Tweet Replies` output to this node.

8. **Merge First Tweet and Replies:**

   - Add a **Merge** node named `Merge first tweet and others`.
   - Configure it to combine outputs from `Get first tweet` and `Fetch tweets which are connected to first tweet`.
   - Connect both nodes accordingly.

9. **Filter Empty Results:**

   - Add a **Filter** node named `Filter empty ones`.
   - Configure it to remove any empty or invalid entries from the merged tweets.
   - Connect `Merge first tweet and others` output to this node.

10. **End Workflow:**

    - Add a **No Operation** node named `No Operation, do nothing`.
    - Connect the `Filter empty ones` output to this node.
    - This node marks the end of the workflow.

11. **Save and Test:**

    - Save the workflow.
    - Test manually by providing a Twitter URL in the `tweet_url` input of the Manual Trigger node.
    - Alternatively, trigger via another workflow passing the same input.

12. **Credentials Setup:**

    - Configure Twitter API credentials on n8n using `twitterapi.io` API keys.
    - Assign these credentials to all HTTP Request nodes calling twitterapi.io.

13. **Optional: Sticky Notes**

    - Add sticky notes near relevant nodes to provide configuration hints or instructions as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                      | Context or Link                                                     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------|
| The workflow relies on [twitterapi.io](https://twitterapi.io) for all Twitter API requests, which simplifies authentication and rate limiting.                                  | https://twitterapi.io                                             |
| Typical fetch time for a 15-tweet thread is approximately 3 seconds, making this workflow very efficient.                                                                          | Performance note                                                  |
| Cost estimation for fetching a 15-tweet thread is around $0.0027, highly cost-effective depending on thread length and reply density.                                           | Cost note                                                        |
| Recommended to create a separate preparatory workflow to collect Twitter URLs (e.g., from Notion or spreadsheets) and then trigger this workflow with those URLs for automation. | Suggested integration strategy                                   |
| Detailed sticky notes are included in the original workflow canvas to assist with setup and understanding.                                                                        | Embedded sticky notes within the n8n canvas                      |

---

**Disclaimer:** The provided text and workflow are generated exclusively from an automated n8n workflow. It complies fully with current content policies and contains no illegal, offensive, or protected content. All data handled is legal and public.