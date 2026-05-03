Automatically Archive Old WordPress Posts to Draft Status

https://n8nworkflows.xyz/workflows/automatically-archive-old-wordpress-posts-to-draft-status-7418


# Automatically Archive Old WordPress Posts to Draft Status

### 1. Workflow Overview

This workflow automates the archival of old WordPress posts by changing their status to draft and tagging them as archived content. It is designed to run quarterly—specifically on the first day of every third month. The main purpose is to keep a WordPress site clean and up-to-date by automatically identifying and archiving posts older than one year.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow quarterly using a cron expression.
- **1.2 Data Retrieval:** Queries WordPress to find all published posts older than 12 months.
- **1.3 Decision Making:** Checks if any posts were found to archive.
- **1.4 Archival Process:** Updates each identified post's status to "draft" and adds relevant tags.
- **1.5 Notification & Logging:** Sends an email notification upon completion if posts were archived, or logs a no-operation if no posts required archiving.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the workflow execution automatically on a quarterly schedule.

- **Nodes Involved:**  
  - Quarterly Trigger

- **Node Details:**  
  - **Quarterly Trigger**  
    - Type: Schedule Trigger  
    - Role: Starts the workflow every 3 months on day 1 at midnight.  
    - Configuration: Cron expression `0 0 1 */3 *` (At 00:00 on day 1 every 3rd month).  
    - Inputs: None (trigger node).  
    - Outputs: Connects to "Find Old Posts" node.  
    - Edge Cases: If n8n instance is down at trigger time, the execution might be missed unless enabled to catch-up.  
    - Version: 1.2 (standard schedule trigger capabilities).

#### 1.2 Data Retrieval

- **Overview:**  
  This block queries WordPress to fetch all published posts older than 12 months from the current date.

- **Nodes Involved:**  
  - Find Old Posts

- **Node Details:**  
  - **Find Old Posts**  
    - Type: WordPress node  
    - Role: Retrieves posts based on age and status criteria.  
    - Configuration:  
      - Operation: `getAll` posts  
      - Filters: Posts with status `publish` and published before the date 12 months ago from workflow execution time (`{{ $now.minus({ months: 12 }).toISO() }}`)  
      - Ordering: Ascending order (oldest posts first)  
    - Inputs: Connected from "Quarterly Trigger".  
    - Outputs: Connects to "Check if Posts Found".  
    - Edge Cases:  
      - No posts returned (empty array).  
      - WordPress API limits or authentication errors.  
      - Date calculation errors if `$now` is undefined or incorrectly formatted.  
    - Version: 1

#### 1.3 Decision Making

- **Overview:**  
  This block evaluates whether any posts were returned from the previous node to decide the next steps.

- **Nodes Involved:**  
  - Check if Posts Found

- **Node Details:**  
  - **Check if Posts Found**  
    - Type: If node  
    - Role: Conditional branching based on the number of posts found.  
    - Configuration:  
      - Condition: Checks if the array length of input data `$json.length` is greater than 0.  
      - True branch: Proceed to archive posts.  
      - False branch: Log that no posts were found.  
    - Inputs: Connected from "Find Old Posts".  
    - Outputs:  
      - True: Connected to "Archive Post".  
      - False: Connected to "Log No Posts".  
    - Edge Cases:  
      - Expression evaluation failure if input data is malformed.  
      - Unexpected data structure.  
    - Version: 2

#### 1.4 Archival Process

- **Overview:**  
  This block updates each identified post by changing its status to "draft" and tagging it as archived.

- **Nodes Involved:**  
  - Archive Post

- **Node Details:**  
  - **Archive Post**  
    - Type: WordPress node  
    - Role: Updates posts individually to archive them.  
    - Configuration:  
      - Operation: `update`  
      - Post ID: Dynamically set to the current post’s `id` (`={{ $json.id }}`).  
      - Update Fields:  
        - Status: `draft` (unpublishes the post)  
        - Tags: Adds `archived,old-content` tags.  
    - Inputs: Connected from "Check if Posts Found" (true branch).  
    - Outputs: Connects to "Send Notification".  
    - Edge Cases:  
      - Failure to update due to permissions or API rate limits.  
      - Posts without IDs or invalid IDs.  
    - Version: 1

#### 1.5 Notification & Logging

- **Overview:**  
  This block handles post-archival notifications via email or logs a no-operation if no posts were archived.

- **Nodes Involved:**  
  - Send Notification  
  - Log No Posts

- **Node Details:**  
  - **Send Notification**  
    - Type: Email Send node  
    - Role: Sends an email confirming the archival process completion.  
    - Configuration:  
      - Subject: "Quarterly Archive Complete"  
      - SMTP Credentials: Uses preconfigured SMTP account.  
      - No email body configured (default or empty).  
    - Inputs: Connected from "Archive Post".  
    - Outputs: None (end of workflow).  
    - Edge Cases:  
      - SMTP authentication or connection errors.  
      - Email delivery failures or spam filtering.  
    - Version: 2.1

  - **Log No Posts**  
    - Type: No Operation (NoOp) node  
    - Role: Does nothing but serves as a placeholder/log for the false condition (no posts to archive).  
    - Configuration: None.  
    - Inputs: Connected from "Check if Posts Found" (false branch).  
    - Outputs: None.  
    - Edge Cases: None.  
    - Version: 1

