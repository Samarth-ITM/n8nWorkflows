Find the best Roblox server for your game using a webhook and Roblox API

https://n8nworkflows.xyz/workflows/find-the-best-roblox-server-for-your-game-using-a-webhook-and-roblox-api-12194


# Find the best Roblox server for your game using a webhook and Roblox API

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Find the best Roblox server for your game using a webhook and Roblox API  
**Workflow name (internal):** Find the best Roblox Servers to Play

This workflow exposes an HTTP endpoint (Webhook) that, given a Roblox **Place ID** (`plid`), fetches the list of **public servers** from the Roblox Games API and selects the “best” server using one of three modes (**auto**, **ping**, **latency**). It then returns either:
- a **redirect** to Roblox’s web start URL (if `ar=true`), or
- a **JSON response** containing a Roblox **deeplink** (`roblox://...`) that starts the experience in the chosen instance.

### 1.1 Input Reception & Validation
Receives query parameters from a webhook call and verifies `plid` exists; otherwise returns a structured error response.

### 1.2 Normalization (“Auto Config”)
Normalizes inputs into consistent fields: `plid`, `act` (least/most), and `mode` (auto/ping/latency), with defaults.

### 1.3 Roblox API Fetch (Public Servers)
Calls `https://games.roblox.com/v1/games/{placeId}/servers/Public` to retrieve available public servers (excluding full servers).

### 1.4 Server Selection (Scoring / Best Pick)
Applies selection logic to pick a single “best” server based on the chosen mode.

### 1.5 Output: Redirect or JSON Deeplink
If `ar=true`, responds with an HTTP redirect to Roblox’s start URL; otherwise returns a JSON payload including `success` and the deeplink in `result`.

---

## 2. Block-by-Block Analysis

### Block 1 — Webhook Entry & Place ID Validation
**Overview:** Receives the incoming request and ensures a `plid` (Place ID) was provided. If missing, it returns an error response immediately.

**Nodes involved:**
- Webhook
- Place ID Set?
- Error #1
- Respond with Error #1

#### Node: **Webhook**
- **Type / role:** `Webhook` (entry point). Receives HTTP requests at a fixed path.
- **Key configuration:**
  - **Path:** `brsf`
  - **Response mode:** `responseNode` (the workflow must end with a Respond to Webhook node on all paths).
- **Inputs / outputs:**
  - **Output →** `Place ID Set?`
- **Edge cases / failures:**
  - If downstream branches do not reach a Respond node, requests may hang until timeout.
  - If n8n is behind a proxy, ensure webhook URL and HTTP method usage match your deployment settings.

#### Node: **Place ID Set?**
- **Type / role:** `IF` node for validating required query parameter.
- **Configuration choices (interpreted):**
  - Checks whether `{{$json.query.plid}}` **exists**.
- **Outputs:**
  - **True →** `Auto Config`
  - **False →** `Error #1`
- **Edge cases:**
  - If `plid` is present but empty string, “exists” may still pass depending on how n8n receives the query; consider adding a “not empty” check if needed.
  - Type validation is **strict** in this node; unexpected non-string formats could fail the condition evaluation in some cases.

#### Node: **Error #1**
- **Type / role:** `Set` node producing a standard error payload.
- **Configuration choices:**
  - Sets:
    - `success = false`
    - `result = "Error #1: Place ID is not set... ?plid=123456789"`
- **Outputs:**
  - **Output →** `Respond with Error #1`
- **Edge cases:**
  - None significant; static message.

#### Node: **Respond with Error #1**
- **Type / role:** `Respond to Webhook` returning the error payload.
- **Configuration choices:**
  - Responds with the incoming JSON from previous node (default behavior).
- **Inputs / outputs:**
  - **Input ←** `Error #1`
  - No downstream output (terminal).
- **Edge cases:**
  - HTTP status code is not explicitly set; by default this is typically `200`. If you want a proper client error, set status code to `400`.

---

### Block 2 — Input Normalization (“Auto Config”)
**Overview:** Consolidates query parameters into normalized fields with defaults, so later nodes don’t need to repeatedly reference webhook query paths.

**Nodes involved:**
- Auto Config

