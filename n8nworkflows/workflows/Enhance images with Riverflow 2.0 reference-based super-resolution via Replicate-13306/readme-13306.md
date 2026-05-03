Enhance images with Riverflow 2.0 reference-based super-resolution via Replicate

https://n8nworkflows.xyz/workflows/enhance-images-with-riverflow-2-0-reference-based-super-resolution-via-replicate-13306


# Enhance images with Riverflow 2.0 reference-based super-resolution via Replicate

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow enhances an image using **Riverflow 2.0 reference-based super-resolution** hosted on **Replicate**. The user supplies an input image plus up to **4 reference images** that act as ground truth for repairing fine details (labels, logos, small text, packaging).

**Typical use cases:**
- Restoring blurry product labels and packaging text
- Improving logos/brand marks
- Fixing small printed details on objects (bottles, boxes, apparel)

**Logical blocks (based on node dependencies):**
1.1 **Input Reception & Sanitization** → collect form inputs and normalize them into Replicate-ready fields  
1.2 **Prediction Creation (Replicate POST)** → submit the job to Replicate’s Riverflow model  
1.3 **Polling Loop (Replicate GET + Wait)** → repeatedly check prediction status until completion  
1.4 **Error Gate & Output Retrieval** → stop on failure, or fetch and return the final enhanced image

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Sanitization
**Overview:** Receives user inputs (image + references), normalizes field names, validates types, and guarantees a strict structure expected by Replicate.

**Nodes involved:**
- **Sticky Note** (documentation)
- **Sticky Note1** (“Receive and process inputs”)
- **Input form**
- **Input Handling**

#### Node: Sticky Note
- **Type / role:** Sticky Note (documentation)
- **Configuration choices:** Explains workflow operation, setup requirement (Replicate API key), and what reference-based super-resolution is.
- **Connections:** None (informational only)
- **Potential failure types:** None
- **Notes:** Contains link: https://replicate.com/sourceful/riverflow-2.0-refsr

#### Node: Sticky Note1
- **Type / role:** Sticky Note (block label)
- **Connections:** None

#### Node: Input form
- **Type / role:** `Form Trigger` — entry point that exposes an n8n form
- **Key configuration:**
  - **Title:** “Input data for Riverflow2.0 refsr”
  - **Description:** “for multiple references, separate with commas (MAX 4)”
  - **Fields (required):**
    - `Image`
    - `Super Resolution References`
- **Input / output:**
  - **Input:** None (trigger)
  - **Output:** Form submission JSON with the two fields as provided by user (field names as labels)
- **Version-specific requirements:** Uses Form Trigger **typeVersion 2.5**
- **Edge cases / failures:**
  - User provides non-URL/non-base64 strings; workflow does not validate URL format, only non-empty string.
  - “Super Resolution References” may be entered as a single string; later code splits by commas.

#### Node: Input Handling
- **Type / role:** `Code` — sanitizes and reshapes the form payload
- **Key configuration choices (interpreted):**
  - Only allows fields normalized to:
    - `image`
    - `super_resolution_references`
  - Normalizes incoming keys: lowercases + replaces whitespace with `_`
  - Validates:
    - `image` must be a **non-empty string**
    - references are converted to an array and capped at **max 4**
  - Outputs:
    - `image` (trimmed string)
    - `super_resolution_refs` (array; always exists, default `[]`)
- **Key expressions/variables used:** Works on `$input.all()`; mutates `item.json`
- **Input / output connections:**
  - Input: **Input form**
  - Output: **POST Request to Replicate API**
- **Version-specific requirements:** Code node **typeVersion 2**
- **Edge cases / failures:**
  - Throws error if `image` missing/empty/non-string → execution stops with a node error (not caught by Stop and Error node).
  - Throws error if more than 4 refs.
  - If user provides references in unusual types (object), it will wrap into `[value]` if truthy; may produce invalid Replicate input.

---

### 2.2 Prediction Creation (Replicate POST)
**Overview:** Submits the sanitized inputs to Replicate to start a Riverflow 2.0 reference-based super-resolution prediction.

**Nodes involved:**
- **Sticky Note2** (“Riverflow 2.0 takes data and begins generations”)
- **POST Request to Replicate API**

#### Node: Sticky Note2
- **Type / role:** Sticky Note (block label)
- **Connections:** None

#### Node: POST Request to Replicate API
- **Type / role:** `HTTP Request` — creates a prediction on Replicate
- **Endpoint:** `POST https://api.replicate.com/v1/models/sourceful/riverflow-2.0-refsr/predictions`
- **Authentication:** Predefined credential type **httpBearerAuth** (Replicate API token)
- **Headers:**
  - `Prefer: wait` (asks Replicate to wait briefly before responding; still returns a prediction object with URLs)
