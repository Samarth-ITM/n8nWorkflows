Hostinger Daily VPS Snapshot and server metrics

https://n8nworkflows.xyz/workflows/hostinger-daily-vps-snapshot-and-server-metrics-3807


# Hostinger Daily VPS Snapshot and server metrics

---

### 1. Workflow Overview

This n8n workflow automates the daily creation of snapshots for Hostinger VPS servers and sends detailed server metrics and execution status notifications via WhatsApp (using the Evolution API). Its primary goal is to maintain up-to-date VPS backups by replacing the previous snapshot (as Hostinger only allows one snapshot per VPS) and provide real-time operational insights and alerting.

**Target Use Cases:**
- Automated VPS backup management for Hostinger users.
- Real-time monitoring and alerting of VPS snapshot status and server metrics.
- Self-hosted automation integration using n8n, suitable for DevOps or system administrators managing VPS infrastructure on Hostinger.

**Logical Blocks:**

- **1.1 Trigger & VPS Listing**: Initiates the workflow either manually or on a daily schedule, then lists all VPS instances.
- **1.2 VPS Filtering & Metrics Retrieval**: Filters running VPS instances and fetches their performance metrics.
- **1.3 Metrics Calculation**: Processes raw metrics into human-readable formats.
- **1.4 Snapshot Creation Loop**: Iteratively creates snapshots for each VPS, handling rate limits by waiting between operations.
- **1.5 Notification Dispatch**: Sends success or error notifications via WhatsApp with detailed snapshot and VPS metrics info.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & VPS Listing

- **Overview:**  
  This block starts the workflow either manually or on a schedule, then retrieves a list of VPS servers from Hostinger via API.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Every day 4:00am (Schedule Trigger)  
  - Hostinger API list VPS (Hostinger API node)  
  - Split Out (Split Out node)  

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - *Type:* Manual Trigger  
    - *Role:* Allows manual initiation for testing/debugging.  
    - *Connections:* Outputs to Hostinger API list VPS.  
    - *Edge Cases:* No input data; manual operation only.

  - **Every day 4:00am**  
    - *Type:* Schedule Trigger  
    - *Role:* Automatically triggers the workflow daily at 4 AM server time.  
    - *Configuration:* Trigger at hour 4, no minute specified (defaults to 0).  
    - *Connections:* Outputs to Hostinger API list VPS.  
    - *Edge Cases:* Timezone considerations; ensure n8n server time matches intended schedule.

  - **Hostinger API list VPS**  
    - *Type:* Hostinger API node (Community node)  
    - *Role:* Fetches the list of VPS instances using Hostinger API’s `listVms` action.  
    - *Credentials:* Hostinger API credential must be configured and selected.  
    - *Connections:* Outputs to Split Out.  
    - *Edge Cases:* API authentication errors, network timeouts, empty VPS list.  
    - *Version Requirements:* Community Hostinger API node installed.

  - **Split Out**  
    - *Type:* Split Out node  
    - *Role:* Flattens the array of VPS instances returned by the API into individual items for subsequent processing.  
    - *Configuration:* Splits on field `response`.  
    - *Connections:* Outputs to Avaliables VPS filter node.  
    - *Edge Cases:* Empty or malformed `response` field.

---

#### 2.2 VPS Filtering & Metrics Retrieval

- **Overview:**  
  Filters VPS instances to keep only those currently running, then fetches server metrics for each running VPS.

- **Nodes Involved:**  
  - Avaliables VPS (Filter node)  
  - Get VPS Metrics (Hostinger API node)  

- **Node Details:**

  - **Avaliables VPS**  
    - *Type:* Filter node  
    - *Role:* Filters VPS where the state equals `"running"`.  
    - *Configuration:* Condition: `{{$json.state}} === "running"` (strict string equality).  
    - *Connections:* Outputs to Get VPS Metrics.  
    - *Edge Cases:* VPS with other states (e.g., stopped, suspended) are excluded; filter requires exact match.

  - **Get VPS Metrics**  
    - *Type:* Hostinger API node  
    - *Role:* Calls `getVmMetrics` API action to retrieve recent metrics for each VPS.  
    - *Parameters:*  
      - `date_from`: 30 minutes ago (ISO timestamp)  
      - `date_to`: current time (ISO timestamp)  
      - `virtualMachineId`: dynamic from filtered VPS item (`{{ $('Avaliables VPS').item.json.id }}`).  
    - *Credentials:* Hostinger API credential reused.  
    - *Connections:* Outputs to Calculate metrics.  
    - *Edge Cases:* API errors, VPS metrics unavailable, invalid time range.

---

#### 2.3 Metrics Calculation

