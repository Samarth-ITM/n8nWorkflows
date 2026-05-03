Create AI-generated Spotify playlists with GPT-5 and email the link

https://n8nworkflows.xyz/workflows/create-ai-generated-spotify-playlists-with-gpt-5-and-email-the-link-11376


# Create AI-generated Spotify playlists with GPT-5 and email the link

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Create AI-generated Spotify playlists with GPT-5 and email the link

**Purpose:**  
This workflow collects event details via an n8n Form, asks an AI agent (GPT‑5) to generate a Spotify playlist concept (name + 18–32 track suggestions), searches Spotify for each track, creates a new playlist in the authenticated Spotify account, adds the found tracks, and emails the Spotify playlist link to the user.

**Target use cases:**
- Event playlist generation (weddings, birthdays, corporate events, themed parties)
- Automated “playlist concierge” for websites (lead capture via email + instant delivery)
- Rapid playlist prototyping for DJs/hosts

### Logical Blocks
**1.1 Form Input Reception**  
Collect user briefing (Occasion, Guests, Personal Touch, Email).

**1.2 AI Playlist Generation (Structured JSON)**  
GPT‑5 generates playlist name, song list, and HTML snippets; output is enforced via a structured parser.

**1.3 Spotify Playlist Creation**  
Get Spotify user ID and create a new public playlist with the AI-generated name.

**1.4 Song Search & URI Extraction**  
Split songs into individual items, search Spotify track catalog, extract each track URI, and aggregate URIs.

**1.5 Add Tracks, Notify User by Email**  
Merge playlist ID + URIs, add tracks to playlist, then email the playlist link.

---

## 2. Block-by-Block Analysis

### 2.1 Block 1 — Form Input Reception
**Overview:** Captures the user’s event briefing and email address via an n8n-hosted form, then starts the workflow.

**Nodes involved:**
- **On form submission** (Form Trigger)

#### Node: On form submission
- **Type / role:** `n8n-nodes-base.formTrigger` — entry point; receives form submissions.
- **Configuration choices:**
  - Form title: **Playlist Maker**
  - Description: “Generate your custom Playlist on Spotify”
  - Fields:
    - Occasion (required)
    - Guests (required)
    - Personal Touch (required)
    - Email (email field; not marked required in JSON)
- **Key variables produced (output JSON):**
  - `$json.Occasion`
  - `$json.Guests`
  - `$json["Personal Touch"]`
  - `$json.Email`
- **Connections:**
  - Output → **AI Agent**
- **Potential failures / edge cases:**
  - Missing/empty Email (since not required): later **Send email** may fail or send nowhere.
  - Form field label includes a space (`Personal Touch`): must be referenced exactly as `$json["Personal Touch"]`.
- **Version notes:** Node typeVersion **2.3**.

---

### 2.2 Block 2 — AI Playlist Generation (Structured JSON)
**Overview:** Uses a GPT‑5-powered agent to generate a playlist name and 18–32 songs (title + artist). Output is constrained to a JSON schema via a structured output parser.

**Nodes involved:**
- **OpenAI Chat Model1**
- **Structured Output Parser1**
- **AI Agent**

#### Node: OpenAI Chat Model1
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — provides the LLM backend to the agent.
- **Configuration choices:**
  - Model: **gpt-5**
  - Options: default/empty
- **Credentials:** OpenAI API credential (“NeoRebels n8n flows”)
- **Connections:**
  - `ai_languageModel` → **AI Agent**
- **Potential failures / edge cases:**
  - Invalid/expired OpenAI key; model access not enabled for the account.
  - Rate limiting / timeouts (large prompts or high traffic).
- **Version notes:** typeVersion **1.2**.

#### Node: Structured Output Parser1
- **Type / role:** `@n8n/n8n-nodes-langchain.outputParserStructured` — validates/parses the agent output into structured JSON.
- **Configuration choices:**
  - JSON schema example enforces keys:
    - `playlist_name` (string)
    - `songs` (array of `{title, artist}`)
    - `songs-website` (HTML `<ul>...</ul>` string)
    - `briefing-website` (HTML `<ul>...</ul>` string)
- **Connections:**
  - `ai_outputParser` → **AI Agent**
- **Potential failures / edge cases:**
  - Model returns invalid JSON or deviates from schema → parser may error or produce empty/partial output depending on parser behavior/version.
  - Keys containing hyphens (`songs-website`, `briefing-website`) can be awkward to reference later; must use bracket notation if needed.
- **Version notes:** typeVersion **1.2**.

