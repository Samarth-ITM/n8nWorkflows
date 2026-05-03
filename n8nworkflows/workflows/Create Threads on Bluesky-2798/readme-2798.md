Create Threads on Bluesky

https://n8nworkflows.xyz/workflows/create-threads-on-bluesky-2798


# Create Threads on Bluesky

### 1. Workflow Overview

This workflow automates the creation of structured, threaded posts on the Bluesky social platform, targeting content creators and social media managers who want precise control over post timing and visibility. It ensures that only the first and last two posts in a thread are visible immediately, while intermediate posts remain hidden, maintaining a clean profile appearance and encouraging engagement.

The workflow is logically divided into the following blocks:

- **1.1 Trigger & Credential Setup:** Scheduled daily trigger and setting Bluesky user credentials.
- **1.2 Authentication:** Creating a Bluesky session to obtain an access token.
- **1.3 Initial Post Creation:** Posting the first visible post that starts the thread.
- **1.4 First Reply Post:** Creating the first hidden reply post linked to the initial post.
- **1.5 Sibling Posts Creation:** Creating a series of hidden sibling reply posts in a loop, maintaining proper parent-child relationships.
- **1.6 Final Visible Posts:** Posting the last two visible posts to conclude the thread.
- **1.7 Timing Control:** Enforcing delays between posts to avoid rate limiting.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Credential Setup

- **Overview:**  
  This block triggers the workflow daily at 9 AM and sets the Bluesky user credentials (handle and app password) required for authentication and posting.

- **Nodes Involved:**  
  - Run Daily at 9 AM  
  - Set Bluesky Credentials  
  - Sticky Note (Bluesky Authentication)

- **Node Details:**

  - **Run Daily at 9 AM**  
    - Type: Schedule Trigger  
    - Role: Triggers workflow execution daily at 9 AM.  
    - Configuration: Interval trigger set to hour 9.  
    - Inputs: None (trigger node).  
    - Outputs: Connects to "Set Bluesky Credentials".  
    - Edge Cases: Workflow will not run outside scheduled time; ensure server time zone matches expectations.

  - **Set Bluesky Credentials**  
    - Type: Set  
    - Role: Stores Bluesky handle and app password as workflow variables.  
    - Configuration: Two string fields: `BlueskyHandle` and `BlueskyAppPassword` to be filled by user.  
    - Inputs: From schedule trigger.  
    - Outputs: Connects to "Create Bluesky Session".  
    - Edge Cases: Missing or incorrect credentials will cause authentication failure downstream.

  - **Sticky Note (Bluesky Authentication)**  
    - Type: Sticky Note  
    - Role: Instructional note reminding to set Bluesky handle and app password.  
    - Inputs/Outputs: None.

---

#### 2.2 Authentication

- **Overview:**  
  This block authenticates with Bluesky by creating a session using the provided credentials, returning an access JWT token for authorized API calls.

- **Nodes Involved:**  
  - Create Bluesky Session  
  - Sticky Note1 (Initial Post identifiers)

- **Node Details:**

  - **Create Bluesky Session**  
    - Type: HTTP Request  
    - Role: Sends POST request to Bluesky API endpoint `/com.atproto.server.createSession` to authenticate.  
    - Configuration:  
      - URL: `https://bsky.social/xrpc/com.atproto.server.createSession`  
      - Method: POST  
      - Body Parameters: `identifier` (Bluesky handle), `password` (app password) from credentials node.  
    - Inputs: From "Set Bluesky Credentials".  
    - Outputs: Connects to "Create Post Text".  
    - Key Expressions: Uses expressions to inject credentials dynamically.  
    - Edge Cases: Authentication failure (wrong credentials), network errors, API downtime.

  - **Sticky Note1 (Initial Post identifiers)**  
    - Type: Sticky Note  
    - Role: Explains that initial post returns URI and CID identifiers.  
    - Inputs/Outputs: None.

---

#### 2.3 Initial Post Creation

- **Overview:**  
  Creates the first visible post of the thread, which acts as the root post for all subsequent replies.

- **Nodes Involved:**  
  - Create Post Text  
  - Create Initial Post  
  - Sticky Note2 (First Reply Post)