- **Body (JSON):**
  - Sends `input.image` from `{{$json.image}}`
  - Sends `input.super_resolution_references` from `{{$json.super_resolution_refs}}`
  - Uses `.toJsonString()` to safely embed strings/arrays as JSON literals
- **Input / output connections:**
  - Input: **Input Handling**
  - Output: **Wait**
- **Version-specific requirements:** HTTP Request **typeVersion 4.4**
- **Edge cases / failures:**
  - 401/403 if bearer token missing/invalid.
  - 400 if Replicate rejects the input format (e.g., invalid image URL, unsupported scheme, empty refs depending on model constraints).
  - Network timeouts / transient 5xx errors from Replicate.
  - If Replicate changes response shape (e.g., `urls.get` missing), later polling breaks.

---

### 2.3 Polling Loop (Replicate GET + Wait)
**Overview:** Repeatedly checks the prediction status using Replicate’s `urls.get` until it is no longer “processing”.

**Nodes involved:**
- **Sticky Note3** (“Polling loop to see if generation is complete”)
- **Wait**
- **GET Request to check status**
- **Check if finished**

#### Node: Sticky Note3
- **Type / role:** Sticky Note (block label)
- **Connections:** None

#### Node: Wait
- **Type / role:** `Wait` — delays execution between polling attempts
- **Key configuration:** Waits **20 seconds**
- **Input / output connections:**
  - Input: **POST Request to Replicate API** (first time), and **Check if finished** (loop)
  - Output: **GET Request to check status**
- **Version-specific requirements:** Wait **typeVersion 1.1**
- **Edge cases / failures:**
  - If the prediction never leaves `processing`, the workflow can loop indefinitely (no max retries / timeout guard in the workflow logic).
  - Execution may hit n8n instance limits (max execution time) depending on hosting configuration.

#### Node: GET Request to check status
- **Type / role:** `HTTP Request` — polls the prediction status
- **Endpoint:** dynamic from the Replicate response: `{{$json.urls.get}}`
- **Authentication:** Predefined credential type **httpBearerAuth**
- **Input / output connections:**
  - Input: **Wait**
  - Output: **Check if finished**
- **Version-specific requirements:** HTTP Request **typeVersion 4.4**
- **Edge cases / failures:**
  - If `$json.urls.get` is undefined (unexpected POST response), expression resolves to empty/invalid URL.
  - 404 if prediction expired or URL invalid.
  - 401/403 if credential invalid.
  - Rate limiting (429) if polling too frequently (20s usually safe, but not guaranteed).

#### Node: Check if finished
- **Type / role:** `IF` — routes based on whether prediction is still processing
- **Condition logic:**
  - If `{{$json.status}} == "processing"` → **true branch**
  - Else → **false branch**
- **Connections:**
  - **True (processing):** to **Wait** (loop)
  - **False (not processing):** to **Check for errors**
- **Version-specific requirements:** IF **typeVersion 2.3**, conditions “version 3”, strict type validation
- **Edge cases / failures:**
  - Replicate statuses can include `starting`, `processing`, `succeeded`, `failed`, `canceled`.  
    This workflow only loops on exactly `processing`; if status is `starting`, it will **not** loop and will fall through to “Check for errors” (and likely be treated as error). This is a key behavior to be aware of.

---

### 2.4 Error Gate & Output Retrieval
**Overview:** Verifies whether the prediction succeeded; on success, extracts the output image URL and downloads the raw image. On failure, stops with a readable error.

**Nodes involved:**
- **Sticky Note4** (“Error checking and ouput”)
- **Check for errors**
- **Stop and Error**
- **Isolate output URL**
- **Retrieve Image**

#### Node: Sticky Note4
- **Type / role:** Sticky Note (block label)
- **Connections:** None

#### Node: Check for errors
- **Type / role:** `IF` — decides success vs failure
- **Condition logic:**
  - If `{{$json.status}} != "succeeded"` → **true branch** (treated as error)
  - Else → **false branch** (success)
- **Connections:**
  - **True (error):** to **Stop and Error**
  - **False (success):** to **Isolate output URL**
- **Version-specific requirements:** IF **typeVersion 2.3**
- **Edge cases / failures:**
  - If status is `starting` (see earlier note), it will be treated as error.
  - If Replicate returns `succeeded` but output is missing/empty, downstream steps fail.

