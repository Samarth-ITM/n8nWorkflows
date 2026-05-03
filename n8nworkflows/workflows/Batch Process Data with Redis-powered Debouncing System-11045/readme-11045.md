Batch Process Data with Redis-powered Debouncing System

https://n8nworkflows.xyz/workflows/batch-process-data-with-redis-powered-debouncing-system-11045


# Batch Process Data with Redis-powered Debouncing System

### 1. Workflow Overview

This workflow implements a **generic debouncing system** using Redis for batch processing of incoming data streams identified by queue IDs. It is designed to aggregate multiple asynchronous data inputs into batches, avoiding redundant or concurrent processing by using Redis data structures and locking mechanisms.

The workflow targets use cases where data arrives in bursts or concurrently, and you wish to buffer and process these data points in controlled batches, ensuring that only one execution handles the batch at a time.

**Logical Blocks:**

- **1.1 Input Reception and Initial Lock Check**  
  Receives input data with a queue identifier, checks if the queue is currently locked to avoid concurrent processing.

- **1.2 Message Aggregation and Last Update Tagging**  
  If unlocked, appends the incoming data to a Redis list and writes a unique execution UUID to a Redis key marking the last update.

- **1.3 Debounce Wait Period**  
  Waits for a configurable debounce duration to allow potential additional incoming data to be collected.

- **1.4 Last Writer Verification and Lock Acquisition**  
  After waiting, verifies if the current execution is the last writer (by matching the UUID). If yes, proceeds; otherwise, terminates.

- **1.5 Batch Retrieval, Processing, and Lock Release**  
  Locks the queue, retrieves all messages from Redis, clears the message list, releases the lock, and outputs the batch for downstream processing.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception and Initial Lock Check

- **Overview:**  
  Receives trigger with input data and queue ID. Checks whether the queue is currently locked to prevent concurrent batch processing.

- **Nodes Involved:**  
  - Trigger  
  - Get lock value  
  - Is lock active?  
  - Wait for lock release (conditional path)  
  - Push to message list (conditional path)

- **Node Details:**

  1. **Trigger**  
     - Type: Execute Workflow Trigger  
     - Role: Entry point, receives parameters `queue_id` and `data` from external invocations.  
     - Inputs: External trigger  
     - Outputs: Sends data to "Get lock value"  
     - Notes: Requires external workflow or system to call with relevant data.

  2. **Get lock value**  
     - Type: Redis (Get)  
     - Role: Reads Redis key `lock_<queue_id>` to check if a processing lock exists.  
     - Key: Dynamic, based on input `queue_id`  
     - Output: Numeric lock value to "Is lock active?"  
     - Edge cases: Redis connection errors, key not existing returns null or zero.

  3. **Is lock active?**  
     - Type: IF node (number > 0)  
     - Role: Determines if lock count > 0 (lock active).  
     - Logic: If true, goes to "Wait for lock release"; else, proceeds to "Push to message list".  
     - Edge cases: Loose type validation handles null or missing data gracefully.

  4. **Wait for lock release**  
     - Type: Wait (time-based pause)  
     - Role: Pauses workflow 1 second before re-checking lock status to avoid race conditions.  
     - Notes: Webhook-based wait node, can cause delays if Redis lock persists long.

  5. **Push to message list**  
     - Type: Redis (List Push)  
     - Role: Appends incoming data to Redis list `messages_<queue_id>`.  
     - Message data is the raw input data.  
     - Edge cases: Redis downtime or list key issues causing failed appends.

---

#### Block 1.2: Message Aggregation and Last Update Tagging

- **Overview:**  
  Generates a unique UUID to mark this execution as the last writer, storing it in Redis under `last_update_<queue_id>`.

- **Nodes Involved:**  
  - Crypto  
  - Set last update uuid

- **Node Details:**

  1. **Crypto**  
     - Type: Crypto (Generate UUID)  
     - Role: Creates a unique identifier for this execution.  
     - Configuration: Action "generate" (UUID).  
     - Output: UUID string.

  2. **Set last update uuid**  
     - Type: Redis (Set)  
     - Role: Sets the Redis key `last_update_<queue_id>` with the UUID from Crypto node.  
     - Ensures this execution is marked as the last data appender.

---

#### Block 1.3: Debounce Wait Period

- **Overview:**  
  Introduces a delay (2 seconds) to allow additional data arrivals before batch processing proceeds.

- **Nodes Involved:**  
  - Wait

- **Node Details:**

  1. **Wait**  
     - Type: Wait  
     - Role: Pauses for 2 seconds as debounce period.  
     - Configured with webhook-based wait to avoid blocking worker threads.

