Turn on a light to a specific color on any update in GitHub repository

https://n8nworkflows.xyz/workflows/turn-on-a-light-to-a-specific-color-on-any-update-in-github-repository-1856


# Turn on a light to a specific color on any update in GitHub repository

### 1. Workflow Overview

This workflow automates the process of turning a smart light to a specific color (red by default) whenever any update occurs in a specified GitHub repository. It is particularly useful for developers or teams who want a visual indicator of repository activity, such as pull requests, issues, pushes, or other GitHub events.

**Logical blocks:**

- **1.1 GitHub Event Trigger:** Watches for any update event on a configured GitHub repository.
- **1.2 Home Automation Action:** Turns on a specified light with a predefined RGB color in Home Assistant.

---

### 2. Block-by-Block Analysis

#### 2.1 GitHub Event Trigger

- **Overview:**  
  This block detects any event update in the designated GitHub repository. It serves as the entry point of the workflow, triggering downstream actions when repository changes occur.

- **Nodes Involved:**  
  - On any update in repository (GitHub Trigger)

- **Node Details:**

  - **Node Name:** On any update in repository  
  - **Type:** GitHub Trigger (n8n-nodes-base.githubTrigger)  
  - **Technical Role:** Listens for GitHub webhook events on a specific repository.  
  - **Configuration:**  
    - Owner: `dummydavid` (GitHub account or organization)  
    - Repository: `DemoRepo` (target repo name)  
    - Events: `*` (all event types, including pushes, pull requests, issues, etc.)  
    - Webhook: Registered automatically; configured with unique webhook ID.  
  - **Key Expressions/Variables:** None (static configuration)  
  - **Input Connections:** None (trigger node)  
  - **Output Connections:** Connected to "Turn a light red" node  
  - **Version Requirements:** Compatible with n8n version supporting GitHub Trigger node v1  
  - **Potential Failure Modes:**  
    - Authentication errors if GitHub credentials are invalid or expired.  
    - Webhook registration failure if repository or owner is misconfigured.  
    - Network issues leading to missed events or delayed triggers.  
  - **Credentials:** Requires valid GitHub API credentials.  
  - **Sub-Workflow:** None

#### 2.2 Home Automation Action

- **Overview:**  
  This block receives the trigger and uses Home Assistant to turn on a specified light and set its color to red (RGB: 255,0,0).

- **Nodes Involved:**  
  - Turn a light red (Home Assistant node)

- **Node Details:**

  - **Node Name:** Turn a light red  
  - **Type:** Home Assistant (n8n-nodes-base.homeAssistant)  
  - **Technical Role:** Calls the `light.turn_on` service on Home Assistant to activate and configure the light color.  
  - **Configuration:**  
    - Domain: `light`  
    - Service: `turn_on`  
    - Service Attributes:  
      - entity_id: `light.lamp` (target light entity)  
      - rgb_color: `[255,0,0]` (red color)  
  - **Key Expressions/Variables:**  
    - The RGB color is set using an expression `={{[255,0,0]}}` for direct RGB array input.  
  - **Input Connections:** From "On any update in repository" node  
  - **Output Connections:** None (terminal node)  
  - **Version Requirements:** Requires Home Assistant API credentials compatible with n8n Home Assistant node v1  
  - **Potential Failure Modes:**  
    - Authentication errors if Home Assistant credentials are invalid or expired.  
    - Service call failure if the specified `entity_id` does not exist or the Home Assistant instance is unreachable.  
    - Incorrect RGB format causing service call rejection.  
  - **Credentials:** Requires valid Home Assistant API credentials.  
  - **Sub-Workflow:** None

---

### 3. Summary Table

| Node Name                | Node Type               | Functional Role                   | Input Node(s)             | Output Node(s)          | Sticky Note                                                                                              |
|--------------------------|-------------------------|---------------------------------|---------------------------|-------------------------|--------------------------------------------------------------------------------------------------------|
| On any update in repository | GitHub Trigger          | Detect GitHub repository updates | None                      | Turn a light red        |                                                                                                        |
| Turn a light red          | Home Assistant          | Turn on light and set color      | On any update in repository | None                    | Configure light here: update `entity_id` if needed. Default color red (255,0,0). See links in notes.    |
| Note                     | Sticky Note             | Documentation note               | None                      | None                    | ## Turn on a light to a specific color on any update in GitHub repository... How it works overview.     |
| Note1                    | Sticky Note             | Configuration guidance           | None                      | None                    | ### Configure light here. Use correct `entity_id`. RGB color picker link included.                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create GitHub Trigger node:**  
   - Add a new node of type **GitHub Trigger**.  
   - Set **Owner** to your GitHub username or organization (e.g., `dummydavid`).  
   - Set **Repository** to the target repository name (e.g., `DemoRepo`).  
   - Select **Events** as `*` to listen to all event types.  
   - Configure or select existing GitHub API credentials with appropriate permissions (read access to repository and webhook creation).  
   - Save the node.

2. **Create Home Assistant node:**  
   - Add a new node of type **Home Assistant**.  
   - Set **Domain** to `light`.  
   - Set **Service** to `turn_on`.  
   - Under **Service Attributes**, add two attributes:  
     - `entity_id` with value of your target light entity ID (default `light.lamp`).  
     - `rgb_color` with value set to `[255,0,0]` (red color). Use expression mode: `={{[255,0,0]}}`.  
   - Configure or select existing Home Assistant API credentials for your instance.  
   - Save the node.

3. **Connect nodes:**  
   - Connect the output of the GitHub Trigger node to the input of the Home Assistant node.

4. **Add notes (optional):**  
   - Add sticky notes for documentation and configuration guidance if desired.

5. **Activate workflow:**  
   - Set the workflow status to active to start listening for GitHub events.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                | Context or Link                                                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| It is likely the name of the light you want to control is not `light.lamp`. Find your light’s `entity_id` in your Home Assistant instance. See community discussion for help. | https://community.home-assistant.io/t/find-the-entity-id-of-a-yeelight-light-in-manual-mode-or-automatic-mode-doesnt-work/165557   |
| To change the light color, use an RGB color picker tool online. Default is red (255,0,0).                                                                                    | https://www.google.com/search?q=rgb+color+picker                                                                                   |
| GitHub credentials must have webhook creation permissions for the repository to register event triggers properly.                                                          | https://docs.n8n.io/integrations/builtin/credentials/github/                                                                       |
| Home Assistant credentials require API access with permissions to control lights.                                                                                           | https://docs.n8n.io/integrations/builtin/credentials/homeassistant/                                                                |