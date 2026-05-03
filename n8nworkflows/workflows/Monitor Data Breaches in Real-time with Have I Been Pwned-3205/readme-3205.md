Monitor Data Breaches in Real-time with Have I Been Pwned

https://n8nworkflows.xyz/workflows/monitor-data-breaches-in-real-time-with-have-i-been-pwned-3205


# Monitor Data Breaches in Real-time with Have I Been Pwned

### 1. Workflow Overview

This workflow is designed to monitor data breaches in real-time using the Have I Been Pwned (HIBP) API. It targets security professionals, developers, and individuals interested in proactive identity protection by automatically detecting new breaches as they are published. The workflow runs every 15 minutes, fetching the latest breach data, comparing it against a cached record of previously alerted breaches, and triggering alerts only when new breaches are detected.

The workflow is logically divided into the following blocks:

- **1.1 Trigger and Data Fetching:** Initiates periodic execution and retrieves the latest breach data from the HIBP API.
- **1.2 Cache Reading and Parsing:** Reads the cached breach data from a local file and prepares it for comparison.
- **1.3 Breach Comparison Logic:** Compares the newly fetched breach with the cached breach to determine if it is new.
- **1.4 Cache Update and Alerting:** Updates the cache with new breach data and triggers alert mechanisms for new breaches.
- **1.5 Initialization and Manual Testing:** Supports manual workflow testing and cache initialization.
- **1.6 Documentation and Support Notes:** Provides user guidance, customization instructions, and author credits.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Data Fetching

**Overview:**  
This block triggers the workflow every 15 minutes and fetches the latest breach data from the Have I Been Pwned API.

**Nodes Involved:**  
- Schedule Trigger  
- Request breaches  
- Add information about the last breach we alerted  

**Node Details:**  

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Periodically triggers the workflow every 15 minutes.  
  - Configuration: Interval set to 15 minutes.  
  - Inputs: None (start node)  
  - Outputs: Connects to "Request breaches" node.  
  - Edge Cases: Workflow execution delay or missed triggers if n8n instance is down.

- **Request breaches**  
  - Type: HTTP Request  
  - Role: Calls the HIBP API endpoint `https://haveibeenpwned.com/api/v3/latestbreach` to fetch the latest breach data.  
  - Configuration: GET request, no authentication required.  
  - Inputs: Triggered by Schedule Trigger.  
  - Outputs: Passes response data to "Read last breach" and "Add information about the last breach we alerted".  
  - Edge Cases: API downtime, network errors, rate limiting by HIBP, malformed responses.

- **Add information about the last breach we alerted**  
  - Type: Merge  
  - Role: Combines the latest breach data with cached breach data for comparison.  
  - Configuration: Mode set to "combine all".  
  - Inputs: Receives latest breach data and cached breach data.  
  - Outputs: Connects to "If - check for new".  
  - Edge Cases: Data format mismatches, empty inputs.

---

#### 1.2 Cache Reading and Parsing

**Overview:**  
Reads the cached breach data from a local JSON file (`cache.json`) and parses it to extract the last alerted breach name.

**Nodes Involved:**  
- Read last breach  
- Get JSON from file  
- Split Out  
- Check for content  
- Set to none  

**Node Details:**  

- **Read last breach**  
  - Type: Read/Write File  
  - Role: Reads the `cache.json` file containing the last alerted breach name.  
  - Configuration: Reads from `./cache.json`.  
  - Inputs: Triggered after "Request breaches".  
  - Outputs: Passes file content to "Get JSON from file".  
  - Edge Cases: File not found, read permission errors, empty file.

- **Get JSON from file**  
  - Type: Extract From File  
  - Role: Parses the file content from JSON string to JSON object.  
  - Configuration: Operation set to "fromJson".  
  - Inputs: Receives file content from "Read last breach".  
  - Outputs: Passes parsed JSON to "Split Out".  
  - Edge Cases: Invalid JSON format, parsing errors.

