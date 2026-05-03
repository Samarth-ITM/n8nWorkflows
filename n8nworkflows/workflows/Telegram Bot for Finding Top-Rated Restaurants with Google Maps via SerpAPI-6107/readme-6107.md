Telegram Bot for Finding Top-Rated Restaurants with Google Maps via SerpAPI

https://n8nworkflows.xyz/workflows/telegram-bot-for-finding-top-rated-restaurants-with-google-maps-via-serpapi-6107


# Telegram Bot for Finding Top-Rated Restaurants with Google Maps via SerpAPI

### 1. Workflow Overview

This workflow implements a **Telegram bot** designed to help users find the top-rated restaurants in a specified area using the Google Maps data accessed via SerpAPI. It primarily targets users who want quick restaurant recommendations by sending a simple Telegram message.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Receives and filters Telegram messages for restaurant queries.
- **1.2 Area Parsing:** Extracts the location (area) from the user's message.
- **1.3 Geocoding:** Converts the extracted area name into geographic coordinates via Nominatim.
- **1.4 Restaurant Search:** Queries SerpAPI’s Google Maps engine to find restaurants in the specified area.
- **1.5 Response Formatting:** Constructs a formatted Telegram message with the top 5 restaurants by rating.
- **1.6 Response Delivery:** Sends the formatted message back to the user on Telegram.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Listens for Telegram messages specifically from an allowed channel and triggers the workflow upon receiving relevant updates.
- **Nodes Involved:** `Telegram Trigger`
- **Node Details:**

  - **Telegram Trigger**
    - Type: Telegram Trigger (Webhook-based)
    - Config: Listens for `message` updates only; restricts messages to a specific Telegram channel (`$env.telegram_channel_id` environment variable).
    - Inputs: Incoming Telegram webhook event (message).
    - Outputs: Passes message JSON to next node.
    - Version: 1.2
    - Edge cases: If environment variable is missing or invalid, no messages will be processed; network or Telegram API downtime can cause trigger failure.
    - Credentials: Uses Telegram API credentials with OAuth2 token.

#### 1.2 Area Parsing

- **Overview:** Extracts the area/location query from the text message. Defaults to "Cairo" if no area is provided.
- **Nodes Involved:** `Parse Area`
- **Node Details:**

  - **Parse Area**
    - Type: Function (JavaScript)
    - Config: Parses incoming Telegram message text expecting format like `/rest <area>`. Uses regex to remove the command prefix and trims whitespace.
    - Key expression:  
      ```js
      const txt = $json.message?.text || '';
      const area = txt.replace(/^(\/rest\s*)/i, '').trim() || 'Cairo';
      return [{ area }];
      ```
    - Inputs: Message JSON from `Telegram Trigger`.
    - Outputs: JSON object with `area` string.
    - Version: 1
    - Edge cases: Missing or malformed commands will default to "Cairo". Bot commands typed incorrectly (e.g., missing `/rest`) will still default, possibly giving irrelevant results.

#### 1.3 Geocoding

- **Overview:** Converts the parsed area string into geographic coordinates using Nominatim OpenStreetMap API.
- **Nodes Involved:** `Geocode (Nominatim)`
- **Node Details:**

  - **Geocode (Nominatim)**
    - Type: HTTP Request
    - Config:  
      - URL: `https://nominatim.openstreetmap.org/search`  
      - Query parameters: `format=json`, `limit=1`, `q={{ $node['Parse Area'].json.area }}`
      - Expects JSON response with location coordinates.
    - Inputs: Receives `area` from `Parse Area`.
    - Outputs: Geocoding response JSON with lat/lon.
    - Version: 4.2
    - Edge cases: If no results found, downstream nodes may fail or return empty results; rate limits or service downtime at Nominatim can cause request failures.
    - Notes: No authentication needed; respects usage policy for Nominatim.

#### 1.4 Restaurant Search

- **Overview:** Uses SerpAPI to search Google Maps for restaurants in the specified area and country, retrieving detailed restaurant data.
- **Nodes Involved:** `Find Restaurants (SerpAPI)`
- **Node Details:**

  - **Find Restaurants (SerpAPI)**
    - Type: HTTP Request
    - Config:  
      - URL: `https://serpapi.com/search.json`  
      - Query parameters include:  
        - `engine=Maps` (Google Maps engine)  
        - `q={{ $env.country_name }}+{{$node["Parse Area"].json.area}}+restaurants` (search query combining country and area)  
        - `hl=en` (language)  
        - `type=search`  
        - `api_key={{ $env.serp_api_key }}`
    - Inputs: Area from `Parse Area`, country name from environment.
    - Outputs: JSON with local Google Maps restaurant results.
    - Version: 4.2
    - Edge cases: Invalid or missing API key leads to errors; API rate limits or network issues can cause failures; if no restaurants found, empty output handled downstream.
    - Credentials: Uses SerpAPI key from environment variable.

