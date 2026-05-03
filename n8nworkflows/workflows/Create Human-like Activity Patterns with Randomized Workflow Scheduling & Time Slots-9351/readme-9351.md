Create Human-like Activity Patterns with Randomized Workflow Scheduling & Time Slots

https://n8nworkflows.xyz/workflows/create-human-like-activity-patterns-with-randomized-workflow-scheduling---time-slots-9351


# Create Human-like Activity Patterns with Randomized Workflow Scheduling & Time Slots

### 1. Workflow Overview

This workflow is designed to create human-like activity patterns by scheduling and executing workflows at randomized times and in randomized time slots. It is particularly suited for scenarios where automation should mimic human unpredictability, such as scheduled tasks, email sending, or batch processing in a distributed or time-sensitive environment.

The workflow consists of several logical blocks:

- **1.1 Initialization & Scheduling**: Sets up the daily schedule trigger and initializes variables and environment for the day’s run.
- **1.2 Daily Planning Generation**: Calculates and creates the activity plan for the day, including randomized time slots and workflow executions.
- **1.3 Planning Persistence and Decision Logic**: Writes planning data to files, reads back the plan, and decides whether to proceed or send a notification of inactivity.
- **1.4 Execution Loop & Sub-Workflow Invocation**: Iterates over planned activities, executing sub-workflows for each scheduled action.
- **1.5 Monitoring, Logging, and Finalization**: Collects execution monitoring data, appends logs, sends success emails, and finalizes the process.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization & Scheduling

- **Overview:**  
  This block sets the daily schedule trigger to start the workflow at 1 AM every day and initializes necessary variables or environment states for the run.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Init  
  - Daily Scheduler

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Starts the workflow daily at 1 AM (default)  
    - Configuration: Triggers automatically at the specified time, no input required  
    - Outputs to: Init node  
    - Edge Cases: Misconfigured timezone may cause trigger to fire at unexpected times  

  - **Init**  
    - Type: Code Node (JavaScript)  
    - Role: Initializes environment variables or context for the day’s run  
    - Configuration: Custom scripting to prepare state  
    - Input: From Schedule Trigger  
    - Output: To Daily Scheduler  
    - Edge Cases: Script errors can stop the workflow, handled with onError continueErrorOutput  

  - **Daily Scheduler**  
    - Type: Code Node (JavaScript)  
    - Role: Generates daily random schedule planning, randomizes tasks and time slots  
    - Configuration: Uses code to produce planning data including time slots and activities  
    - Input: From Init  
    - Output: Two outputs—successful planning leads to Send Planning and Write Log File nodes; failure leads to ERROR: Daily Scheduler node  
    - Edge Cases: Logic errors, data inconsistencies, or exceptions in code can trigger error branch  

---

#### 2.2 Daily Planning Generation

- **Overview:**  
  This block processes the scheduled data created by the Daily Scheduler and persists the planning information to files. It also sends the planned schedule via email.

- **Nodes Involved:**  
  - Send planning (Email Send)  
  - Write Log File (Read/Write File)  
  - Write Planning File (Read/Write File)

- **Node Details:**

  - **Send planning**  
    - Type: Email Send  
    - Role: Sends an email containing the day’s planned schedule  
    - Configuration: Uses configured SMTP or email credentials; email body populated with planning details  
    - Input: From Daily Scheduler  
    - Output: To Write Log File node  
    - Edge Cases: Email sending failure due to SMTP issues or invalid recipient addresses  

  - **Write Log File**  
    - Type: Read/Write File  
    - Role: Logs the planning process details to a file on disk or storage  
    - Configuration: Writes to a predefined log file path  
    - Input: From Send planning  
    - Output: To Write Planning File  
    - Edge Cases: File system permissions, disk space, or path errors  

  - **Write Planning File**  
    - Type: Read/Write File  
    - Role: Writes the finalized planning data to a persistent planning file  
    - Configuration: Uses a specific file path and format (e.g., JSON or CSV)  
    - Input: From Write Log File  
    - Output: To Go/no go (Switch) node for decision  
    - Edge Cases: File write failures, concurrent access issues  