- **Split Out**  
  - Type: Split Out  
  - Role: Extracts the `data` field from the parsed JSON for further processing.  
  - Configuration: Field to split out is `data`.  
  - Inputs: Receives parsed JSON from "Get JSON from file".  
  - Outputs: Connects to "Check for content".  
  - Edge Cases: Missing `data` field, empty arrays.

- **Check for content**  
  - Type: If  
  - Role: Checks if the cached breach data exists (non-empty).  
  - Configuration: Condition checks if `lastItem` exists and is non-empty.  
  - Inputs: Receives split data from "Split Out".  
  - Outputs:  
    - True branch: Passes data to "Add information about the last breach we alerted".  
    - False branch: Connects to "Set to none".  
  - Edge Cases: Empty cache file or missing data.

- **Set to none**  
  - Type: Set  
  - Role: Sets a default value `lastItem` to "none" when cache is empty.  
  - Configuration: Assigns `lastItem` = "none".  
  - Inputs: Triggered when cache is empty.  
  - Outputs: Connects to "Add information about the last breach we alerted".  
  - Edge Cases: Ensures workflow continues gracefully with empty cache.

---

#### 1.3 Breach Comparison Logic

**Overview:**  
Compares the latest breach name with the cached breach name to determine if the breach is new or already alerted.

**Nodes Involved:**  
- If - check for new  
- Set breach name  
- Old breach  
- New breach  

**Node Details:**  

- **If - check for new**  
  - Type: If  
  - Role: Compares cached breach name (`lastItem`) with the latest breach name (`Name`).  
  - Configuration: Condition checks if `lastItem` is not equal to `Name`.  
  - Inputs: Receives combined data from "Add information about the last breach we alerted".  
  - Outputs:  
    - True branch (new breach): Connects to "Set breach name" and "New breach".  
    - False branch (old breach): Connects to "Old breach".  
  - Edge Cases: Case sensitivity, missing fields.

- **Set breach name**  
  - Type: Set  
  - Role: Updates the `lastItem` variable with the new breach name.  
  - Configuration: Assigns `lastItem` = `Name` from the latest breach.  
  - Inputs: Triggered when a new breach is detected.  
  - Outputs: Connects to "Convert to File".  
  - Edge Cases: Missing `Name` field.

- **Old breach**  
  - Type: NoOp  
  - Role: Placeholder node indicating the breach has been seen before; no action taken.  
  - Inputs: Triggered when breach is not new.  
  - Outputs: None.  
  - Notes: Contains a sticky note explaining this is a previously alerted breach.

- **New breach**  
  - Type: NoOp  
  - Role: Placeholder node indicating a new breach detected; place to add alerting nodes.  
  - Inputs: Triggered when breach is new.  
  - Outputs: None by default; users can add alert nodes here.  
  - Notes: Contains a sticky note prompting users to add alert mechanisms (Slack, email, etc.).

---

#### 1.4 Cache Update and Alerting

**Overview:**  
Converts the updated breach name to JSON file format and writes it to the cache file to persist the latest alerted breach.

**Nodes Involved:**  
- Convert to File  
- Write breach name to file  

**Node Details:**  

- **Convert to File**  
  - Type: Convert To File  
  - Role: Converts the updated breach name JSON data to a file format suitable for writing.  
  - Configuration: Operation set to "toJson", binary property name set to `data`.  
  - Inputs: Receives updated breach name from "Set breach name".  
  - Outputs: Connects to "Write breach name to file".  
  - Edge Cases: Conversion errors, data format issues.

- **Write breach name to file**  
  - Type: Read/Write File  
  - Role: Writes the updated breach name JSON file to `./cache.json`.  
  - Configuration: Operation set to "write", file path `./cache.json`.  
  - Inputs: Receives file data from "Convert to File".  
  - Outputs: None (end of cache update flow).  
  - Edge Cases: File write permission errors, disk space issues.

---

#### 1.5 Initialization and Manual Testing