#### Node: **Auto Config**
- **Type / role:** `Set` node for computed parameters.
- **Configuration choices (interpreted):**
  - `plid = {{$json.query.plid}}`
  - `act = {{$if($json.act,$json.act,"most")}}`
    - **Important:** This references `$json.act` (not `$json.query.act`). Unless something upstream populates `act`, this will usually default to `"most"`. If you intended `?act=least`, you likely want `{{$if($json.query.act, $json.query.act, "most")}}`.
  - `mode = {{$if($json.query.mode, $json.query.mode,"auto")}}`
- **Outputs:**
  - **Output →** `Fetch Public Servers`
- **Edge cases / failures:**
  - Expression mismatch for `act` can silently ignore the caller’s query parameter.
  - If `plid` includes non-numeric content, Roblox API may return 400/404; the workflow does not currently validate numeric format.

---

### Block 3 — Roblox Public Server Fetch
**Overview:** Calls the Roblox Games API to fetch public servers for the given place, excluding full games, and ordering by player count.

**Nodes involved:**
- Fetch Public Servers

#### Node: **Fetch Public Servers**
- **Type / role:** `HTTP Request` node calling Roblox API.
- **Configuration choices (interpreted):**
  - **URL:** `https://games.roblox.com/v1/games/{{ $json.plid }}/servers/Public`
  - **Query parameters:**
    - `cursor` (declared but left empty; no pagination is implemented)
    - `sortOrder = {{ $if($json.act == "least","Asc","Desc") }}`
    - `excludeFullGames = true`
  - **Headers:** Adds browser-like headers (origin/referer/user-agent, etc.) and `cache-control: no-cache`
  - **Auth:** none (intentionally)
- **Inputs / outputs:**
  - **Input ←** `Auto Config`
  - **Output →** `Filter For The Best`
- **Edge cases / failures:**
  - Roblox may rate-limit or block requests; browser headers help but do not guarantee stability.
  - If Roblox returns non-200 responses (403/429/5xx), the node will fail unless “Continue On Fail” is enabled (it is not).
  - Pagination not handled: only the first page of servers is considered. Popular experiences may require cursor iteration to find better candidates.

---

### Block 4 — Best Server Selection Logic
**Overview:** Evaluates returned server list and picks one server according to the selected mode: `ping`, `latency`, or `auto`.

**Nodes involved:**
- Filter For The Best

#### Node: **Filter For The Best**
- **Type / role:** `Code` node implementing custom ranking logic.
- **Configuration choices / key variables:**
  - Reads:
    - `const input = items[0].json;`
    - `const servers = input.data;`
    - `const mode = $('Auto Config').first().json.mode || "auto";`
  - If no servers: returns an object `{success:false, result:"No servers found..."}` (but see note below about output shape).
  - Selection algorithm:
    - **mode = "ping"**: smallest `ping`
    - **mode = "latency"**: smallest `ping`, tie-breaker highest `fps`
    - **mode = "auto"**:
      - Prefer server with ping at least 5ms lower than current best
      - If within ±5ms, pick higher `playing` (more players) to avoid “dead” servers
  - Returns:
    - `success: true`
    - `selected_mode`
    - `best_server_id`
    - `stats: {ping, fps, players, maxPlayers}`
    - `all_details: bestServer`
- **Inputs / outputs:**
  - **Input ←** `Fetch Public Servers`
  - **Output →** `Auto Redirect?`
- **Edge cases / potential failures:**
  - **n8n Code node output shape:** In n8n, a Code node normally must return an array of items like `return [{ json: {...} }]`. This code returns a plain object. Depending on n8n version/settings, this may throw “Code node must return an array” or behave unexpectedly. If you encounter issues, wrap returns as:
    - `return [{ json: { ... } }];`
  - Assumes `ping`, `fps`, and `playing` exist; if Roblox API omits them or returns null, comparisons can yield incorrect results or `NaN`.

---

### Block 5 — Response Routing (Redirect vs JSON Deeplink)
**Overview:** If `ar=true` was provided, responds with an HTTP redirect to Roblox’s web start endpoint; otherwise returns JSON with a `roblox://` deeplink.

**Nodes involved:**
- Auto Redirect?
- Respond to Webhook
- Success
- Respond with Success

#### Node: **Auto Redirect?**
- **Type / role:** `IF` node determining output mode.
- **Configuration choices:**
  - Checks that `$('Webhook').item.json.query.ar` exists **and** is boolean true.
  - Uses **loose** type validation.
- **Inputs / outputs:**
  - **Input ←** `Filter For The Best`
  - **True →** `Respond to Webhook` (redirect)
  - **False →** `Success` (JSON deeplink)