- **Overview:**  
  Converts raw VPS metrics into readable, formatted values for inclusion in notifications.

- **Nodes Involved:**  
  - Calculate metrics (Set node)  

- **Node Details:**

  - **Calculate metrics**  
    - *Type:* Set node (data transformation)  
    - *Role:* Assigns and formats multiple metrics from VPS JSON data into user-friendly strings:  
      - VPS ID, plan, hostname  
      - CPU count  
      - Disk size (GB)  
      - External IP address  
      - Operating system name  
      - CPU usage (last value with a percent sign)  
      - RAM usage: formatted as GB used, percentage used, and GB free  
      - Disk usage: used GB, percentage used, GB free  
      - Uptime: days and hours  
    - *Expressions:* Uses last values from metric arrays and calculations for percentages and unit conversions.  
    - *Connections:* Outputs to Loop Over Items node.  
    - *Edge Cases:* Missing or incomplete metric data, division by zero if memory/disk info missing.

---

#### 2.4 Snapshot Creation Loop

- **Overview:**  
  Processes each VPS in batches, inserting a wait time to respect API limits, creates a snapshot, and handles continuation.

- **Nodes Involved:**  
  - Loop Over Items (Split In Batches node)  
  - WhatsApp Message (success) (Evolution API node)  
  - Create Snapshot (Hostinger API node)  
  - Wait 2 seg (Wait node)  
  - Next VPS (NoOp node)  
  - WhatsApp Message (error) (Evolution API node)  

- **Node Details:**

  - **Loop Over Items**  
    - *Type:* Split In Batches node  
    - *Role:* Processes VPS items one by one (default batch size assumed 1) to avoid API flooding.  
    - *Connections:*  
      - Main output #0 to WhatsApp Message (success)  
      - Main output #1 to Create Snapshot  
    - *Edge Cases:* Large VPS lists may cause long execution times.

  - **WhatsApp Message (success)**  
    - *Type:* Evolution API node  
    - *Role:* Sends a WhatsApp message reporting snapshot success including VPS metrics.  
    - *Message Content:* Includes snapshot status, timestamp (São Paulo timezone), server name, IP, CPUs, RAM, disk usage, OS, and uptime.  
    - *Credentials:* Evolution API credential required.  
    - *Connections:* None (end node).  
    - *Edge Cases:* API authentication errors, network issues, invalid phone numbers.

  - **Create Snapshot**  
    - *Type:* Hostinger API node  
    - *Role:* Creates a snapshot for the VPS (`createSnapshot` action).  
    - *Parameters:* VPS ID dynamically from current item.  
    - *Credentials:* Hostinger API credential reused.  
    - *Connections:*  
      - On success: to Next VPS node  
      - On error: to WhatsApp Message (error) node (using `continueOnError` to avoid workflow stop).  
    - *Edge Cases:* API errors, snapshot creation failures, rate limiting.

  - **Wait 2 seg**  
    - *Type:* Wait node  
    - *Role:* Pauses workflow execution for 2 seconds between VPS snapshot operations to respect API limits.  
    - *Connections:* Outputs to Loop Over Items (to continue next batch).  
    - *Edge Cases:* Workflow timeout if many VPS processed.

  - **Next VPS**  
    - *Type:* NoOp node (placeholder)  
    - *Role:* Connects Create Snapshot success output to Wait 2 seg, enabling sequential processing.  
    - *Connections:* Outputs to Wait 2 seg.  

  - **WhatsApp Message (error)**  
    - *Type:* Evolution API node  
    - *Role:* Sends an error notification if snapshot creation fails, including VPS metrics and error status.  
    - *Credentials:* Evolution API credential reused.  
    - *Connections:* None (end node).  
    - *Edge Cases:* Same as WhatsApp Message (success).

---

#### 2.5 Sticky Notes (Documentation & Instruction)

- **Overview:**  
  The workflow includes multiple sticky notes describing prerequisites, VPS metrics available, credential setup, and usage instructions to help users configure and understand the workflow.

- **Nodes Involved:**  
  - Sticky Note6 (Header instructions)  
  - Sticky Note1 (Hostinger API credential reminder)  
  - Sticky Note2 (Hostinger API credential reminder)  
  - Sticky Note3 (Hostinger API credential reminder)  
  - Sticky Note4 (Notification message instructions)  
  - Sticky Note5 (Empty/placeholder)  
  - Sticky Note (VPS Metrics available description)  

- **Node Details:**  
  - These notes do not have inputs or outputs; purely for user reference.  
  - Provide setup steps, API key link, notification configuration, and detailed metrics descriptions.

---

### 3. Summary Table