#### Node: AI Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates prompt + model + output parsing.
- **Configuration choices (interpreted):**
  - Prompt injects form fields into a `<TASK>` block:
    - `$occasion = {{ $json.Occasion }}`
    - `$audience = {{ $json.Guests }}`
    - `$additional-input = {{ $json["Personal Touch"] }}`
  - Forces a strict JSON response format (playlist_name + songs[] + 2 HTML fields).
  - Safety/style constraints:
    - “Do not acknowledge the prompt”
    - Output briefing/title in the same language as user input
    - No swear words; replace with asterisks if present
  - System message defines a “DJ expert” role and instructs playlists to be **18–32 tracks**.
  - `hasOutputParser: true` (uses Structured Output Parser1)
- **Inputs:**
  - Main input from **On form submission**
  - LLM input from **OpenAI Chat Model1** (`ai_languageModel`)
  - Output parser input from **Structured Output Parser1** (`ai_outputParser`)
- **Outputs / downstream:**
  - Main output → **Get Spotify User ID**
  - Main output → **Split Out**
- **Potential failures / edge cases:**
  - If any of the form fields are empty, the prompt may become underspecified (playlist quality issues).
  - If parser fails, downstream nodes referencing `output.*` may break.
  - Track list length outside 18–32 could occur if the model disobeys; downstream still works but playlist size differs.
- **Version notes:** typeVersion **2**.

---

### 2.3 Block 3 — Spotify Playlist Creation
**Overview:** Uses the Spotify OAuth credential to identify the current user and create a new playlist named after the AI output.

**Nodes involved:**
- **Get Spotify User ID**
- **Create Spotify Playlist**
- **Extract Playlist ID**

#### Node: Get Spotify User ID
- **Type / role:** `n8n-nodes-base.httpRequest` — calls Spotify Web API to get the authenticated user profile.
- **Configuration choices:**
  - GET `https://api.spotify.com/v1/me`
  - Auth: predefined credential type `spotifyOAuth2Api`
- **Connections:**
  - Input ← **AI Agent**
  - Output → **Create Spotify Playlist**
- **Potential failures / edge cases:**
  - OAuth token expired/revoked; insufficient scopes.
  - Spotify API downtime / 429 rate limits.
- **Version notes:** typeVersion **4.2**.

#### Node: Create Spotify Playlist
- **Type / role:** `n8n-nodes-base.httpRequest` — creates a playlist for a user.
- **Configuration choices:**
  - POST `https://api.spotify.com/v1/users/{{$json["id"]}}/playlists`
  - JSON body:
    - `name`: `{{ $('AI Agent').all()[0].json.output.playlist_name }}`
    - `description`: “Created by n8n workflow”
    - `public`: `true`
  - Auth: Spotify OAuth2 credential
- **Connections:**
  - Input ← **Get Spotify User ID** (provides `$json.id`)
  - Output → **Extract Playlist ID**
- **Potential failures / edge cases:**
  - Missing scope: playlist creation typically needs `playlist-modify-public` (or `playlist-modify-private` for private playlists).
  - If **AI Agent** output is missing `playlist_name`, the expression can resolve to empty and Spotify may reject or create with a default.
  - Using `$('AI Agent').all()[0]` assumes at least one item exists; if the agent node produces no items, expression fails.
- **Version notes:** typeVersion **4.2**.

#### Node: Extract Playlist ID
- **Type / role:** `n8n-nodes-base.set` — normalizes playlist ID into a dedicated field.
- **Configuration choices:**
  - Sets `playlistId = {{$json["id"]}}` (from the create-playlist response)
- **Connections:**
  - Input ← **Create Spotify Playlist**
  - Output → **Merge** (input 0)
- **Potential failures / edge cases:**
  - If Spotify response doesn’t include `id` (error object), `playlistId` becomes undefined.
- **Version notes:** typeVersion **3.4**.

---

### 2.4 Block 4 — Song Search & URI Extraction
**Overview:** Iterates through the AI-generated songs, searches Spotify for each, extracts the first matching track URI, and aggregates URIs into an array.

**Nodes involved:**
- **Split Out**
- **Search Spotify for Song**
- **Extract Track URI**
- **Build URIs Array**

#### Node: Split Out
- **Type / role:** `n8n-nodes-base.splitOut` — converts an array into multiple items (one per song).
- **Configuration choices:**
  - Field to split: `output.songs`
- **Connections:**
  - Input ← **AI Agent**
  - Output → **Search Spotify for Song**
- **Potential failures / edge cases:**
  - If `output.songs` is missing/not an array (parser failure), split produces no items or errors.
- **Version notes:** typeVersion **1**.

