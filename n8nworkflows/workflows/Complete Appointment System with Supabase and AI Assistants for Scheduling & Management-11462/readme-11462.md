Complete Appointment System with Supabase and AI Assistants for Scheduling & Management

https://n8nworkflows.xyz/workflows/complete-appointment-system-with-supabase-and-ai-assistants-for-scheduling---management-11462


# Complete Appointment System with Supabase and AI Assistants for Scheduling & Management

### 1. Workflow Overview

This workflow is a **Complete Appointment System** designed for managing appointment scheduling, rescheduling, cancellation, and information retrieval using **Supabase** as the backend database and **AI assistants** (via Ollama Qwen3 language models) for intelligent decision-making. It listens for incoming HTTP POST requests on a webhook and processes multiple types of appointment-related actions in a unified workflow.

**Target Use Cases:**  
- Organizations or services handling appointments such as interview scheduling, class enrollment, or client meetings.  
- Automating appointment lifecycle management: creation, rescheduling, canceling, and querying user/class information.  
- Integrating AI for smart scheduling and data handling.

**Logical Blocks:**  

- **1.1 Input Reception and Action Routing**  
  Receives incoming API requests via Webhook, sets up action instructions, and routes to specific logic branches using a Switch node.

- **1.2 Appointment Scheduling Block**  
  Assigns interviewers based on availability, creates appointment records, and updates Supabase tables accordingly.

- **1.3 Appointment Rescheduling Block**  
  Handles rescheduling requests by finding suitable alternative slots, updating records, reinserting previous slots, and deleting old slots.

- **1.4 Appointment Cancellation Block**  
  Manages appointment cancellations, reassigns available slots back to interviewers, and deletes enroller records.

- **1.5 Class List Retrieval Block**  
  Retrieves a list of classes for a given user.

- **1.6 Enroller Information Retrieval Block**  
  Fetches detailed information about an enroller.

- **1.7 Email Validation and Notification Blocks**  
  Checks presence of email addresses before sending notifications for booking, rescheduling, and cancellation; skips sending if email is missing or already processed.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception and Action Routing

**Overview:**  
Receives appointment-related requests via a webhook, sets action instructions, and directs workflow execution based on the requested action type.

**Nodes Involved:**  
- Webhook  
- Appointment Actions (Set node)  
- Switch  

**Node Details:**  

- **Webhook**  
  - Type: HTTP Webhook (POST)  
  - Role: Entry point for receiving JSON requests with action name and arguments.  
  - Config: Path set for unique webhook ID, HTTP method POST, responseMode set to responseNode (responds after workflow completes).  
  - Input: Incoming HTTP POST JSON body.  
  - Output: JSON passed to Appointment Actions node.  
  - Edge Cases: Invalid HTTP method, malformed JSON, missing required fields.

- **Appointment Actions**  
  - Type: Set  
  - Role: Defines string instructions per action ("set_appointment", "reschedule", "cancel", "get_list", "get_user_info") embedding the incoming request parameters for AI processing.  
  - Config: Assigns multiple string fields with instructions templated using expressions referencing webhook JSON body args.  
  - Key Expressions: `{{$json.body.args.<field>}}` for dynamic substitution.  
  - Input: Data from Webhook node.  
  - Output: Augmented data with AI instruction fields passed to Switch node.

- **Switch**  
  - Type: Switch (version 3.2)  
  - Role: Routes execution based on the webhook body’s `name` field which specifies the action (e.g., set_appointment, reschedule, cancel, get_list, get_user_info).  
  - Config: Multiple strict string equality rules matching the `name`.  
  - Input: Data from Appointment Actions node.  
  - Output: Routes to one of the five AI agents or processes depending on the action.

---

#### 2.2 Appointment Scheduling Block

**Overview:**  
For "set_appointment" requests, assigns the earliest available interviewer, creates the appointment record in "Unitek_enrollers," and deletes the interviewer's availability slot.

**Nodes Involved:**  
- Assign Interviewer & Schedule (LangChain Agent)  
- Load Interviewers Table (Supabase)  
- Delete Interviewer Record (Supabase)  
- Create Interview Record (Supabase)  
- Respond Appointment Booking  
- Appointment Email (If node)  
- Skip If Email Exists (NoOp)  
- Skip If No Email (NoOp)  

**Node Details:**  

