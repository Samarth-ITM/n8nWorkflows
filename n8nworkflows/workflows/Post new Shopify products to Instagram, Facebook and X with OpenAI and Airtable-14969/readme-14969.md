Post new Shopify products to Instagram, Facebook and X with OpenAI and Airtable

https://n8nworkflows.xyz/workflows/post-new-shopify-products-to-instagram--facebook-and-x-with-openai-and-airtable-14969


# Post new Shopify products to Instagram, Facebook and X with OpenAI and Airtable

Let me analyze this n8n workflow thoroughly and produce a comprehensive, structured reference document. I need to go through every node, understand its role, connections, configuration, and then organize the information according to the required sections.

Let me first map out all nodes and their connections:

**Nodes (in order of flow):**

1. Schedule — Poll Every 10 Minutes (scheduleTrigger)
2. Shopify — Fetch Recent Products (API) (httpRequest)
3. Code — Extract & Validate Products (code)
4. Split In Batches — One Product at a Time (splitInBatches)
5. Airtable — Check If Already Posted (airtable)
6. IF — Not Yet Posted (Dedup Check) (if)
7. Set — Store Product Fields as Variables (set)
8. HTTP — Fetch Product Image Binary1 (httpRequest)
9. Upload a File (uploadToUrl)
10. Set — Store Hosted Image URL (set)
11. Code — Fan Out to 3 Platform Items (code)
12. Switch — Route to Platform Branches (switch)
13. OpenAI — Instagram Caption (openAi)
14. OpenAI — Facebook Caption (openAi)
15. OpenAI — Twitter/X Caption (openAi)
16. Instagram — Create Media Container1 (httpRequest)
17. Instagram — Publish Container (httpRequest)
18. Facebook — Publish Photo to Page (httpRequest)
19. Twitter/X — Publish Product Tweet1 (twitter)
20. Airtable — Log Posted Product (airtable)

**Sticky Notes:**
- 📝 Note — Poll, Fetch & Deduplicate
- 📝 Note — Set, Upload & Switch Route
- 📝 Note — 3 Independent AI Caption Chains
- 📝 Note — Post to Platforms & Log to Airtable
- 📋 MAIN — Workflow Overview1

Let me map which nodes each sticky note covers:

Note — Poll, Fetch & Deduplicate: covers Schedule, Shopify, Split In Batches, Airtable Search, IF node

Note — Set, Upload & Switch Route: covers Set (Product Fields), HTTP Fetch, Upload a File, Set (Hosted Image URL), Code (Fan Out), Switch

Note — 3 Independent AI Caption Chains: covers the 3 OpenAI nodes

Note — Post to Platforms & Log to Airtable: covers Instagram Create Container, Instagram Publish Container, Facebook Publish Photo, Twitter/X Publish Tweet, Airtable Log

MAIN — Workflow Overview1: general overview covering all nodes

Now let me build the document.

**Connections mapping:**

- Schedule → Shopify Fetch
- Shopify Fetch → Code Extract & Validate
- Code Extract & Validate → Split In Batches
- Split In Batches → Airtable Check If Already Posted
- Airtable Check If Already Posted → IF Not Yet Posted
- IF Not Yet Posted → Set Store Product Fields as Variables
- Set Store Product Fields → HTTP Fetch Product Image Binary
- HTTP Fetch Product Image Binary → Upload a File
- Upload a File → Set Store Hosted Image URL
- Set Store Hosted Image URL → Code Fan Out to 3 Platform Items
- Code Fan Out → Switch Route to Platform Branches
- Switch → (Instagram output) → OpenAI Instagram Caption
- Switch → (Facebook output) → OpenAI Facebook Caption
- Switch → (Twitter output) → OpenAI Twitter/X Caption
- OpenAI Instagram Caption → Instagram Create Media Container1
- OpenAI Facebook Caption → Facebook Publish Photo to Page
- OpenAI Twitter/X Caption → Twitter/X Publish Product Tweet1
- Instagram Create Media Container1 → Instagram Publish Container
- Instagram Publish Container → Airtable Log Posted Product
- Facebook Publish Photo to Page → Airtable Log Posted Product
- Twitter/X Publish Product Tweet1 → Airtable Log Posted Product

Let me now document each node in detail.

**Schedule — Poll Every 10 Minutes**
- Type: scheduleTrigger v1.2
- Role: Entry point, triggers workflow every 10 minutes
- Config: interval field = minutes, minutesInterval = 10

**Shopify — Fetch Recent Products (API)**
- Type: httpRequest v4.2
- Role: Calls Shopify Admin API to get recently published products
- URL: `https://{{ $vars.SHOPIFY_STORE_DOMAIN }}/admin/api/2024-01/products.json`
- Method: GET (default)
- Query params: status=active, published_status=published, limit=10, published_at_min (computed as 15 min ago), fields=id,title,body_html,vendor,handle,images,variants,tags,product_type
- Headers: X-Shopify-Access-Token from $vars.SHOPIFY_ACCESS_TOKEN, Content-Type: application/json
- Timeout: 15000ms
- Edge cases: API errors if token invalid, store domain misconfigured, rate limits, no products returned

**Code — Extract & Validate Products**
- Type: code v2
- Role: Parses Shopify response, validates each product (must have title + image + price >= $1), cleans HTML from description, maps to clean structure
- Output fields per item: productId (string), title, description (plain text, truncated to 450 chars), price ($X.XX), rawPrice (float), productUrl, imageUrl, allImages (up to 3), tags (up to 6, trimmed), vendor, productType
- Returns empty array if no products found (workflow stops)

**Split In Batches — One Product at a Time**
- Type: splitInBatches v3
- Role: Processes products one at a time to avoid API rate limits
- No special config (default batch size)

**Airtable — Check If Already Posted**
- Type: airtable v2
- Role: Search in ProductPostLog table for existing ProductID
- Operation: search
- Filter formula: `={ProductID}='{{ $json.productId }}'`
- Options: fields to return: ["ProductID"]
- Base/table: needs to be configured (list mode, empty values)
- If found, the IF node will skip; if not found (id is empty), proceeds

**IF — Not Yet Posted (Dedup Check)**
- Type: if v2
- Role: Checks if Airtable search returned no record (meaning product not yet posted)
- Condition: $json.id is empty (string operation isEmpty)
- True branch → continues to Set node
- False branch → product already posted, stops

**Set — Store Product Fields as Variables**
- Type: set v3.4
- Role: Maps product fields from the batch item into clean workflow variables for downstream use
- Assignments:
  - productId from Split In Batches item
  - title
  - description
  - price
  - productUrl
  - imageUrl
  - tags (joined with comma)
  - vendor
  - productType

**HTTP — Fetch Product Image Binary1**
- Type: httpRequest v4.2
- Role: Downloads the product image binary from imageUrl
- URL: dynamic from $json.imageUrl
- Options: timeout 20000ms, response format = file, max redirects = 5
- Output: binary data of the image