#### Node: Search Spotify for Song
- **Type / role:** `n8n-nodes-base.httpRequest` — searches Spotify track catalog.
- **Configuration choices:**
  - GET `https://api.spotify.com/v1/search?q={{$json["title"]}}%20{{$json["artist"]}}&type=track&limit=1`
  - Auth: Spotify OAuth2
  - `onError: continueRegularOutput` (important): the workflow won’t hard-fail on search errors.
- **Connections:**
  - Input ← **Split Out**
  - Output → **Extract Track URI**
- **Potential failures / edge cases:**
  - Track not found: returns empty `tracks.items`.
  - API rate limiting (429) or transient errors; because it continues, later extraction may yield `not_found`.
  - Query is not URL-encoded beyond replacing space with `%20`; special characters (e.g., “Beyoncé”, “AC/DC”, quotes) may reduce match accuracy. Consider `encodeURIComponent()` in production.
- **Version notes:** typeVersion **4.2**.

#### Node: Extract Track URI
- **Type / role:** `n8n-nodes-base.set` — extracts first track URI or sets a sentinel.
- **Configuration choices:**
  - `trackUri = {{$json["tracks"]["items"][0]["uri"] || "not_found"}}`
- **Connections:**
  - Input ← **Search Spotify for Song**
  - Output → **Build URIs Array**
- **Potential failures / edge cases:**
  - If `tracks` or `items` is missing due to an API error payload, the expression can still throw depending on n8n’s safe navigation behavior. (Here it directly indexes `[0]` before `||`, which can be risky.)
- **Version notes:** typeVersion **3.4**.

#### Node: Build URIs Array
- **Type / role:** `n8n-nodes-base.code` — aggregates valid URIs across all items into one array.
- **Configuration choices:**
  - Filters items where `trackUri` starts with `spotify:track:`
  - Returns single item: `{ uris: [ ... ] }`
- **Connections:**
  - Input ← **Extract Track URI**
  - Output → **Merge** (input 1)
- **Potential failures / edge cases:**
  - If all songs are `not_found`, `uris` becomes `[]` and adding tracks may fail or create an empty playlist.
- **Version notes:** typeVersion **2**.

---

### 2.5 Block 5 — Add Tracks, Merge Data, Email Notification
**Overview:** Combines playlist ID and track URIs, adds tracks to the Spotify playlist, then emails the Spotify link to the user.

**Nodes involved:**
- **Merge**
- **Add Tracks to Playlist**
- **Send email**

#### Node: Merge
- **Type / role:** `n8n-nodes-base.merge` — combines two data streams.
- **Configuration choices:**
  - Mode: **combine**
  - Combine by: **combineAll** (pairs all inputs into a single combined item set)
- **Connections:**
  - Input 0 ← **Extract Playlist ID**
  - Input 1 ← **Build URIs Array**
  - Output → **Add Tracks to Playlist**
- **Potential failures / edge cases:**
  - If one branch produces no items (e.g., URIs branch fails), combine may output nothing and downstream won’t run.
- **Version notes:** typeVersion **3.2**.

#### Node: Add Tracks to Playlist
- **Type / role:** `n8n-nodes-base.httpRequest` — adds track URIs to playlist.
- **Configuration choices:**
  - POST `https://api.spotify.com/v1/playlists/{{$json["playlistId"]}}/tracks`
  - Body parameter `uris = {{$json["uris"]}}`
  - Auth: Spotify OAuth2
- **Connections:**
  - Input ← **Merge**
  - Output → **Send email**
- **Potential failures / edge cases:**
  - Missing scope: requires playlist modification scope (public/private depending on playlist).
  - If `uris` is empty, Spotify may reject (or accept but do nothing). Consider guarding with an IF node.
  - If `playlistId` missing, request URL becomes invalid.
- **Version notes:** typeVersion **4.2**.

#### Node: Send email
- **Type / role:** `n8n-nodes-base.emailSend` — sends the final Spotify link to the user.
- **Configuration choices:**
  - To: `{{ $('On form submission').item.json.Email }}`
  - Subject: “Your custom playlist is ready!”
  - HTML body:
    - References playlist name: `{{ $('AI Agent').item.json.output.playlist_name }}`
    - References Spotify link: `{{ $('Create Spotify Playlist').item.json.external_urls.spotify }}`
- **Credentials:** SMTP account
- **Connections:**
  - Input ← **Add Tracks to Playlist**
  - No outputs (end)