---

#### Block 1.4: Last Writer Verification and Lock Acquisition

- **Overview:**  
  After waiting, verifies if the current execution is the last writer by comparing stored UUIDs. If true, proceeds to acquire lock; else, terminates.

- **Nodes Involved:**  
  - Get last update uuid  
  - Am I last?  
  - Set lock

- **Node Details:**

  1. **Get last update uuid**  
     - Type: Redis (Get)  
     - Role: Reads key `last_update_<queue_id>` to retrieve the UUID currently marked as last writer.

  2. **Am I last?**  
     - Type: IF node  
     - Role: Compares UUID from "Crypto" node with the one retrieved from Redis.  
     - Condition: Equals comparison, case sensitive and strict validation.  
     - Output: If true, proceeds to "Set lock"; else, workflow ends.

  3. **Set lock**  
     - Type: Redis (Increment)  
     - Role: Increments Redis key `lock_<queue_id>` to lock the queue for exclusive processing.

---

#### Block 1.5: Batch Retrieval, Processing, and Lock Release

- **Overview:**  
  Retrieves all messages from Redis list, clears the list, releases processing lock, and splits batch into individual messages for downstream handling.

- **Nodes Involved:**  
  - Get messages  
  - Clear messages  
  - Release lock  
  - Split messages

- **Node Details:**

  1. **Get messages**  
     - Type: Redis (List Get)  
     - Role: Fetches all entries stored in `messages_<queue_id>` Redis list.  
     - Output: Array of messages.

  2. **Clear messages**  
     - Type: Redis (Delete)  
     - Role: Deletes Redis list `messages_<queue_id>` to clear buffer after retrieval.

  3. **Release lock**  
     - Type: Redis (Delete)  
     - Role: Deletes the Redis key `lock_<queue_id>` to release processing lock.

  4. **Split messages**  
     - Type: SplitOut  
     - Role: Splits the batch array into individual messages for parallel downstream processing.  
     - Output: Each message outgoing data item.

- **Notes:**  
  The final batch messages are output for further workflow nodes to consume.

---

### 3. Summary Table

| Node Name           | Node Type                 | Functional Role                         | Input Node(s)              | Output Node(s)               | Sticky Note                                                                                   |
|---------------------|---------------------------|---------------------------------------|----------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| Trigger             | Execute Workflow Trigger  | Entry point, receives input data      | -                          | Get lock value              |                                                                                              |
| Get lock value      | Redis (Get)               | Checks if queue lock is active        | Trigger                    | Is lock active?             | Check if this queue's list of messages are being read at the moment                          |
| Is lock active?     | IF                        | Branches based on lock status         | Get lock value             | Wait for lock release, Push to message list |                                                                                              |
| Wait for lock release | Wait                     | Waits 1 second to re-check lock       | Is lock active?            | Get lock value              |                                                                                              |
| Push to message list | Redis (List Push)         | Appends incoming data to Redis list   | Is lock active?            | Crypto                     |                                                                                              |
| Crypto              | Crypto (UUID generate)    | Generates unique execution UUID       | Push to message list       | Set last update uuid        |                                                                                              |
| Set last update uuid | Redis (Set)               | Marks last writer for debounce        | Crypto                     | Wait                       |                                                                                              |
| Wait                | Wait                      | Debounce delay (2 seconds)             | Set last update uuid       | Get last update uuid        | Debounce period - adjust as needed                                                          |
| Get last update uuid | Redis (Get)               | Retrieves last writer UUID             | Wait                       | Am I last?                 |                                                                                              |
| Am I last?          | IF                        | Checks if current execution is last   | Get last update uuid       | Set lock                   |                                                                                              |
| Set lock            | Redis (Increment)         | Locks queue for batch processing       | Am I last?                 | Get messages               |                                                                                              |
| Get messages        | Redis (List Get)          | Gets all buffered messages             | Set lock                   | Clear messages             |                                                                                              |
| Clear messages      | Redis (Delete)            | Clears message list after retrieval    | Get messages               | Release lock               |                                                                                              |
| Release lock        | Redis (Delete)            | Releases processing lock                | Clear messages             | Split messages             |                                                                                              |
| Split messages      | SplitOut                  | Outputs individual messages from batch | Release lock               | -                         | This will be executed for every message in the queue - connect your further logic here.     |
| Sticky Note (26...)  | Sticky Note               | Explains entire generic debouncer flow | -                          | -                          | Generic debouncer explanation and step-by-step flow                                         |
| Sticky Note1         | Sticky Note               | Indicates debounce period               | -                          | -                          | Debounce period - adjust as needed                                                          |
| Sticky Note2         | Sticky Note               | Notes on lock check                     | -                          | -                          | Check if this queue's list of messages are being read at the moment                         |
| Sticky Note3         | Sticky Note               | Notes on downstream split message usage | -                          | -                          | This will be executed for every message in the queue - connect your further logic here.     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**  
   - Type: Execute Workflow Trigger  
   - Configure inputs: `queue_id` (string), `data` (any)  
   - Position: Start of workflow