- **Assign Interviewer & Schedule**  
  - Type: LangChain Agent using Ollama Chat Model (Qwen3:14b)  
  - Role: Processes the "set_appointment" instructions, queries available interviewers, and decides which interviewer to assign.  
  - Config: Text input is the "set_appointment" instructions from Appointment Actions node, with no think mode enabled (`/no_think`).  
  - Input Connections: Switch node (for set_appointment action), also linked to Load Interviewers Table, Delete Interviewer Record, Create Interview Record as AI tools.  
  - Output: Passes data to Respond Appointment Booking node.  
  - Edge Cases: No interviewers available (returns error JSON), AI model timeout, malformed instructions.

- **Load Interviewers Table**  
  - Type: Supabase Tool (Get All)  
  - Role: Retrieves all available interviewers from "Unitek_interviewers" table.  
  - Config: Return all records, no filters.  
  - Credentials: Supabase API credentials configured.  
  - Input: Triggered as AI tool node by agent for data querying.  
  - Output: Supplies interviewers data for AI agent decision.

- **Delete Interviewer Record**  
  - Type: Supabase Tool (Delete)  
  - Role: Deletes interviewer’s availability slot once assigned.  
  - Config: Filters on interviewer name and availability date (dynamic values).  
  - Input: Data from AI agent outputs.  
  - Output: Confirms deletion.

- **Create Interview Record**  
  - Type: Supabase Tool (Create)  
  - Role: Creates a new appointment record in "Unitek_enrollers" with appointment details.  
  - Config: Sets fields such as enroller name, interviewer name, date, email, program, location, phone number, nationality id from webhook args and AI output.  
  - Input: Data from AI agent outputs.  
  - Output: Confirms creation.

- **Respond Appointment Booking**  
  - Type: Respond to Webhook  
  - Role: Sends final response to webhook caller with appointment info or errors.  
  - Input: From Assign Interviewer & Schedule node.

- **Appointment Email (If Node)**  
  - Type: If  
  - Role: Checks if email exists in the incoming request.  
  - Config: Condition tests if email string exists and is non-empty.  
  - Input: From Respond Appointment Booking node.  
  - Output: Routes to email sending or skip nodes.

- **Skip If Email Exists / Skip If No Email (NoOp)**  
  - Type: No Operation  
  - Role: Control flow nodes to manage email sending skip logic.  
  - Used to ensure email notifications are sent only if an email address is provided.

---

#### 2.3 Appointment Rescheduling Block

**Overview:**  
Handles rescheduling requests by finding suitable new appointment slots, reinserting old slots, updating enroller records, and deleting the new slot from interviewers.

**Nodes Involved:**  
- AI Rescheduling Agent (LangChain Agent)  
- Load Interviewers Table1 (Supabase)  
- Reinsert Previous Interview Slot (Supabase)  
- Update Enroller Record (Supabase)  
- Load Enroller Record (Supabase)  
- Delete New Interview Slot (Supabase)  
- Respond Appointment Reschedule  
- Reschedule Email (If)  
- Skip Reschedule If Email Exists (NoOp)  
- Skip Reschedule If No Email (NoOp)  

**Node Details:**  

- **AI Rescheduling Agent**  
  - Type: LangChain Agent using Ollama Chat Model1  
  - Role: Processes rescheduling instructions and manages logic to find closest available slots.  
  - Config: Uses instruction text from Appointment Actions node for "reschedule" action.  
  - Input: Routed from Switch node.  
  - Output: Passes data to Supabase nodes for table operations and final webhook response.

- **Load Interviewers Table1**  
  - Same as Load Interviewers Table, but used here for rescheduling context.

- **Reinsert Previous Interview Slot**  
  - Type: Supabase Tool (Create)  
  - Role: Inserts a new availability slot for the previous interviewer after rescheduling.  
  - Config: Sets interviewer name, available date, and unique ID from AI output.  
  - Input: Data from AI agent.  
  - Output: Confirms insertion.

- **Update Enroller Record**  
  - Type: Supabase Tool (Update)  
  - Role: Updates the enroller’s appointment details with new interviewer and date.  
  - Config: Filters by nationality id; updates interviewer name and date fields.  
  - Input: Data from AI agent.  
  - Output: Confirms update.

- **Load Enroller Record**  
  - Type: Supabase Tool (Get)  
  - Role: Fetches current enroller record for verification and use within rescheduling logic.  
  - Input: Webhook arguments.