**Upload a File**
- Type: uploadToUrl v1
- Role: Uploads the binary image to UploadToURL CDN to get a public HTTPS URL
- No special parameters (uses credentials)
- Credentials: uploadToUrlApi (named "uploadtourl - Deepanshi")
- Output: JSON with hosted URL (url field, or data.url, file.url, link variations)

**Set — Store Hosted Image URL**
- Type: set v3.4
- Role: Stores the hosted image URL and re-stores all product fields for downstream nodes
- Assignments:
  - hostedImageUrl: tries multiple paths ($json.url, $json.data.url, $json.file.url, $json.link), fallbacks to original imageUrl
  - productId, title, description, price, productUrl, tags, vendor (all from previous Set node)

**Code — Fan Out to 3 Platform Items**
- Type: code v2
- Role: Creates 3 items from the single product item, each with a "platform" field set to "instagram", "facebook", or "twitter"
- This enables the Switch node to route to the correct branch

**Switch — Route to Platform Branches**
- Type: switch v3
- Role: Routes items based on the "platform" field to 3 outputs: Instagram, Facebook, Twitter
- Rules: 
  - Instagram: platform equals "instagram"
  - Facebook: platform equals "facebook"
  - Twitter: platform equals "twitter"
- Fallback output: extra (unused)

**OpenAI — Instagram Caption**
- Type: @n8n/n8n-nodes-langchain.openAi v1.4
- Role: Generates emoji-rich, storytelling Instagram caption with 10+ hashtags
- Resource: chat
- Credentials: OpenAI API (named "OpenAi - HSR")
- Note: The system prompt and user message are not explicitly shown in the JSON but should be configured with platform-specific instructions

**OpenAI — Facebook Caption**
- Type: @n8n/n8n-nodes-langchain.openAi v1.4
- Role: Generates conversational, CTA-driven Facebook post copy
- Resource: chat

**OpenAI — Twitter/X Caption**
- Type: @n8n/n8n-nodes-langchain.openAi v1.4
- Role: Generates short tweet (under 260 characters) with price and link
- Resource: chat

**Instagram — Create Media Container1**
- Type: httpRequest v4.2
- Role: Step 1 of Instagram publishing - creates a media container via Graph API
- URL: https://graph.facebook.com/v19.0/YOUR_IG_ACCOUNT_ID/media
- Method: POST
- Query params: image_url (from hostedImageUrl), caption (from OpenAI output $json.choices[0].message.content), access_token (YOUR_FB_ACCESS_TOKEN)
- Timeout: 15000ms

**Instagram — Publish Container**
- Type: httpRequest v4.2
- Role: Step 2 of Instagram publishing - publishes the media container
- URL: https://graph.facebook.com/v19.0/YOUR_IG_ACCOUNT_ID/media_publish
- Method: POST
- Query params: creation_id (from $json.id of container step), access_token (YOUR_FB_ACCESS_TOKEN)

**Facebook — Publish Photo to Page**
- Type: httpRequest v4.2
- Role: Posts image + caption to Facebook Page
- URL: https://graph.facebook.com/v19.0/YOUR_FB_PAGE_ID/photos
- Method: POST
- Query params: url (hostedImageUrl), message (from OpenAI output), access_token (YOUR_FB_ACCESS_TOKEN)

**Twitter/X — Publish Product Tweet1**
- Type: twitter v1
- Role: Posts the tweet text via native Twitter node
- Parameters: text = $json.choices[0].message.content
- Credentials: twitterOAuth1Api (named "X OAuth account")

**Airtable — Log Posted Product**
- Type: airtable v2
- Role: Creates a record in ProductPostLog with product details and post statuses
- Operation: create
- Columns/fields:
  - Title (from Set node)
  - ImageURL (hostedImageUrl)
  - PostedAt (ISO timestamp)
  - ProductID
  - TwitterStatus: "posted"
  - FacebookStatus: "posted"
  - InstagramStatus: "posted"
- Matching columns: ["ProductID"] (for upsert behavior)
- Credentials: airtableTokenApi (named "Airtable Personal Access Token account")

Now let me structure the documentation.

---

**Sticky notes coverage:**

"📝 Note — Poll, Fetch & Deduplicate" covers:
- Schedule — Poll Every 10 Minutes
- Shopify — Fetch Recent Products (API)
- Split In Batches — One Product at a Time
- Airtable — Check If Already Posted
- IF — Not Yet Posted (Dedup Check)

"📝 Note — Set, Upload & Switch Route" covers:
- Set — Store Product Fields as Variables
- HTTP — Fetch Product Image Binary1
- Upload a File
- Set — Store Hosted Image URL
- Code — Fan Out to 3 Platform Items
- Switch — Route to Platform Branches

"📝 Note — 3 Independent AI Caption Chains" covers:
- OpenAI — Instagram Caption
- OpenAI — Facebook Caption
- OpenAI — Twitter/X Caption

"📝 Note — Post to Platforms & Log to Airtable" covers:
- Instagram — Create Media Container1
- Instagram — Publish Container
- Facebook — Publish Photo to Page
- Twitter/X — Publish Product Tweet1
- Airtable — Log Posted Product

"📋 MAIN — Workflow Overview1" covers all nodes (general overview)

Now I need to build the complete document. Let me do it carefully, ensuring all nodes are covered, all sticky notes are mapped, and all details are captured.

Let me also consider edge cases and failure types for each node, and version-specific requirements.

I'll now write the full document.## 1. Workflow Overview

**Purpose**  
This workflow automates social-media promotion for every newly published Shopify product. On a 10‑minute schedule it polls the Shopify Admin API, deduplicates against an Airtable log, enriches each product with a publicly hosted image, generates platform‑specific AI captions, and publishes to Instagram, Facebook, and X (Twitter) while recording the result back in Airtable.

**Logical Blocks**

| Block | Functional Role |
|-------|-----------------|
| **1.1 – Poll, Fetch & Deduplicate** | Schedule trigger → Shopify API call → product validation → batch splitting → Airtable dedup check |
| **1.2 – Data Enrichment & Media Hosting** | Map product fields to variables → download image binary → upload to CDN → store hosted URL |
| **1.3 – Platform Fan‑Out** | Duplicate enriched item into three platform‑tagged items → Switch routes each to its own branch |
| **1.4 – Platform‑Specific AI Captions** | Three independent OpenAI chat calls, each with a tailored system prompt (Instagram, Facebook, X) |
| **1.5 – Multi‑Platform Publishing** | Instagram two‑step Graph API publish, Facebook photo post, X tweet |
| **1.6 – Result Logging** | Single Airtable record creation/upsert capturing product ID, timestamps, hosted image URL, and per‑platform status |

---

## 2. Block‑by‑Block Analysis

### Block 1.1 – Poll, Fetch & Deduplicate

**Overview**  
A schedule trigger runs every 10 minutes, calls the Shopify Admin REST API for products published in the last 15 minutes, validates and normalizes the data, processes one product at a time, and checks Airtable to skip products already posted.

**Nodes Involved**