---

### 3. Summary Table

| Node Name          | Node Type              | Functional Role               | Input Node(s)       | Output Node(s)         | Sticky Note                                                                                                      |
|--------------------|------------------------|------------------------------|---------------------|------------------------|-----------------------------------------------------------------------------------------------------------------|
| Quarterly Trigger   | Schedule Trigger       | Initiates quarterly workflow | None                | Find Old Posts         |                                                                                                                 |
| Find Old Posts      | WordPress              | Retrieves old published posts | Quarterly Trigger   | Check if Posts Found   |                                                                                                                 |
| Check if Posts Found| If                     | Checks if posts were found   | Find Old Posts       | Archive Post, Log No Posts |                                                                                                                 |
| Archive Post       | WordPress              | Updates posts to draft status | Check if Posts Found | Send Notification      |                                                                                                                 |
| Send Notification  | Email Send             | Sends completion notification| Archive Post         | None                   |                                                                                                                 |
| Log No Posts       | No Operation (NoOp)    | Logs when no posts found     | Check if Posts Found | None                   |                                                                                                                 |
| Sticky Note        | Sticky Note            | Workflow overview and tips   | None                | None                   | ## Workflow Overview  Author: David Olusola  This workflow automatically archives old WordPress posts every quarter. Schedule: Runs on the 1st day of every 3rd month (quarterly) Steps: 1. Trigger executes quarterly 2. Finds old posts from WordPress 3. Archives the identified posts --- Need n8n Coaching? 🎯 Get Professional Help: - Workflow optimization - Advanced automation strategies - Custom node development - Error handling & debugging - Performance improvements 📧 [Contact for n8n coaching and consultation services!](mailto:david@daexai.com) --- 💡 Tips: - Add filters to "Find Old Posts" to specify criteria - Consider adding error handling nodes - Test with a smaller batch first |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Quarterly Trigger" Node:**  
   - Type: Schedule Trigger  
   - Parameters: Set the cron expression to `0 0 1 */3 *` (runs at midnight on the first day every 3rd month).  
   - No credentials required.  

2. **Create "Find Old Posts" Node:**  
   - Type: WordPress  
   - Credentials: Configure WordPress API credentials with appropriate permissions.  
   - Operation: `getAll`  
   - Options:  
     - Order: Ascending  
     - Before: Use expression `{{ $now.minus({ months: 12 }).toISO() }}` to filter posts published before 12 months ago.  
     - Status: `publish`  
   - Connect input from "Quarterly Trigger".

3. **Create "Check if Posts Found" Node:**  
   - Type: If  
   - Condition: Use expression `{{ $json.length }}` with operator "greater than" 0.  
   - Connect input from "Find Old Posts".

4. **Create "Archive Post" Node:**  
   - Type: WordPress  
   - Credentials: Same as "Find Old Posts".  
   - Operation: `update`  
   - Post ID: Use expression `={{ $json.id }}` to update each post dynamically.  
   - Update Fields:  
     - Status: `draft`  
     - Tags: `archived,old-content`  
   - Connect input from "Check if Posts Found" (true branch).

5. **Create "Send Notification" Node:**  
   - Type: Email Send  
   - Credentials: Configure SMTP credentials (e.g., username, password, host, port).  
   - Subject: "Quarterly Archive Complete"  
   - Connect input from "Archive Post".

6. **Create "Log No Posts" Node:**  
   - Type: No Operation (NoOp)  
   - No parameters needed.  
   - Connect input from "Check if Posts Found" (false branch).

7. **Connect Nodes:**  
   - Connect "Quarterly Trigger" output to "Find Old Posts".  
   - Connect "Find Old Posts" output to "Check if Posts Found".  
   - Connect "Check if Posts Found" true output to "Archive Post".  
   - Connect "Archive Post" output to "Send Notification".  
   - Connect "Check if Posts Found" false output to "Log No Posts".

8. **Add a Sticky Note (Optional):**  
   - Add a sticky note describing the workflow overview, schedule, and tips for future improvements.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                  |
|----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Workflow automatically archives old WordPress posts quarterly by changing their status to draft and tagging as archived. | Workflow purpose                                |
| Runs on the 1st day of every 3rd month (quarterly) using cron `0 0 1 */3 *`.                                           | Scheduling setup                                |
| Suggestions: Add more filters to "Find Old Posts" for finer control, add error handling nodes, and test with small batches first. | Optimization tips                               |
| Author & Contact for n8n coaching: David Olusola, email: david@daexai.com                                            | Author and consulting services                   |

---

*Disclaimer:* The provided content originates solely from an automated workflow created with n8n, a workflow automation tool. It strictly adheres to current content policies and contains no illegal, offensive, or protected material. All handled data is legal and public.