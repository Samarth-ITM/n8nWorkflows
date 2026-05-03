Generate multiple profile picture avatars with free public APIs (no keys)

https://n8nworkflows.xyz/workflows/generate-multiple-profile-picture-avatars-with-free-public-apis--no-keys--12125


# Generate multiple profile picture avatars with free public APIs (no keys)

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Workflow name:** *Simple Profile Picture Generator (No API Keys Needed)*  
**Purpose:** Generate a gallery of multiple profile-picture/avatar styles (12 variants) using **free public avatar/image APIs without any API keys**, then render them as an HTML grid with download links.

**Target use cases**
- Quickly generating placeholder avatars for apps, demos, internal tools
- Comparing multiple avatar styles for a consistent “seed” (same input produces consistent results)
- Producing downloadable avatar images without storing files

### 1.1 Manual Execution (Entry Point)
Starts the workflow when the user clicks **Execute workflow** in n8n.

### 1.2 Avatar URL Generation (API URL Builder)
Builds a `seed`, randomly selects a `gender`, and constructs an array of 12 avatar/image URLs across several public providers.

### 1.3 HTML Gallery Rendering (Presentation Builder)
Creates a responsive HTML page that shows avatars in cards, includes metadata (gender + seed), and adds “Download” buttons.

### 1.4 Display Output in n8n
Renders the generated HTML in an n8n HTML view node.

---

## 2. Block-by-Block Analysis

### Block 1 — Manual Execution (Entry Point)

**Overview:** Provides a manual trigger so the workflow runs on demand from the editor UI.  
**Nodes involved:**  
- **When clicking ‘Execute workflow’**

#### Node: When clicking ‘Execute workflow’
- **Type / role:** `Manual Trigger` (`n8n-nodes-base.manualTrigger`) — starts execution manually.
- **Configuration choices:** Default configuration, no parameters.
- **Key variables/expressions:** None.
- **Inputs:** None (entry node).
- **Outputs:** Sends a single empty item to the next node.
- **Connections:**
  - Output → **Call the Profile APIs**
- **Version notes:** TypeVersion `1` (standard).
- **Edge cases / failures:** None typical; only runs in editor/manual context (not scheduled).

**Sticky note covering this area (context):**
- “## 1. Call the APIs to get random profile images” (section marker)

---

### Block 2 — Avatar URL Generation (API URL Builder)

**Overview:** Generates a normalized `seed` and randomly selects `gender`, then builds an `apis` array containing 12 avatar/image URLs from RandomUser, DiceBear styles, RoboHash sets, and UI Avatars.  
**Nodes involved:**  
- **Call the Profile APIs**

#### Node: Call the Profile APIs
- **Type / role:** `Code` (`n8n-nodes-base.code`) — constructs data (URLs + metadata) in JavaScript.
- **Configuration choices (interpreted):**
  - `userInput` is currently set to an **empty string** (`const userInput = '';`).
  - Generates a random number `randomNum` between 0 and 9999.
  - Computes `seed`:
    - If `userInput` is non-empty: lowercases and replaces whitespace with `-`
    - Otherwise: `random${randomNum}` (e.g., `random4821`)
  - Randomly selects a `gender` (`male` or `female`).
  - Creates an `apis` array of 12 objects: `{ name, url }`
    - **RandomUser** portrait: `https://randomuser.me/api/portraits/{women|men}/{0-98}.jpg`
    - **DiceBear** (v7) styles: `avataaars`, `bottts`, `pixel-art`, `lorelei`, `initials` with `png?seed=...&size=200`
    - **RoboHash**: `set=set1` (Robot) and `set=set2` (Monster), `size=200x200`
    - **UI Avatars**: multiple background colors, `size=200&background=...&color=fff`
  - Returns one item: `{ apis, gender, seed }`
- **Key expressions/variables used:**
  - `seed`, `gender`, `apis`
  - Uses `Math.random()`, string normalization via regex `replace(/\s+/g, '-')`