- **Potential failures / edge cases:**
  - Missing/invalid recipient email (Email not required in the form).
  - SMTP auth errors, blocked ports, provider restrictions (Gmail often requires app passwords/OAuth).
  - Expression dependency: if **Create Spotify Playlist** didn’t return `external_urls.spotify` (error payload), email will contain blank/invalid link.
- **Version notes:** typeVersion **2.1**.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On form submission | n8n-nodes-base.formTrigger | Collect user briefing + email; workflow entry | — | AI Agent | ### 1. Form Input & AI Generation<br>Captures user input and uses AI to generate a playlist concept. |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Generate playlist name + songs (structured JSON) | On form submission; OpenAI Chat Model1; Structured Output Parser1 | Get Spotify User ID; Split Out | ### 1. Form Input & AI Generation<br>Captures user input and uses AI to generate a playlist concept. |
| OpenAI Chat Model1 | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider (GPT‑5) | — | AI Agent | ### 1. Form Input & AI Generation<br>Captures user input and uses AI to generate a playlist concept. |
| Structured Output Parser1 | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce/parse structured JSON output | — | AI Agent | ### 1. Form Input & AI Generation<br>Captures user input and uses AI to generate a playlist concept. |
| Get Spotify User ID | n8n-nodes-base.httpRequest | Fetch Spotify `/me` profile (user id) | AI Agent | Create Spotify Playlist | ### 2. Playlist Creation<br>Authenticates with Spotify and creates a new playlist on the user's account. |
| Create Spotify Playlist | n8n-nodes-base.httpRequest | Create playlist with AI name | Get Spotify User ID | Extract Playlist ID | ### 2. Playlist Creation<br>Authenticates with Spotify and creates a new playlist on the user's account. |
| Extract Playlist ID | n8n-nodes-base.set | Save `playlistId` for later merge | Create Spotify Playlist | Merge | ### 2. Playlist Creation<br>Authenticates with Spotify and creates a new playlist on the user's account. |
| Split Out | n8n-nodes-base.splitOut | Split songs array into per-song items | AI Agent | Search Spotify for Song | ### 3. Song Search & URI Extraction<br>Searches Spotify for each recommended song and extracts track URIs. |
| Search Spotify for Song | n8n-nodes-base.httpRequest | Search Spotify tracks by title+artist | Split Out | Extract Track URI | ### 3. Song Search & URI Extraction<br>Searches Spotify for each recommended song and extracts track URIs. |
| Extract Track URI | n8n-nodes-base.set | Extract `spotify:track:*` URI or `not_found` | Search Spotify for Song | Build URIs Array | ### 3. Song Search & URI Extraction<br>Searches Spotify for each recommended song and extracts track URIs. |
| Build URIs Array | n8n-nodes-base.code | Aggregate valid track URIs into `uris[]` | Extract Track URI | Merge | ### 4. Add Tracks & Merge Data<br>Combines all track URIs and adds them to the newly created playlist. |
| Merge | n8n-nodes-base.merge | Combine playlistId + uris into one item | Extract Playlist ID; Build URIs Array | Add Tracks to Playlist | ### 4. Add Tracks & Merge Data<br>Combines all track URIs and adds them to the newly created playlist. |
| Add Tracks to Playlist | n8n-nodes-base.httpRequest | Add `uris[]` to playlist | Merge | Send email | ### 4. Add Tracks & Merge Data<br>Combines all track URIs and adds them to the newly created playlist. |
| Send email | n8n-nodes-base.emailSend | Email playlist link to user | Add Tracks to Playlist | — | ### 5. Email Notification<br>Sends the user an email with their new playlist link. <br>(Alternatively use Gmail or Outlook node) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials**
   1. **Spotify OAuth2** (in n8n Credentials)
      - Create Spotify app at https://developer.spotify.com/dashboard
      - Copy Client ID/Secret into n8n Spotify OAuth2 credential
      - Authorize in n8n
      - Ensure scopes include playlist creation/modification (commonly `playlist-modify-public`; add private scope if needed).
   2. **OpenAI API**
      - Create API key: https://platform.openai.com/settings/organization/api-keys
      - Add to n8n OpenAI credential.
   3. **SMTP**
      - Configure SMTP host/port/user/password/sender identity in n8n SMTP credential.

2) **Add Form Trigger**
   - Node: **Form Trigger** named **On form submission**
   - Configure:
     - Title: `Playlist Maker`
     - Description: `Generate your custom Playlist on Spotify`
     - Fields:
       - Occasion (required)
       - Guests (required)
       - Personal Touch (required)
       - Email (type email; strongly consider marking required)