- **Edge cases:**
  - Query parameters are typically strings (`"true"`, `"1"`). With loose validation, this may still pass, but behavior can vary. If it fails unexpectedly, consider checking string values explicitly (e.g., `"true"`).

#### Node: **Respond to Webhook** (redirect)
- **Type / role:** `Respond to Webhook` with HTTP redirect.
- **Configuration choices:**
  - **Respond with:** `redirect`
  - **Redirect URL:**
    - `https://www.roblox.com/games/start?placeId={{ $('Auto Config').item.json.plid }}&instanceId={{ $json.best_server_id }}`
- **Inputs / outputs:**
  - **Input ←** `Auto Redirect?` (true branch)
  - Terminal.
- **Edge cases:**
  - If `best_server_id` is missing due to upstream failure/shape issues, redirect URL will be malformed.

#### Node: **Success**
- **Type / role:** `Set` node building a consistent success payload with deeplink.
- **Configuration choices:**
  - `success = true`
  - `result = roblox://experiences/start?placeId={{ $('Auto Config').item.json.plid }}&gameInstanceId={{ $json.best_server_id }}`
- **Inputs / outputs:**
  - **Input ←** `Auto Redirect?` (false branch)
  - **Output →** `Respond with Success`
- **Edge cases:**
  - Deeplink schema must be handled by the client OS/browser; many browsers won’t open `roblox://` automatically without user interaction.

#### Node: **Respond with Success**
- **Type / role:** `Respond to Webhook` returning JSON.
- **Configuration choices:**
  - Default JSON response from input.
- **Inputs / outputs:**
  - **Input ←** `Success`
  - Terminal.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook | Webhook | Entry point HTTP endpoint (`/brsf`) | — | Place ID Set? | # Best Roblox Server Finder\nThis workflow helps gamers find the best Roblox server so that you can have a smooth Experience playing the game of your choice.\n\n## Setup\nNo Setup is Required. Having your Roblox Cookies are optional.\n\n## Example Usage:\n`https://YOUR_INSTANCE.com/brsf?plid=1234567890&mode=auto&ar=true` |
| Place ID Set? | IF | Validate required `plid` query parameter | Webhook | Auto Config; Error #1 | # Best Roblox Server Finder\nThis workflow helps gamers find the best Roblox server so that you can have a smooth Experience playing the game of your choice.\n\n## Setup\nNo Setup is Required. Having your Roblox Cookies are optional.\n\n## Example Usage:\n`https://YOUR_INSTANCE.com/brsf?plid=1234567890&mode=auto&ar=true` |
| Error #1 | Set | Build error payload when `plid` missing | Place ID Set? (false) | Respond with Error #1 |  |
| Respond with Error #1 | Respond to Webhook | Return error JSON to caller | Error #1 | — |  |
| Auto Config | Set | Normalize inputs (`plid`, `act`, `mode`) | Place ID Set? (true) | Fetch Public Servers |  |
| Fetch Public Servers | HTTP Request | Call Roblox Games API for public servers | Auto Config | Filter For The Best | Keep in mind, authorization is **not required!** |
| Filter For The Best | Code | Select best server based on mode logic | Fetch Public Servers | Auto Redirect? | # Modes\n- Ping Mode: Purely selects the lowest numerical ping.\n- Latency Mode: Lowest ping + Highest FPS as a tie-breaker.\n- Auto Mode: If pings are within 5ms of each other, it prioritizes the server with the most players to ensure the game isn't "dead." |
| Auto Redirect? | IF | Decide redirect vs JSON based on `ar` | Filter For The Best | Respond to Webhook; Success |  |
| Respond to Webhook | Respond to Webhook | HTTP redirect to Roblox start URL | Auto Redirect? (true) | — |  |
| Success | Set | Build success JSON + `roblox://` deeplink | Auto Redirect? (false) | Respond with Success |  |
| Respond with Success | Respond to Webhook | Return success JSON to caller | Success | — | # Response Format\n```json\n{\n  \"succsess\":true,\n  \"result\":\"roblox:// Deeplink\"\n}\n``` |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name it: **Find the best Roblox Servers to Play**
   - Ensure it is set to Active only when ready to receive requests.

2) **Add “Webhook” node**
   - Node type: **Webhook**
   - **Path:** `brsf`
   - **Response Mode:** `Using Respond to Webhook node` (responseNode)
   - Connect: **Webhook → Place ID Set?**

