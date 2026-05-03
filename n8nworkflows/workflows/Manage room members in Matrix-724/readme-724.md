Manage room members in Matrix

https://n8nworkflows.xyz/workflows/manage-room-members-in-matrix-724


# Manage room members in Matrix

### 1. Workflow Overview

This workflow automates the creation of a new Matrix chat room, invites members from an existing room (excluding the workflow executor), and sends a welcome message in the newly created room. It is designed for managing room membership dynamically within the Matrix ecosystem, particularly useful for onboarding users or synchronizing members across rooms.

The workflow can be logically divided into the following blocks:

- **1.1 Input Trigger:** Manual initiation of the workflow.
- **1.2 Room Creation:** Creation of a new Matrix room with specified name and alias.
- **1.3 User Retrieval:** Fetching information about the current user and members of an existing room.
- **1.4 Member Filtering and Invitation:** Filtering out the current user and inviting other members to the newly created room.
- **1.5 Welcome Message Sending:** Sending a greeting message to the newly created room.
- **1.6 No Operation Handling:** Placeholder for cases where no invitation is sent.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Trigger

- **Overview:** This block starts the workflow manually when triggered by the user.
- **Nodes Involved:** `On clicking 'execute'`
- **Node Details:**
  - **Type:** Manual Trigger
  - **Role:** Initiates the workflow execution on user command.
  - **Configuration:** No parameters required.
  - **Inputs:** None (trigger node).
  - **Outputs:** Connects to the Room Creation node (`Matrix`).
  - **Failure cases:** None expected.
  - **Version requirements:** Standard n8n node.

#### 2.2 Room Creation

- **Overview:** Creates a new Matrix room named "n8n" with the alias "#discussion-n8n".
- **Nodes Involved:** `Matrix`
- **Node Details:**
  - **Type:** Matrix node, resource: `room`
  - **Role:** Creates a new room on the Matrix server.
  - **Configuration:**
    - `roomName`: "n8n"
    - `roomAlias`: "discussion-n8n"
  - **Credentials:** Uses `matrixApi` credential for authentication.
  - **Inputs:** Triggered by `On clicking 'execute'`.
  - **Outputs:** Passes newly created room details to `Matrix1`.
  - **Edge cases:**
    - Room alias already in use.
    - Authentication failure.
    - Network or API timeouts.
  - **Version requirements:** Compatible with Matrix API v1.

#### 2.3 User Retrieval

- **Overview:** Retrieves info about the current user and members of an existing Matrix room.
- **Nodes Involved:** `Matrix1`, `Matrix2`
- **Node Details:**

  - **Matrix1**
    - **Type:** Matrix node, resource: `account`
    - **Role:** Fetches current authenticated user details.
    - **Configuration:** No parameters; uses `matrixApi` credential.
    - **Input:** From `Matrix` node output.
    - **Output:** User details passed to `Matrix2`.
    - **Failure handling:** `continueOnFail` enabled to allow workflow continuation on failure.
    - **Edge cases:** Authentication errors, API failures.

  - **Matrix2**
    - **Type:** Matrix node, resource: `roomMember`
    - **Role:** Retrieves members of a specified existing Matrix room.
    - **Configuration:**
      - `roomId`: Fixed to `!cMUIsUgevrhCoeMkSG:matrix.org` (a pre-existing room).
      - No filters applied.
      - Uses `matrixApi` credential.
    - **Input:** From `Matrix1`.
    - **Output:** Member list passed to `IF` node.
    - **Edge cases:** Room not found, permission denied, API errors.

#### 2.4 Member Filtering and Invitation

- **Overview:** Filters out the current user from the member list and invites others to the newly created room.
- **Nodes Involved:** `IF`, `Matrix3`, `NoOp`
- **Node Details:**

  - **IF**
    - **Type:** Conditional node
    - **Role:** Compares `user_id` of current user (`Matrix1`) against each room member (`Matrix2`) to exclude the current user from invitations.
    - **Configuration:**
      - Condition: Check if `Matrix1.user_id` is not equal to `Matrix2.user_id`.
    - **Input:** From `Matrix2`.
    - **Outputs:** 
      - True branch: Passes user ID to `Matrix3` for invitation.
      - False branch: Passes control to `NoOp`.
    - **Edge cases:** Missing or malformed user IDs.

  - **Matrix3**
    - **Type:** Matrix node, resource: `room`, operation: `invite`
    - **Role:** Invites filtered user(s) to the newly created room.
    - **Configuration:**
      - `roomId`: Taken from `Matrix` node output (`room_id` of created room).
      - `userId`: Taken from `IF` node output (`user_id`).
      - Uses `matrixApi` credential.
    - **Input:** True branch of `IF`.
    - **Output:** Passes control to `Matrix4`.
    - **Edge cases:** Invitation failure due to permissions, invalid user IDs, network errors.

  - **NoOp**
    - **Type:** No Operation node
    - **Role:** Placeholder to handle false branch of `IF` (i.e., when no invitation is needed).
    - **Input:** False branch of `IF`.
    - **Output:** None.
    - **Edge cases:** None.

