Standup bot (1/4): Initialize

https://n8nworkflows.xyz/workflows/standup-bot--1-4---initialize-1472


# Standup bot (1/4): Initialize

### 1. Workflow Overview

This workflow, titled **"Standup Bot - Initialize"**, serves as the initial setup step for a Mattermost Standup Bot. Its primary purpose is to generate a default configuration file containing vital connection and authentication parameters needed by subsequent workflows of the bot. This configuration is saved as a JSON file on the local filesystem and must be run once manually during the setup phase. The workflow logically consists of three main blocks:

- **1.1 Manual Trigger:** Entry point for human initiation.
- **1.2 Configuration Setup:** Defines and sets default configuration values.
- **1.3 Configuration File Creation:** Converts the configuration to binary form and writes it to disk.

---

### 2. Block-by-Block Analysis

#### 2.1 Manual Trigger

- **Overview:**  
  This block allows the workflow to be started manually by a user. It acts as the entry point for the initialization process.

- **Nodes Involved:**  
  - `On clicking 'execute'`

- **Node Details:**

  - **Node Name:** On clicking 'execute'  
    - **Type:** Manual Trigger  
    - **Role:** Starts the workflow on manual user execution within n8n's editor or UI.  
    - **Configuration:** No parameters configured; default manual trigger settings.  
    - **Key Expressions/Variables:** None.  
    - **Connections:** Output connects to `Use Default Config` node.  
    - **Version:** Compatible with n8n v1.x and above; no special version requirements.  
    - **Potential Failures:** None expected; manual trigger does not fail but workflow halts until triggered.  
    - **Sub-workflow:** None.

#### 2.2 Configuration Setup

- **Overview:**  
  This block sets the default configuration parameters for the Mattermost Standup Bot. It defines all necessary keys and tokens required for authentication and communication.

- **Nodes Involved:**  
  - `Use Default Config`

- **Node Details:**

  - **Node Name:** Use Default Config  
    - **Type:** Set  
    - **Role:** Defines the key configuration parameters used by the bot, such as tokens, URLs, and user IDs.  
    - **Configuration:**  
      - Sets string values for:  
        - `config.slashCmdToken` (Mattermost Slash Command token)  
        - `config.mattermostBaseUrl` (Base URL of the Mattermost instance)  
        - `config.botUserToken` (Mattermost bot user token)  
        - `config.n8nWebhookUrl` (Webhook URL for the action workflow)  
        - `config.botUserId` (User ID of the Mattermost bot)  
      - `keepOnlySet` is enabled, so only these values pass forward.  
    - **Key Expressions/Variables:** Static string values are hardcoded; no dynamic expressions used.  
    - **Connections:** Output connects to `Move Binary Data` node.  
    - **Version:** Compatible with n8n v1.x; no special version requirements.  
    - **Potential Failures:** Misconfiguration or incorrect token values will propagate invalid config but no runtime failure at this node.  
    - **Sub-workflow:** None.

#### 2.3 Configuration File Creation

- **Overview:**  
  This block converts the JSON configuration data to binary format and writes it to a file on the local filesystem. This step ensures the config is stored persistently for use by other workflows.

- **Nodes Involved:**  
  - `Move Binary Data`  
  - `Write Binary File`

- **Node Details:**

  - **Node Name:** Move Binary Data  
    - **Type:** Move Binary Data  
    - **Role:** Converts JSON data into a UTF-8 encoded binary file stream suitable for writing to disk.  
    - **Configuration:**  
      - Mode set to `jsonToBinary`  
      - Encoding set to `utf8`  
      - File name set to `standup-bot-config.json`  
    - **Key Expressions/Variables:** None; static parameters.  
    - **Connections:** Output connects to `Write Binary File` node.  
    - **Version:** Compatible with n8n v1.x; no additional requirements.  
    - **Potential Failures:** Encoding errors if data is malformed; unlikely here as data is static and well-formed.  
    - **Sub-workflow:** None.

  - **Node Name:** Write Binary File  
    - **Type:** Write Binary File  
    - **Role:** Writes the binary data to a file at `/home/node/.n8n/standup-bot-config.json`.  
    - **Configuration:**  
      - File path explicitly set to `/home/node/.n8n/standup-bot-config.json` (absolute path).  
    - **Key Expressions/Variables:** None.  
    - **Connections:** Terminal node; no outputs.  
    - **Version:** Requires n8n environment to have write permissions to `/home/node/.n8n/` directory.  
    - **Potential Failures:**  
      - Filesystem permission errors.  
      - Disk full or write errors.  
      - Path does not exist or inaccessible.  
    - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name             | Node Type         | Functional Role                      | Input Node(s)            | Output Node(s)          | Sticky Note                                                                                                                   |