| # | Node |
|---|------|
| 1 | Schedule — Poll Every 10 Minutes |
| 2 | Shopify — Fetch Recent Products (API) |
| 3 | Code — Extract & Validate Products |
| 4 | Split In Batches — One Product at a Time |
| 5 | Airtable — Check If Already Posted |
| 6 | IF — Not Yet Posted (Dedup Check) |

---

#### Node: Schedule — Poll Every 10 Minutes
- **Type:** `n8n-nodes-base.scheduleTrigger` (v1.2)  
- **Role:** Entry point; fires the workflow every 10 minutes.  
- **Configuration:** Interval set to *minutes* with `minutesInterval = 10`.  
- **Connections:** Output → *Shopify — Fetch Recent Products (API)*  
- **Edge Cases:** If the workflow execution exceeds 10 minutes, overlapping runs may occur. Use n8n’s execution concurrency settings to prevent duplicate runs.

---

#### Node: Shopify — Fetch Recent Products (API)
- **Type:** `n8n-nodes-base.httpRequest` (v4.2)  
- **Role:** Retrieves recently published active products from Shopify.  
- **Configuration:**  
  - **URL:** `https://{{ $vars.SHOPIFY_STORE_DOMAIN }}/admin/api/2024-01/products.json` (dynamic from env‑var).  
  - **Method:** GET (default).  
  - **Query Parameters:**  
    - `status = active`  
    - `published_status = published`  
    - `limit = 10`  
    - `published_at_min = {{ new Date(Date.now() - 15 * 60 * 1000).toISOString() }}` (products published in last 15 min)  
    - `fields = id,title,body_html,vendor,handle,images,variants,tags,product_type`  
  - **Headers:** `X-Shopify-Access-Token = {{ $vars.SHOPIFY_ACCESS_TOKEN }}`, `Content-Type = application/json`.  
  - **Timeout:** 15 000 ms.  
- **Connections:** Input ← *Schedule*; Output → *Code — Extract & Validate Products*  
- **Edge Cases:**  
  - 401/403 if token lacks `read_products` scope.  
  - Rate‑limit (429) from Shopify; will fail the execution.  
  - Empty `products` array leads downstream code to return `[]`, stopping the workflow.

---

#### Node: Code — Extract & Validate Products
- **Type:** `n8n-nodes-base.code` (v2)  
- **Role:** Parses the Shopify JSON response, filters out invalid products (missing title, no image, price < $1), strips HTML from description, and maps to a clean schema.  
- **Output Schema (per item):**  
  - `productId` (string)  
  - `title`  
  - `description` (plain text, ≤ 450 chars)  
  - `price` (formatted `$X.XX`)  
  - `rawPrice` (float)  
  - `productUrl` (constructed from vendor + handle)  
  - `imageUrl` (first image `src`)  
  - `allImages` (up to 3 image URLs)  
  - `tags` (up to 6, trimmed)  
  - `vendor`  
  - `productType`  
- **Logic Detail:** Returns `[]` when no valid products, which terminates the branch.  
- **Connections:** Input ← *Shopify Fetch*; Output → *Split In Batches*  
- **Edge Cases:** Products with HTML entities or exotic characters may produce unexpected description text. Price parsing may fail if `variants[0].price` is null.

---

#### Node: Split In Batches — One Product at a Time
- **Type:** `n8n-nodes-base.splitInBatches` (v3)  
- **Role:** Ensures each product is processed sequentially (default batch size 1), preventing API rate‑limit bursts.  
- **Connections:** Input ← *Code — Extract & Validate Products*; Output → *Airtable — Check If Already Posted*  
- **Edge Cases:** If upstream provides zero items, the node does nothing and the branch ends.

---

#### Node: Airtable — Check If Already Posted
- **Type:** `n8n-nodes-base.airtable` (v2)  
- **Role:** Searches the `ProductPostLog` table for an existing record with the same `ProductID`.  
- **Configuration:**  
  - **Operation:** `search`  
  - **Filter By Formula:** `={ProductID}='{{ $json.productId }}'`  
  - **Options → Fields:** `["ProductID"]` (minimizes returned data).  
  - **Base/Table:** Must be selected at runtime (currently set to list mode with empty values).  
- **Connections:** Input ← *Split In Batches*; Output → *IF — Not Yet Posted*  
- **Edge Cases:** If Airtable credentials are invalid or the base/table IDs are missing, the node errors. An empty result set passes an item with no `id` field, which the IF node uses to infer “not posted”.

---

#### Node: IF — Not Yet Posted (Dedup Check)
- **Type:** `n8n-nodes-base.if` (v2)  
- **Role:** Passes items where no matching Airtable record was found (i.e., `$json.id` is empty). Products already in the log are dropped.  
- **Configuration:**  
  - **Condition:** `{{ $json.id }}` **is empty** (string operation `isEmpty`).  
- **Connections:** Input ← *Airtable — Check If Already Posted*; **True** output → *Set — Store Product Fields as Variables*; **False** output → (end of branch).  
- **Edge Cases:** If Airtable returns a record, the item is filtered out; the workflow simply stops for that product.

---

### Block 1.2 – Data Enrichment & Media Hosting

**Overview**  
Collects validated product data into workflow variables, downloads the product image, re‑uploads it to a public CDN (UploadToURL), and stores the resulting hosted URL for downstream use.

**Nodes Involved**

| # | Node |
|---|------|
| 7 | Set — Store Product Fields as Variables |
| 8 | HTTP — Fetch Product Image Binary1 |
| 9 | Upload a File |
| 10 | Set — Store Hosted Image URL |

---

#### Node: Set — Store Product Fields as Variables
- **Type:** `n8n-nodes-base.set` (v3.4)  
- **Role:** Maps each relevant field from the batch item to named variables for clean downstream reference.  
- **Assignments:**  
  - `productId` ← Split In Batches item `productId`  
  - `title` ← item `title`  
  - `description` ← item `description`  
  - `price` ← item `price`  
  - `productUrl` ← item `productUrl`  
  - `imageUrl` ← item `imageUrl`  
  - `tags` ← item `tags.join(', ')`  
  - `vendor` ← item `vendor`  
  - `productType` ← item `productType`  
- **Connections:** Input ← *IF — Not Yet Posted*; Output → *HTTP — Fetch Product Image Binary1*  
- **Edge Cases:** If `tags` is unexpectedly non‑array the `.join` call will throw.

---

#### Node: HTTP — Fetch Product Image Binary1
- **Type:** `n8n-nodes-base.httpRequest` (v4.2)  
- **Role:** Downloads the product image binary for re‑upload to CDN.  
- **Configuration:**  
  - **URL:** `{{ $json.imageUrl }}` (dynamic).  
  - **Response Format:** File (binary).  
  - **Timeout:** 20 000 ms; max redirects = 5.  
- **Connections:** Input ← *Set — Store Product Fields as Variables*; Output → *Upload a File*  
- **Edge Cases:** If `imageUrl` is broken or returns a non‑image content type, the downstream upload may fail. Large images may exceed memory limits on small n8n instances.