#### 2.5 Welcome Message Sending

- **Overview:** Sends a fixed welcome text message to the newly created room.
- **Nodes Involved:** `Matrix4`
- **Node Details:**
  - **Type:** Matrix node, resource: `room`, operation: `sendMessage`
  - **Role:** Posts a welcome message "Welcome to n8n!" in the created room.
  - **Configuration:**
    - `roomId`: Uses the room ID from `Matrix` node output.
    - `text`: "Welcome to n8n!"
    - Uses `matrixApi` credential.
  - **Input:** From `Matrix3` (after successful invitation).
  - **Output:** End of workflow.
  - **Edge cases:** Message sending failure due to permissions or network issues.

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                   | Input Node(s)           | Output Node(s)       | Sticky Note                               |
|-----------------------|---------------------|---------------------------------|------------------------|----------------------|-------------------------------------------|
| On clicking 'execute' | Manual Trigger      | Starts workflow manually         | None                   | Matrix               |                                           |
| Matrix                | Matrix (room create) | Creates new room                 | On clicking 'execute'   | Matrix1               |                                           |
| Matrix1               | Matrix (account)     | Fetches current user info        | Matrix                  | Matrix2               |                                           |
| Matrix2               | Matrix (roomMember)  | Retrieves members of existing room | Matrix1                 | IF                    |                                           |
| IF                    | Conditional          | Filters out current user         | Matrix2                 | Matrix3, NoOp          |                                           |
| Matrix3               | Matrix (room invite) | Invites filtered members         | IF (true branch)        | Matrix4                |                                           |
| Matrix4               | Matrix (sendMessage) | Sends welcome message            | Matrix3                 | None                  |                                           |
| NoOp                  | No Operation         | Handles no-invitation branch     | IF (false branch)       | None                  |                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**
   - Name: `On clicking 'execute'`.
   - Type: Manual Trigger.
   - No parameters needed.

2. **Create Matrix Node for Room Creation**
   - Name: `Matrix`.
   - Type: Matrix.
   - Resource: `room`.
   - Parameters:
     - `roomName`: "n8n"
     - `roomAlias`: "discussion-n8n"
   - Credentials: Configure `matrixApi` with appropriate Matrix server credentials.
   - Connect output from `On clicking 'execute'` to this node.

3. **Create Matrix Node to Fetch Current User**
   - Name: `Matrix1`.
   - Type: Matrix.
   - Resource: `account`.
   - Credentials: Use the same `matrixApi`.
   - Set `continueOnFail` to true (to allow workflow to proceed even if this node fails).
   - Connect output of `Matrix` to `Matrix1`.

4. **Create Matrix Node to Get Members of Existing Room**
   - Name: `Matrix2`.
   - Type: Matrix.
   - Resource: `roomMember`.
   - Parameters:
     - `roomId`: `!cMUIsUgevrhCoeMkSG:matrix.org` (replace with your actual existing room ID).
     - Leave filters empty.
   - Credentials: Use `matrixApi`.
   - Connect output of `Matrix1` to `Matrix2`.

5. **Add IF Node to Filter Current User**
   - Name: `IF`.
   - Type: If.
   - Condition:
     - String comparison: Check if `{{$node["Matrix1"].json["user_id"]}}` is **not equal** to `{{$node["Matrix2"].json["user_id"]}}`.
   - Connect output of `Matrix2` to `IF`.

6. **Create Matrix Node to Invite Members**
   - Name: `Matrix3`.
   - Type: Matrix.
   - Resource: `room`.
   - Operation: `invite`.
   - Parameters:
     - `roomId`: Set to `{{$node["Matrix"].json["room_id"]}}` to target the newly created room.
     - `userId`: Set to `{{$node["IF"].json["user_id"]}}` to invite filtered members.
   - Credentials: Use `matrixApi`.
   - Connect `IF` true output to `Matrix3`.

7. **Create Matrix Node to Send Welcome Message**
   - Name: `Matrix4`.
   - Type: Matrix.
   - Resource: `room`.
   - Operation: `sendMessage`.
   - Parameters:
     - `roomId`: `{{$node["Matrix"].json["room_id"]}}`
     - `text`: "Welcome to n8n!"
   - Credentials: Use `matrixApi`.
   - Connect output of `Matrix3` to `Matrix4`.

8. **Create No Operation Node**
   - Name: `NoOp`.
   - Type: No Operation.
   - Purpose: To handle the false branch of the IF node where no invitation is needed.
   - Connect `IF` false output to `NoOp`.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                     |
|-----------------------------------------------------------------------------------------------|----------------------------------------------------|
| The workflow requires a valid Matrix API credential (`matrixApi`) configured in n8n.          | Credential setup for Matrix integration             |
| The fixed existing room ID `!cMUIsUgevrhCoeMkSG:matrix.org` should be replaced with your own. | Change to your target room for member invitations  |
| This workflow demonstrates basic room and membership management in Matrix via n8n's Matrix node. | Useful for automation in Matrix-powered chat systems |
| Matrix API documentation: https://matrix.org/docs/api/client-server/                          | For advanced configurations and troubleshooting    |