- **Delete New Interview Slot**  
  - Type: Supabase Tool (Delete)  
  - Role: Deletes the newly assigned interviewer’s availability slot after rescheduling confirmation.  
  - Config: Filters by interviewer name and date (from AI output).  
  - Input: Data from AI agent.

- **Respond Appointment Reschedule**  
  - Type: Respond to Webhook  
  - Role: Sends rescheduling result back to the requester.

- **Reschedule Email (If Node)**  
  - Type: If  
  - Role: Checks presence of email to decide on sending reschedule notification.

- **Skip Reschedule If Email Exists / Skip Reschedule If No Email (NoOp)**  
  - Control nodes for email notification flow.

---

#### 2.4 Appointment Cancellation Block

**Overview:**  
Processes cancellation requests by retrieving enroller info, reinserting availability slots, deleting appointment records, and handling response notifications.

**Nodes Involved:**  
- AI Cancellation Agent (LangChain Agent)  
- Fetch Enroller Profile (Supabase)  
- Insert Available Interviewer (Supabase)  
- Delete Enroller Record (Supabase)  
- Respond Appointment Cancellation  
- Cancel Email (If)  
- Skip Cancellation If Email Exists (NoOp)  
- Skip Cancellation If No Email (NoOp)  

**Node Details:**  

- **AI Cancellation Agent**  
  - Type: LangChain Agent using Ollama Chat Model2  
  - Role: Executes cancellation instructions, manages data consistency in tables.  
  - Input: Routed from Switch node (cancel action).  
  - Output: Passes data for DB updates and webhook response.

- **Fetch Enroller Profile**  
  - Type: Supabase Tool (Get)  
  - Role: Retrieves enroller information by nationality id.  
  - Input: Webhook args.

- **Insert Available Interviewer**  
  - Type: Supabase Tool (Create)  
  - Role: Adds availability slot back to interviewers table for canceled appointment date.  
  - Config: Requires unique ID, interviewer name, and date.

- **Delete Enroller Record**  
  - Type: Supabase Tool (Delete)  
  - Role: Deletes the enroller’s appointment record.

- **Respond Appointment Cancellation**  
  - Type: Respond to Webhook  
  - Role: Sends cancellation confirmation.

- **Cancel Email (If Node)**  
  - Type: If  
  - Role: Conditional email notification sending based on presence of email.

- **Skip Cancellation If Email Exists / Skip Cancellation If No Email (NoOp)**  
  - Control nodes for cancellation email flow.

---

#### 2.5 Class List Retrieval Block

**Overview:**  
Fetches and returns the list of classes for a given enroller using AI to process instructions and Supabase data queries.

**Nodes Involved:**  
- AI Class List Agent (LangChain Agent)  
- Fetch Enroller Classes (Supabase)  
- Respond Class List Request  

**Node Details:**  

- **AI Class List Agent**  
  - Type: LangChain Agent using Ollama Chat Model3  
  - Role: Processes "get_list" instructions and queries classes table.  
  - Input: Routed from Switch node.  
  - Output: Data passed to respond node.

- **Fetch Enroller Classes**  
  - Type: Supabase Tool (Get)  
  - Role: Retrieves class data filtered by nationality number.  
  - Input: Webhook args.

- **Respond Class List Request**  
  - Type: Respond to Webhook  
  - Role: Sends list of classes as response.

---

#### 2.6 Enroller Information Retrieval Block

**Overview:**  
Fetches detailed information about an enroller upon request and returns it via webhook.

**Nodes Involved:**  
- AI Enroller Info Agent (LangChain Agent)  
- Fetch Enroller Record (Supabase)  
- Respond Enroller Info Request  

**Node Details:**  

- **AI Enroller Info Agent**  
  - Type: LangChain Agent using Ollama Chat Model4  
  - Role: Processes "get_user_info" instructions and queries enrollers table.  
  - Input: Routed from Switch node.  
  - Output: Data passed to respond node.

- **Fetch Enroller Record**  
  - Type: Supabase Tool (Get)  
  - Role: Retrieves enroller data filtered by nationality id.  
  - Input: Webhook args.

- **Respond Enroller Info Request**  
  - Type: Respond to Webhook  
  - Role: Sends detailed user info response.

---

### 3. Summary Table