---

#### 2.3 Planning Persistence and Decision Logic

- **Overview:**  
  Reads back the planning file, evaluates whether there are planned activities, and branches the workflow accordingly to either proceed with execution or send a "Nothing Planned" email notification.

- **Nodes Involved:**  
  - Go/no go (Switch)  
  - Read Planning File (Read/Write File)  
  - NOTHING Planned Email (Email Send)

- **Node Details:**

  - **Read Planning File**  
    - Type: Read/Write File  
    - Role: Reads the planning file created earlier to retrieve the list of scheduled activities  
    - Configuration: Reads from the planning file path  
    - Input: From Append Planning File  
    - Output: To LOOP SUCCESS email node  
    - Edge Cases: File read failures, missing file, or corrupted data  

  - **Go/no go**  
    - Type: Switch  
    - Role: Evaluates if the planning file contains any planned items  
    - Configuration: Conditional logic based on file data presence or count  
    - Input: From Write Planning File  
    - Outputs:  
      - If planned activities exist, proceeds to Split Out node  
      - Else, routes to NOTHING Planned Email node  
    - Edge Cases: Expression evaluation failure  

  - **NOTHING Planned Email**  
    - Type: Email Send  
    - Role: Sends notification email indicating no activities were planned for the day  
    - Configuration: Email body and recipients configured to notify relevant parties  
    - Input: From Go/no go (switch, no activities branch)  
    - Output: None (terminal)  
    - Edge Cases: Email sending failure  

---

#### 2.4 Execution Loop & Sub-Workflow Invocation

- **Overview:**  
  This block splits the planned activities into individual items, loops over them in batches, and executes a sub-workflow for each scheduled activity to carry out the planned tasks.

- **Nodes Involved:**  
  - Split Out (Split Out)  
  - Loop Over Items (Split In Batches)  
  - Execute sub-process (Execute Workflow)  
  - Monitoring (Code)  
  - Get Sub-workflow Name (n8n node)  
  - ERROR: Execute Sub-worflow (Stop and Error)

- **Node Details:**

  - **Split Out**  
    - Type: Split Out  
    - Role: Splits the planning data into discrete items for processing  
    - Input: From Go/no go (switch, positive branch)  
    - Output: To Loop Over Items  
    - Edge Cases: Empty input leads to no output  

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Role: Processes planning items one batch at a time to control execution load  
    - Input: From Split Out  
    - Outputs:  
      - To Aggregate node for collecting results  
      - To Execute sub-process node for running each sub-workflow  
    - Edge Cases: Batch size misconfiguration, partial failures in batch processing  

  - **Execute sub-process**  
    - Type: Execute Workflow  
    - Role: Invokes a sub-workflow dynamically for each planned activity  
    - Configuration: Uses dynamic parameters possibly set by Get Sub-workflow Name  
    - Input: From Loop Over Items  
    - Outputs:  
      - Success to Monitoring node  
      - Failure to Get Sub-workflow Name (possibly for error handling)  
    - Error Handling: onError set to continueErrorOutput for resilience  
    - Edge Cases: Sub-workflow failure, authentication issues, timeout  

  - **Monitoring**  
    - Type: Code Node  
    - Role: Processes monitoring data returned from sub-workflow executions and appends to planning file  
    - Input: From Execute sub-process success output  
    - Output: To Append Planning File  
    - Error Handling: onError continueErrorOutput  
    - Edge Cases: Script or data errors  

  - **Get Sub-workflow Name**  
    - Type: n8n Core Node (n8n)  
    - Role: Possibly retrieves or sets the name of the sub-workflow to execute or handles error continuation  
    - Input: From Execute sub-process failure output  
    - Output: To ERROR: Execute Sub-worflow  
    - Edge Cases: Misconfiguration or missing sub-workflow names  

  - **ERROR: Execute Sub-worflow**  
    - Type: Stop and Error  
    - Role: Stops the workflow with error related to sub-workflow execution failure  
    - Input: From Get Sub-workflow Name  
    - Output: None (terminal)  

---

#### 2.5 Monitoring, Logging, and Finalization