|-----------------------|-------------------|------------------------------------|--------------------------|-------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| On clicking 'execute'  | Manual Trigger    | Manual entry point for workflow    | —                        | Use Default Config       |                                                                                                                               |
| Use Default Config     | Set               | Defines default bot configuration  | On clicking 'execute'     | Move Binary Data         | This node holds default config values: slashCmdToken, mattermostBaseUrl, botUserToken, n8nWebhookUrl, botUserId                |
| Move Binary Data       | Move Binary Data  | Converts JSON config to binary     | Use Default Config        | Write Binary File        | Converts JSON config to UTF-8 encoded binary data for file writing                                                             |
| Write Binary File      | Write Binary File | Writes config to filesystem        | Move Binary Data          | —                       | Writes the config file to /home/node/.n8n/standup-bot-config.json; requires proper filesystem permissions                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow and name it**: "Standup Bot - Initialize".

2. **Add a Manual Trigger node**:  
   - Name it `On clicking 'execute'`.  
   - Leave all parameters as default (no inputs required). This node will allow manual workflow start.

3. **Add a Set node**:  
   - Name it `Use Default Config`.  
   - Configure to set the following string values:  
     - `config.slashCmdToken`: `"xxxxx"` (replace with your Mattermost Slash Command token)  
     - `config.mattermostBaseUrl`: `"https://mattermost.yourdomain.tld"` (replace with your Mattermost base URL)  
     - `config.botUserToken`: `"xxxxx"` (replace with your Mattermost bot user token)  
     - `config.n8nWebhookUrl`: `"https://n8n.yourdomain.tld/webhook/standup-bot/action/f6f9b174745fa4651f750c36957d674c"` (replace with your actual webhook URL)  
     - `config.botUserId`: `"xxxxx"` (replace with your Mattermost bot user ID)  
   - Enable the option **"Keep Only Set"** to ensure only these values proceed forward.

4. **Connect `On clicking 'execute'` node's main output to `Use Default Config` node's input.**

5. **Add a Move Binary Data node**:  
   - Name it `Move Binary Data`.  
   - Set the mode to `jsonToBinary`.  
   - Configure options:  
     - Encoding: `utf8`  
     - File Name: `standup-bot-config.json`.

6. **Connect `Use Default Config` node's main output to `Move Binary Data` node's input.**

7. **Add a Write Binary File node**:  
   - Name it `Write Binary File`.  
   - Set the file name to `/home/node/.n8n/standup-bot-config.json`.  
   - Ensure that the n8n runtime environment has write permissions to this path.

8. **Connect `Move Binary Data` node's main output to `Write Binary File` node's input.**

9. **Save the workflow.**

10. **Run the workflow manually once** using the manual trigger to generate the configuration file.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                               |
|---------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| This workflow is part 1 of 4 for the Mattermost Standup Bot. The configuration file must be created first. | Initial setup step; subsequent workflows depend on this config file.                                         |
| The config file path is `/home/node/.n8n/standup-bot-config.json`. Ensure n8n has filesystem access here. | Important for operational success; missing permissions cause failures.                                       |
| Replace placeholder values (`xxxxx`, URLs) with actual tokens and URLs before running the workflow.      | Prevents authentication and connectivity issues with Mattermost and n8n webhooks.                            |
| For webhook URL, use the URL to the "Action from MM" webhook in the "Standup Bot - Worker" workflow.     | Ensures Mattermost actions reach the worker workflow correctly.                                              |
| Mattermost tokens and user IDs can be found or generated in the Mattermost system administration panel. | Useful for accurate configuration.                                                                           |

---

This documentation provides a complete, structured reference to understand, reproduce, and maintain the Standup Bot initialization workflow in n8n.