- **Inputs:** One item from Manual Trigger (content unused).
- **Outputs:** One JSON item:
  - `json.apis` (array of `{name, url}`)
  - `json.gender` (string)
  - `json.seed` (string)
- **Connections:**
  - Input ← **When clicking ‘Execute workflow’**
  - Output → **Generate HTML Format**
- **Version notes:** TypeVersion `2` (Code node).
- **Edge cases / potential failures:**
  - **External availability/rate limiting:** These are public endpoints; they can throttle or change behavior.
  - **RandomUser image index range:** Uses `Math.floor(Math.random() * 99)` → 0–98. If the service changes valid indices, some images may 404.
  - **Seed characters:** If `userInput` includes special characters beyond spaces, they are not URL-encoded here. Most services tolerate it, but for safety you may want `encodeURIComponent(seed)` in URLs.
  - **Inconsistent file types:** Some URLs return `.jpg` while download filename is set to `.png` later (cosmetic but can confuse users).

**Sticky note covering this area (context):**
- Main workflow note contains customization guidance: change `userInput`, fix `gender`, edit `apis`, change sizes.

---

### Block 3 — HTML Gallery Rendering (Presentation Builder)

**Overview:** Converts the `apis` list into a full HTML document with CSS styling, responsive grid layout, metadata display, and download links per avatar.  
**Nodes involved:**  
- **Generate HTML Format**

#### Node: Generate HTML Format
- **Type / role:** `Code` (`n8n-nodes-base.code`) — builds an HTML string from incoming JSON.
- **Configuration choices (interpreted):**
  - Reads from input item:
    - `const apis = $input.item.json.apis;`
    - `const gender = $input.item.json.gender;`
    - `const seed = $input.item.json.seed;`
  - Constructs a complete HTML document:
    - Responsive grid (`grid-template-columns: repeat(auto-fill, minmax(260px, 1fr))`)
    - Card layout with image + API name + download link
    - Metadata header showing gender and seed
  - For each API entry:
    - `<img src="...">` with `loading="lazy"`
    - A download link:
      - `href` is the same URL
      - `download="..."` uses slugified api name + seed, ends with `.png`
      - `target="_blank"`
  - Returns `{ html }` as JSON.
- **Key expressions/variables used:**
  - `$input.item.json.*`
  - `apis.map(...).join('')`
  - `api.name.toLowerCase().replace(/\s+/g, '-')`
- **Inputs:** One item from **Call the Profile APIs**.
- **Outputs:** One item containing `json.html` (string).
- **Connections:**
  - Input ← **Call the Profile APIs**
  - Output → **Show the Profile Pictures**
- **Version notes:** TypeVersion `2`.
- **Edge cases / potential failures:**
  - **Missing fields:** If `apis` is missing or not an array, `apis.map` throws and the node fails.
  - **HTML injection risk (low here):** If `seed` or `api.name` came from untrusted user input, it could inject HTML. In this workflow, `seed` can be customized manually in code; if you later take user input from a webhook/form, you should escape output.
  - **Download attribute limitations:** Some browsers ignore `download` for cross-origin URLs; the link may open the image instead of forcing a download.

**Sticky note covering this area (context):**
- “## 2. Generate HTML format to show the profile images”

---

### Block 4 — Display Output in n8n

**Overview:** Renders the generated HTML inside n8n so the user can view the gallery and click download links.  
**Nodes involved:**  
- **Show the Profile Pictures**

#### Node: Show the Profile Pictures
- **Type / role:** `HTML` (`n8n-nodes-base.html`) — displays HTML content.
- **Configuration choices (interpreted):**
  - HTML field is set via expression: `{{ $json.html }}`
- **Inputs:** One item containing `json.html`.
- **Outputs:** Typically none further (end of workflow).
- **Connections:**
  - Input ← **Generate HTML Format**
- **Version notes:** TypeVersion `1.2`.
- **Edge cases / potential failures:**
  - If `json.html` is missing/empty, the rendered page will be blank.
  - External images may fail to load due to provider downtime, CORS policies, or blocked mixed content (if served over http; here URLs are https).

