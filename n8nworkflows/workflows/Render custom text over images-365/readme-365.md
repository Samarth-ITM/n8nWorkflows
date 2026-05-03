Render custom text over images

https://n8nworkflows.xyz/workflows/render-custom-text-over-images-365


# Render custom text over images

---

### 1. Workflow Overview

This workflow automates the creation and sharing of a custom cocktail recipe image every Friday at 6 PM. It retrieves a random cocktail’s data from an external API, generates a branded image with the cocktail details using Bannerbear, and posts the image to a Rocket.Chat channel.

Logical blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow weekly on Fridays at 6 PM using a Cron node.

- **1.2 Data Retrieval:** Fetches a random cocktail’s data (name, image, recipe) from TheCocktailDB API via an HTTP Request node.

- **1.3 Image Generation:** Sends the cocktail data to Bannerbear to generate a customized image based on a predefined template.

- **1.4 Image Sharing:** Posts the generated image to a specified Rocket.Chat channel.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  Triggers the workflow execution every Friday at 18:00 to automate content creation and sharing.

- **Nodes Involved:**  
  - Cron

- **Node Details:**

  - **Cron**  
    - Type: Trigger node for scheduled execution  
    - Configuration: Set to trigger *every week* on weekday 5 (Friday) at 18:00 (6 PM)  
    - Inputs: None (trigger node)  
    - Outputs: Connected to HTTP Request node  
    - Version: 1  
    - Edge cases:  
      - If the n8n instance is offline or paused at trigger time, the workflow will not run.  
      - Timezone considerations depend on n8n server settings; mismatch can cause unexpected trigger time.  
    - No sub-workflow references.

#### 1.2 Data Retrieval

- **Overview:**  
  Retrieves data for a random cocktail including its name, image URL, and recipe instructions from TheCocktailDB API.

- **Nodes Involved:**  
  - HTTP Request

- **Node Details:**

  - **HTTP Request**  
    - Type: HTTP Request node to call external REST API  
    - Configuration:  
      - URL: `https://www.thecocktaildb.com/api/json/v1/1/random.php`  
      - Method: GET (default)  
      - No authentication or additional headers  
    - Inputs: Triggered by Cron node  
    - Outputs: JSON response containing an array under `drinks` with cocktail data  
    - Version: 1  
    - Key expressions:  
      - Access data using expressions like `$node["HTTP Request"].json["drinks"][0]["strDrink"]` for cocktail name  
    - Edge cases:  
      - API downtime or rate limiting may cause request failures or empty responses  
      - Unexpected response format can cause expression errors downstream  
      - Network latency or timeout issues  
    - No sub-workflow references.

#### 1.3 Image Generation

- **Overview:**  
  Uses the Bannerbear node to generate a custom image by injecting the cocktail’s image, name, and recipe text into a predefined template.

- **Nodes Involved:**  
  - Bannerbear

- **Node Details:**

  - **Bannerbear**  
    - Type: Bannerbear node for dynamic image creation  
    - Configuration:  
      - Template ID: (must be set to a valid Bannerbear template configured to accept fields `cocktail-image`, `title`, and `recipe`)  
      - Modifications:  
        - `cocktail-image`: image URL from the HTTP Request node (`strDrinkThumb`)  
        - `title`: cocktail name (`strDrink`)  
        - `recipe`: cocktail instructions (`strInstructions`)  
      - Additional field: `waitForImage` enabled to ensure the image is processed before continuing  
    - Inputs: Receives cocktail data from HTTP Request node  
    - Outputs: JSON containing the generated image URL under `image_url`  
    - Version: 1  
    - Credentials: Bannerbear API key required  
    - Edge cases:  
      - Invalid or empty template ID will cause failure  
      - Missing or malformed modifications can lead to incomplete image generation  
      - Bannerbear API rate limits or downtime  
      - Large text inputs may overflow template text areas if template is not designed for variable length  
    - No sub-workflow references.

#### 1.4 Image Sharing

- **Overview:**  
  Shares the generated cocktail image by posting it as an attachment to a Rocket.Chat channel.