---

#### Node: Upload a File
- **Type:** `n8n-nodes-uploadtourl.uploadToUrl` (v1)  
- **Role:** Uploads the binary image to the UploadToURL CDN and returns a public HTTPS URL.  
- **Configuration:** No parameters required; relies on stored credentials.  
- **Credentials:** `uploadToUrlApi` – named “uploadtourl - Deepanshi”.  
- **Connections:** Input ← *HTTP — Fetch Product Image Binary1*; Output → *Set — Store Hosted Image URL*  
- **Edge Cases:** Service downtime or invalid credentials will cause a fatal error. The returned JSON structure may vary; the downstream Set node handles multiple possible key paths.

---

#### Node: Set — Store Hosted Image URL
- **Type:** `n8n-nodes-base.set` (v3.4)  
- **Role:** Persists the CDN‑hosted URL and re‑maps all product fields so they survive the fan‑out step.  
- **Assignments:**  
  - `hostedImageUrl` ← first of `$json.url`, `$json.data.url`, `$json.file.url`, `$json.link`, or fallback to original `imageUrl`.  
  - Re‑assigns `productId`, `title`, `description`, `price`, `productUrl`, `tags`, `vendor` from the previous Set node.  
- **Connections:** Input ← *Upload a File*; Output → *Code — Fan Out to 3 Platform Items*  
- **Edge Cases:** If none of the URL keys exist, the node falls back to the original Shopify image URL, which may not be publicly accessible for some stores.

---

### Block 1.3 – Platform Fan‑Out

**Overview**  
Duplicates the enriched product item three times, each tagged with a `platform` field (`instagram`, `facebook`, `twitter`), then routes each item through a Switch node to the appropriate caption‑generation branch.

**Nodes Involved**

| # | Node |
|---|------|
| 11 | Code — Fan Out to 3 Platform Items |
| 12 | Switch — Route to Platform Branches |

---

#### Node: Code — Fan Out to 3 Platform Items
- **Type:** `n8n-nodes-base.code` (v2)  
- **Role:** Creates three items from the single enriched product, each with an added `platform` property.  
- **Logic:**  
  ```js
  const item = $input.first().json;
  return [
    { json: { ...item, platform: 'instagram' } },
    { json: { ...item, platform: 'facebook' } },
    { json: { ...item, platform: 'twitter' } }
  ];
  ```
- **Connections:** Input ← *Set — Store Hosted Image URL*; Output → *Switch — Route to Platform Branches*  
- **Edge Cases:** If `item` contains unexpected nested objects, shallow spread may not deep‑clone; downstream mutations could affect other branches.

---

#### Node: Switch — Route to Platform Branches
- **Type:** `n8n-nodes-base.switch` (v3)  
- **Role:** Routes each item based on the `platform` field to one of three outputs.  
- **Configuration:**  
  - **Output 0 (Instagram):** `platform` equals `instagram` (case‑insensitive).  
  - **Output 1 (Facebook):** `platform` equals `facebook`.  
  - **Output 2 (Twitter):** `platform` equals `twitter`.  
  - **Fallback:** Extra output (unused).  
- **Connections:** Input ← *Code — Fan Out*; Output 0 → *OpenAI — Instagram Caption*; Output 1 → *OpenAI — Facebook Caption*; Output 2 → *OpenAI — Twitter/X Caption*.  
- **Edge Cases:** If the `platform` field is missing or miss‑pelled, the item is routed to the fallback output and will be lost.

---

### Block 1.4 – Platform‑Specific AI Captions

**Overview**  
Three independent OpenAI chat completions, each with a tailored system prompt to generate platform‑appropriate captions.

**Nodes Involved**

| # | Node |
|---|------|
| 13 | OpenAI — Instagram Caption |
| 14 | OpenAI — Facebook Caption |
| 15 | OpenAI — Twitter/X Caption |

---

#### Node: OpenAI — Instagram Caption
- **Type:** `@n8n/n8n-nodes-langchain.openAi` (v1.4)  
- **Role:** Generates an emoji‑rich, storytelling Instagram caption with 10+ hashtags.  
- **Configuration:** Resource = `chat`.  
- **Credentials:** OpenAI API (named “OpenAi - HSR”).  
- **Expected Prompt Structure (to be configured by user):**  
  - System message: “You are a social media copywriter for Instagram. Write a storytelling caption with emojis and at least 10 hashtags for the following product.”  
  - User message: should include `title`, `description`, `price`, `vendor`, `tags`, `productUrl`.  
- **Connections:** Input ← *Switch Output 0*; Output → *Instagram — Create Media Container1*  
- **Edge Cases:** If the OpenAI API key is invalid or rate‑limited, the node fails and stops the Instagram branch. Long responses may exceed Instagram caption limits if not constrained in the prompt.

---

#### Node: OpenAI — Facebook Caption
- **Type:** `@n8n/n8n-nodes-langchain.openAi` (v1.4)  
- **Role:** Generates a conversational, CTA‑driven Facebook post copy.  
- **Configuration:** Resource = `chat`.  
- **Credentials:** Same OpenAI API credential.  
- **Expected Prompt Structure:**  
  - System: “You are a Facebook page manager. Write a friendly, CTA‑driven post for the product. Include a short link preview‑friendly description.”  
  - User: product fields.  
- **Connections:** Input ← *Switch Output 1*; Output → *Facebook — Publish Photo to Page*  
- **Edge Cases:** Facebook may reject posts with overly long messages or disallowed content.

---

#### Node: OpenAI — Twitter/X Caption
- **Type:** `@n8n/n8n-nodes-langchain.openAi` (v1.4)  
- **Role:** Generates a concise tweet (< 260 characters) including price and link.  
- **Configuration:** Resource = `chat`.  
- **Credentials:** Same OpenAI API credential.  
- **Expected Prompt Structure:**  
  - System: “You are a Twitter copywriter. Write a punchy tweet under 260 characters for the product, including price and a link.”  
  - User: product fields.  
- **Connections:** Input ← *Switch Output 2*; Output → *Twitter/X — Publish Product Tweet1*  
- **Edge Cases:** If the generated tweet exceeds 280 characters (X limit), the Twitter node will fail. Prompt should enforce a 260‑character limit to leave room for the auto‑appended link.

---

### Block 1.5 – Multi‑Platform Publishing

**Overview**  
Executes platform‑specific API calls to publish the product: Instagram two‑step (container creation + publish), Facebook photo post, and X tweet.

**Nodes Involved**

| # | Node |
|---|------|
| 16 | Instagram — Create Media Container1 |
| 17 | Instagram — Publish Container |
| 18 | Facebook — Publish Photo to Page |
| 19 | Twitter/X — Publish Product Tweet1 |

---