- **Node Details:**

  - **Create Post Text**  
    - Type: Code  
    - Role: Constructs JSON payload for the initial post.  
    - Configuration:  
      - Creates a post text string: "[initial post - visible]" (customizable).  
      - Sets `repo` to Bluesky handle.  
      - Sets collection to `app.bsky.feed.post`.  
      - Sets `createdAt` to current timestamp.  
    - Inputs: From "Create Bluesky Session".  
    - Outputs: Connects to "Create Initial Post".  
    - Edge Cases: Expression failures if credentials missing.

  - **Create Initial Post**  
    - Type: HTTP Request  
    - Role: Sends POST request to create the initial post on Bluesky.  
    - Configuration:  
      - URL: `https://bsky.social/xrpc/com.atproto.repo.createRecord`  
      - Method: POST  
      - JSON Body: Payload from "Create Post Text".  
      - Authorization Header: Bearer token from "Create Bluesky Session".  
    - Inputs: From "Create Post Text".  
    - Outputs: Connects to "Create Reply Text".  
    - Edge Cases: API errors, token expiration, rate limiting.

  - **Sticky Note2 (First Reply Post)**  
    - Type: Sticky Note  
    - Role: Explains that first reply post sets ROOT and PARENT to initial post URI and CID.  
    - Inputs/Outputs: None.

---

#### 2.4 First Reply Post

- **Overview:**  
  Creates the first hidden reply post linked directly to the initial post, establishing the thread structure.

- **Nodes Involved:**  
  - Create Reply Text  
  - Create Reply

- **Node Details:**

  - **Create Reply Text**  
    - Type: Code  
    - Role: Constructs JSON payload for the first reply post.  
    - Configuration:  
      - Text: "[reply post - hidden]" (customizable).  
      - Sets `repo` to Bluesky handle.  
      - Sets collection to `app.bsky.feed.post`.  
      - Sets `reply.root` and `reply.parent` to initial post's URI and CID.  
      - Sets `createdAt` to 1 second in the future to enforce timing delay.  
    - Inputs: From "Create Initial Post".  
    - Outputs: Connects to "Create Reply".  
    - Edge Cases: Expression failures if initial post data missing.

  - **Create Reply**  
    - Type: HTTP Request  
    - Role: Sends POST request to create the first reply post.  
    - Configuration:  
      - URL: `https://bsky.social/xrpc/com.atproto.repo.createRecord`  
      - Method: POST  
      - JSON Body: Payload from "Create Reply Text".  
      - Authorization Header: Bearer token from "Create Bluesky Session".  
    - Inputs: From "Create Reply Text".  
    - Outputs: Connects to "Create Sibling Text".  
    - Edge Cases: API errors, token expiration, rate limiting.

---

#### 2.5 Sibling Posts Creation

- **Overview:**  
  Creates a series of hidden sibling reply posts forming the body of the thread. Posts are created sequentially with proper parent-child relationships and timing delays to avoid rate limits.

- **Nodes Involved:**  
  - Create Sibling Text  
  - Create Sibling  
  - Create Sibling Array  
  - Loop Posts  
  - Create Sibling Text (Loop)  
  - Create Post  
  - Wait  
  - Sticky Note3 (Sibling Post)  
  - Sticky Note6 (Sibling Posts Loop)

