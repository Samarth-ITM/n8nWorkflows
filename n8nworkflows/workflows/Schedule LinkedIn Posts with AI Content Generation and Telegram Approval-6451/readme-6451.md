Schedule LinkedIn Posts with AI Content Generation and Telegram Approval

https://n8nworkflows.xyz/workflows/schedule-linkedin-posts-with-ai-content-generation-and-telegram-approval-6451


# Schedule LinkedIn Posts with AI Content Generation and Telegram Approval

### 1. Workflow Overview

This workflow automates the scheduling and posting of LinkedIn content with AI-generated text, followed by human approval via Telegram before the post is published. It is designed for use cases involving social media teams or individuals who want to maintain a consistent, engaging LinkedIn presence in the fields of UCaaS, AI, Contact Centers, and technology business sectors.

The workflow logic is divided into the following functional blocks:

- **1.1 Schedule and Date Filtering:** Triggers the workflow on scheduled days and times, filtering out weekends and specific days not intended for posting.
- **1.2 Prompt Preparation and AI Content Generation:** Selects from a set of predefined prompts to generate unique LinkedIn post content using OpenAI's GPT model.
- **1.3 Telegram Approval Process:** Sends the AI-generated draft to a Telegram user/chat for approval and waits for a response.
- **1.4 Scheduled Posting:** Upon approval, schedules the post to be published at a random time on the selected day.
- **1.5 LinkedIn Posting and Notification:** Publishes the approved post to LinkedIn and sends a Telegram notification confirming the post is active.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule and Date Filtering

- **Overview:** This block initiates the workflow on a cron schedule Monday to Thursday at 8 AM UTC, computes the current day adjusted for Central Time, and skips execution on Fridays, Saturdays, and Sundays.
- **Nodes Involved:**  
  - Schedule Trigger  
  - Get Day (Code)  
  - Check Day (If)  

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Starts the workflow every Monday to Thursday at 8:00 AM UTC (cron: `0 8 * * 1-4`).  
    - Configuration: Cron expression set to trigger at 8 AM UTC on weekdays Mon-Thu.  
    - Inputs: None (trigger node)  
    - Outputs: Connected to Get Day node  
    - Edge Cases: If the system time or timezone shifts, cron may misalign; ensure server timezone settings are consistent.  

  - **Get Day**  
    - Type: Code (JavaScript)  
    - Role: Determines whether the current day (adjusted to Central Time) is a posting day.  
    - Configuration: Converts UTC day to Central Time day by subtracting 5 hours; flags days Sunday (0), Friday (5), and Saturday (6) as skip days.  
    - Key Variables: `skipToday` boolean output indicating if workflow should skip posting today.  
    - Inputs: Trigger from Schedule Trigger  
    - Outputs: Connected to Check Day node  
    - Edge Cases: Handles day wrap-around when converting timezones; failure if date operations throw errors or if DST changes are not accounted for.  

  - **Check Day**  
    - Type: If  
    - Role: Branches the workflow based on `skipToday` flag; stops workflow if today is a skip day, otherwise continues.  
    - Configuration: Condition checks if `skipToday` is true; if yes, stops forward flow.  
    - Inputs: From Get Day  
    - Outputs:  
      - True branch: Ends workflow (no connections)  
      - False branch: Proceeds to Prompts node  
    - Edge Cases: Misconfigured condition could cause unintended skips or runs.  

#### 2.2 Prompt Preparation and AI Content Generation

- **Overview:** This block sets multiple detailed AI prompts for LinkedIn post generation, randomly selects one prompt, and sends it to OpenAI GPT-4 for content creation.
- **Nodes Involved:**  
  - Prompts (Set)  
  - Random Prompt (Code)  
  - OpenAI Message (OpenAI GPT-4 node)  