#### Node: Stop and Error
- **Type / role:** `Stop and Error` — terminates execution with a custom error message
- **Error message expression:** `Error occurred while generating:  {{ $json.error }}`
- **Connections:** Input from **Check for errors** (error branch)
- **Edge cases / failures:**
  - If `$json.error` is missing, message becomes less informative (blank after colon).

#### Node: Isolate output URL
- **Type / role:** `Set` — maps Replicate output to a single `url` field
- **Key configuration:**
  - Sets `url` to `{{$json.output[0]}}`
  - Assumes Replicate returns an `output` array and the first element is the final image URL
- **Connections:**
  - Input from **Check for errors** (success branch)
  - Output to **Retrieve Image**
- **Version-specific requirements:** Set **typeVersion 3.4**
- **Edge cases / failures:**
  - If `output` is not an array, or empty, `output[0]` is undefined → next HTTP request fails.
  - Some Replicate models return different output structures (object, single string, multiple images).

#### Node: Retrieve Image
- **Type / role:** `HTTP Request` — downloads the resulting image as “raw image” response
- **Endpoint:** `{{$json.url}}`
- **Connections:** Input from **Isolate output URL**
- **Version-specific requirements:** HTTP Request **typeVersion 4.4**
- **Edge cases / failures:**
  - Output URL may expire (Replicate delivery URLs can be time-limited).
  - Depending on n8n HTTP node defaults, the response may be treated as text unless configured to download as file/binary in some setups; confirm your instance behavior if you need binary output.

---

### 2.5 Example / Visual Notes (Non-executable)
**Overview:** These nodes provide example images and expected output preview in the canvas.

**Nodes involved:**
- **Sticky Note7** (“Examples”)
- **Sticky Note10** (initial image preview link)
- **Sticky Note11** (reference image preview link)
- **Sticky Note6** (output image preview link)

All are **Sticky Notes** with no connections and no runtime behavior.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | stickyNote | Workflow description, setup notes, model context |  |  | ## Reference-based super-resolution using Riverflow 2.0 … More details at [Replicate API - riverflow-2.0-refsr](https://replicate.com/sourceful/riverflow-2.0-refsr). |
| Input form | formTrigger | Entry form to collect image + references |  | Input Handling | ### Receive and process inputs |
| Input Handling | code | Normalize/validate inputs; enforce refs array max 4 | Input form | POST Request to Replicate API | ### Receive and process inputs |
| POST Request to Replicate API | httpRequest | Create Replicate prediction for riverflow-2.0-refsr | Input Handling | Wait | ### Riverflow 2.0 takes data and begins generations |
| Wait | wait | Delay between polling attempts | POST Request to Replicate API; Check if finished | GET Request to check status | ### Polling loop to see if generation is complete |
| GET Request to check status | httpRequest | Poll prediction status via `urls.get` | Wait | Check if finished | ### Polling loop to see if generation is complete |
| Check if finished | if | Loop while status == processing | GET Request to check status | Wait; Check for errors | ### Polling loop to see if generation is complete |
| Check for errors | if | Route succeeded vs error | Check if finished | Stop and Error; Isolate output URL | ### Error checking and ouput |
| Stop and Error | stopAndError | Stop workflow with error message | Check for errors |  | ### Error checking and ouput |
| Isolate output URL | set | Extract `output[0]` into `url` | Check for errors | Retrieve Image | ### Error checking and ouput |
| Retrieve Image | httpRequest | Download final enhanced image | Isolate output URL |  | ### Error checking and ouput |
| Sticky Note1 | stickyNote | Block label |  |  | ### Receive and process inputs |
| Sticky Note2 | stickyNote | Block label |  |  | ### Riverflow 2.0 takes data and begins generations |
| Sticky Note3 | stickyNote | Block label |  |  | ### Polling loop to see if generation is complete |
| Sticky Note4 | stickyNote | Block label |  |  | ### Error checking and ouput |
| Sticky Note7 | stickyNote | Examples section label |  |  | ## Examples |
| Sticky Note10 | stickyNote | Example initial image |  |  | ### Initial Image — https://replicate.delivery/pbxt/OW3DXNmlv67ojQJla9m3QSARjk7Porx0DONS6GKcu6CV2mfo/Replicate%20x%20Sourceful%20%284%29.png |
| Sticky Note11 | stickyNote | Example reference image |  |  | ### Super resolution ref — https://replicate.delivery/pbxt/OW3DX8gMvTqrpi1YuacjIqS7277BuwDOSs73ZwBxErzpzuEn/Replicate%20x%20Sourceful%20%283%29.png |
| Sticky Note6 | stickyNote | Example output image |  |  | ### Output Image — https://replicate.delivery/xezq/MfYWhveDa9n32E3KTgVs4y4gfYpKyScUg4XTpCAH00fZlsNYB/tmp5zly0jjj.webp |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: **“Reference-Based Super-Resolution using Riverflow 2.0 (Image Enhancer/Upscaler)”**
- (Optional) In workflow settings, keep **Binary Mode: separate** to match the original behavior.