- **Nodes Involved:**  
  - Rocketchat

- **Node Details:**

  - **Rocketchat**  
    - Type: Rocket.Chat node to send messages with attachments  
    - Configuration:  
      - Channel: Must be set to the target Rocket.Chat channel name or ID  
      - Attachments: Single image attachment with URL taken from Bannerbear output (`image_url`)  
    - Inputs: Receives image URL from Bannerbear node  
    - Outputs: None (end of workflow)  
    - Version: 1  
    - Credentials: Rocket.Chat API credentials required  
    - Edge cases:  
      - Invalid or unauthorized channel name causes posting failure  
      - Network issues or Rocket.Chat API downtime  
      - Image URL invalid or expired (if Bannerbear images are temporary)  
    - No sub-workflow references.

---

### 3. Summary Table

| Node Name    | Node Type           | Functional Role                  | Input Node(s) | Output Node(s) | Sticky Note          |
|--------------|---------------------|--------------------------------|---------------|----------------|----------------------|
| Cron         | Cron Trigger        | Scheduled weekly trigger        | -             | HTTP Request   |                      |
| HTTP Request | HTTP Request        | Fetch random cocktail data      | Cron          | Bannerbear     |                      |
| Bannerbear   | Bannerbear          | Generate customized cocktail image | HTTP Request  | Rocketchat     |                      |
| Rocketchat   | Rocket.Chat Message | Share generated image to chat   | Bannerbear    | -              |                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cron Node**  
   - Add a Cron node named "Cron"  
   - Set trigger time: Weekly on Friday (weekday 5), hour 18  
   - Leave other settings default  
   - No credentials required

2. **Create HTTP Request Node**  
   - Add HTTP Request node named "HTTP Request"  
   - Set URL to `https://www.thecocktaildb.com/api/json/v1/1/random.php`  
   - Method: GET (default)  
   - Connect "Cron" node’s main output to this node’s input  
   - No authentication required

3. **Create Bannerbear Node**  
   - Add Bannerbear node named "Bannerbear"  
   - Set Template ID to your valid Bannerbear cocktail template ID  
   - Under Modifications:  
     - Add modification for `cocktail-image` → set to expression: `{{$node["HTTP Request"].json["drinks"][0]["strDrinkThumb"]}}`  
     - Add modification for `title` → set to expression: `{{$node["HTTP Request"].json["drinks"][0]["strDrink"]}}`  
     - Add modification for `recipe` → set to expression: `{{$node["HTTP Request"].json["drinks"][0]["strInstructions"]}}`  
   - Enable "Wait for image" in additional fields to ensure image generation completes before proceeding  
   - Connect "HTTP Request" node output to this node’s input  
   - Set Bannerbear API credentials

4. **Create Rocket.Chat Node**  
   - Add Rocket.Chat node named "Rocketchat"  
   - Set target channel (e.g., `#general` or specific channel ID)  
   - Under Attachments: add an attachment with imageUrl set to expression: `{{$node["Bannerbear"].json["image_url"]}}`  
   - Connect "Bannerbear" node output to this node’s input  
   - Set Rocket.Chat API credentials

5. **Finalize and Activate Workflow**  
   - Check all connections: Cron → HTTP Request → Bannerbear → Rocketchat  
   - Save workflow  
   - Activate workflow to enable scheduled execution

---

### 5. General Notes & Resources

| Note Content                                                             | Context or Link                                         |
|--------------------------------------------------------------------------|--------------------------------------------------------|
| Bannerbear template must be pre-configured with the fields: `cocktail-image`, `title`, `recipe` | Bannerbear official docs: https://www.bannerbear.com/docs/ |
| Rocket.Chat API credentials require appropriate permissions and token setup | Rocket.Chat API guide: https://developer.rocket.chat/api/rest-api/ |
| Timezone of the Cron node depends on n8n server settings; verify to ensure correct trigger timing | n8n documentation on Cron node                         |
| TheCocktailDB is a free API for cocktail data: https://www.thecocktaildb.com/api.php | API documentation                                       |

---