- **Node Details:**

  - **Prompts**  
    - Type: Set  
    - Role: Defines six detailed AI prompts focused on creating engaging LinkedIn posts about UCaaS, AI, Contact Center, and tech topics with specific tone and formatting instructions.  
    - Configuration: Six string variables `prompt1` to `prompt6` holding full prompt texts, including tone, style, content instructions, and topic focus.  
    - Inputs: False branch output from Check Day  
    - Outputs: Connected to Random Prompt  
    - Notes: Sticky Note advises setting OpenAI prompts here.  
    - Edge Cases: Potential for very long strings causing node input size limits; ensure prompts are updated carefully to avoid formatting errors.  

  - **Random Prompt**  
    - Type: Code (JavaScript)  
    - Role: Randomly selects one prompt from the six defined in the previous node to ensure variation in generated posts.  
    - Configuration: Picks a random index to select a prompt; outputs selected prompt as `selectedPrompt`.  
    - Inputs: From Prompts  
    - Outputs: Connected to OpenAI Message  
    - Notes: Sticky Note highlights purpose is to make posts look unique.  
    - Edge Cases: Array empty or undefined prompts could break selection; ensure prompts are always set.  

  - **OpenAI Message**  
    - Type: OpenAI GPT-4 (Langchain integration)  
    - Role: Sends the randomly selected prompt to OpenAI’s GPT-4 model for generation of LinkedIn post content.  
    - Configuration: Uses model `chatgpt-4o-latest`; message content set via expression referencing `Random Prompt` output.  
    - Credentials: OpenAI API credentials required and configured.  
    - Inputs: From Random Prompt  
    - Outputs: Connected to Random Time node  
    - Edge Cases: API authentication failure, rate limiting, or prompt length issues; retry logic recommended.  

#### 2.3 Telegram Approval Process

- **Overview:** Sends the AI-generated LinkedIn post draft via Telegram, waits for user approval before continuing.
- **Nodes Involved:**  
  - Random Time (Code)  
  - Send message and wait for response (Telegram)  
  - Telegram Approved (If)  

- **Node Details:**

  - **Random Time**  
    - Type: Code (JavaScript)  
    - Role: Selects a random posting time from predefined Central Time options (09:00, 11:00, 15:00), converts it to UTC ISO format for scheduling.  
    - Configuration: Defines an array of times, maps selection to UTC, outputs both local and UTC time strings.  
    - Inputs: From OpenAI Message  
    - Outputs: Connected to Send message and wait for response  
    - Notes: Sticky Note indicates this node selects a random time.  
    - Edge Cases: Time conversion errors possible with DST changes; ensure server time sync.  

  - **Send message and wait for response**  
    - Type: Telegram node (send and wait for approval)  
    - Role: Sends the AI-generated post draft along with the scheduled time to Telegram chat; waits for user approval response.  
    - Configuration: Message template includes the post, scheduled time, and an approval prompt; uses double approval type.  
    - Credentials: Telegram API token configured.  
    - Inputs: From Random Time  
    - Outputs: Connected to Telegram Approved  
    - Notes: Sticky Note includes message template preview.  
    - Edge Cases: Telegram API limits, network errors, or no response timeout should be handled.  

  - **Telegram Approved**  
    - Type: If  
    - Role: Checks the Telegram approval response boolean `data.approved` to branch workflow.  
    - Configuration: Passes workflow forward only if approved is true; otherwise ends.  
    - Inputs: From Send message and wait for response  
    - Outputs:  
      - True branch: Connects to Wait for Time Selected node  
      - False branch: Ends workflow  
    - Edge Cases: Incorrect parsing of approval data could cause logical errors.  

#### 2.4 Scheduled Posting

- **Overview:** Waits until the randomly selected time to execute the LinkedIn post creation.
- **Nodes Involved:**  
  - Wait for Time Selected (Wait)  

- **Node Details:**

  - **Wait for Time Selected**  
    - Type: Wait node  
    - Role: Pauses workflow execution until the scheduled UTC time calculated earlier.  
    - Configuration: Uses ISO time string `postTimeUTC_ISO` from Random Time node.  
    - Inputs: From Telegram Approved (true branch)  
    - Outputs: Connects to Create a post node  
    - Notes: Sticky Note explains the time expression used for waiting.  
    - Edge Cases: Workflow persistence required; if the workflow or n8n instance restarts, state may be lost unless persisted.  

#### 2.5 LinkedIn Posting and Notification

- **Overview:** Posts the approved content to LinkedIn and sends a confirmation message via Telegram.
- **Nodes Involved:**  
  - Create a post (LinkedIn)  
  - Notify Post Active (Telegram)  