**Overview:**  
Supports manual workflow testing and cache initialization by writing an empty JSON object to the cache file.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Set empty json  
- Convert json to file  
- Write cache.json  

**Node Details:**  

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual execution of the workflow for testing purposes.  
  - Inputs: None (start node).  
  - Outputs: Connects to "Set empty json".  
  - Edge Cases: Manual trigger only; no automatic scheduling.

- **Set empty json**  
  - Type: Set  
  - Role: Sets an empty JSON object `{}` as string in the `data` field.  
  - Configuration: Assigns `data` = `"{}"`.  
  - Inputs: Triggered by manual trigger.  
  - Outputs: Connects to "Convert json to file".  
  - Edge Cases: Ensures cache file is reset.

- **Convert json to file**  
  - Type: Convert To File  
  - Role: Converts the empty JSON string to a file format.  
  - Configuration: Operation set to "toJson".  
  - Inputs: Receives empty JSON string from "Set empty json".  
  - Outputs: Connects to "Write cache.json".  
  - Edge Cases: Conversion errors unlikely.

- **Write cache.json**  
  - Type: Read/Write File  
  - Role: Writes the empty JSON file to `./cache.json` to reset the cache.  
  - Configuration: Operation set to "write", file path `./cache.json`.  
  - Inputs: Receives file data from "Convert json to file".  
  - Outputs: None (end of manual reset flow).  
  - Edge Cases: File write permission errors.

---

#### 1.6 Documentation and Support Notes

**Overview:**  
Contains sticky notes providing workflow description, instructions, and author support information.

**Nodes Involved:**  
- Sticky Note (overview)  
- Sticky Note1 (cache explanation)  
- Sticky Note2 (old breach explanation)  
- Sticky Note3 (new breach alert explanation)  
- Sticky Note4 (cache cleanup instruction)  
- Sticky Note6 (author support and credits)  

**Node Details:**  

- **Sticky Note**  
  - Content: Overview of workflow purpose and cache mechanism.  
  - Position: Near the top-left of the workflow.  

- **Sticky Note1**  
  - Content: Explains saving the breach name for next run comparison.  
  - Position: Near cache update nodes.  

- **Sticky Note2**  
  - Content: Explains the old breach path where no alert is sent.  
  - Position: Near "Old breach" node.  

- **Sticky Note3**  
  - Content: Explains the new breach path and alerting opportunity.  
  - Position: Near "New breach" node.  

- **Sticky Note4**  
  - Content: Instructions to delete `cache.json` to reset alerts.  
  - Position: Near "Request breaches" node.  