#### Node: Instagram — Create Media Container1
- **Type:** `n8n-nodes-base.httpRequest` (v4.2)  
- **Role:** Step 1 of Instagram publishing – creates a media container via the Graph API.  
- **Configuration:**  
  - **URL:** `https://graph.facebook.com/v19.0/YOUR_IG_ACCOUNT_ID/media`  
  - **Method:** POST  
  - **Query Parameters:**  
    - `image_url` ← `{{ $('Set — Store Hosted Image URL').item.json.hostedImageUrl }}`  
    - `caption` ← `{{ $json.choices[0].message.content }}` (OpenAI output)  
    - `access_token` ← `YOUR_FB_ACCESS_TOKEN`  
  - **Timeout:** 15 000 ms.  
- **Connections:** Input ← *OpenAI — Instagram Caption*; Output → *Instagram — Publish Container*  
- **Edge Cases:** `YOUR_IG_ACCOUNT_ID` and `YOUR_FB_ACCESS_TOKEN` are placeholders; must be replaced with real values. If the image URL is not publicly accessible, the container creation fails. Facebook API may return an `id` field which is passed to the publish step.

---

#### Node: Instagram — Publish Container
- **Type:** `n8n-nodes-base.httpRequest` (v4.2)  
- **Role:** Step 2 of Instagram publishing – publishes the media container created in the previous step.  
- **Configuration:**  
  - **URL:** `https://graph.facebook.com/v19.0/YOUR_IG_ACCOUNT_ID/media_publish`  
  - **Method:** POST  
  - **Query Parameters:**  
    - `creation_id` ← `{{ $json.id }}` (container ID from previous step)  
    - `access_token` ← `YOUR_FB_ACCESS_TOKEN`  
  - **Timeout:** 15 000 ms.  
- **Connections:** Input ← *Instagram — Create Media Container1*; Output → *Airtable — Log Posted Product*  
- **Edge Cases:** If the container is not ready (still processing), the publish call returns an error. Retry logic or a wait step may be needed for large images.

---

#### Node: Facebook — Publish Photo to Page
- **Type:** `n8n-nodes-base.httpRequest` (v4.2)  
- **Role:** Posts the product image and AI caption to the Facebook Page.  
- **Configuration:**  
  - **URL:** `https://graph.facebook.com/v19.0/YOUR_FB_PAGE_ID/photos`  
  - **Method:** POST  
  - **Query Parameters:**  
    - `url` ← `{{ $('Set — Store Hosted Image URL').item.json.hostedImageUrl }}`  
    - `message` ← `{{ $json.choices[0].message.content }}`  
    - `access_token` ← `YOUR_FB_ACCESS_TOKEN`  
  - **Timeout:** 15 000 ms.  
- **Connections:** Input ← *OpenAI — Facebook Caption*; Output → *Airtable — Log Posted Product*  
- **Edge Cases:** If `YOUR_FB_PAGE_ID` or `YOUR_FB_ACCESS_TOKEN` are invalid, the request returns a 403 or 401. The hosted image URL must be publicly accessible.

---

#### Node: Twitter/X — Publish Product Tweet1
- **Type:** `n8n-nodes-base.twitter` (v1)  
- **Role:** Publishes the AI‑generated tweet text via native X integration.  
- **Configuration:**  
  - `text` ← `{{ $json.choices[0].message.content }}`  
  - `additionalFields` – none.  
- **Credentials:** `twitterOAuth1Api` – named “X OAuth account”.  
- **Connections:** Input ← *OpenAI — Twitter/X Caption*; Output → *Airtable — Log Posted Product*  
- **Edge Cases:** If the tweet exceeds 280 characters, the node will return an API error. The OpenAI prompt should enforce a 260‑character limit to allow for the product URL length.

---

### Block 1.6 – Result Logging

**Overview**  
After all three platform posts (Instagram, Facebook, X) complete, a single Airtable record is created/upserted to log the product, its hosted image, and the per‑platform post status.

**Nodes Involved**

| # | Node |
|---|------|
| 20 | Airtable — Log Posted Product |

---

#### Node: Airtable — Log Posted Product
- **Type:** `n8n-nodes-base.airtable` (v2)  
- **Role:** Persists a deduplication record with product metadata and post statuses.  
- **Configuration:**  
  - **Operation:** `create` (acts as upsert because `matchingColumns` is set).  
  - **Columns:**  
    - `Title` ← `{{ $('Set — Store Hosted Image URL').item.json.title }}`  
    - `ImageURL` ← `{{ $('Set — Store Hosted Image URL').item.json.hostedImageUrl }}`  
    - `PostedAt` ← `{{ new Date().toISOString() }}`  
    - `ProductID` ← `{{ $('Set — Store Hosted Image URL').item.json.productId }}`  
    - `TwitterStatus` ← `"posted"`  
    - `FacebookStatus` ← `"posted"`  
    - `InstagramStatus` ← `"posted"`  
  - **Matching Columns:** `["ProductID"]` – enables upsert behavior (updates if already exists).  
  - **Base/Table:** Must be selected (list mode, currently empty).  