#### 1.5 Response Formatting

- **Overview:** Sorts and formats the top five restaurants into a Markdown message suitable for Telegram, including ratings, contact info, service options, and Google Maps links.
- **Nodes Involved:** `Format Reply`
- **Node Details:**

  - **Format Reply**
    - Type: Function (JavaScript)
    - Config:  
      - Sorts restaurants by rating descending.  
      - Limits to top 5.  
      - Creates clickable Google Maps links based on GPS coordinates.  
      - Displays service options (dine-in, takeaway) with checkmark icons.  
      - Formats message using Markdown for Telegram.
    - Key expression snippet:
      ```js
      const area = $node['Parse Area'].json.area;
      const list = ($json.local_results || [])
        .sort((a, b) => (b.rating || 0) - (a.rating || 0))
        .slice(0, 5)
        .map((p, i) => {
          // build mapUrl and icons...
          return `${i + 1}. *${p.title}* ⭐${p.rating || '-'}\n...`;
        }).join('\n\n');
      return [{
        chat_id: $node['Telegram Trigger'].json.message.chat.id,
        text: `🍽 *Best Restaurants in ${area} – Top 5*\n\n${list}`,
        parse_mode: 'Markdown'
      }];
      ```
    - Inputs: Restaurant results JSON from SerpAPI.
    - Outputs: Telegram message JSON with `chat_id`, `text`, and parse mode.
    - Version: 1
    - Edge cases: Missing fields like rating or phone handled gracefully with fallbacks; empty restaurant list results in empty message body.
  
#### 1.6 Response Delivery

- **Overview:** Sends the formatted message back to the user’s Telegram chat.
- **Nodes Involved:** `Send to Telegram`
- **Node Details:**

  - **Send to Telegram**
    - Type: Telegram node (message send)
    - Config:  
      - `chatId` and `text` are dynamically set from incoming JSON.  
      - `appendAttribution` disabled to avoid extra footer text.
    - Inputs: Takes formatted message JSON from `Format Reply`.
    - Outputs: Confirmation of Telegram message sent.
    - Version: 1.2
    - Credentials: Uses same Telegram API credentials as trigger.
    - Edge cases: Message may fail if chat ID invalid or blocked bot; Telegram API limits or downtime can cause failure.

---

### 3. Summary Table

| Node Name             | Node Type                 | Functional Role                 | Input Node(s)       | Output Node(s)      | Sticky Note                                           |
|-----------------------|---------------------------|--------------------------------|---------------------|---------------------|-------------------------------------------------------|
| Telegram Trigger       | Telegram Trigger           | Receive Telegram messages       | —                   | Parse Area          |                                                       |
| Parse Area            | Function                  | Parse location from message     | Telegram Trigger    | Geocode (Nominatim) | ## Parse Area<br>Process Input                         |
| Geocode (Nominatim)    | HTTP Request              | Convert area to geocoordinates  | Parse Area          | Find Restaurants    | ## Geocode<br>Convert area to map point                |
| Find Restaurants (SerpAPI)| HTTP Request            | Search top restaurants via API  | Geocode (Nominatim) | Format Reply        | ## Search<br>Find Top 5 rated resturants              |
| Format Reply           | Function                  | Format Telegram message         | Find Restaurants    | Send to Telegram    | ## Response<br>Prepare response                        |
| Send to Telegram       | Telegram                  | Send message back to user       | Format Reply        | —                   | ## Respond<br>Send report to telegram                  |
| Sticky Note            | Sticky Note               | Prerequisites notes             | —                   | —                   | ## Prerequisites <br>**SerpAPI**: api key<br>**Countery Name**: eg.. Egypt |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**
   - Type: `Telegram Trigger`
   - Parameters:
     - Updates: Select only `message`
     - Additional Fields: Set `chatIds` to environment variable `telegram_channel_id`
   - Credentials: Link your Telegram Bot API credential
   - Position: Starting point