- **Node Details:**

  - **Create Sibling Text**  
    - Type: Code  
    - Role: Creates JSON payload for the first sibling post after the first reply.  
    - Configuration:  
      - Text: "[first sibling - hidden]" (customizable).  
      - Sets `reply.root` to initial post URI/CID.  
      - Sets `reply.parent` to first reply post URI/CID.  
      - Sets `createdAt` to 2 seconds in the future.  
    - Inputs: From "Create Reply".  
    - Outputs: Connects to "Create Sibling".  
    - Edge Cases: Expression failures if reply post data missing.

  - **Create Sibling**  
    - Type: HTTP Request  
    - Role: Sends POST request to create the first sibling post.  
    - Configuration:  
      - URL: `https://bsky.social/xrpc/com.atproto.repo.createRecord`  
      - Method: POST  
      - JSON Body: Payload from "Create Sibling Text".  
      - Authorization Header: Bearer token from "Create Bluesky Session".  
    - Inputs: From "Create Sibling Text".  
    - Outputs: Connects to "Create Sibling Array".  
    - Edge Cases: API errors, token expiration, rate limiting.

  - **Create Sibling Array**  
    - Type: Code  
    - Role: Defines an array of sibling posts to create in the loop, including hidden and visible posts.  
    - Configuration:  
      - Array of 9 items with `id` and `name` fields representing post text placeholders.  
      - Last two posts marked as visible.  
    - Inputs: From "Create Sibling".  
    - Outputs: Connects to "Loop Posts".  
    - Edge Cases: None significant.

  - **Loop Posts**  
    - Type: Split In Batches  
    - Role: Iterates over sibling posts array to create each post sequentially.  
    - Configuration: Default batch size (1) to enforce sequential execution.  
    - Inputs: From "Create Sibling Array".  
    - Outputs: Two outputs: first empty (unused), second connects to "Create Sibling Text (Loop)".  
    - Edge Cases: Loop failures if input array empty or malformed.

  - **Create Sibling Text (Loop)**  
    - Type: Code  
    - Role: Creates JSON payload for each sibling post in the loop.  
    - Configuration:  
      - Text: Uses current item's `name` field as post text.  
      - For first iteration, sets `reply.parent` to first sibling post URI/CID; for subsequent iterations, uses previous post's URI/CID.  
      - Sets `reply.root` to initial post URI/CID.  
      - Sets `createdAt` to 2 seconds in the future.  
    - Inputs: From second output of "Loop Posts".  
    - Outputs: Connects to "Create Post".  
    - Edge Cases: Expression failures if previous post data missing.

  - **Create Post**  
    - Type: HTTP Request  
    - Role: Sends POST request to create each sibling post in the loop.  
    - Configuration:  
      - URL: `https://bsky.social/xrpc/com.atproto.repo.createRecord`  
      - Method: POST  
      - JSON Body: Payload from "Create Sibling Text (Loop)".  
      - Authorization Header: Bearer token from "Create Bluesky Session".  
    - Inputs: From "Create Sibling Text (Loop)".  
    - Outputs: Connects to "Wait".  
    - Edge Cases: API errors, token expiration, rate limiting.

  - **Wait**  
    - Type: Wait  
    - Role: Enforces a 2-second delay between sibling post creations to avoid rate limiting.  
    - Configuration: Wait time set to 2 seconds.  
    - Inputs: From "Create Post".  
    - Outputs: Connects back to "Loop Posts" to continue iteration.  
    - Edge Cases: Workflow pause; long delays may impact throughput.

  - **Sticky Note3 (Sibling Post)**  
    - Type: Sticky Note  
    - Role: Explains setting of ROOT and PARENT for sibling posts.  
    - Inputs/Outputs: None.

  - **Sticky Note6 (Sibling Posts Loop)**  
    - Type: Sticky Note  
    - Role: Details loop logic for sibling posts, emphasizing parent-child linkage and use of previous post identifiers.  
    - Inputs/Outputs: None.

---

#### 2.6 Final Visible Posts

- **Overview:**  
  The last two posts in the sibling array are visible posts, concluding the thread with visible content to followers.

- **Nodes Involved:**  
  - Included in "Create Sibling Array" and processed in the loop in 2.5.

- **Node Details:**  
  - The last two items in the sibling array have names indicating visible posts: "[sibling nine - visible]" and "[sibling ten - visible]".  
  - These posts are created with the same parent-child logic and timing delays as hidden posts.  
  - This design ensures only the first and last two posts are visible immediately.

---

#### 2.7 Timing Control

- **Overview:**  
  Timing delays are enforced between posts to prevent API rate limiting and ensure proper chronological order.

- **Nodes Involved:**  
  - Wait node  
  - Timestamp calculations in code nodes (Create Reply Text, Create Sibling Text, Create Sibling Text (Loop))

- **Node Details:**

  - **Wait**  
    - Type: Wait  
    - Role: Pauses workflow for 2 seconds between each sibling post creation.  
    - Configuration: Fixed 2-second delay.  
    - Inputs: From "Create Post".  
    - Outputs: Back to "Loop Posts".  
    - Edge Cases: Delays may accumulate; consider exponential backoff for failures.

  - **Timestamp Calculations**  
    - In code nodes, `createdAt` timestamps are set to current time plus 1 or 2 seconds to ensure posts are chronologically ordered and spaced.  
    - Edge Cases: System clock skew may affect timestamps.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                      | Input Node(s)              | Output Node(s)            | Sticky Note                                                                                          |