- **Credentials:** `airtableTokenApi` – named “Airtable Personal Access Token account”.  
- **Connections:** Receives items from *Instagram — Publish Container*, *Facebook — Publish Photo to Page*, and *Twitter/X — Publish Product Tweet1* (all three converge).  
- **Edge Cases:** If any upstream platform post fails, that branch does not reach this node, so statuses may be inaccurate. Consider adding error‑handling branches that still log a “failed” status.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|-----------|-----------|-----------------|----------------|-----------------|-------------|
| Schedule — Poll Every 10 Minutes | scheduleTrigger (v1.2) | Entry point; triggers every 10 min | – | Shopify — Fetch Recent Products (API) | 📝 Note — Poll, Fetch & Deduplicate: Schedule Trigger runs every 10 minutes — no webhook needed; fetches products published in the last 15 minutes. |
| Shopify — Fetch Recent Products (API) | httpRequest (v4.2) | Calls Shopify Admin API for recently published products | Schedule — Poll Every 10 Minutes | Code — Extract & Validate Products | 📝 Note — Poll, Fetch & Deduplicate: Shopify Node fetches products using published_at_min filter and status=active. |
| Code — Extract & Validate Products | code (v2) | Parses Shopify response, validates products, maps to clean schema | Shopify — Fetch Recent Products (API) | Split In Batches — One Product at a Time | 📝 Note — Poll, Fetch & Deduplicate: Code node extracts and validates products (must have title, image, price ≥ $1). |
| Split In Batches — One Product at a Time | splitInBatches (v3) | Processes products sequentially (one per iteration) | Code — Extract & Validate Products | Airtable — Check If Already Posted | 📝 Note — Poll, Fetch & Deduplicate: Split In Batches processes each new product one at a time to avoid API rate limits. |
| Airtable — Check If Already Posted | airtable (v2) | Searches ProductPostLog for existing ProductID | Split In Batches — One Product at a Time | IF — Not Yet Posted (Dedup Check) | 📝 Note — Poll, Fetch & Deduplicate: Airtable Search looks up ProductID; if found, IF node skips it. |
| IF — Not Yet Posted (Dedup Check) | if (v2) | Filters out products already in the log | Airtable — Check If Already Posted | Set — Store Product Fields as Variables (true) | 📝 Note — Poll, Fetch & Deduplicate: Only truly new products proceed. |
| Set — Store Product Fields as Variables | set (v3.4) | Maps product fields to workflow variables | IF — Not Yet Posted (Dedup Check) | HTTP — Fetch Product Image Binary1 | 📝 Note — Set, Upload & Switch Route: Set Node maps and stores all product fields into clean workflow variables. |
| HTTP — Fetch Product Image Binary1 | httpRequest (v4.2) | Downloads product image binary | Set — Store Product Fields as Variables | Upload a File | 📝 Note — Set, Upload & Switch Route: Downloads the product image binary. |
| Upload a File | uploadToUrl (v1) | Uploads image to CDN and returns hosted URL | HTTP — Fetch Product Image Binary1 | Set — Store Hosted Image URL | 📝 Note — Set, Upload & Switch Route: Uploads image to UploadToURL CDN; hosted URL stored. |
| Set — Store Hosted Image URL | set (v3.4) | Persists hosted image URL and re‑maps all product fields | Upload a File | Code — Fan Out to 3 Platform Items | 📝 Note — Set, Upload & Switch Route: Stores hosted image URL and re‑maps product fields. |
| Code — Fan Out to 3 Platform Items | code (v2) | Creates three items with platform tag (instagram, facebook, twitter) | Set — Store Hosted Image URL | Switch — Route to Platform Branches | 📝 Note — Set, Upload & Switch Route: Switch Node routes enriched product data into 3 simultaneous branches. |
| Switch — Route to Platform Branches | switch (v3) | Routes items to Instagram, Facebook, or X branch based on `platform` field | Code — Fan Out to 3 Platform Items | OpenAI — Instagram Caption (output 0), OpenAI — Facebook Caption (output 1), OpenAI — Twitter/X Caption (output 2) | 📝 Note — Set, Upload & Switch Route: Routes data into 3 independent branches. |
| OpenAI — Instagram Caption | @n8n/n8n-nodes-langchain.openAi (v1.4) | Generates Instagram‑style caption (emoji‑rich, 10+ hashtags) | Switch — Route to Platform Branches (output 0) | Instagram — Create Media Container1 | 📝 Note — 3 Independent AI Caption Chains: Instagram branch uses emoji‑rich storytelling style, 10+ hashtags. |
| OpenAI — Facebook Caption | @n8n/n8n-nodes-langchain.openAi (v1.4) | Generates Facebook‑style caption (conversational, CTA‑driven) | Switch — Route to Platform Branches (output 1) | Facebook — Publish Photo to Page | 📝 Note — 3 Independent AI Caption Chains: Facebook branch uses conversational CTA‑driven copy. |
| OpenAI — Twitter/X Caption | @n8n/n8n-nodes-langchain.openAi (v1.4) | Generates X‑style caption (≤ 260 chars, includes price & link) | Switch — Route to Platform Branches (output 2) | Twitter/X — Publish Product Tweet1 | 📝 Note — 3 Independent AI Caption Chains: Twitter/X branch uses punchy under‑260‑char copy with price + link. |
| Instagram — Create Media Container1 | httpRequest (v4.2) | Creates Instagram media container via Graph API | OpenAI — Instagram Caption | Instagram — Publish Container | 📝 Note — Post to Platforms & Log to Airtable: Two‑step Graph API flow (create container → publish). |
| Instagram — Publish Container | httpRequest (v4.2) | Publishes Instagram media container | Instagram — Create Media Container1 | Airtable — Log Posted Product | 📝 Note — Post to Platforms & Log to Airtable: Container ID from step 1 is passed to step 2. |
| Facebook — Publish Photo to Page | httpRequest (v4.2) | Posts image + caption to Facebook Page | OpenAI — Facebook Caption | Airtable — Log Posted Product | 📝 Note — Post to Platforms & Log to Airtable: Graph API /photos endpoint posts image + caption. |
| Twitter/X — Publish Product Tweet1 | twitter (v1) | Publishes tweet via native X node | OpenAI — Twitter/X Caption | Airtable — Log Posted Product | 📝 Note — Post to Platforms & Log to Airtable: Native Twitter node posts the punchy tweet. |
| Airtable — Log Posted Product | airtable (v2) | Creates/upserts ProductPostLog record with statuses and metadata | Instagram — Publish Container, Facebook — Publish Photo to Page, Twitter/X — Publish Product Tweet1 | – | 📝 Note — Post to Platforms & Log to Airtable: Final Airtable record created/updated with ProductID, post statuses per platform, timestamp, and hosted image URL — prevents re‑processing. |
| 📝 Note — Poll, Fetch & Deduplicate | stickyNote (v1) | Documentation note | – | – | (self) |
| 📝 Note — Set, Upload & Switch Route | stickyNote (v1) | Documentation note | – | – | (self) |
| 📝 Note — 3 Independent AI Caption Chains | stickyNote (v1) | Documentation note | – | – | (self) |
| 📝 Note — Post to Platforms & Log to Airtable | stickyNote (v1) | Documentation note | – | – | (self) |
| 📋 MAIN — Workflow Overview1 | stickyNote (v1) | High‑level overview note | – | – | 🛍️ E-commerce Product Launch — Auto-Post to Instagram, Facebook & X. Architecture: Schedule Trigger (poll), Split In Batches, Airtable dedup, Switch to 3 branches, 3x OpenAI calls, UploadToURL CDN, Airtable log. |

---

## 4. Reproducing the Workflow from Scratch

Below is a step‑by‑step guide to recreate the entire workflow manually in n8n.

### Prerequisites
- n8n instance (self‑hosted or cloud) with the **UploadToURL** community node installed.
- Credentials configured:  
  - Shopify Admin API (Personal Access Token with `read_products` scope)  
  - Airtable Personal Access Token (with write access to the target base)  
  - OpenAI API key  
  - Facebook Graph API Page token (with `pages_manage_posts` and `instagram_content_publish` permissions)  
  - X (Twitter) OAuth1 credentials  
  - UploadToURL API credentials  
- Environment variables (or workflow static data):  
  - `SHOPIFY_STORE_DOMAIN`  
  - `SHOPIFY_ACCESS_TOKEN`  
  - `IG_ACCOUNT_ID`  
  - `FB_ACCESS_TOKEN`  
  - `FB_PAGE_ID`

### Step 1 – Create Schedule Trigger
1. Add **Schedule Trigger** node.  
2. Set interval to **Every 10 minutes** (field: minutes, value: 10).  
3. Name: `Schedule — Poll Every 10 Minutes`.

### Step 2 – Add Shopify Fetch Node
4. Add **HTTP Request** node.  
5. Name: `Shopify — Fetch Recent Products (API)`.  
6. Set **Method** to GET.  
7. Set **URL** to `https://{{ $vars.SHOPIFY_STORE_DOMAIN }}/admin/api/2024-01/products.json`.  
8. Enable **Send Query Parameters** and add:  
   - `status` = `active`  
   - `published_status` = `published`  
   - `limit` = `10`  
   - `published_at_min` = `{{ new Date(Date.now() - 15 * 60 * 1000).toISOString() }}`  
   - `fields` = `id,title,body_html,vendor,handle,images,variants,tags,product_type`  