| Node Name                 | Node Type                     | Functional Role                              | Input Node(s)                 | Output Node(s)                         | Sticky Note                                                                                             |
|---------------------------|-------------------------------|----------------------------------------------|------------------------------|---------------------------------------|-------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                | Manual workflow start                         | None                         | Hostinger API list VPS                 |                                                                                                       |
| Every day 4:00am          | Schedule Trigger               | Scheduled daily start at 4 AM                 | None                         | Hostinger API list VPS                 |                                                                                                       |
| Hostinger API list VPS    | Hostinger API node             | Lists all VPS instances                        | When clicking ‘Test workflow’, Every day 4:00am | Split Out                              | Sticky Note1, Sticky Note2, Sticky Note3 (credential reminders)                                        |
| Split Out                 | Split Out                      | Splits VPS array into individual items        | Hostinger API list VPS        | Avaliables VPS                        |                                                                                                       |
| Avaliables VPS            | Filter                        | Filters VPS with state "running"               | Split Out                    | Get VPS Metrics                      |                                                                                                       |
| Get VPS Metrics           | Hostinger API node             | Retrieves VPS metrics for each running VPS    | Avaliables VPS               | Calculate metrics                    | Sticky Note (metrics description)                                                                     |
| Calculate metrics         | Set                           | Formats and calculates readable metrics       | Get VPS Metrics              | Loop Over Items                     |                                                                                                       |
| Loop Over Items           | Split In Batches               | Processes VPS items sequentially               | Calculate metrics            | WhatsApp Message (success), Create Snapshot |                                                                                                       |
| WhatsApp Message (success)| Evolution API node             | Sends success notification via WhatsApp       | Loop Over Items              | None                                | Sticky Note4 (notification message instructions)                                                      |
| Create Snapshot           | Hostinger API node             | Creates a snapshot for the VPS                  | Loop Over Items              | Next VPS (success), WhatsApp Message (error) (on error) | Sticky Note1, Sticky Note2, Sticky Note3 (credential reminders)                                        |
| Wait 2 seg                | Wait                          | Waits 2 seconds between VPS snapshot creations | Next VPS                    | Loop Over Items                      |                                                                                                       |
| Next VPS                  | NoOp                          | Placeholder to connect snapshot creation to wait | Create Snapshot             | Wait 2 seg                          |                                                                                                       |
| WhatsApp Message (error)  | Evolution API node             | Sends error notification via WhatsApp          | Create Snapshot (on error)   | None                                | Sticky Note4 (notification message instructions)                                                      |
| Sticky Note6              | Sticky Note                   | Workflow overview and setup instructions       | None                        | None                                | See detailed content in Section 5                                                                     |
| Sticky Note1              | Sticky Note                   | Hostinger API credential reminder               | None                        | None                                |                                                                                                       |
| Sticky Note2              | Sticky Note                   | Hostinger API credential reminder               | None                        | None                                |                                                                                                       |
| Sticky Note3              | Sticky Note                   | Hostinger API credential reminder               | None                        | None                                |                                                                                                       |
| Sticky Note4              | Sticky Note                   | Notification message setup instructions         | None                        | None                                |                                                                                                       |
| Sticky Note5              | Sticky Note                   | Empty/placeholder                               | None                        | None                                |                                                                                                       |
| Sticky Note               | Sticky Note                   | VPS Metrics available description                | None                        | None                                |                                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: "When clicking ‘Test workflow’"  
   - No parameters.  

2. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Name: "Every day 4:00am"  
   - Parameters: Trigger at hour 4 (daily at 4:00 AM).  

3. **Create Hostinger API Node to List VPS**  
   - Type: Hostinger API (Community node)  
   - Name: "Hostinger API list VPS"  
   - Parameters: `vpsAction` set to `listVms`  
   - Credentials: Create and assign Hostinger API credential with API key from https://hpanel.hostinger.com/profile/api  
   - Connect outputs of Manual Trigger and Schedule Trigger to this node.  

4. **Add Split Out Node**  
   - Type: Split Out  
   - Name: "Split Out"  
   - Parameters: Field to split out = `response` (the VPS array)  
   - Connect output of Hostinger API list VPS to this node.  

5. **Add Filter Node to Select Running VPS**  
   - Type: Filter  
   - Name: "Avaliables VPS"  
   - Condition: `$json.state === "running"` (strict string equality)  
   - Connect output of Split Out node to this node.  

6. **Create Hostinger API Node to Get VPS Metrics**  
   - Type: Hostinger API  
   - Name: "Get VPS Metrics"  
   - Parameters:  
     - `vpsAction`: `getVmMetrics`  
     - `virtualMachineId`: expression `={{ $('Avaliables VPS').item.json.id }}`  
     - `date_from`: `={{ $now.minus({ minute: 30 }).toISO() }}`  
     - `date_to`: `={{ $now.toISO() }}`  
   - Credentials: Reuse Hostinger API credential  
   - Connect output of Avaliables VPS to this node.  