- **Sticky Note6**  
  - Content: Author introduction, support links, and encouragement to support the creator.  
  - Link: [xqus.com](https://xqus.com), [Gumroad](https://xqus.gumroad.com/l/hasgi)  
  - Position: Near manual trigger node.

---

### 3. Summary Table

| Node Name                          | Node Type           | Functional Role                                | Input Node(s)                          | Output Node(s)                          | Sticky Note                                                                                      |
|-----------------------------------|---------------------|-----------------------------------------------|--------------------------------------|---------------------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’      | Manual Trigger      | Manual start for testing and cache reset      | None                                 | Set empty json                        |                                                                                                |
| Set empty json                    | Set                 | Sets empty JSON string for cache reset        | When clicking ‘Test workflow’         | Convert json to file                  |                                                                                                |
| Convert json to file              | Convert To File     | Converts JSON string to file format            | Set empty json                       | Write cache.json                     |                                                                                                |
| Write cache.json                 | Read/Write File     | Writes empty cache file to reset cache         | Convert json to file                 | None                                |                                                                                                |
| Schedule Trigger                 | Schedule Trigger    | Triggers workflow every 15 minutes             | None                                 | Request breaches                    |                                                                                                |
| Request breaches                | HTTP Request       | Fetches latest breach data from HIBP API       | Schedule Trigger                    | Read last breach, Add information about the last breach we alerted |                                                                                                |
| Read last breach               | Read/Write File     | Reads cached breach data from `cache.json`     | Request breaches                   | Get JSON from file                  |                                                                                                |
| Get JSON from file             | Extract From File   | Parses cached file content to JSON             | Read last breach                   | Split Out                         |                                                                                                |
| Split Out                     | Split Out           | Extracts `data` field from cached JSON         | Get JSON from file                 | Check for content                  |                                                                                                |
| Check for content             | If                  | Checks if cached breach data exists             | Split Out                         | Add information about the last breach we alerted (true), Set to none (false) |                                                                                                |
| Set to none                   | Set                 | Sets `lastItem` to "none" if cache empty       | Check for content (false branch)   | Add information about the last breach we alerted | File was empty.                                                                                |
| Add information about the last breach we alerted | Merge               | Combines latest breach and cached breach data | Request breaches, Check for content | If - check for new                |                                                                                                |
| If - check for new            | If                  | Compares cached breach name with latest breach | Add information about the last breach we alerted | Set breach name, New breach (true), Old breach (false) |                                                                                                |
| Set breach name               | Set                 | Updates cached breach name with new breach     | If - check for new (true branch)   | Convert to File                   |                                                                                                |
| Convert to File               | Convert To File     | Converts updated breach name to file format    | Set breach name                   | Write breach name to file          |                                                                                                |
| Write breach name to file     | Read/Write File     | Writes updated breach name to `cache.json`     | Convert to File                   | None                            |                                                                                                |
| New breach                   | NoOp                | Placeholder for new breach alerting             | If - check for new (true branch)   | None                            | Send alert. Add alert nodes here (Slack, email, etc.).                                        |
| Old breach                   | NoOp                | Placeholder for already alerted breach          | If - check for new (false branch)  | None                            | Already alerted.                                                                              |
| Sticky Note                  | Sticky Note         | Workflow overview and cache explanation         | None                             | None                            | ### Receive an alert when new breaches are added to haveibeenpwned.com. This workflow demonstrates how we can receive alerts when new breaches are added to haveibeenpwned.com. It also demonstrates a simple method for caching data between executions. |
| Sticky Note1                 | Sticky Note         | Explains saving breach name for next run        | None                             | None                            | ### Save the name of the breach. We will check it the next time the workflow runs to see if we have a new breach. |
| Sticky Note2                 | Sticky Note         | Explains old breach path                         | None                             | None                            | ### This breach has been seen before. If we end up here it means that the latest breach has been seen before. |
| Sticky Note3                 | Sticky Note         | Explains new breach alerting                     | None                             | None                            | ### This is a new breach - send alert. If we end up here it means that the latest breach is new. Time to send some alerts to Slack, or Discord or something. |
| Sticky Note4                 | Sticky Note         | Instructions to reset cache                       | None                             | None                            | ### Clean up the cache. Delete the `./cache.json` file. This will make sure the alert is triggered on the next run. |
| Sticky Note6                 | Sticky Note         | Author support and credits                        | None                             | None                            | ## Support My Work! ❤️ 👋 Hello! I'm Audun / xqus 🔗 My work: [xqus.com](https://xqus.com) 💸 n8n shop: [xqus.gumroad.com](https://xqus.gumroad.com) If you find this workflow helpful, consider downloading or purchasing it on [Gumroad](https://xqus.gumroad.com/l/hasgi). Your support helps me create more useful n8n workflows and resources for the community. -Thanks a lot! 🙌 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set interval to every 15 minutes.  
   - This node will start the workflow automatically.

2. **Add an HTTP Request node named "Request breaches"**  
   - Connect it to the Schedule Trigger.  
   - Set method to GET.  
   - Set URL to `https://haveibeenpwned.com/api/v3/latestbreach`.  
   - No authentication needed.

3. **Add a Read/Write File node named "Read last breach"**  
   - Connect it to "Request breaches".  
   - Configure to read from file `./cache.json`.  
   - Operation: Read.

4. **Add an Extract From File node named "Get JSON from file"**  
   - Connect it to "Read last breach".  
   - Operation: fromJson.

5. **Add a Split Out node named "Split Out"**  
   - Connect it to "Get JSON from file".  
   - Field to split out: `data`.

6. **Add an If node named "Check for content"**  
   - Connect it to "Split Out".  
   - Condition: Check if `lastItem` exists (string exists).  
   - True branch connects to next step.  
   - False branch connects to "Set to none".

7. **Add a Set node named "Set to none"**  
   - Connect false branch of "Check for content" to this node.  
   - Assign `lastItem` = "none".  
   - Connect output to next step.

8. **Add a Merge node named "Add information about the last breach we alerted"**  
   - Connect true branch of "Check for content" and output of "Set to none" to this node.  
   - Also connect "Request breaches" output to this node.  
   - Mode: Combine all.

9. **Add an If node named "If - check for new"**  
   - Connect output of the Merge node to this node.  
   - Condition: Check if `lastItem` (cached breach name) is NOT EQUAL to `Name` (latest breach name).  
   - True branch: New breach detected.  
   - False branch: Old breach.

10. **Add a Set node named "Set breach name"**  
    - Connect true branch of "If - check for new" to this node.  
    - Assign `lastItem` = `Name` from latest breach.

11. **Add a Convert To File node named "Convert to File"**  
    - Connect "Set breach name" output to this node.  
    - Operation: toJson.  
    - Binary property name: `data`.

12. **Add a Read/Write File node named "Write breach name to file"**  
    - Connect "Convert to File" output to this node.  
    - Operation: Write.  
    - File name: `./cache.json`.

13. **Add a NoOp node named "New breach"**  
    - Connect true branch of "If - check for new" to this node (parallel to "Set breach name").  
    - This node is a placeholder for alerting mechanisms.  
    - Add your alert nodes (email, Slack, etc.) after this node as needed.

14. **Add a NoOp node named "Old breach"**  
    - Connect false branch of "If - check for new" to this node.  
    - Represents no action needed for already alerted breaches.

15. **Add a Manual Trigger node named "When clicking ‘Test workflow’"**  
    - For manual testing and cache reset.

16. **Add a Set node named "Set empty json"**  
    - Connect manual trigger to this node.  
    - Assign `data` = `"{}"` (empty JSON string).

17. **Add a Convert To File node named "Convert json to file"**  
    - Connect "Set empty json" to this node.  
    - Operation: toJson.

18. **Add a Read/Write File node named "Write cache.json"**  
    - Connect "Convert json to file" to this node.  
    - Operation: Write.  
    - File name: `./cache.json`.  
    - This resets the cache file.

19. **Add Sticky Notes as per the original workflow**  
    - Add notes for overview, cache explanation, old/new breach explanation, cache reset instructions, and author support.  
    - Position them near relevant nodes for clarity.

20. **Credential Setup**  
    - No credentials required for the HIBP API endpoint used.  
    - If adding alert nodes (Slack, email, etc.), configure their respective credentials accordingly.

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow demonstrates a simple cache mechanism to avoid redundant alerts for the same breach.                       | Workflow description                                                                             |
| To reset alerts, delete the `./cache.json` file or use the manual trigger to write an empty JSON cache.                   | Sticky Note4                                                                                     |
| Customize alerting by adding nodes (Email, Slack, Teams, Discord, Telegram, Webhook, SMS) after the "New breach" node.    | Workflow customization instructions                                                             |
| Support the author Audun / xqus by visiting [xqus.com](https://xqus.com) and [Gumroad](https://xqus.gumroad.com/l/hasgi). | Sticky Note6                                                                                    |
| The HIBP API endpoint used does not require an API key.                                                                   | Setup instructions                                                                              |
| The workflow runs every 15 minutes by default; adjust the Schedule Trigger node to change frequency.                      | Setup instructions                                                                              |

---

This document provides a comprehensive understanding of the workflow, enabling reproduction, modification, and troubleshooting by advanced users and AI agents alike.