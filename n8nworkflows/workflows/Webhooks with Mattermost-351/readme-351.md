Webhooks with Mattermost

https://n8nworkflows.xyz/workflows/webhooks-with-mattermost-351


# Webhooks with Mattermost

### 1. Workflow Overview

This workflow is designed to respond to Mattermost slash commands via a webhook. When a user triggers the slash command in Mattermost, the workflow fetches a random cocktail recipe from an external API and posts the drink name, instructions, serving glass, and an image back to the same Mattermost channel. It is structured into three logical blocks:  
- **1.1 Input Reception:** Receiving the slash command HTTP POST webhook from Mattermost.  
- **1.2 External API Data Retrieval:** Fetching a random cocktail recipe from TheCocktailDB API.  
- **1.3 Mattermost Message Posting:** Formatting and sending the cocktail details back into the originating Mattermost channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
This block captures the incoming HTTP POST request triggered by the Mattermost slash command. This node acts as the entry point, receiving the command payload and extracting necessary information (such as the channel ID) for subsequent processing.

- **Nodes Involved:**  
  - `Webhook`

- **Node Details:**  

| Node Name | Details |
|---|---|
| Webhook | 
- **Type:** Webhook node, acts as an HTTP endpoint for external triggers.  
- **Configuration:**  
  - HTTP method: POST  
  - Path: `webhook` (meaning the webhook URL ends with `/webhook`)  
  - No additional options enabled.  
- **Key Expressions/Variables:** None directly used here, but the node outputs the POST body JSON, including Mattermost slash command data such as `channel_id`.  
- **Input Connections:** None (trigger node).  
- **Output Connections:** Sends output to the `HTTP Request` node.  
- **Edge Cases/Potential Failures:**  
  - Invalid or missing POST data could disrupt downstream processing.  
  - Unauthorized or unexpected POST requests if webhook URL is not secured.  
  - HTTP method other than POST will not trigger the node.  
- **Version Requirements:** Compatible with n8n version supporting Webhook node (version 1+).  
- **Sub-workflow:** None.

---

#### 2.2 External API Data Retrieval

- **Overview:**  
This block fetches a random cocktail recipe by making an HTTP GET request to TheCocktailDB API. The returned JSON contains detailed information about a drink, which will be used in the message posted back to Mattermost.

- **Nodes Involved:**  
  - `HTTP Request`

- **Node Details:**  

| Node Name | Details |
|---|---|
| HTTP Request | 
- **Type:** HTTP Request node, used for fetching data from external APIs.  
- **Configuration:**  
  - URL: `https://www.thecocktaildb.com/api/json/v1/1/random.php` (endpoint returning a random cocktail).  
  - HTTP method: GET (default).  
  - No authentication or additional HTTP options configured.  
- **Key Expressions/Variables:** None required; static API endpoint is used.  
- **Input Connections:** Receives input from the `Webhook` node.  
- **Output Connections:** Sends output to the `Mattermost` node.  
- **Edge Cases/Potential Failures:**  
  - API downtime or network issues causing request failure or timeout.  
  - Unexpected API response structure or empty data.  
  - Rate limiting by TheCocktailDB API.  
- **Version Requirements:** Compatible with n8n HTTP Request node version 1+.  
- **Sub-workflow:** None.

---

#### 2.3 Mattermost Message Posting

- **Overview:**  
This block formats data from the cocktail API response and posts a message to the Mattermost channel that issued the slash command. The message includes the drink name, instructions, serving glass, and an image attachment.

- **Nodes Involved:**  
  - `Mattermost`

- **Node Details:**  