| Node Name                  | Node Type                  | Functional Role                            | Input Node(s)             | Output Node(s)                       | Sticky Note                                                                                      |
|----------------------------|----------------------------|--------------------------------------------|---------------------------|-------------------------------------|-------------------------------------------------------------------------------------------------|
| Webhook                    | HTTP Webhook               | Receives incoming appointment requests     |                           | Appointment Actions                 | See Sticky Note: Workflow purpose, setup, and overview                                          |
| Appointment Actions        | Set                        | Sets AI instructions per action             | Webhook                   | Switch                            | See Sticky Note: Workflow purpose, setup, and overview                                          |
| Switch                    | Switch                     | Routes requests based on action type        | Appointment Actions       | Assign Interviewer & Schedule, AI Rescheduling Agent, AI Cancellation Agent, AI Class List Agent, AI Enroller Info Agent |                                                                                                 |
| Assign Interviewer & Schedule | LangChain Agent (Ollama)  | Handles appointment scheduling logic        | Switch                    | Respond Appointment Booking        | Sticky Note1: Schedule Interviews Section                                                      |
| Load Interviewers Table    | Supabase Tool (Get All)    | Loads available interviewers                 | AI tool from Assign Interviewer & Schedule | Assign Interviewer & Schedule | Sticky Note1: Schedule Interviews Section                                                      |
| Delete Interviewer Record  | Supabase Tool (Delete)     | Deletes assigned interviewer availability    | AI tool from Assign Interviewer & Schedule | Assign Interviewer & Schedule | Sticky Note1: Schedule Interviews Section                                                      |
| Create Interview Record    | Supabase Tool (Create)     | Creates appointment record                    | AI tool from Assign Interviewer & Schedule | Assign Interviewer & Schedule | Sticky Note1: Schedule Interviews Section                                                      |
| Respond Appointment Booking | Respond to Webhook         | Sends booking confirmation                    | Assign Interviewer & Schedule | Appointment Email                 | Sticky Note2: Email Validation Section                                                        |
| Appointment Email          | If                         | Checks if email exists for booking            | Respond Appointment Booking | Skip If Email Exists, Skip If No Email | Sticky Note2: Email Validation Section                                                        |
| Skip If Email Exists       | NoOp                       | Skips if email already exists                 | Appointment Email          |                                   | Sticky Note2: Email Validation Section                                                        |
| Skip If No Email           | NoOp                       | Skips if email missing                        | Appointment Email          |                                   | Sticky Note2: Email Validation Section                                                        |
| AI Rescheduling Agent      | LangChain Agent (Ollama)   | Handles rescheduling logic                     | Switch                    | Respond Appointment Reschedule     | Sticky Note3: Appointment Rescheduling Section                                                |
| Load Interviewers Table1   | Supabase Tool (Get All)    | Loads interviewers for rescheduling           | AI tool from AI Rescheduling Agent | AI Rescheduling Agent            | Sticky Note3: Appointment Rescheduling Section                                                |
| Reinsert Previous Interview Slot | Supabase Tool (Create)     | Reinserts old availability slot after reschedule | AI tool from AI Rescheduling Agent | AI Rescheduling Agent            | Sticky Note3: Appointment Rescheduling Section                                                |
| Update Enroller Record     | Supabase Tool (Update)     | Updates enroller appointment details          | AI tool from AI Rescheduling Agent | AI Rescheduling Agent            | Sticky Note3: Appointment Rescheduling Section                                                |
| Load Enroller Record       | Supabase Tool (Get)        | Loads current enroller record                  | AI tool from AI Rescheduling Agent | AI Rescheduling Agent            | Sticky Note3: Appointment Rescheduling Section                                                |
| Delete New Interview Slot  | Supabase Tool (Delete)     | Deletes newly assigned interviewer slot        | AI tool from AI Rescheduling Agent | AI Rescheduling Agent            | Sticky Note3: Appointment Rescheduling Section                                                |
| Respond Appointment Reschedule | Respond to Webhook         | Sends reschedule confirmation                   | AI Rescheduling Agent      | Reschedule Email                   | Sticky Note4: Appointment Reschedule Notification Section                                     |
| Reschedule Email           | If                         | Checks email presence for reschedule notification | Respond Appointment Reschedule | Skip Reschedule If Email Exists, Skip Reschedule If No Email | Sticky Note4: Appointment Reschedule Notification Section                                     |
| Skip Reschedule If Email Exists | NoOp                       | Skip sending reschedule email if already sent   | Reschedule Email           |                                   | Sticky Note4: Appointment Reschedule Notification Section                                     |
| Skip Reschedule If No Email | NoOp                       | Skip sending reschedule email if no email       | Reschedule Email           |                                   | Sticky Note4: Appointment Reschedule Notification Section                                     |
| AI Cancellation Agent     | LangChain Agent (Ollama)   | Handles cancellation logic                      | Switch                    | Respond Appointment Cancellation   | Sticky Note5: Appointment Cancellation Section                                               |
| Fetch Enroller Profile     | Supabase Tool (Get)        | Retrieves enroller info for cancellation        | AI tool from AI Cancellation Agent | AI Cancellation Agent           | Sticky Note5: Appointment Cancellation Section                                               |
| Insert Available Interviewer | Supabase Tool (Create)     | Re-adds interviewer availability after cancel   | AI tool from AI Cancellation Agent | AI Cancellation Agent           | Sticky Note5: Appointment Cancellation Section                                               |
| Delete Enroller Record    | Supabase Tool (Delete)     | Deletes enroller record on cancellation          | AI tool from AI Cancellation Agent | AI Cancellation Agent           | Sticky Note5: Appointment Cancellation Section                                               |
| Respond Appointment Cancellation | Respond to Webhook         | Sends cancellation confirmation                   | AI Cancellation Agent      | Cancel Email                      | Sticky Note6: Cancellation Email Handling                                                   |
| Cancel Email              | If                         | Checks email presence for cancellation notification | Respond Appointment Cancellation | Skip Cancellation If Email Exists, Skip Cancellation If No Email | Sticky Note6: Cancellation Email Handling                                                   |
| Skip Cancellation If Email Exists | NoOp                       | Skip sending cancellation email if already sent   | Cancel Email               |                                   | Sticky Note6: Cancellation Email Handling                                                   |
| Skip Cancellation If No Email | NoOp                       | Skip sending cancellation email if no email       | Cancel Email               |                                   | Sticky Note6: Cancellation Email Handling                                                   |
| AI Class List Agent       | LangChain Agent (Ollama)   | Handles fetching class list                        | Switch                    | Respond Class List Request         | Sticky Note7: Class List Retrieval                                                          |
| Fetch Enroller Classes    | Supabase Tool (Get)        | Gets classes for enroller                          | AI tool from AI Class List Agent | AI Class List Agent              | Sticky Note7: Class List Retrieval                                                          |
| Respond Class List Request | Respond to Webhook         | Sends list of classes                               | AI Class List Agent        |                                 | Sticky Note7: Class List Retrieval                                                          |
| AI Enroller Info Agent    | LangChain Agent (Ollama)   | Provides detailed enroller info                      | Switch                    | Respond Enroller Info Request     | Sticky Note8: Enroller Information Retrieval                                               |
| Fetch Enroller Record     | Supabase Tool (Get)        | Fetches enroller detailed record                    | AI tool from AI Enroller Info Agent | AI Enroller Info Agent           | Sticky Note8: Enroller Information Retrieval                                               |
| Respond Enroller Info Request | Respond to Webhook         | Sends detailed enroller information                  | AI Enroller Info Agent     |                                 | Sticky Note8: Enroller Information Retrieval                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Type: HTTP Webhook  
   - HTTP Method: POST  
   - Path: Unique path (e.g., "46161ac6-9838-4b3d-ad4a-c8d62362a656")  
   - Response Mode: responseNode