- **Overview:**  
  Aggregates results from all sub-workflow executions, reads final planning and log files, merges data, extracts information, and sends final success notifications.

- **Nodes Involved:**  
  - Aggregate (Aggregate)  
  - Read Final Planning File (Read/Write File)  
  - Read Log File (Read/Write File)  
  - Merge (Merge)  
  - Extract from File (Extract from File)  
  - FINAL SUCCESS Email (Email Send)  
  - LOOP SUCCESS email (Email Send)  
  - Return Monitoring Data (Code)

- **Node Details:**

  - **Aggregate**  
    - Type: Aggregate  
    - Role: Collects batch processing results into a single data set  
    - Input: From Loop Over Items (batch success output)  
    - Output: To Read Final Planning File  
    - Edge Cases: Mismatched or missing aggregation data  

  - **Read Final Planning File**  
    - Type: Read/Write File  
    - Role: Reads the final planning file for summary and reporting  
    - Input: From Aggregate  
    - Outputs:  
      - To Merge node (main output 0)  
      - To Extract from File node (main output 1)  
    - Edge Cases: File access issues  

  - **Read Log File**  
    - Type: Read/Write File  
    - Role: Reads the log file for merging with final planning data  
    - Input: Independently triggered, connected to Merge node input 2  
    - Output: To Merge  
    - Edge Cases: File access or format issues  

  - **Merge**  
    - Type: Merge  
    - Role: Combines data from final planning, extracted data, and logs into a comprehensive dataset  
    - Inputs:  
      - Final Planning File (index 0)  
      - Extract from File (index 1)  
      - Read Log File (index 2)  
    - Output: To FINAL SUCCESS Email  
    - Edge Cases: Incompatible data formats or empty inputs  

  - **Extract from File**  
    - Type: Extract from File  
    - Role: Extracts specific data or metadata from the final planning file for reporting  
    - Input: From Read Final Planning File  
    - Output: To Merge  
    - Edge Cases: Extraction errors if file format changes  

  - **FINAL SUCCESS Email**  
    - Type: Email Send  
    - Role: Sends a final success notification email indicating completion of all scheduled activities  
    - Input: From Merge  
    - Output: None (terminal)  
    - Edge Cases: Email sending failures  

  - **LOOP SUCCESS email**  
    - Type: Email Send  
    - Role: Sends email notification after successful loop processing, uses environment variables after Split node  
    - Input: From Read Planning File  
    - Output: To Return Monitoring Data  
    - Edge Cases: Email failure  

  - **Return Monitoring Data**  
    - Type: Code Node  
    - Role: Returns or prepares monitoring data for next loop iteration or finalization  
    - Input: From LOOP SUCCESS email  
    - Output: To Loop Over Items  
    - Edge Cases: Script errors  

---

### 3. Summary Table