|-------------------------|---------------------|------------------------------------|----------------------------|---------------------------|----------------------------------------------------------------------------------------------------|
| Run Daily at 9 AM       | Schedule Trigger    | Triggers workflow daily at 9 AM    | None                       | Set Bluesky Credentials    | ## Trigger                                                                                         |
| Set Bluesky Credentials  | Set                 | Stores Bluesky handle and password | Run Daily at 9 AM           | Create Bluesky Session     | ## Bluesky Authentication<br>Set your Bluesky social link and also your App Password.              |
| Create Bluesky Session   | HTTP Request        | Authenticates and creates session  | Set Bluesky Credentials     | Create Post Text           |                                                                                                    |
| Create Post Text         | Code                | Constructs initial post payload     | Create Bluesky Session      | Create Initial Post        |                                                                                                    |
| Create Initial Post      | HTTP Request        | Creates initial visible post        | Create Post Text            | Create Reply Text          | ## Initial Post [A]<br>When the first post is created two identifiers are returned:<br>- URI<br>- CID |
| Create Reply Text        | Code                | Constructs first reply post payload | Create Initial Post         | Create Reply               | ## First Reply Post [B]<br>Here we set the 'ROOT' and the 'PARENT' values.<br>We use both URI and CID as ROOT and PARENT, as this is the first child of the root post (Initial Post [A]).<br>We receive a new URI and CID in return. |
| Create Reply             | HTTP Request        | Creates first hidden reply post     | Create Reply Text           | Create Sibling Text        |                                                                                                    |
| Create Sibling Text      | Code                | Constructs first sibling post payload | Create Reply               | Create Sibling             | ## Sibling Post [C]<br>Set 'ROOT' using URI/CID from the root post (Initial Post [A]).<br>For the PARENT, we use the URI and CID returned by the preceding post (First Reply Post [B]). |
| Create Sibling           | HTTP Request        | Creates first sibling post           | Create Sibling Text         | Create Sibling Array       |                                                                                                    |
| Create Sibling Array     | Code                | Defines array of sibling posts       | Create Sibling              | Loop Posts                 |                                                                                                    |
| Loop Posts              | Split In Batches    | Iterates over sibling posts          | Create Sibling Array        | Create Sibling Text (Loop) | ## Sibling Posts using Loop node [D]<br>Here we set the 'ROOT' using both URI and CID from the root post (Initial Post [A]), and for all future siblings.<br>For the PARENT, we use the URI and CID returned by the preceding post.<br>So the first loop iteration gets it from the 'Create Sibling' node, and after that from the 'Create Post' node. |
| Create Sibling Text (Loop) | Code              | Constructs sibling post payload in loop | Loop Posts                 | Create Post                |                                                                                                    |
| Create Post              | HTTP Request        | Creates sibling post in loop         | Create Sibling Text (Loop)  | Wait                      |                                                                                                    |
| Wait                    | Wait                | Enforces delay between posts         | Create Post                 | Loop Posts                 |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 9 AM (adjust hour as needed).  
   - Connect output to next node.

2. **Create a Set node for Bluesky Credentials**  
   - Add two string fields:  
     - `BlueskyHandle` (e.g., your Bluesky username)  
     - `BlueskyAppPassword` (your Bluesky app password)  
   - Connect input from Schedule Trigger node.  
   - Connect output to authentication node.

3. **Create an HTTP Request node to authenticate (Create Bluesky Session)**  
   - Method: POST  
   - URL: `https://bsky.social/xrpc/com.atproto.server.createSession`  
   - Body Parameters (JSON):  
     - `identifier`: expression referencing `BlueskyHandle` from Set node  
     - `password`: expression referencing `BlueskyAppPassword` from Set node  
   - Send body as JSON.  
   - Connect input from Set node.  
   - Connect output to initial post preparation node.

4. **Create a Code node to build initial post payload (Create Post Text)**  
   - JavaScript code:  
     - Define post text (e.g., "[initial post - visible]").  
     - Set `repo` to Bluesky handle from credentials.  
     - Set collection to `app.bsky.feed.post`.  
     - Set `createdAt` to current timestamp.  
   - Connect input from authentication node.  
   - Connect output to HTTP request node for initial post.

5. **Create an HTTP Request node to create the initial post (Create Initial Post)**  
   - Method: POST  
   - URL: `https://bsky.social/xrpc/com.atproto.repo.createRecord`  
   - JSON Body: expression referencing output of "Create Post Text" node.  
   - Headers: Authorization Bearer token from authentication node.  
   - Connect input from "Create Post Text".  
   - Connect output to first reply post preparation node.