3) **Add “Place ID Set?” node**
   - Node type: **IF**
   - Condition: **String → exists**
   - Left value expression: `{{$json.query.plid}}`
   - True branch → **Auto Config**
   - False branch → **Error #1**

4) **Add “Error #1” node**
   - Node type: **Set**
   - Add fields:
     - `success` (Boolean) = `false`
     - `result` (String) = `Error #1: Place ID is not set. You must use the query parameter "plid" in the URL. Example: "?plid=123456789"`
   - Connect: **Error #1 → Respond with Error #1**

5) **Add “Respond with Error #1” node**
   - Node type: **Respond to Webhook**
   - Use default “Respond with JSON” behavior (no redirect)
   - (Optional improvement) Set HTTP status code to **400**.

6) **Add “Auto Config” node**
   - Node type: **Set**
   - Add fields (expressions):
     - `plid` (String): `{{$json.query.plid}}`
     - `act` (String): `{{$if($json.act,$json.act,"most")}}`  
       - If you want the query parameter, use: `{{$if($json.query.act, $json.query.act, "most")}}`
     - `mode` (String): `{{$if($json.query.mode, $json.query.mode,"auto")}}`
   - Connect: **Auto Config → Fetch Public Servers**

7) **Add “Fetch Public Servers” node**
   - Node type: **HTTP Request**
   - Method: **GET**
   - URL (expression): `https://games.roblox.com/v1/games/{{$json.plid}}/servers/Public`
   - Enable “Send Query Parameters”:
     - `cursor` (leave empty / optional)
     - `sortOrder`: `{{$if($json.act == "least","Asc","Desc")}}`
     - `excludeFullGames`: `true`
   - Enable “Send Headers” and add:
     - `cache-control: no-cache`
     - `origin: https://www.roblox.com`
     - `referer: https://www.roblox.com/`
     - `user-agent: Mozilla/5.0 ...`
     - (Optional) other sec-fetch/sec-ch-ua headers as in the workflow
   - No credentials required.
   - Connect: **Fetch Public Servers → Filter For The Best**

8) **Add “Filter For The Best” node**
   - Node type: **Code (JavaScript)**
   - Paste the selection logic.
   - If your n8n requires array output, return items as:
     - `return [{ json: { ... } }];`
   - Connect: **Filter For The Best → Auto Redirect?**

9) **Add “Auto Redirect?” node**
   - Node type: **IF**
   - Conditions (AND):
     - Boolean “exists”: `{{ $('Webhook').item.json.query.ar }}`
     - Boolean “is true”: `{{ $('Webhook').item.json.query.ar }}`
   - True branch → **Respond to Webhook**
   - False branch → **Success**

10) **Add “Respond to Webhook” node (redirect)**
   - Node type: **Respond to Webhook**
   - Respond with: **Redirect**
   - Redirect URL (expression):
     - `https://www.roblox.com/games/start?placeId={{ $('Auto Config').item.json.plid }}&instanceId={{ $json.best_server_id }}`

11) **Add “Success” node**
   - Node type: **Set**
   - Fields:
     - `success` (Boolean) = `true`
     - `result` (String, expression):
       - `roblox://experiences/start?placeId={{ $('Auto Config').item.json.plid }}&gameInstanceId={{ $json.best_server_id }}`
   - Connect: **Success → Respond with Success**

12) **Add “Respond with Success” node**
   - Node type: **Respond to Webhook**
   - Default JSON response.

13) **Test**
   - Call:
     - `https://YOUR_INSTANCE.com/brsf?plid=1234567890&mode=auto`
   - Redirect test:
     - `https://YOUR_INSTANCE.com/brsf?plid=1234567890&mode=auto&ar=true`

**Credentials:** None required (no Roblox auth in this workflow).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Best Roblox Server Finder: workflow to find best server for smooth gameplay. Setup not required; Roblox cookies optional. Example: `https://YOUR_INSTANCE.com/brsf?plid=1234567890&mode=auto&ar=true` | Sticky note near entry nodes |
| Keep in mind, authorization is **not required!** | Sticky note near HTTP request |
| Modes: Ping (lowest ping), Latency (lowest ping + highest FPS tie-breaker), Auto (within 5ms prefer higher players) | Sticky note describing selection behavior |
| Response format note shows `{ "succsess": true, "result": "roblox:// Deeplink" }` (typo: “succsess” should be “success” to match workflow output) | Sticky note near response nodes |