2) **Add a Form Trigger node**
- Node type: **Form Trigger**
- Form title: `Input data for Riverflow2.0 refsr`
- Form description: `for multiple references, separate with commas (MAX 4)`
- Add fields (both required):
  - Field label: `Image`
  - Field label: `Super Resolution References`

3) **Add a Code node: “Input Handling”**
- Connect: **Input form → Input Handling**
- Paste logic that:
  - Normalizes incoming field names to snake_case
  - Allows only `image` and `super_resolution_references`
  - Ensures `image` is a non-empty string
  - Converts references into an array by splitting comma-separated strings
  - Enforces **max 4** references
  - Outputs:
    - `image`
    - `super_resolution_refs` (array, default `[]`)

4) **Create Replicate credential (Bearer token)**
- Credential type: **HTTP Bearer Auth**
- Name: `Replicate API key`
- Token: your Replicate API token from https://replicate.com/

5) **Add HTTP Request node: “POST Request to Replicate API”**
- Connect: **Input Handling → POST Request to Replicate API**
- Method: **POST**
- URL: `https://api.replicate.com/v1/models/sourceful/riverflow-2.0-refsr/predictions`
- Authentication: **Predefined Credential Type → HTTP Bearer Auth** (`Replicate API key`)
- Send Headers: **true**
- Add header: `Prefer = wait`
- Send Body: **true**
- Body type: **JSON**
- JSON body (using expressions):
  - `input.image` = `{{ $json.image.toJsonString() }}`
  - `input.super_resolution_references` = `{{ $json.super_resolution_refs.toJsonString() }}`

6) **Add a Wait node**
- Connect: **POST Request to Replicate API → Wait**
- Wait amount: **20 seconds**

7) **Add HTTP Request node: “GET Request to check status”**
- Connect: **Wait → GET Request to check status**
- Method: **GET**
- URL: `{{ $json.urls.get }}`
- Authentication: **HTTP Bearer Auth** (`Replicate API key`)

8) **Add IF node: “Check if finished”**
- Connect: **GET Request to check status → Check if finished**
- Condition: String equals
  - Left: `{{ $json.status }}`
  - Operation: `equals`
  - Right: `processing`
- True branch (processing): connect to **Wait** (to form the loop)
- False branch: connect to **Check for errors** (next step)

9) **Add IF node: “Check for errors”**
- Connect: **Check if finished (false output) → Check for errors**
- Condition: String not equals
  - Left: `{{ $json.status }}`
  - Operation: `notEquals`
  - Right: `succeeded`
- True branch (not succeeded): connect to **Stop and Error**
- False branch (succeeded): connect to **Isolate output URL**

10) **Add Stop and Error node**
- Name: `Stop and Error`
- Error message: `Error occurred while generating:  {{ $json.error }}`

11) **Add Set node: “Isolate output URL”**
- Connect: **Check for errors (succeeded branch) → Isolate output URL**
- Add a string field:
  - Name: `url`
  - Value: `{{ $json.output[0] }}`

12) **Add HTTP Request node: “Retrieve Image”**
- Connect: **Isolate output URL → Retrieve Image**
- Method: **GET**
- URL: `{{ $json.url }}`
- (If you need binary output explicitly: configure the node to download the response as a file/binary, depending on your n8n version’s HTTP node options.)

13) **Add sticky notes (optional, for parity)**
- Add notes matching the original block labels and example image links, especially the model link:
  - https://replicate.com/sourceful/riverflow-2.0-refsr

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Add Replicate API key for HTTP requests | https://replicate.com/ |
| Riverflow 2.0 Reference-Based Super-Resolution model page | https://replicate.com/sourceful/riverflow-2.0-refsr |
| Example images embedded in canvas notes (initial, reference, output) | replicate.delivery links shown in Sticky Note10 / Sticky Note11 / Sticky Note6 |
| Important behavior: polling only loops on status == `processing` (not `starting`) | Consider adjusting IF logic if Replicate returns `starting` in your runs |
| No max retry/timeout guard in polling loop | Add a counter or max execution time strategy if needed in production |