2. **Create Redis Node "Get lock value"**  
   - Operation: Get  
   - Key: `lock_{{ $json.queue_id }}`  
   - Key Type: String  
   - Credentials: Configure Redis account  
   - Connect Trigger → Get lock value

3. **Create IF Node "Is lock active?"**  
   - Condition: `$json.data > 0` (number greater than zero)  
   - Connect Get lock value → Is lock active?

4. **Create Wait Node "Wait for lock release"**  
   - Duration: 1 second  
   - Webhook enabled  
   - Connect Is lock active? (true branch) → Wait for lock release

5. **Connect Wait for lock release → Get lock value** (loop back to re-check lock)

6. **Create Redis Node "Push to message list"**  
   - Operation: Push (tail)  
   - List: `messages_{{ $json.queue_id }}`  
   - Message Data: `{{ $json.data }}`  
   - Credentials: Redis account  
   - Connect Is lock active? (false branch) → Push to message list

7. **Create Crypto Node "Crypto"**  
   - Action: Generate UUID  
   - Connect Push to message list → Crypto

8. **Create Redis Node "Set last update uuid"**  
   - Operation: Set  
   - Key: `last_update_{{ $json.queue_id }}`  
   - Value: `{{ $json.data }}` (UUID from Crypto)  
   - Credentials: Redis account  
   - Connect Crypto → Set last update uuid

9. **Create Wait Node "Wait"**  
   - Duration: 2 seconds (debounce period)  
   - Webhook enabled  
   - Connect Set last update uuid → Wait

10. **Create Redis Node "Get last update uuid"**  
    - Operation: Get  
    - Key: `last_update_{{ $json.queue_id }}`  
    - Credentials: Redis account  
    - Connect Wait → Get last update uuid

11. **Create IF Node "Am I last?"**  
    - Condition: Equals  
    - Left Value: UUID from Crypto (saved earlier, use expression referencing Crypto node output)  
    - Right Value: UUID from Get last update uuid  
    - Case sensitive, strict validation  
    - Connect Get last update uuid → Am I last?

12. **Create Redis Node "Set lock"**  
    - Operation: Increment  
    - Key: `lock_{{ $json.queue_id }}`  
    - Credentials: Redis account  
    - Connect Am I last? (true branch) → Set lock

13. **Create Redis Node "Get messages"**  
    - Operation: List Get (get all list elements)  
    - Key: `messages_{{ $json.queue_id }}`  
    - Credentials: Redis account  
    - Connect Set lock → Get messages

14. **Create Redis Node "Clear messages"**  
    - Operation: Delete  
    - Key: `messages_{{ $json.queue_id }}`  
    - Credentials: Redis account  
    - Connect Get messages → Clear messages

15. **Create Redis Node "Release lock"**  
    - Operation: Delete  
    - Key: `lock_{{ $json.queue_id }}`  
    - Credentials: Redis account  
    - Connect Clear messages → Release lock

16. **Create SplitOut Node "Split messages"**  
    - Field to split out: `data` (the list array from Get messages)  
    - Connect Release lock → Split messages

17. **Downstream nodes:** Use outputs from "Split messages" to process individual data items as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                                     |
|----------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| This workflow is a generic debouncer implementation using Redis to aggregate and batch-process concurrent data inputs. | Explained in the large sticky note attached to the workflow.                                                       |
| Debounce period is configurable; here set to 2 seconds, adjust based on traffic and processing needs.                 | Sticky Note1 near the Wait node.                                                                                   |
| Lock-checking logic prevents race conditions and ensures only one execution processes the batch at a time.            | Sticky Note2 near the Get lock value and Is lock active? nodes.                                                    |
| The SplitOut node outputs individual messages for further processing—connect your business logic here.                 | Sticky Note3 near the Split messages node.                                                                          |
| Redis credentials must be configured with appropriate host, port, and auth for connectivity.                          | Credential setup required before running the workflow.                                                              |

---

**Disclaimer:**  
The provided text is exclusively derived from an n8n automated workflow. It complies strictly with applicable content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.