9. Enable **Send Headers** and add:  
   - `X-Shopify-Access-Token` = `{{ $vars.SHOPIFY_ACCESS_TOKEN }}`  
   - `Content-Type` = `application/json`  
10. Set **Timeout** to 15000 ms.  
11. Connect output of *Schedule* → input of this node.

### Step 3 – Add Code Node for Product Extraction
12. Add **Code** node.  
13. Name: `Code — Extract & Validate Products`.  
14. Paste the JavaScript code that extracts `products` from the Shopify response, filters for title + image + price ≥ $1, strips HTML from `body_html`, truncates description to 450 chars, and maps each product to the schema described in section 2.  
15. Connect *Shopify Fetch* → *Code Extract*.

### Step 4 – Add Split In Batches Node
16. Add **Split In Batches** node.  
17. Name: `Split In Batches — One Product at a Time`.  
18. Keep default batch size (1).  
19. Connect *Code Extract* → *Split In Batches*.

### Step 5 – Add Airtable Search Node (Dedup Check)
20. Add **Airtable** node.  
21. Name: `Airtable — Check If Already Posted`.  
22. Set **Operation** to `search`.  
23. Select the target **Base** and **Table** (`ProductPostLog`).  
24. Set **Filter By Formula** to `={ProductID}='{{ $json.productId }}'`.  
25. Under **Options → Fields**, add `ProductID`.  
26. Connect *Split In Batches* → *Airtable Search*.

### Step 6 – Add IF Node (Not Yet Posted)
27. Add **IF** node.  
28. Name: `IF — Not Yet Posted (Dedup Check)`.  
29. Add condition: **Left Value** = `{{ $json.id }}`, **Operation** = `is empty`.  
30. Connect *Airtable Search* → *IF* (input).  
31. Connect **True** branch of IF → next node (Set Product Fields).

### Step 7 – Add Set Node (Product Fields)
32. Add **Set** node.  
33. Name: `Set — Store Product Fields as Variables`.  
34. Add assignments:  
   - `productId` (string) ← `{{ $('Split In Batches — One Product at a Time').item.json.productId }}`  
   - `title` ← `{{ $('Split In Batches — One Product at a Time').item.json.title }}`  
   - `description` ← `{{ $('Split In Batches — One Product at a Time').item.json.description }}`  
   - `price` ← `{{ $('Split In Batches — One Product at a Time').item.json.price }}`  
   - `productUrl` ← `{{ $('Split In Batches — One Product at a Time').item.json.productUrl }}`  
   - `imageUrl` ← `{{ $('Split In Batches — One Product at a Time').item.json.imageUrl }}`  
   - `tags` (string) ← `{{ $('Split In Batches — One Product at a Time').item.json.tags.join(', ') }}`  
   - `vendor` ← `{{ $('Split In Batches — One Product at a Time').item.json.vendor }}`  
   - `productType` ← `{{ $('Split In Batches — One Product at a Time').item.json.productType }}`  

### Step 8 – Add HTTP Request to Fetch Image Binary
35. Add **HTTP Request** node.  
36. Name: `HTTP — Fetch Product Image Binary1`.  
37. Set **URL** to `{{ $json.imageUrl }}`.  
38. Under **Options → Response → Response Format**, select **File**.  
39. Set **Timeout** to 20000 ms, **Max Redirects** to 5.  
40. Connect *Set Product Fields* → *HTTP Fetch Image*.

### Step 9 – Add UploadToURL Node
41. Add **UploadToURL** node (community node `n8n-nodes-uploadtourl`).  
42. Name: `Upload a File`.  
43. Select the previously configured **UploadToURL API** credential.  
44. Connect *HTTP Fetch Image* → *Upload a File*.

### Step 10 – Add Set Node (Hosted Image URL)
45. Add **Set** node.  
46. Name: `Set — Store Hosted Image URL`.  
47. Add assignments:  
   - `hostedImageUrl` ← `{{ $json?.url ?? $json?.data?.url ?? $json?.file?.url ?? $json?.link ?? $('Set — Store Product Fields as Variables').item.json.imageUrl }}`  
   - `productId` ← `{{ $('Set — Store Product Fields as Variables').item.json.productId }}`  
   - `title` ← `{{ $('Set — Store Product Fields as Variables').item.json.title }}`  
   - `description` ← `{{ $('Set — Store Product Fields as Variables').item.json.description }}`  
   - `price` ← `{{ $('Set — Store Product Fields as Variables').item.json.price }}`  
   - `productUrl` ← `{{ $('Set — Store Product Fields as Variables').item.json.productUrl }}`  
   - `tags` ← `{{ $('Set — Store Product Fields as Variables').item.json.tags }}`  
   - `vendor` ← `{{ $('Set — Store Product Fields as Variables').item.json.vendor }}`  
48. Connect *Upload a File* → *Set Hosted Image URL*.

### Step 11 – Add Code Node (Fan Out)
49. Add **Code** node.  
50. Name: `Code — Fan Out to 3 Platform Items`.  
51. Paste the JavaScript that spreads the incoming item into three objects, each with a `platform` field set to `instagram`, `facebook`, or `twitter`.  
52. Connect *Set Hosted Image URL* → *Fan Out*.

### Step 12 – Add Switch Node (Platform Routing)
53. Add **Switch** node.  
54. Name: `Switch — Route to Platform Branches`.  
55. Configure three rules:  
   - Rule 0: `platform` equals `instagram` → output **Instagram**  
   - Rule 1: `platform` equals `facebook` → output **Facebook**  
   - Rule 2: `platform` equals `twitter` → output **Twitter**  
   - Fallback output: **extra** (unused).  
56. Connect *Fan Out* → *Switch*.

### Step 13 – Add OpenAI Caption Nodes (One Per Platform)

**Instagram Caption**  
57. Add **OpenAI** node (LangChain OpenAI).  
58. Name: `OpenAI — Instagram Caption`.  
59. Set **Resource** to `chat`.  
60. Select your **OpenAI API** credential.  
61. Configure messages:  
   - System: “You are a social media copywriter for Instagram. Write a storytelling caption with emojis and at least 10 hashtags for the following product.”  
   - User: Include `{{ $json.title }}`, `{{ $json.description }}`, `{{ $json.price }}`, `{{ $json.vendor }}`, `{{ $json.tags }}`, `{{ $json.productUrl }}`.  
62. Connect **Switch Output 0 (Instagram)** → this node.

**Facebook Caption**  
63. Add **OpenAI** node.  
64. Name: `OpenAI — Facebook Caption`.  
65. Same configuration as above, but system prompt: “You are a Facebook page manager. Write a friendly, CTA‑driven post for the product. Include a short link preview‑friendly description.”  
66. Connect **Switch Output 1 (Facebook)** → this node.