2. **Create Set Node "Appointment Actions":**  
   - Create multiple string fields:  
     - `set_appointment`: Template instructions with variables from webhook body args (name, nationality_number, phone_number, email, location, program) instructing AI to assign earliest available interviewer and create appointment.  
     - `reschedule`: Instructions to reschedule appointment with preferred date and steps including table lookups and updates.  
     - `cancel`: Instructions to cancel appointment, reinsert availability, and delete enroller record.  
     - `get_list`: Instructions to get class list for user.  
     - `get_user_info`: Instructions to get detailed user information.

3. **Create Switch Node:**  
   - Input: Output of Appointment Actions  
   - Conditions: Strict string equals on `{{$json.body.name}}` for values: "set_appointment", "reschedule", "cancel", "get_list", "get_user_info"  
   - Each output branch will connect to respective AI agent nodes.

4. **Create Ollama Chat Model Nodes (Qwen3:14b):**  
   - One per action:  
     - "Assign Interviewer & Schedule" (for set_appointment)  
     - "AI Rescheduling Agent" (for reschedule)  
     - "AI Cancellation Agent" (for cancel)  
     - "AI Class List Agent" (for get_list)  
     - "AI Enroller Info Agent" (for get_user_info)  
   - Configure each to receive the corresponding instruction string from Appointment Actions node.