3) **Add AI model + parser**
   1. Node: **OpenAI Chat Model** named **OpenAI Chat Model1**
      - Model: `gpt-5`
      - Select OpenAI credentials
   2. Node: **Structured Output Parser** named **Structured Output Parser1**
      - Provide JSON schema example with keys:
        - `playlist_name`
        - `songs` array of `{title, artist}`
        - `songs-website`
        - `briefing-website`

4) **Add AI Agent**
   - Node: **AI Agent** (LangChain Agent)
   - Connect:
     - **On form submission** → **AI Agent** (main)
     - **OpenAI Chat Model1** → **AI Agent** (`ai_languageModel`)
     - **Structured Output Parser1** → **AI Agent** (`ai_outputParser`)
   - Configure the agent prompt to inject:
     - `{{ $json.Occasion }}`, `{{ $json.Guests }}`, `{{ $json["Personal Touch"] }}`
   - Ensure it outputs the required JSON format and includes playlist length guidance (18–32 tracks).

5) **Spotify: get user + create playlist**
   1. Node: **HTTP Request** named **Get Spotify User ID**
      - GET `https://api.spotify.com/v1/me`
      - Authentication: Spotify OAuth2 credential
   2. Node: **HTTP Request** named **Create Spotify Playlist**
      - POST `https://api.spotify.com/v1/users/{{$json["id"]}}/playlists`
      - Send JSON body:
        - `name`: `{{ $('AI Agent').all()[0].json.output.playlist_name }}`
        - `description`: `Created by n8n workflow`
        - `public`: `true`
      - Authentication: Spotify OAuth2
   3. Node: **Set** named **Extract Playlist ID**
      - Set field `playlistId` to `{{ $json["id"] }}`

6) **Spotify: search each song and collect URIs**
   1. Node: **Split Out** named **Split Out**
      - Field to split: `output.songs`
   2. Node: **HTTP Request** named **Search Spotify for Song**
      - GET `https://api.spotify.com/v1/search?q={{$json["title"]}}%20{{$json["artist"]}}&type=track&limit=1`
      - Authentication: Spotify OAuth2
      - Error handling: set to “continue on error” (equivalent to `continueRegularOutput`)
   3. Node: **Set** named **Extract Track URI**
      - Set `trackUri` to `{{ $json["tracks"]["items"][0]["uri"] || "not_found" }}`
   4. Node: **Code** named **Build URIs Array**
      - JS code:
        - Filter items where `trackUri` starts with `spotify:track:`
        - Return one item: `{ uris: [...] }`

7) **Merge playlistId + uris**
   - Node: **Merge** named **Merge**
   - Mode: **combine**
   - Combine by: **combineAll**
   - Connect:
     - **Extract Playlist ID** → **Merge** (Input 0)
     - **Build URIs Array** → **Merge** (Input 1)

8) **Add tracks to playlist**
   - Node: **HTTP Request** named **Add Tracks to Playlist**
   - POST `https://api.spotify.com/v1/playlists/{{$json["playlistId"]}}/tracks`
   - Send body parameter:
     - `uris`: `{{ $json["uris"] }}`
   - Authentication: Spotify OAuth2

9) **Send email with Spotify link**
   - Node: **Send Email** named **Send email**
   - To: `{{ $('On form submission').item.json.Email }}`
   - Subject: `Your custom playlist is ready!`
   - HTML body example:
     - `Your playlist "{{ $('AI Agent').item.json.output.playlist_name }}" is ready.`
     - `Listen on Spotify: {{ $('Create Spotify Playlist').item.json.external_urls.spotify }}`
   - Select SMTP credential

10) **Connect the flow**
   - On form submission → AI Agent  
   - AI Agent → Get Spotify User ID → Create Spotify Playlist → Extract Playlist ID → Merge → Add Tracks to Playlist → Send email  
   - AI Agent → Split Out → Search Spotify for Song → Extract Track URI → Build URIs Array → Merge  

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Spotify Developer Dashboard setup (Client ID/Secret, OAuth) | https://developer.spotify.com/dashboard |
| OpenAI API Keys creation | https://platform.openai.com/settings/organization/api-keys |
| Test the workflow here: “PlaylistMaker AI” | https://playlistmaker.ai |
| Contact: ufuk@neorebels.com | mailto:ufuk@neorebels.com |
| LinkedIn: Ufuk Oeren | https://www.linkedin.com/in/ufuk-oeren/ |
| SMTP can be replaced by Gmail/Outlook nodes | Mentioned in sticky notes and “Send email” node comment |
| Workflow prerequisites summary (Spotify OAuth2, OpenAI key, SMTP) | Included in sticky note content (Prerequisites & Required Setup) |