**Sticky note covering this area (context):**
- “## 3. Show the results and download the profile picture”

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point |  | Call the Profile APIs | ## 1. Call the APIs to get random profile images |
| Call the Profile APIs | Code | Build seed/gender and generate avatar API URLs | When clicking ‘Execute workflow’ | Generate HTML Format | ## 1. Call the APIs to get random profile images |
| Generate HTML Format | Code | Convert avatar URLs into a styled HTML gallery | Call the Profile APIs | Show the Profile Pictures | ## 2. Generate HTML format to show the profile images |
| Show the Profile Pictures | HTML | Render the final HTML gallery in n8n | Generate HTML Format |  | ## 3. Show the results and download the profile picture |
| Sticky Note | Sticky Note | Documentation / instructions |  |  | ## Simple Profile Picture Generator (No API Keys Needed) … (How it works / How to Set Up / Customize) |
| Sticky Note1 | Sticky Note | Section label |  |  | ## 1. Call the APIs to get random profile images |
| Sticky Note2 | Sticky Note | Section label |  |  | ## 2. Generate HTML format to show the profile images |
| Sticky Note3 | Sticky Note | Section label |  |  | ## 3. Show the results and download the profile picture |
| Sticky Note4 | Sticky Note | External reference |  |  | ## Here's How  \n\n@[youtube](bspOMa5LwM0) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Simple Profile Picture Generator (No API Keys Needed)**
   - No special settings required.

2. **Add node: Manual Trigger**
   - Node type: **Manual Trigger**
   - Name: **When clicking ‘Execute workflow’**
   - Keep defaults.

3. **Add node: Code (URL generator)**
   - Node type: **Code**
   - Name: **Call the Profile APIs**
   - Paste/configure JavaScript logic that:
     - Sets `const userInput = ''` (or your chosen seed keyword)
     - Builds `seed` from `userInput` (or `random####`)
     - Chooses `gender` randomly (`male`/`female`) or set fixed
     - Builds an array `apis` with 12 `{name, url}` entries from:
       - RandomUser portraits
       - DiceBear (7.x) styles: avataaars, bottts, pixel-art, lorelei, initials
       - RoboHash (set1 + set2)
       - UI Avatars (multiple background colors)
     - Returns: `{ json: { apis, gender, seed } }`

4. **Connect nodes**
   - Connect **When clicking ‘Execute workflow’ → Call the Profile APIs**

5. **Add node: Code (HTML builder)**
   - Node type: **Code**
   - Name: **Generate HTML Format**
   - Configure code to:
     - Read `apis`, `gender`, `seed` from `$input.item.json`
     - Build a full HTML document string (CSS + grid + cards)
     - For each `api` create:
       - `<img src="api.url" ...>`
       - `<a href="api.url" download="..." target="_blank">Download</a>`
     - Return `{ json: { html } }`

6. **Connect nodes**
   - Connect **Call the Profile APIs → Generate HTML Format**

7. **Add node: HTML (renderer)**
   - Node type: **HTML**
   - Name: **Show the Profile Pictures**
   - Set **HTML** parameter to an expression: `{{ $json.html }}`

8. **Connect nodes**
   - Connect **Generate HTML Format → Show the Profile Pictures**

9. **Credentials**
   - No credentials are required (all endpoints are public; no auth).

10. **Run**
   - Click **Execute workflow**
   - The HTML node should render the gallery and allow opening/downloading each avatar.

**Optional hardening (recommended if you later add real user input via webhook):**
- URL-encode the seed: `encodeURIComponent(seed)` in all URL templates.
- Escape seed/name when injecting into HTML text nodes/attributes.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ## Simple Profile Picture Generator (No API Keys Needed) — explains flow, setup, and customization (custom seed, fixed gender, add/remove APIs, change size). | Workflow sticky note (internal documentation) |
| ## Here's How — @[youtube](bspOMa5LwM0) | https://www.youtube.com/watch?v=bspOMa5LwM0 (video reference from sticky note) |