| Node Name               | Node Type               | Functional Role                            | Input Node(s)                 | Output Node(s)               | Sticky Note                                   |
|-------------------------|-------------------------|--------------------------------------------|------------------------------|------------------------------|-----------------------------------------------|
| Schedule Trigger        | Schedule Trigger        | Triggers workflow daily at 1 AM             | -                            | Init                         | At 1am every day (default)                     |
| Init                   | Code                    | Initializes environment for daily run       | Schedule Trigger             | Daily Scheduler              |                                               |
| Daily Scheduler        | Code                    | Generates daily randomized schedule         | Init                         | Send planning, Write Log File, ERROR: Daily Scheduler |                                               |
| Send planning          | Email Send              | Sends planning email                         | Daily Scheduler              | Write Log File               |                                               |
| Write Log File         | Read/Write File         | Writes log of planning process               | Send planning                | Write Planning File          |                                               |
| Write Planning File    | Read/Write File         | Persists schedule planning data              | Write Log File               | Go/no go                    |                                               |
| Go/no go               | Switch                  | Decides if plan exists; proceed or notify   | Write Planning File          | Split Out, NOTHING Planned Email |                                               |
| NOTHING Planned Email  | Email Send              | Sends "nothing planned" notification email  | Go/no go                    | -                           |                                               |
| Split Out              | Split Out               | Splits planning data into individual items  | Go/no go                    | Loop Over Items             |                                               |
| Loop Over Items        | Split In Batches        | Loops over items in batches                   | Split Out                   | Aggregate, Execute sub-process |                                               |
| Execute sub-process    | Execute Workflow        | Runs sub-workflows per planned item          | Loop Over Items             | Monitoring, Get Sub-workflow Name |                                               |
| Monitoring             | Code                    | Processes monitoring data from sub-workflows| Execute sub-process          | Append Planning File, ERROR: Monitoring |                                               |
| Append Planning File   | Read/Write File         | Appends monitoring data to planning file    | Monitoring                  | Read Planning File          |                                               |
| Read Planning File     | Read/Write File         | Reads planning file for loop success email  | Append Planning File         | LOOP SUCCESS email          |                                               |
| LOOP SUCCESS email     | Email Send              | Sends success notification after loop       | Read Planning File          | Return Monitoring Data      | Use $env after Split                           |
| Return Monitoring Data | Code                    | Returns monitoring data for next loop        | LOOP SUCCESS email          | Loop Over Items             |                                               |
| Get Sub-workflow Name  | n8n                     | Retrieves sub-workflow name or error handler| Execute sub-process          | ERROR: Execute Sub-worflow  |                                               |
| ERROR: Execute Sub-worflow | Stop and Error       | Stops workflow on sub-workflow execution error | Get Sub-workflow Name      | -                           |                                               |
| ERROR: Daily Scheduler | Stop and Error          | Stops workflow on daily scheduler error      | Daily Scheduler             | -                           |                                               |
| ERROR: Monitoring      | Stop and Error          | Stops workflow on monitoring error           | Monitoring                  | -                           |                                               |
| Aggregate              | Aggregate               | Aggregates batch processing results          | Loop Over Items             | Read Final Planning File    |                                               |
| Read Final Planning File | Read/Write File       | Reads final planning file                      | Aggregate                   | Merge (main 0), Extract from File (main 1) |                                               |
| Extract from File      | Extract from File       | Extracts data from final planning file        | Read Final Planning File    | Merge (main 1)              |                                               |
| Read Log File          | Read/Write File         | Reads log file for merge                      | -                          | Merge (main 2)              |                                               |
| Merge                  | Merge                   | Combines final planning, extract, and log data| Read Final Planning File, Extract from File, Read Log File | FINAL SUCCESS Email |                                               |
| FINAL SUCCESS Email    | Email Send              | Sends final success notification              | Merge                      | -                           |                                               |
| Sticky Note            | Sticky Note             | -                                              | -                          | -                           |                                               |
| Sticky Note1           | Sticky Note             | -                                              | -                          | -                           |                                               |
| Sticky Note2           | Sticky Note             | -                                              | -                          | -                           |                                               |
| Sticky Note3           | Sticky Note             | -                                              | -                          | -                           |                                               |
| Sticky Note5           | Sticky Note             | -                                              | -                          | -                           |                                               |
| Sticky Note6           | Sticky Note             | -                                              | -                          | -                           |                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set trigger time to 1 AM daily (default)  
   - Connect output to Init node  

2. **Create the Init node:**  
   - Type: Code (JavaScript)  
   - Add initialization script for environment or variables  
   - Connect output to Daily Scheduler node  

3. **Create the Daily Scheduler node:**  
   - Type: Code (JavaScript)  
   - Script generates randomized daily activity schedule with time slots  
   - Output 1 (success) connects to Send planning and Write Log File nodes  
   - Output 2 (error) connects to ERROR: Daily Scheduler node  

4. **Create the Send planning node:**  
   - Type: Email Send  
   - Configure SMTP or email credentials  
   - Compose email with daily planning details from Daily Scheduler output  
   - Connect output to Write Log File node  

5. **Create the Write Log File node:**  
   - Type: Read/Write File  
   - Configure file path for log file  
   - Connect output to Write Planning File node  

6. **Create the Write Planning File node:**  
   - Type: Read/Write File  
   - Configure file path for storing planning data  
   - Connect output to Go/no go node  