6. **Create a Code node to build first reply post payload (Create Reply Text)**  
   - JavaScript code:  
     - Define reply text (e.g., "[reply post - hidden]").  
     - Set `repo` to Bluesky handle.  
     - Set collection to `app.bsky.feed.post`.  
     - Set `reply.root` and `reply.parent` to initial post URI and CID from "Create Initial Post".  
     - Set `createdAt` to current time + 1 second.  
   - Connect input from "Create Initial Post".  
   - Connect output to HTTP request node for first reply.

7. **Create an HTTP Request node to create first reply post (Create Reply)**  
   - Method: POST  
   - URL: `https://bsky.social/xrpc/com.atproto.repo.createRecord`  
   - JSON Body: expression referencing "Create Reply Text".  
   - Headers: Authorization Bearer token from authentication node.  
   - Connect input from "Create Reply Text".  
   - Connect output to first sibling post preparation node.

8. **Create a Code node to build first sibling post payload (Create Sibling Text)**  
   - JavaScript code:  
     - Define sibling text (e.g., "[first sibling - hidden]").  
     - Set `repo` to Bluesky handle.  
     - Set collection to `app.bsky.feed.post`.  
     - Set `reply.root` to initial post URI/CID.  
     - Set `reply.parent` to first reply post URI/CID.  
     - Set `createdAt` to current time + 2 seconds.  
   - Connect input from "Create Reply".  
   - Connect output to HTTP request node for first sibling.

9. **Create an HTTP Request node to create first sibling post (Create Sibling)**  
   - Method: POST  
   - URL: `https://bsky.social/xrpc/com.atproto.repo.createRecord`  
   - JSON Body: expression referencing "Create Sibling Text".  
   - Headers: Authorization Bearer token from authentication node.  
   - Connect input from "Create Sibling Text".  
   - Connect output to code node defining sibling posts array.

10. **Create a Code node to define sibling posts array (Create Sibling Array)**  
    - JavaScript code:  
      - Define array of posts with `id` and `name` fields, including hidden and visible posts.  
    - Connect input from "Create Sibling".  
    - Connect output to Split In Batches node.

11. **Create a Split In Batches node to loop over sibling posts (Loop Posts)**  
    - Default batch size (1) for sequential processing.  
    - Connect input from "Create Sibling Array".  
    - Connect second output to code node creating sibling post payload in loop.

12. **Create a Code node to build sibling post payload in loop (Create Sibling Text (Loop))**  
    - JavaScript code:  
      - Use current item `name` as post text.  
      - For first iteration, set `reply.parent` to first sibling post URI/CID; else use previous post URI/CID.  
      - Set `reply.root` to initial post URI/CID.  
      - Set `createdAt` to current time + 2 seconds.  
    - Connect input from second output of "Loop Posts".  
    - Connect output to HTTP request node creating sibling post.

13. **Create an HTTP Request node to create sibling post in loop (Create Post)**  
    - Method: POST  
    - URL: `https://bsky.social/xrpc/com.atproto.repo.createRecord`  
    - JSON Body: expression referencing "Create Sibling Text (Loop)".  
    - Headers: Authorization Bearer token from authentication node.  
    - Connect input from "Create Sibling Text (Loop)".  
    - Connect output to Wait node.

14. **Create a Wait node to enforce delay between posts (Wait)**  
    - Wait time: 2 seconds.  
    - Connect input from "Create Post".  
    - Connect output back to "Loop Posts" to continue iteration.

15. **Add Sticky Notes** (optional for documentation)  
    - Add notes explaining Bluesky authentication, initial post identifiers, reply post linkage, sibling post logic, and loop behavior as per original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow creates Bluesky threads with controlled visibility: first and last two posts visible, others hidden. | Workflow purpose and design principle.                                                             |
| Suggested enhancements include error handling with retries, input validation, notifications, and monitoring. | Workflow improvement ideas for robustness and observability.                                       |
| Setup requires Bluesky account, handle, and app password; credentials must be entered in the workflow. | User setup instructions.                                                                            |
| Timing delays of 1-2 seconds between posts prevent rate limiting by Bluesky API.                         | Rate limiting mitigation strategy.                                                                 |
| Bluesky API endpoints used: `/com.atproto.server.createSession` for auth, `/com.atproto.repo.createRecord` for posts. | API integration details.                                                                            |
| For more info on Bluesky API and authentication, refer to official Bluesky developer documentation.     | External resource (not linked here, user to consult Bluesky docs).                                 |

---

This document fully describes the "Create Threads on Bluesky" workflow, enabling advanced users and AI agents to understand, reproduce, and modify it with confidence.