- **Node Details:**

  - **Create a post**  
    - Type: LinkedIn node  
    - Role: Publishes the AI-generated and approved LinkedIn post content under the specified LinkedIn user account.  
    - Configuration: Text set via expression from OpenAI Message content; person ID specified to identify LinkedIn profile.  
    - Credentials: LinkedIn OAuth2 credentials required.  
    - Inputs: From Wait for Time Selected  
    - Outputs: Connects to Notify Post Active  
    - Edge Cases: LinkedIn API limits, authentication expiry, or content rejection errors possible.  

  - **Notify Post Active**  
    - Type: Telegram node  
    - Role: Sends a notification message to Telegram to confirm that the LinkedIn post is now active.  
    - Configuration: Simple text message "Personal LinkedIn Post Active" sent to dedicated chat ID.  
    - Credentials: Telegram API token configured.  
    - Inputs: From Create a post  
    - Outputs: None (end of workflow)  
    - Edge Cases: Telegram message failure or chat ID changes.  

---

### 3. Summary Table

| Node Name                      | Node Type                      | Functional Role                                    | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                                           |
|--------------------------------|--------------------------------|--------------------------------------------------|-------------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger               | Triggers workflow Mon-Thu 8 AM UTC                | None                          | Get Day                       | The schdule trigger checks to see if the time is 8AM                                                                  |
| Get Day                       | Code                          | Computes current day adjusted to Central Time     | Schedule Trigger              | Check Day                     | Check to see if this is a day we want to run the workflow. We are set to avoid Friday-Sunday. We also convert UTC to Central Time |
| Check Day                     | If                            | Skips workflow on weekends                         | Get Day                      | Prompts (false branch)        |                                                                                                                       |
| Prompts                      | Set                           | Defines six detailed AI prompts                    | Check Day                    | Random Prompt                 | Set your OpenAI prompts to set your LinkedIn post.                                                                     |
| Random Prompt                 | Code                          | Selects one prompt randomly for variation          | Prompts                      | OpenAI Message                | Select a random prompt so that the posts look unique.                                                                   |
| OpenAI Message               | OpenAI GPT-4                  | Generates LinkedIn post content from prompt        | Random Prompt                | Random Time                  |                                                                                                                       |
| Random Time                  | Code                          | Picks random posting time and converts to UTC ISO  | OpenAI Message              | Send message and wait for response | Select a Random Time                                                                                                   |
| Send message and wait for response | Telegram (send & wait)       | Sends draft post to Telegram and waits for approval | Random Time                  | Telegram Approved             | Here is your AI-generated LinkedIn post draft: \n\n{{ $('OpenAI Message').item.json.message.content }}\n\nApprove?          |
| Telegram Approved            | If                            | Checks if Telegram approval was granted            | Send message and wait for response | Wait for Time Selected (true branch) |                                                                                                                       |
| Wait for Time Selected       | Wait                          | Waits until the scheduled posting time             | Telegram Approved            | Create a post                 | At Specified Time expression\n\n{{ new Date().toISOString().split('T')[0] + 'T' + $('Random Time').item.json.postTime + ':00Z' }}\n\nTo test change to wait for 5 seconds then revert back |
| Create a post                | LinkedIn                      | Publishes post to LinkedIn                          | Wait for Time Selected       | Notify Post Active            |                                                                                                                       |
| Notify Post Active           | Telegram                      | Sends confirmation message to Telegram             | Create a post                | None                         |                                                                                                                       |
| When clicking ‘Execute workflow’ | Manual Trigger               | Manual trigger alternative entry point             | None                        | Get Day                      |                                                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Configure cron expression: `0 8 * * 1-4` to run at 8:00 AM UTC Monday through Thursday.  
   - Connect output to Get Day node.  

2. **Create Get Day node (Code)**  
   - Type: Code  
   - Paste JavaScript code to:  
     - Get current UTC day and hour  
     - Convert to Central Time day by subtracting 5 hours  
     - Mark Sunday (0), Friday (5), Saturday (6) as skip days  
     - Output JSON `{ skipToday: boolean }`  
   - Connect output to Check Day node.  