7. **Create the Go/no go node:**  
   - Type: Switch  
   - Add condition to check if planning data contains activities  
   - If yes, connect to Split Out node  
   - If no, connect to NOTHING Planned Email node  

8. **Create the NOTHING Planned Email node:**  
   - Type: Email Send  
   - Configure SMTP credentials and recipient(s)  
   - Compose "Nothing Planned" notification  
   - Terminal node (no output)  

9. **Create the Split Out node:**  
   - Type: Split Out  
   - Connect input from Go/no go (yes branch)  
   - Connect output to Loop Over Items node  

10. **Create the Loop Over Items node:**  
    - Type: Split In Batches  
    - Default batch size or as required  
    - Connect output 1 to Aggregate node  
    - Connect output 2 to Execute sub-process node  

11. **Create the Execute sub-process node:**  
    - Type: Execute Workflow  
    - Configure to execute a sub-workflow dynamically determined per item  
    - Set onError to continueErrorOutput  
    - Connect output 1 (success) to Monitoring node  
    - Connect output 2 (error) to Get Sub-workflow Name node  

12. **Create the Monitoring node:**  
    - Type: Code (JavaScript)  
    - Processes monitoring results from sub-workflow execution  
    - Set onError to continueErrorOutput  
    - Connect output 1 to Append Planning File node  
    - Connect output 2 to ERROR: Monitoring node  

13. **Create the Append Planning File node:**  
    - Type: Read/Write File  
    - Append monitoring data to planning file  
    - Connect output to Read Planning File node  

14. **Create the Read Planning File node:**  
    - Type: Read/Write File  
    - Reads the updated planning file  
    - Connect output to LOOP SUCCESS email node  

15. **Create the LOOP SUCCESS email node:**  
    - Type: Email Send  
    - Configure to send success notification after batch processing  
    - Use environment variables for dynamic content  
    - Connect output to Return Monitoring Data node  

16. **Create the Return Monitoring Data node:**  
    - Type: Code (JavaScript)  
    - Prepares monitoring data for next loop iteration  
    - Connect output back to Loop Over Items node to continue loop  

17. **Create the Get Sub-workflow Name node:**  
    - Type: n8n core node (n8n)  
    - Retrieves or validates sub-workflow name for execution errors  
    - Connect output to ERROR: Execute Sub-worflow node  

18. **Create the ERROR: Execute Sub-worflow node:**  
    - Type: Stop and Error  
    - Ends workflow on sub-workflow execution failure  

19. **Create the ERROR: Daily Scheduler node:**  
    - Type: Stop and Error  
    - Ends workflow on daily scheduler error  

20. **Create the ERROR: Monitoring node:**  
    - Type: Stop and Error  
    - Ends workflow on monitoring error  

21. **Create the Aggregate node:**  
    - Type: Aggregate  
    - Aggregates batch processing results  
    - Connect output to Read Final Planning File node  

22. **Create the Read Final Planning File node:**  
    - Type: Read/Write File  
    - Reads final planning data  
    - Connect outputs:  
      - Main output 0 to Merge node  
      - Main output 1 to Extract from File node  

23. **Create the Extract from File node:**  
    - Type: Extract from File  
    - Extracts specific data from planning file  
    - Connect output to Merge node  

24. **Create the Read Log File node:**  
    - Type: Read/Write File  
    - Reads the log file for final merge  
    - Connect output to Merge node  

25. **Create the Merge node:**  
    - Type: Merge  
    - Merges final planning, extracted data, and logs  
    - Connect output to FINAL SUCCESS Email node  

26. **Create the FINAL SUCCESS Email node:**  
    - Type: Email Send  
    - Sends final success notification email  
    - Terminal node (no output)  

27. **Add Sticky Notes strategically:**  
    - Document key points, trigger times, usage of environment variables, or important comments in the flow  

---

### 5. General Notes & Resources

| Note Content                                                                            | Context or Link                                      |
|-----------------------------------------------------------------------------------------|-----------------------------------------------------|
| "At 1am every day (default)"                                                           | Schedule Trigger node note                           |
| "Use $env after Split"                                                                  | LOOP SUCCESS email node note                         |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.