5. **Appointment Scheduling Sub-Workflow:**  
   - Create Supabase Tool node to get all from "Unitek_interviewers".  
   - Create Supabase Tool node to delete interviewer record (filter by interviewer name and availability).  
   - Create Supabase Tool node to create a new record in "Unitek_enrollers" with appointment data.  
   - Connect these Supabase nodes as AI tools triggered by the "Assign Interviewer & Schedule" agent.  
   - Create Respond to Webhook node to send back booking confirmation.  
   - Create If node "Appointment Email" to check existence of email in webhook args.  
   - Create two NoOp nodes to skip email sending if email exists or missing.  
   - Connect Respond node output to If node, and then to NoOps accordingly.

6. **Appointment Rescheduling Sub-Workflow:**  
   - Create Supabase Tool nodes:  
     - Get all from "Unitek_interviewers"  
     - Create new availability for previous interviewer (Reinsert Previous Interview Slot)  
     - Update enroller record in "Unitek_enrollers" (update interviewer name and date)  
     - Get enroller record by nationality id  
     - Delete new interviewer availability slot  
   - Connect these as AI tools triggered by "AI Rescheduling Agent".  
   - Create Respond to Webhook node for reschedule confirmation.  
   - Create If node "Reschedule Email" to check email presence.  
   - Create two NoOp nodes to skip email sending as needed.

7. **Appointment Cancellation Sub-Workflow:**  
   - Create Supabase Tool nodes:  
     - Get enroller profile by nationality id  
     - Insert available interviewer slot (unique id, name, date)  
     - Delete enroller record  
   - Connect these as AI tools triggered by "AI Cancellation Agent".  
   - Create Respond to Webhook node for cancellation confirmation.  
   - Create If node "Cancel Email" to check email presence.  
   - Create two NoOp nodes to skip email sending accordingly.

8. **Class List Retrieval Sub-Workflow:**  
   - Create Supabase Tool node to get classes from "Unitek_classes" filtered by nationality number.  
   - Connect as AI tool triggered by "AI Class List Agent".  
   - Create Respond to Webhook node to send class list.

9. **Enroller Information Retrieval Sub-Workflow:**  
   - Create Supabase Tool node to get enroller record filtered by nationality id.  
   - Connect as AI tool triggered by "AI Enroller Info Agent".  
   - Create Respond to Webhook node to send detailed information.

10. **Credential Setup:**  
    - Configure Supabase API credentials with correct API keys and project URLs.  
    - Configure Ollama nodes to connect to the Qwen3:14b model with necessary access.

11. **Connections:**  
    - Connect nodes according to the logic detailed above, with AI agents triggering Supabase nodes as AI tools, and Supabase nodes feeding data back to agents.  
    - Switch node routes action to corresponding AI agent.  
    - AI agent main output connects to Respond nodes.  
    - Respond nodes connect to respective email validation If nodes and NoOp nodes for flow control.

12. **Testing:**  
    - Test each action by sending POST requests with JSON bodies including `name` (action) and `args` with required fields as per instructions.

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow is designed for organizations or services managing appointments such as interviews, class enrollments, or client meetings. | Sticky Note (Node: Sticky Note at position [-912,-368])                                         |
| Requires n8n environment with configured Supabase access and Ollama Qwen3 language model credentials.                                |                                                                                                 |
| Instructions embedded in the Set node ("Appointment Actions") guide AI agents to follow strict table checking, updating, and error handling. |                                                                                                 |
| See Supabase documentation for table schema design: interviewers, enrollers, classes.                                                | https://supabase.com/docs                                                                        |
| Ollama Qwen3 14b is used as the language model for AI agents.                                                                        | https://ollama.com/models/qwen3                                                                 |

---

**Disclaimer:** The provided content originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.