2. **Add Function Node "Parse Area"**
   - Type: `Function`
   - Parameters: Insert code to extract the area from Telegram message text:
     ```js
     const txt = $json.message?.text || '';
     const area = txt.replace(/^(\/rest\s*)/i, '').trim() || 'Cairo';
     return [{ area }];
     ```
   - Connect output of `Telegram Trigger` to this node.

3. **Add HTTP Request Node "Geocode (Nominatim)"**
   - Type: `HTTP Request`
   - Parameters:
     - URL: `https://nominatim.openstreetmap.org/search`
     - Query Parameters:
       - `format` = `json`
       - `limit` = `1`
       - `q` = `={{$node['Parse Area'].json.area}}`
     - Response Format: JSON
   - Connect output of `Parse Area` to this node.

4. **Add HTTP Request Node "Find Restaurants (SerpAPI)"**
   - Type: `HTTP Request`
   - Parameters:
     - URL: `https://serpapi.com/search.json`
     - Query Parameters:
       - `engine` = `Maps`
       - `q` = `={{ $env.country_name }}+{{$node["Parse Area"].json.area}}+restaurants`
       - `hl` = `en`
       - `type` = `search`
       - `api_key` = `={{ $env.serp_api_key }}`
     - Response Format: JSON
   - Connect output of `Geocode (Nominatim)` to this node.

5. **Add Function Node "Format Reply"**
   - Type: `Function`
   - Parameters: Insert code to format top 5 restaurants into a Markdown Telegram message:
     ```js
     const area = $node['Parse Area'].json.area;
     const icon = v => v ? '✅' : '❌';
     const list = ($json.local_results || [])
       .sort((a, b) => (b.rating || 0) - (a.rating || 0))
       .slice(0, 5)
       .map((p, i) => {
         const { latitude: lat, longitude: lng } = p.gps_coordinates || {};
         const mapUrl = lat && lng ? `https://maps.google.com/?q=${lat},${lng}` : (p.link || '');
         const dineIn = icon(p.service_options?.dine_in);
         const takeAway = icon(p.service_options?.takeaway);
         return `${i + 1}. *${p.title}* ⭐${p.rating || '-'}\n`
              + `${p.type || ''} | ${p.open_state || ''}\n`
              + `☎️ ${p.phone || '—'} ${p.website ? `| 🌐 ${p.website}` : ''}\n`
              + `🍽️ Dine-in: ${dineIn} | Takeaway: ${takeAway}\n`
              + `📍 [On the Map](${mapUrl})`;
       }).join('\n\n');
     return [{
       chat_id: $node['Telegram Trigger'].json.message.chat.id,
       text: `🍽 *Best Restaurants in ${area} – Top 5*\n\n${list}`,
       parse_mode: 'Markdown'
     }];
     ```
   - Connect output of `Find Restaurants (SerpAPI)` to this node.

6. **Add Telegram Node "Send to Telegram"**
   - Type: `Telegram`
   - Parameters:
     - `chatId`: `={{$json.chat_id}}`
     - `text`: `={{$json.text}}`
     - Additional Fields: Disable `appendAttribution`
   - Credentials: Use the same Telegram API credentials
   - Connect output of `Format Reply` to this node.

7. **Set Environment Variables:**
   - `telegram_channel_id`: Telegram channel or chat ID allowed to send messages to the bot.
   - `serp_api_key`: Valid SerpAPI key with access to Google Maps engine.
   - `country_name`: Country for the search (e.g., "Egypt") to scope SerpAPI queries.

8. **Optional: Add Sticky Notes for Documentation**
   - Add notes describing each block for easier maintenance and clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                    |
|-----------------------------------------------------------------------------------------------|---------------------------------------------------|
| Workflow requires valid SerpAPI key and Telegram API credentials configured as environment vars| Prerequisites                                      |
| Country name environment variable scopes restaurant search geographically                      | Geographic scoping                                 |
| Uses Nominatim OpenStreetMap API for geocoding without authentication                         | Geocoding service details                          |
| Message formatting uses Telegram Markdown syntax with links to Google Maps                     | Telegram message formatting                         |
| SerpAPI docs: https://serpapi.com/search-api                                                | External API documentation                         |
| Telegram Bot API docs: https://core.telegram.org/bots/api                                    | Telegram integration details                       |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created using n8n, an integration and automation tool. This process strictly complies with relevant content policies and contains no illegal, offensive, or protected material. All data processed is legal and publicly available.