| Node Name | Details |
|---|---|
| Mattermost | 
- **Type:** Mattermost node, responsible for sending messages to Mattermost channels.  
- **Configuration:**  
  - Message text uses expressions to dynamically insert data from the `HTTP Request` node JSON response:  
    ```
    Why not try {{$node["HTTP Request"].json["drinks"][0]["strDrink"]}}?
    {{$node["HTTP Request"].json["drinks"][0]["strInstructions"]}} Serve in {{$node["HTTP Request"].json["drinks"][0]["strGlass"]}}.
    ```  
  - Channel ID is dynamically set from the webhook payload’s `channel_id` field:  
    `={{$node["Webhook"].json["body"]["channel_id"]}}`  
  - Attachments include an image URL pointing to the drink’s thumbnail:  
    ```
    image_url: {{$node["HTTP Request"].json["drinks"][0]["strDrinkThumb"]}}
    ```  
  - Credentials: Uses a pre-configured Mattermost API credential named `Mattermost`.  
  - No additional options or advanced settings configured.  
- **Key Expressions/Variables:** See above; uses JSON paths to deeply access the API response and webhook body.  
- **Input Connections:** Receives input from the `HTTP Request` node.  
- **Output Connections:** None (terminal node).  
- **Edge Cases/Potential Failures:**  
  - Invalid or expired Mattermost API credentials causing authentication failures.  
  - Incorrect/empty channel ID resulting in message delivery failure.  
  - API limits or network errors posting message.  
  - If the API response lacks expected fields, message may have missing content or formatting issues.  
- **Version Requirements:** Compatible with n8n Mattermost node version 1+.  
- **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name     | Node Type        | Functional Role                 | Input Node(s) | Output Node(s) | Sticky Note                                                                                                              |
|---------------|------------------|--------------------------------|---------------|----------------|--------------------------------------------------------------------------------------------------------------------------|
| Webhook       | Webhook          | Receive slash command trigger   | None          | HTTP Request   |                                                                                                                          |
| HTTP Request  | HTTP Request     | Fetch random cocktail data      | Webhook       | Mattermost     |                                                                                                                          |
| Mattermost    | Mattermost       | Post message to Mattermost      | HTTP Request  | None           |                                                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook Node:**  
   - Add a new **Webhook** node.  
   - Set HTTP Method to `POST`.  
   - Set the path to `webhook`. This will be the endpoint URL (e.g., `https://<n8n-instance>/webhook`).  
   - No additional options needed.  

2. **Create the HTTP Request Node:**  
   - Add an **HTTP Request** node.  
   - Set the URL to `https://www.thecocktaildb.com/api/json/v1/1/random.php`.  
   - Use the default GET method.  
   - Leave other options default (no authentication).  
   - Connect the output of the Webhook node to the input of this node.

3. **Create the Mattermost Node:**  
   - Add a **Mattermost** node.  
   - Configure the **Credentials** with your Mattermost API credential (OAuth2 or personal access token). Create this credential if not already set.  
   - Set the **Channel ID** parameter to: `={{$node["Webhook"].json["body"]["channel_id"]}}` to dynamically post in the channel from which the slash command originated.  
   - Set the **Message** parameter to the following expression to format the cocktail info:  
     ```
     Why not try {{$node["HTTP Request"].json["drinks"][0]["strDrink"]}}?
     {{$node["HTTP Request"].json["drinks"][0]["strInstructions"]}} Serve in {{$node["HTTP Request"].json["drinks"][0]["strGlass"]}}.
     ```  
   - Add an attachment with the image URL set to:  
     ```
     {{$node["HTTP Request"].json["drinks"][0]["strDrinkThumb"]}}
     ```  
   - Connect the output of the HTTP Request node to the input of the Mattermost node.

4. **Activate the Workflow:**  
   - Save and activate the workflow.  
   - Use the webhook URL plus `/webhook` to configure your Mattermost slash command integration.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                  |
|--------------------------------------------------------------------------------------------------|-------------------------------------------------|
| This workflow is inspired by a similar workflow but is triggered specifically by a slash command. | Original workflow referenced by the user note.  |
| Mattermost API credential setup is required to enable posting messages.                           | n8n Mattermost node documentation: https://docs.n8n.io/integrations/n8n-nodes-base.mattermost/ |
| TheCocktailDB API provides free public endpoints for cocktail recipes.                           | https://www.thecocktaildb.com/api.php            |

---

This documentation fully describes the “Mattermost Webhook” workflow, enabling users to understand, reproduce, and modify it effectively.