**Twitter/X Caption**  
67. Add **OpenAI** node.  
68. Name: `OpenAI — Twitter/X Caption`.  
69. System prompt: “You are a Twitter copywriter. Write a punchy tweet under 260 characters for the product, including the price and a link.”  
70. Connect **Switch Output 2 (Twitter)** → this node.

### Step 14 – Add Instagram Publishing Nodes

**Create Media Container**  
71. Add **HTTP Request** node.  
72. Name: `Instagram — Create Media Container1`.  
73. Set **Method** to POST.  
74. Set **URL** to `https://graph.facebook.com/v19.0/{{ $vars.IG_ACCOUNT_ID }}/media`.  
75. Enable **Send Query Parameters** and add:  
   - `image_url` = `{{ $('Set — Store Hosted Image URL').item.json.hostedImageUrl }}`  
   - `caption` = `{{ $json.choices[0].message.content }}`  
   - `access_token` = `{{ $vars.FB_ACCESS_TOKEN }}`  
76. Set **Timeout** to 15000 ms.  
77. Connect *OpenAI — Instagram Caption* → this node.

**Publish Container**  
78. Add **HTTP Request** node.  
79. Name: `Instagram — Publish Container`.  
80. Set **Method** to POST.  
81. Set **URL** to `https://graph.facebook.com/v19.0/{{ $vars.IG_ACCOUNT_ID }}/media_publish`.  
82. Add query parameters:  
   - `creation_id` = `{{ $json.id }}`  
   - `access_token` = `{{ $vars.FB_ACCESS_TOKEN }}`  
83. Connect *Instagram — Create Media Container1* → this node.

### Step 15 – Add Facebook Publishing Node
84. Add **HTTP Request** node.  
85. Name: `Facebook — Publish Photo to Page`.  
86. Set **Method** to POST.  
87. Set **URL** to `https://graph.facebook.com/v19.0/{{ $vars.FB_PAGE_ID }}/photos`.  
88. Add query parameters:  
   - `url` = `{{ $('Set — Store Hosted Image URL').item.json.hostedImageUrl }}`  
   - `message` = `{{ $json.choices[0].message.content }}`  
   - `access_token` = `{{ $vars.FB_ACCESS_TOKEN }}`  
89. Connect *OpenAI — Facebook Caption* → this node.

### Step 16 – Add Twitter Publishing Node
90. Add **Twitter** node.  
91. Name: `Twitter/X — Publish Product Tweet1`.  
92. Set **Text** to `{{ $json.choices[0].message.content }}`.  
93. Select your **X OAuth1** credential.  
94. Connect *OpenAI — Twitter/X Caption* → this node.

### Step 17 – Add Airtable Log Node
95. Add **Airtable** node.  
96. Name: `Airtable — Log Posted Product`.  
97. Set **Operation** to `create`.  
98. Select your **Airtable** credential (Personal Access Token).  
99. Choose the target **Base** and **Table** (`ProductPostLog`).  
100. Under **Columns**, map:  
   - `Title` ← `{{ $('Set — Store Hosted Image URL').item.json.title }}`  
   - `ImageURL` ← `{{ $('Set — Store Hosted Image URL').item.json.hostedImageUrl }}`  
   - `PostedAt` ← `{{ new Date().toISOString() }}`  
   - `ProductID` ← `{{ $('Set — Store Hosted Image URL').item.json.productId }}`  
   - `TwitterStatus` ← `posted`  
   - `FacebookStatus` ← `posted`  
   - `InstagramStatus` ← `posted`  
101. Set **Matching Columns** to `["ProductID"]` to enable upsert behavior.  
102. Connect outputs from *Instagram — Publish Container*, *Facebook — Publish Photo to Page*, and *Twitter/X — Publish Product Tweet1* → this node.

### Step 18 – Add Sticky Notes (Optional)
103. Add four **Sticky Note** nodes with content matching the documentation notes in the workflow JSON (Poll, Fetch & Deduplicate; Set, Upload & Switch Route; 3 Independent AI Caption Chains; Post to Platforms & Log to Airtable) and position them over the corresponding node groups.  
104. Add one main overview sticky note summarizing architecture and setup requirements.

### Step 19 – Save & Activate
105. Save the workflow.  
106. Ensure all credentials are valid and environment variables are set.  
107. Activate the workflow to start the 10‑minute polling cycle.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Shopify Admin API version used is `2024-01`. Ensure your store supports this version. | https://shopify.dev/docs/admin-api/rest/reference/products |
| Airtable table must be named `ProductPostLog` with fields: `ProductID` (primary), `Title`, `PostedAt`, `InstagramStatus`, `FacebookStatus`, `TwitterStatus`, `ImageURL`. | https://airtable.com/developers |
| UploadToURL is a community node (`n8n-nodes-uploadtourl`). Install via n8n community nodes before running. | https://www.npmjs.com/package/n8n-nodes-uploadtourl |
| Facebook Graph API requires a Page token with `pages_manage_posts` and `instagram_content_publish` scopes. | https://developers.facebook.com/docs/pages/access-tokens |
| Instagram publishing uses the Content Publishing API (two‑step container flow). Your Instagram account must be a Business or Creator account linked to a Facebook Page. | https://developers.facebook.com/docs/instagram-api/guides/content-publishing |
| Twitter (X) OAuth1 credentials must have read and write permissions. | https://developer.twitter.com/en/docs/authentication/oauth-1-0a |
| OpenAI model choice is not specified in the workflow JSON; configure the desired model (e.g., `gpt-4o-mini`) in each OpenAI node. | https://platform.openai.com/docs/models |
| The workflow uses `{{ $vars.SHOPIFY_STORE_DOMAIN }}` and `{{ $vars.SHOPIFY_ACCESS_TOKEN }}` as expression references. Set these as workflow variables or replace them with static values. | – |
| Placeholder strings `YOUR_IG_ACCOUNT_ID`, `YOUR_FB_PAGE_ID`, and `YOUR_FB_ACCESS_TOKEN` in the HTTP Request nodes must be replaced with real values or variable expressions before activation. | – |
| If a product image URL is not publicly accessible (e.g., Shopify CDN requires authentication), the UploadToURL step may fail. Consider alternative CDN or signed URL strategies. | – |
| The Airtable log sets all three platform statuses to `"posted"` regardless of individual success. To capture failures, add error‑handling branches that write a `"failed"` status. | – |
| n8n’s `splitInBatches` node processes one item at a time by default. Increasing batch size may cause rate‑limit errors with Shopify or social APIs. | – |
| The workflow assumes all three platform posts succeed before reaching the Airtable log. In production, consider using `Set` nodes or merge nodes to collect results and conditionally log partial failures. | – |
| The `Code — Fan Out to 3 Platform Items` node uses a shallow spread (`...item`). If the product object contains nested objects that are later mutated, consider deep‑cloning to avoid cross‑branch contamination. | – |