7. **Add Set Node to Calculate and Format Metrics**  
   - Type: Set  
   - Name: "Calculate metrics"  
   - Assign fields with expressions to extract and format:  
     - `id`, `plan`, `hostname`, `cpus`, `disk`, `IP`, `OS` from VPS JSON  
     - `cpu_usage`, `ram_usage`, `disco`, `uptime` from metrics JSON with calculations as per expressions in the original workflow  
   - Connect output of Get VPS Metrics to this node.  

8. **Add Split In Batches Node for Sequential Processing**  
   - Type: Split In Batches  
   - Name: "Loop Over Items"  
   - Default batch size (1) is suitable to avoid API overload  
   - Connect output of Calculate metrics to this node.  

9. **Add WhatsApp Message Node for Success Notifications**  
   - Type: Evolution API  
   - Name: "WhatsApp Message (success)"  
   - Parameters:  
     - Resource: `messages-api`  
     - RemoteJid: target WhatsApp number (e.g., `55119999999`)  
     - Message Text: Use template that includes snapshot status, date/time (São Paulo timezone), VPS plan, hostname, IP, CPUs, RAM, disk usage, OS, uptime (use expressions to reference 'Create Snapshot' and 'Calculate metrics' nodes)  
   - Credentials: Configure Evolution API credential with proper authentication  
   - Connect main output #0 of Loop Over Items to this node.  

10. **Add Hostinger API Node for Creating Snapshot**  
    - Type: Hostinger API  
    - Name: "Create Snapshot"  
    - Parameters:  
      - `vpsAction`: `createSnapshot`  
      - `subcategory`: `snapshots`  
      - `virtualMachineId`: expression `={{ $json.id }}` from current item  
    - Credentials: Reuse Hostinger API credential  
    - Set "On Error" workflow setting to "Continue on Error" to avoid stopping workflow on failure  
    - Connect main output #1 of Loop Over Items to this node.  

11. **Add NoOp Node as Placeholder**  
    - Type: NoOp  
    - Name: "Next VPS"  
    - Connect success output of Create Snapshot to this node.  

12. **Add Wait Node to Pause 2 Seconds Between Snapshots**  
    - Type: Wait  
    - Name: "Wait 2 seg"  
    - Parameters: Amount = 2 seconds  
    - Connect output of Next VPS node to this node.  

13. **Connect Wait Node Back to Loop Over Items**  
    - Connect output of Wait 2 seg back to Loop Over Items (to process next batch).  

14. **Add WhatsApp Message Node for Error Notifications**  
    - Type: Evolution API  
    - Name: "WhatsApp Message (error)"  
    - Parameters:  
      - Resource: `messages-api`  
      - RemoteJid: target WhatsApp number (same as success message)  
      - Message Text: Template including error snapshot status, date/time, VPS plan, hostname, IP (similar to success message but indicating error)  
    - Credentials: Use same Evolution API credential  
    - Connect error output of Create Snapshot to this node.  

15. **Add Sticky Notes for Documentation**  
    - Add sticky notes with:  
      - Workflow overview and setup instructions  
      - VPS metrics available description  
      - Hostinger API credential reminders  
      - Notification message instructions  
    - Position them near related nodes for user clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                        | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Hostinger API key must be generated from https://hpanel.hostinger.com/profile/api and added as credential in n8n.                                                                                                                                 | Hostinger API credential setup                                                                         |
| Evolution API credentials required for WhatsApp notifications; alternatively, other notification methods may be used by adapting message templates.                                                                                                | Notification channel setup                                                                              |
| Hostinger allows only one snapshot per VPS; new snapshot replaces the previous one automatically.                                                                                                                                                   | Important Hostinger limitation                                                                         |
| VPS Metrics available include snapshot status, snapshot date/time, server name, external IP, CPU count, RAM usage/free, disk usage/free, OS and version, uptime (days, hours).                                                                      | Metrics description (Sticky Note)                                                                      |
| Workflow created by Leandro Melo. Payment support via Paypal/Pix: contato@sistemanaweb.com                                                                                                                                                          | Project credits                                                                                        |
| [Community Nodes required: n8n-nodes-hostinger-api and n8n-nodes-evolution-api]                                                                                                                                                                      | Node installation prerequisite                                                                         |
| WhatsApp message examples use São Paulo timezone formatting for timestamps. Adjust workflow expressions if running in different timezones.                                                                                                         | Timezone considerations                                                                                 |

---

**Disclaimer:** The provided text is exclusively extracted from an automated workflow built with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---