3. **Create Check Day node (If)**  
   - Type: If  
   - Condition: Check if `skipToday` is true  
   - True branch: No output (ends workflow)  
   - False branch: Connect to Prompts node  

4. **Create Prompts node (Set)**  
   - Type: Set  
   - Define six string variables `prompt1` to `prompt6` with detailed prompts for LinkedIn posts on AI, UCaaS, contact centers, including tone, style, and content instructions.  
   - Connect output to Random Prompt node.  

5. **Create Random Prompt node (Code)**  
   - Type: Code  
   - JavaScript code to select a random prompt from the six defined in Prompts node and output as `selectedPrompt`.  
   - Connect output to OpenAI Message node.  

6. **Create OpenAI Message node**  
   - Type: OpenAI GPT-4 (Langchain)  
   - Model: `chatgpt-4o-latest`  
   - Message content: Set to expression referencing `selectedPrompt` from Random Prompt node.  
   - Set OpenAI API credentials.  
   - Connect output to Random Time node.  

7. **Create Random Time node (Code)**  
   - Type: Code  
   - JavaScript code to randomly choose a time from `['09:00', '11:00', '15:00']` Central Time, convert to UTC ISO string, output both local and UTC times.  
   - Connect output to Send message and wait for response node.  

8. **Create Send message and wait for response node (Telegram)**  
   - Type: Telegram (Send and Wait)  
   - Message:  
     ```
     Here is your AI-generated LinkedIn post draft:

     {{ $('OpenAI Message').item.json.message.content }}

     Scheduled Post Time (Central Time): {{ $json.postTimeCentral }}

     Approve?
     ```  
   - Set approval type to double approval.  
   - Set Telegram API credentials.  
   - Connect output to Telegram Approved node.  

9. **Create Telegram Approved node (If)**  
   - Type: If  
   - Condition: `{{$json.data.approved}}` equals true  
   - True branch: Connect to Wait for Time Selected node  
   - False branch: No output (ends workflow)  

10. **Create Wait for Time Selected node (Wait)**  
    - Type: Wait  
    - Configure to wait for specific time using expression:  
      `{{ $('Random Time').item.json.postTimeUTC_ISO }}`  
    - Connect output to Create a post node.  

11. **Create Create a post node (LinkedIn)**  
    - Type: LinkedIn node  
    - Text: Set expression to `{{ $('OpenAI Message').item.json.message.content }}`  
    - Person: Set LinkedIn person ID (e.g., your own LinkedIn profile ID)  
    - Set LinkedIn OAuth2 credentials.  
    - Connect output to Notify Post Active node.  

12. **Create Notify Post Active node (Telegram)**  
    - Type: Telegram Send Message  
    - Message text: "Personal LinkedIn Post Active"  
    - Chat ID: Set to your Telegram chat ID  
    - Set Telegram API credentials.  
    - No further connection (end of workflow).  

13. **Optional: Create Manual Trigger node**  
    - Type: Manual Trigger  
    - Connect output to Get Day node for manual workflow execution.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                            | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| The schedule trigger checks to see if the time is 8AM                                                                                                | Sticky Note near Schedule Trigger node                                                                           |
| Check to see if this is a day we want to run the workflow. We are set to avoid Friday-Sunday. We also convert UTC to Central Time                   | Sticky Note near Get Day node                                                                                     |
| Set your OpenAI prompts to set your LinkedIn post.                                                                                                  | Sticky Note near Prompts node                                                                                     |
| Select a random prompt so that the posts look unique.                                                                                               | Sticky Note near Random Prompt node                                                                               |
| Select a Random Time                                                                                                                                | Sticky Note near Random Time node                                                                                 |
| At Specified Time expression:<br> `{{ new Date().toISOString().split('T')[0] + 'T' + $('Random Time').item.json.postTime + ':00Z' }}`<br>To test change to wait for 5 seconds then revert back | Sticky Note near Wait for Time Selected node                                                                      |
| Telegram message template includes the AI-generated LinkedIn post draft, scheduled time, and approval prompt.                                       | Sticky Note near Send message and wait for response node                                                         |

---

**Disclaimer:** The provided text stems exclusively from an automated workflow created with n8n, a workflow automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and publicly available.