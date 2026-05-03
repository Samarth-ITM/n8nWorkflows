Forecast demand, optimize pricing, and engage customers with GPT‑4.1, Postgres, email, and Slack

https://n8nworkflows.xyz/workflows/forecast-demand--optimize-pricing--and-engage-customers-with-gpt-4-1--postgres--email--and-slack-12791


# Forecast demand, optimize pricing, and engage customers with GPT‑4.1, Postgres, email, and Slack

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title (given):** Forecast demand, optimize pricing, and engage customers with GPT‑4.1, Postgres, email, and Slack  
**Workflow name (JSON):** E-commerce Intelligence & Automated Customer Engagement Platform

**Purpose:**  
This workflow ingests e-commerce events via a webhook (orders, reviews, inventory updates, and social media mentions), normalizes and validates them, runs AI-powered analyses (demand forecasting, review sentiment, product recommendations), applies pricing/promotion/replenishment business rules, stores results in Postgres (CRM + inventory), and notifies stakeholders via Slack and customers via email. It includes an exception path with a manual review wait-state.

### Logical blocks
1.1 **Multi-channel ingestion & configuration** → Webhook trigger + centralized thresholds/parameters.  
1.2 **Routing, normalization & validation** → Switch by `dataType`, normalize fields per stream, validate required fields.  
1.3 **AI-powered intelligence (parallel)** → Three AI agents (forecast, sentiment, recommendations) using a shared OpenAI chat model + structured output parsers.  
1.4 **Consolidation & deduplication** → Merge AI outputs + remove duplicates by `(orderId, customerId)`.  
1.5 **Business rules (pricing → promotions → replenishment)** → Code nodes compute final price, promotions, and replenishment recommendations.  
1.6 **Exception handling & human-in-the-loop** → If low stock/negative sentiment/action required, wait for manual review.  
1.7 **Persistence & engagement distribution** → Store to Postgres tables, Slack alerts, and email campaigns.  
1.8 **Inventory stream persistence & alerting** → Inventory updates stored and supply chain notified.  
1.9 **Social stream alerting** → Social mentions forwarded to customer support via Slack.

---

## 2. Block-by-Block Analysis

### 1.1 Multi-channel ingestion & configuration
**Overview:** Receives inbound event payloads and defines global thresholds (low stock, negative sentiment, etc.) referenced later by rules/conditions.  
**Nodes involved:** `Webhook - Ingest Data`, `Workflow Configuration`

#### Webhook - Ingest Data
- **Type / role:** `Webhook` trigger; entry point for all data streams.
- **Key configuration:**
  - Method: `POST`
  - Path: `ecommerce-data` (creates a URL like `/webhook/ecommerce-data`)
  - Response mode: `lastNode`
  - Response data: `firstEntryBinary` (unusual for JSON-only payloads; see edge cases)
- **Input/Output:**
  - No input (trigger).
  - Output → `Workflow Configuration`.
- **Edge cases / failures:**
  - If callers send JSON but you respond with `firstEntryBinary`, response may be empty or unexpected unless binary data is attached.
  - Large payloads may hit n8n body size limits.
  - Missing `dataType` later causes routing to drop (no matching switch rule).

#### Workflow Configuration
- **Type / role:** `Set` node; stores tunable workflow constants.
- **Key configuration (fields created):**
  - `lowStockThreshold` = 10
  - `negativeSentimentThreshold` = -0.5
  - `highDemandThreshold` = 100
  - `apiRateLimit` = 100
  - `batchSize` = 50
  - `retryAttempts` = 3
  - `retryDelay` = 5000 (ms)
- **Expressions/variables:** Later referenced via `$('Workflow Configuration').first().json.<field>`.
- **Input/Output:**
  - Input ← `Webhook - Ingest Data`
  - Output → `Route by Data Type`
- **Edge cases:**
  - If the workflow is executed with multiple items, `first()` may not correspond to the same item context you expect. Here it’s used as “global config”, which is typical but should be understood.

---

### 1.2 Routing, normalization & validation
**Overview:** Routes events into four streams and standardizes payloads into consistent schemas before validation filters gate downstream processing.  
**Nodes involved:** `Route by Data Type`, `Normalize Orders`, `Validate Orders`, `Normalize Reviews`, `Validate Reviews`, `Normalize Inventory`, `Validate Inventory`, `Normalize Social Media`, `Validate Social Media`

#### Route by Data Type
- **Type / role:** `Switch`; routes by `{{$json.dataType}}`.
- **Rules / outputs (renamed outputs):**
  - `orders` when `dataType == "order"`
  - `reviews` when `dataType == "review"`
  - `inventory` when `dataType == "inventory"`
  - `social` when `dataType == "social_media"`
- **Input/Output:**
  - Input ← `Workflow Configuration`
  - Outputs → respective normalize nodes.
- **Edge cases:**
  - Any other `dataType` produces no output (silent drop unless an additional “fallback” output is added).

#### Normalize Orders
- **Type / role:** `Set`; maps possible field variants to canonical order schema.
- **Key mappings (examples):**
  - `orderId = $json.id || $json.order_id`
  - `customerId = $json.customer_id || $json.customerId`
  - `orderDate = $json.created_at || $json.orderDate || $now`
  - `totalAmount = $json.total || $json.amount || 0`
  - `products = $json.items || $json.products || []`
  - `dataType = "order"`
- **Connections:** Input ← `Route by Data Type (orders)`; Output → `Validate Orders`
- **Edge cases:**
  - `orderDate` becomes a string; ensure downstream DB column types align (timestamp vs text).
  - `products` is an array but later CRM insert stores `recommendations`, not `products`.

#### Validate Orders
- **Type / role:** `Filter`; ensures minimum order integrity.
- **Conditions (AND):**
  - `orderId` not empty
  - `customerId` not empty
  - `totalAmount > 0`
- **Connections:** Output → `AI Agent - Demand Forecasting` and `AI Agent - Product Recommendations` (parallel)
- **Edge cases:**
  - If `totalAmount` is a string, loose coercion may cause unexpected results; here it is set as `number` in normalization.

#### Normalize Reviews
- **Type / role:** `Set`; canonical review schema.
- **Key mappings:**
  - `reviewId = $json.id || $json.review_id`
  - `productId = $json.product_id || $json.productId`
  - `rating = $json.rating || $json.stars || 0`
  - `reviewText = $json.text || $json.comment || $json.review || ''`
  - `dataType = "review"`
- **Connections:** Output → `Validate Reviews`
- **Edge cases:** Rating defaults to 0; may skew sentiment prompt context if not present.

#### Validate Reviews
- **Type / role:** `Filter`; requires review identity and content.
- **Conditions (AND):** `reviewId`, `productId`, `reviewText` not empty
- **Connections:** Output → `AI Agent - Sentiment Analysis`
- **Edge cases:** Strict validation enabled; missing fields will drop items.

#### Normalize Inventory
- **Type / role:** `Set`; canonical inventory schema.
- **Key mappings:**
  - `productId = $json.id || $json.product_id || $json.sku`
  - `currentStock = $json.stock || $json.quantity || 0`
  - `reorderPoint = $json.reorder_point || $json.minStock || 20`
  - `warehouseLocation` default `'default'`
  - `dataType = "inventory"`
- **Connections:** Output → `Validate Inventory`

#### Validate Inventory
- **Type / role:** `Filter`; requires `productId` and non-negative stock.
- **Conditions:** `productId` not empty; `currentStock >= 0`
- **Connections:** Output → `Store in Inventory Database`
- **Edge cases:** No AI forecasting is connected to inventory stream directly; `forecastedDemand` stored later may be missing unless the inbound inventory payload already contains it.

#### Normalize Social Media
- **Type / role:** `Set`; canonical social post schema.
- **Key mappings:**
  - `postId = $json.id || $json.post_id`
  - `platform = $json.platform || $json.source || 'unknown'`
  - `content = $json.text || $json.message || $json.content || ''`
  - `engagement = $json.likes + $json.shares + $json.comments || 0`
  - `dataType = "social_media"`
- **Connections:** Output → `Validate Social Media`
- **Edge cases (important):**
  - If any of `likes/shares/comments` are `undefined`, `undefined + number` becomes `NaN`, which is falsy, so it falls back to `0`. This hides partial engagement data; consider `(likes||0)+(shares||0)+(comments||0)`.

#### Validate Social Media
- **Type / role:** `Filter`; requires `postId` and `content`.
- **Connections:** Output → `Notify Customer Support Team`

---

### 1.3 AI-powered intelligence analysis (parallel)
**Overview:** Uses a shared OpenAI chat model (`gpt-4.1-mini`) with three LangChain Agent nodes to generate structured analytics.  
**Nodes involved:** `OpenAI Chat Model`, `AI Agent - Demand Forecasting`, `Structured Output - Forecast`, `AI Agent - Sentiment Analysis`, `Structured Output - Sentiment`, `AI Agent - Product Recommendations`, `Structured Output - Recommendations`

#### OpenAI Chat Model
- **Type / role:** `lmChatOpenAi`; provides the language model to the agent nodes via the `ai_languageModel` connection.
- **Key configuration:** model = `gpt-4.1-mini`
- **Credentials:** OpenAI API credential (“OpenAi account”).
- **Connections:** Outputs its model to all three AI Agent nodes.
- **Edge cases:** Model availability / permission errors; rate limiting; token limits if `JSON.stringify($json)` is large.

#### AI Agent - Demand Forecasting
- **Type / role:** LangChain `agent`; generates demand forecast from order data.
- **Prompt input:** `Order Data: {{ JSON.stringify($json) }}`
- **System message:** forecasting tasks + requires structured output (forecast, confidence, actions, risk factors).
- **Output parsing:** Connected to `Structured Output - Forecast` via `ai_outputParser`.
- **Connections:** Input ← `Validate Orders`; Output → `Merge Analysis Results` (input index 1).
- **Edge cases:**
  - Forecast schema expects `forecastedDemand` etc.; if the agent returns malformed JSON, parser fails.
  - Using per-order data may not be sufficient for “forecast”; typically you’d aggregate.

#### Structured Output - Forecast
- **Type / role:** Structured output parser; validates/coerces model output into a schema example.
- **Schema example fields:** `forecastedDemand`, `confidenceLevel`, `recommendedStockLevel`, `trendDirection`, `riskFactors`
- **Connections:** Feeds parser into the forecasting agent.
- **Edge cases:** Example-based schema is permissive; still can fail if output is not JSON-like.

#### AI Agent - Sentiment Analysis
- **Type / role:** LangChain `agent`; analyzes review sentiment.
- **Prompt input:** `Review: {{ $json.reviewText }} / Rating: {{ $json.rating }}/5`
- **System message:** sentiment score (-1..1), classification, themes, urgency, actionRequired.
- **Output parsing:** `Structured Output - Sentiment`
- **Connections:** Input ← `Validate Reviews`; Output → `Merge Analysis Results` (input index 0)

#### Structured Output - Sentiment
- **Type / role:** Structured output parser; schema example includes:
  - `sentimentScore`, `classification`, `keyThemes[]`, `urgency`, `actionRequired`
- **Edge cases:** `actionRequired` is boolean in example, but later an IF compares it to string `"true"` (see exceptions block).

#### AI Agent - Product Recommendations
- **Type / role:** LangChain `agent`; generates top recommendations from customer order payload.
- **Prompt input:** `Customer Order: {{ JSON.stringify($json) }}`
- **Output parsing:** `Structured Output - Recommendations` (manual JSON schema)
- **Connections:** Input ← `Validate Orders`; Output → `Merge Analysis Results` (input index 2)

#### Structured Output - Recommendations
- **Type / role:** Structured output parser with **manual JSON Schema**:
  - `recommendations[]` (objects with `productId`, `productName`, `score`, `reasoning`)
  - `conversionProbability` number
  - `upsellOpportunities[]` string
- **Edge cases:** If model returns extra fields, parser behavior depends on node version; ensure compatibility with strict schema expectations.

---

### 1.4 Consolidation & deduplication
**Overview:** Combines outputs from the three AI agents and removes duplicate order/customer pairs before applying pricing logic.  
**Nodes involved:** `Merge Analysis Results`, `Remove Duplicate Records`

#### Merge Analysis Results
- **Type / role:** `Merge` with `numberInputs=3`; consolidates sentiment + forecast + recommendations.
- **Connections:**
  - Input 0 ← `AI Agent - Sentiment Analysis`
  - Input 1 ← `AI Agent - Demand Forecasting`
  - Input 2 ← `AI Agent - Product Recommendations`
  - Output → `Remove Duplicate Records`
- **Edge cases (important):**
  - These three streams originate from different triggers (`reviews` vs `orders`). Without a common merge key and synchronized batching, merges may misalign items or wait indefinitely for missing inputs (depending on merge mode defaults). Consider splitting into separate workflows or using keys + “Merge by Key” patterns.

#### Remove Duplicate Records
- **Type / role:** `Remove Duplicates`; dedupes by selected fields.
- **Configuration:** compare selected fields: `orderId, customerId`
- **Connections:** Output → `Apply Pricing Rules`
- **Edge cases:** If merged items don’t include `orderId/customerId` (e.g., sentiment-only), dedupe may behave unexpectedly.

---

### 1.5 Business rules: pricing → promotions → replenishment
**Overview:** Applies deterministic business logic in sequence using Code nodes.  
**Nodes involved:** `Apply Pricing Rules`, `Apply Promotion Rules`, `Apply Replenishment Rules`

#### Apply Pricing Rules
- **Type / role:** `Code` (JS); computes `finalPrice` based on demand/inventory/competitor and margin protection.
- **Key inputs expected in `$json`:**
  - `basePrice` or `price`
  - `demandForecast` (string `high/medium/low` or numeric 0..1)
  - `inventoryLevel`
  - `competitorPrice`, `minMargin`, `cost`
- **Key outputs added:**
  - `finalPrice`, `priceAdjustment`, `priceChangePercent`, `margin`, `adjustmentReason`, `pricingTimestamp`
- **Connections:** Output → `Apply Promotion Rules`
- **Edge cases:**
  - If `basePrice` is 0, percent calculations divide by zero (Infinity/NaN).
  - `demandForecast` from AI forecast node is named `forecastedDemand` in the parser example, not `demandForecast`; this mismatch means demand-based rules may not trigger unless upstream maps it.

#### Apply Promotion Rules
- **Type / role:** `Code` (JS); applies promotions by segment/category/seasonality/inventory.
- **Key inputs expected:**
  - `customerSegment`, `productCategory`, `inventoryLevel`, `price`, `purchaseHistory`
- **Outputs:**
  - `promotion` object (applied/type/discount/finalPrice/reasons/urgency)
  - `promotionApplied: true`, `processedAt`
- **Connections:** Output → `Apply Replenishment Rules`
- **Edge cases:**
  - Uses `productPrice = data.price || ... || 0`; if upstream only has `finalPrice`, promotions may compute against 0 unless mapped.

#### Apply Replenishment Rules
- **Type / role:** `Code` (JS); calculates reorder quantities, reorder point, urgency.
- **Key inputs expected:**
  - `currentStock`, `demandForecast` (monthly), `leadTimeDays`, `safetyStockLevel`, supplier constraints
- **Outputs:** `replenishment` object including `orderQuantity`, `shouldReorder`, `urgency`, `reorderPoint`, etc.
- **Connections:** Output → `Check for Exceptions`
- **Edge cases:**
  - If `demandForecast` is 0, `dailyDemand` is 0 and `daysUntilStockout = currentStock / 0` becomes Infinity; code guards only partially.
  - Forecast naming mismatch again (`forecastedDemand` vs `demandForecast`).

---

### 1.6 Exception handling & human-in-the-loop
**Overview:** Flags items requiring attention and optionally pauses execution awaiting manual approval before persisting to CRM.  
**Nodes involved:** `Check for Exceptions`, `Wait for Manual Review`

#### Check for Exceptions
- **Type / role:** `IF`; routes to manual review vs direct persistence.
- **Conditions (OR):**
  1. `currentStock < lowStockThreshold` (from Workflow Configuration)
  2. `sentimentScore < negativeSentimentThreshold`
  3. `actionRequired == "true"` (string compare)
- **Connections:**
  - **True** → `Wait for Manual Review`
  - **False** → `Store in CRM Database`
- **Edge cases:**
  - `actionRequired` is likely boolean; comparing to string `"true"` may fail. Prefer `={{ $json.actionRequired === true }}`.
  - `currentStock` might not exist on order/review derived items; condition may evaluate as `null < 10` (true/false depending on coercion).

#### Wait for Manual Review
- **Type / role:** `Wait`; pauses up to 24 units (node uses `resumeAmount: 24` with `limitWaitTime: true`).
- **Resume mechanism:** `webhook` resume (a generated resume URL).
- **Connections:** Output → `Store in CRM Database`
- **Edge cases:**
  - If resume webhook is never called, execution ends after wait limit.
  - Ensure the manual review system can call the resume webhook with correct execution context.

---

### 1.7 Persistence & customer engagement distribution (CRM + email)
**Overview:** Stores consolidated customer/order insights in a CRM table and triggers a personalized email campaign.  
**Nodes involved:** `Store in CRM Database`, `Prepare Campaign Data`, `Send Email Campaign`

#### Store in CRM Database
- **Type / role:** `Postgres`; inserts into a CRM table (table name placeholder).
- **Operation:** (not explicitly set; default in node is typically **insert**)
- **Columns mapped:**
  - `status, orderId, orderDate, customerId, totalAmount, sentimentScore, recommendations`
- **Connections:** Output → `Prepare Campaign Data`
- **Requirements:**
  - Postgres credentials must be configured in n8n (not included in JSON).
  - Table must exist and columns must match types (e.g., `recommendations` likely JSONB/text).
- **Edge cases:**
  - Placeholder table name must be replaced.
  - If `recommendations` is an array/object, Postgres insert may fail unless column is JSON/JSONB or you stringify.

#### Prepare Campaign Data
- **Type / role:** `Set`; prepares email fields.
- **Key fields:**
  - `customerEmail = $json.email || $json.customerEmail`
  - `customerName = $json.name || $json.customerName`
  - `recommendationsList = $json.recommendations`
  - `campaignId = {{$now.toFormat('yyyyMMdd')}}-personalized`
- **Connections:** Output → `Send Email Campaign`
- **Edge cases:**
  - Email node uses `{{$json.recommendations}}` directly in HTML but this node creates `recommendationsList` (name mismatch).
  - Ensure recommendations are formatted as HTML `<li>` items; otherwise the email renders raw JSON.

#### Send Email Campaign
- **Type / role:** `Email Send`; sends personalized email.
- **Configuration:**
  - To: `{{$json.customerEmail}}`
  - From: placeholder `<__PLACEHOLDER_VALUE__Sender email address__>`
  - Subject: “Personalized Recommendations Just for You”
  - HTML template references `{{$json.recommendations}}` and `{{$json.promotionalOffers}}`
- **Edge cases:**
  - `promotionalOffers` is never set in the workflow as provided.
  - Requires SMTP/email credentials configured in n8n for the Email node.

---

### 1.8 Inventory stream persistence & supply chain alerting
**Overview:** Stores inventory updates and posts an alert to a supply chain Slack channel.  
**Nodes involved:** `Store in Inventory Database`, `Notify Supply Chain Team`

#### Store in Inventory Database
- **Type / role:** `Postgres`; updates inventory records (by productId).
- **Operation:** `update`
- **Matching:** `matchingColumns: ["productId"]`
- **Columns mapped:** `productId, lastUpdated, currentStock, reorderPoint, forecastedDemand`
- **Connections:** Output → `Notify Supply Chain Team`
- **Edge cases:**
  - Placeholder table name must be replaced.
  - If `productId` doesn’t exist in table, update affects 0 rows (no upsert). Consider “insert or update”.
  - `forecastedDemand` may be missing from inventory payload unless provided externally.

#### Notify Supply Chain Team
- **Type / role:** `Slack` message (OAuth2).
- **Channel:** placeholder Slack channel ID.
- **Message template:** includes product name, stock, reorder point, forecasted demand.
- **Credentials:** Slack OAuth2 credential (“Slack account”).
- **Edge cases:** Missing `productName` (inventory normalization sets it, good). Slack API permission issues.

---

### 1.9 Social stream alerting (customer support)
**Overview:** Sends Slack notification for social mentions to support for monitoring/response.  
**Nodes involved:** `Notify Customer Support Team`

#### Notify Customer Support Team
- **Type / role:** `Slack` message (OAuth2).
- **Channel:** placeholder support channel ID.
- **Message template:** platform/author/content/engagement.
- **Connections:** Input ← `Validate Social Media`
- **Edge cases:** Slack throttling if high volume; consider batching or rate limiting.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook - Ingest Data | Webhook | Ingest external e-commerce events | — | Workflow Configuration | ## Multi-Channel Data Ingestion  \n**Why:** Webhook triggers capture real-time events from orders, reviews, inventory, and social platforms, ensuring immediate response to business-critical changes. |
| Workflow Configuration | Set | Central thresholds & tunables | Webhook - Ingest Data | Route by Data Type | ## Multi-Channel Data Ingestion  \n**Why:** Webhook triggers capture real-time events from orders, reviews, inventory, and social platforms, ensuring immediate response to business-critical changes. |
| Route by Data Type | Switch | Route payloads to the correct stream | Workflow Configuration | Normalize Orders; Normalize Reviews; Normalize Inventory; Normalize Social Media | ## Multi-Channel Data Ingestion  \n**Why:** Webhook triggers capture real-time events from orders, reviews, inventory, and social platforms, ensuring immediate response to business-critical changes. |
| Normalize Orders | Set | Standardize order fields | Route by Data Type | Validate Orders | ## Multi-Channel Data Ingestion  \n**Why:** Webhook triggers capture real-time events from orders, reviews, inventory, and social platforms, ensuring immediate response to business-critical changes. |
| Validate Orders | Filter | Gate invalid orders | Normalize Orders | AI Agent - Demand Forecasting; AI Agent - Product Recommendations | ## Multi-Channel Data Ingestion  \n**Why:** Webhook triggers capture real-time events from orders, reviews, inventory, and social platforms, ensuring immediate response to business-critical changes. |
| Normalize Reviews | Set | Standardize review fields | Route by Data Type | Validate Reviews | ## Multi-Channel Data Ingestion  \n**Why:** Webhook triggers capture real-time events from orders, reviews, inventory, and social platforms, ensuring immediate response to business-critical changes. |
| Validate Reviews | Filter | Gate invalid reviews | Normalize Reviews | AI Agent - Sentiment Analysis | ## Multi-Channel Data Ingestion  \n**Why:** Webhook triggers capture real-time events from orders, reviews, inventory, and social platforms, ensuring immediate response to business-critical changes. |
| Normalize Inventory | Set | Standardize inventory fields | Route by Data Type | Validate Inventory | ## Multi-Channel Data Ingestion  \n**Why:** Webhook triggers capture real-time events from orders, reviews, inventory, and social platforms, ensuring immediate response to business-critical changes. |
| Validate Inventory | Filter | Gate invalid inventory updates | Normalize Inventory | Store in Inventory Database | ## Multi-Channel Data Ingestion  \n**Why:** Webhook triggers capture real-time events from orders, reviews, inventory, and social platforms, ensuring immediate response to business-critical changes. |
| Normalize Social Media | Set | Standardize social mention fields | Route by Data Type | Validate Social Media | ## Multi-Channel Data Ingestion  \n**Why:** Webhook triggers capture real-time events from orders, reviews, inventory, and social platforms, ensuring immediate response to business-critical changes. |
| Validate Social Media | Filter | Gate invalid social mentions | Normalize Social Media | Notify Customer Support Team | ## Multi-Channel Data Ingestion  \n**Why:** Webhook triggers capture real-time events from orders, reviews, inventory, and social platforms, ensuring immediate response to business-critical changes. |
| OpenAI Chat Model | lmChatOpenAi | Shared LLM for AI agents | — | AI Agent - Sentiment Analysis; AI Agent - Demand Forecasting; AI Agent - Product Recommendations | ## AI-Powered Intelligence Analysis  \n**Why:** Parallel AI agents process data for sentiment analysis, pricing strategy, promotional targeting, and inventory forecasting, transforming raw data into actionable insights. |
| AI Agent - Sentiment Analysis | LangChain Agent | Review sentiment analysis | Validate Reviews (+ OpenAI model) | Merge Analysis Results | ## AI-Powered Intelligence Analysis  \n**Why:** Parallel AI agents process data for sentiment analysis, pricing strategy, promotional targeting, and inventory forecasting, transforming raw data into actionable insights. |
| Structured Output - Sentiment | Structured Output Parser | Enforce sentiment JSON structure | — (connected as agent output parser) | AI Agent - Sentiment Analysis | ## AI-Powered Intelligence Analysis  \n**Why:** Parallel AI agents process data for sentiment analysis, pricing strategy, promotional targeting, and inventory forecasting, transforming raw data into actionable insights. |
| AI Agent - Demand Forecasting | LangChain Agent | Demand forecasting from orders | Validate Orders (+ OpenAI model) | Merge Analysis Results | ## AI-Powered Intelligence Analysis  \n**Why:** Parallel AI agents process data for sentiment analysis, pricing strategy, promotional targeting, and inventory forecasting, transforming raw data into actionable insights. |
| Structured Output - Forecast | Structured Output Parser | Enforce forecast JSON structure | — (connected as agent output parser) | AI Agent - Demand Forecasting | ## AI-Powered Intelligence Analysis  \n**Why:** Parallel AI agents process data for sentiment analysis, pricing strategy, promotional targeting, and inventory forecasting, transforming raw data into actionable insights. |
| AI Agent - Product Recommendations | LangChain Agent | Personalized recommendations | Validate Orders (+ OpenAI model) | Merge Analysis Results | ## AI-Powered Intelligence Analysis  \n**Why:** Parallel AI agents process data for sentiment analysis, pricing strategy, promotional targeting, and inventory forecasting, transforming raw data into actionable insights. |
| Structured Output - Recommendations | Structured Output Parser | Enforce recommendations JSON schema | — (connected as agent output parser) | AI Agent - Product Recommendations | ## AI-Powered Intelligence Analysis  \n**Why:** Parallel AI agents process data for sentiment analysis, pricing strategy, promotional targeting, and inventory forecasting, transforming raw data into actionable insights. |
| Merge Analysis Results | Merge | Combine AI outputs | AI Agent - Sentiment Analysis; AI Agent - Demand Forecasting; AI Agent - Product Recommendations | Remove Duplicate Records | ## AI-Powered Intelligence Analysis  \n**Why:** Parallel AI agents process data for sentiment analysis, pricing strategy, promotional targeting, and inventory forecasting, transforming raw data into actionable insights. |
| Remove Duplicate Records | Remove Duplicates | Deduplicate by order/customer | Merge Analysis Results | Apply Pricing Rules | ## Automated Engagement Distribution  \n**Why:** Coordinated delivery through CRM, email campaigns, and team notifications ensures stakeholders receive relevant updates while customers experience personalized interactions. |
| Apply Pricing Rules | Code | Dynamic pricing computation | Remove Duplicate Records | Apply Promotion Rules | ## Automated Engagement Distribution  \n**Why:** Coordinated delivery through CRM, email campaigns, and team notifications ensures stakeholders receive relevant updates while customers experience personalized interactions. |
| Apply Promotion Rules | Code | Promotion selection & discounting | Apply Pricing Rules | Apply Replenishment Rules | ## Automated Engagement Distribution  \n**Why:** Coordinated delivery through CRM, email campaigns, and team notifications ensures stakeholders receive relevant updates while customers experience personalized interactions. |
| Apply Replenishment Rules | Code | Replenishment quantity/urgency | Apply Promotion Rules | Check for Exceptions | ## Automated Engagement Distribution  \n**Why:** Coordinated delivery through CRM, email campaigns, and team notifications ensures stakeholders receive relevant updates while customers experience personalized interactions. |
| Check for Exceptions | IF | Route to manual review vs auto-store | Apply Replenishment Rules | Wait for Manual Review; Store in CRM Database | ## Automated Engagement Distribution  \n**Why:** Coordinated delivery through CRM, email campaigns, and team notifications ensures stakeholders receive relevant updates while customers experience personalized interactions. |
| Wait for Manual Review | Wait | Human approval gate | Check for Exceptions (true) | Store in CRM Database | ## Automated Engagement Distribution  \n**Why:** Coordinated delivery through CRM, email campaigns, and team notifications ensures stakeholders receive relevant updates while customers experience personalized interactions. |
| Store in CRM Database | Postgres | Persist customer/order insights | Check for Exceptions (false) or Wait for Manual Review | Prepare Campaign Data | ## Automated Engagement Distribution  \n**Why:** Coordinated delivery through CRM, email campaigns, and team notifications ensures stakeholders receive relevant updates while customers experience personalized interactions. |
| Prepare Campaign Data | Set | Shape fields for outbound email | Store in CRM Database | Send Email Campaign | ## Automated Engagement Distribution  \n**Why:** Coordinated delivery through CRM, email campaigns, and team notifications ensures stakeholders receive relevant updates while customers experience personalized interactions. |
| Send Email Campaign | Email Send | Send personalized email | Prepare Campaign Data | — | ## Automated Engagement Distribution  \n**Why:** Coordinated delivery through CRM, email campaigns, and team notifications ensures stakeholders receive relevant updates while customers experience personalized interactions. |
| Store in Inventory Database | Postgres | Update inventory table | Validate Inventory | Notify Supply Chain Team |  |
| Notify Supply Chain Team | Slack | Alert supply chain on inventory | Store in Inventory Database | — |  |
| Notify Customer Support Team | Slack | Alert support on social mentions | Validate Social Media | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create workflow**
   1. New workflow name: *E-commerce Intelligence & Automated Customer Engagement Platform* (or your preferred name).

2) **Add trigger: Webhook**
   1. Add node: **Webhook**
   2. Set **HTTP Method**: `POST`
   3. Set **Path**: `ecommerce-data`
   4. Set **Response Mode**: `Last Node`
   5. Set **Response Data**: (match original) `First entry binary`  
      - If you expect JSON responses, change this to a JSON response mode instead.

3) **Add configuration constants**
   1. Add node: **Set** named `Workflow Configuration`
   2. Add numeric fields:
      - `lowStockThreshold` = 10
      - `negativeSentimentThreshold` = -0.5
      - `highDemandThreshold` = 100
      - `apiRateLimit` = 100
      - `batchSize` = 50
      - `retryAttempts` = 3
      - `retryDelay` = 5000
   3. Enable “Include Other Fields”.
   4. Connect: `Webhook - Ingest Data` → `Workflow Configuration`

4) **Route by payload type**
   1. Add node: **Switch** named `Route by Data Type`
   2. Add 4 rules (string equals) on `={{ $json.dataType }}`:
      - `"order"` → output name `orders`
      - `"review"` → output name `reviews`
      - `"inventory"` → output name `inventory`
      - `"social_media"` → output name `social`
   3. Connect: `Workflow Configuration` → `Route by Data Type`

5) **Orders stream: normalize + validate**
   1. Add **Set** `Normalize Orders` with fields:
      - `orderId` (string) `={{ $json.id || $json.order_id }}`
      - `customerId` (string) `={{ $json.customer_id || $json.customerId }}`
      - `orderDate` (string) `={{ $json.created_at || $json.orderDate || $now }}`
      - `totalAmount` (number) `={{ $json.total || $json.amount || 0 }}`
      - `status` (string) `={{ $json.status || 'pending' }}`
      - `products` (array) `={{ $json.items || $json.products || [] }}`
      - `dataType` (string) `order`
      - Include other fields = true
   2. Add **Filter** `Validate Orders` with conditions:
      - `orderId` not empty
      - `customerId` not empty
      - `totalAmount` > 0
   3. Connect: `Route by Data Type (orders)` → `Normalize Orders` → `Validate Orders`

6) **Reviews stream: normalize + validate**
   1. Add **Set** `Normalize Reviews`:
      - `reviewId` `={{ $json.id || $json.review_id }}`
      - `productId` `={{ $json.product_id || $json.productId }}`
      - `customerId` `={{ $json.customer_id || $json.customerId }}`
      - `rating` (number) `={{ $json.rating || $json.stars || 0 }}`
      - `reviewText` `={{ $json.text || $json.comment || $json.review || '' }}`
      - `reviewDate` `={{ $json.created_at || $json.date || $now }}`
      - `dataType` = `review`
   2. Add **Filter** `Validate Reviews`:
      - `reviewId` not empty
      - `productId` not empty
      - `reviewText` not empty
   3. Connect: `Route by Data Type (reviews)` → `Normalize Reviews` → `Validate Reviews`

7) **Inventory stream: normalize + validate**
   1. Add **Set** `Normalize Inventory`:
      - `productId` `={{ $json.id || $json.product_id || $json.sku }}`
      - `productName` `={{ $json.name || $json.product_name || '' }}`
      - `currentStock` (number) `={{ $json.stock || $json.quantity || 0 }}`
      - `reorderPoint` (number) `={{ $json.reorder_point || $json.minStock || 20 }}`
      - `lastUpdated` `={{ $json.updated_at || $now }}`
      - `warehouseLocation` `={{ $json.location || $json.warehouse || 'default' }}`
      - `dataType` = `inventory`
   2. Add **Filter** `Validate Inventory`:
      - `productId` not empty
      - `currentStock >= 0`
   3. Connect: `Route by Data Type (inventory)` → `Normalize Inventory` → `Validate Inventory`

8) **Social stream: normalize + validate**
   1. Add **Set** `Normalize Social Media`:
      - `postId` `={{ $json.id || $json.post_id }}`
      - `platform` `={{ $json.platform || $json.source || 'unknown' }}`
      - `content` `={{ $json.text || $json.message || $json.content || '' }}`
      - `author` `={{ $json.author || $json.username || $json.user || '' }}`
      - `timestamp` `={{ $json.created_at || $json.timestamp || $now }}`
      - `engagement` (number) `={{ $json.likes + $json.shares + $json.comments || 0 }}`
      - `dataType` = `social_media`
   2. Add **Filter** `Validate Social Media`:
      - `postId` not empty
      - `content` not empty
   3. Connect: `Route by Data Type (social)` → `Normalize Social Media` → `Validate Social Media`

9) **Set up OpenAI model credential**
   1. Create credentials: **OpenAI API**
   2. Add node: **OpenAI Chat Model** (`lmChatOpenAi`)
   3. Select model: `gpt-4.1-mini`
   4. Attach OpenAI credentials.

10) **AI agents + structured parsers**
   1. Add **Structured Output Parser** nodes:
      - `Structured Output - Forecast` with the forecast JSON example
      - `Structured Output - Sentiment` with the sentiment JSON example
      - `Structured Output - Recommendations` with the provided manual JSON schema
   2. Add **AI Agent** nodes:
      - `AI Agent - Demand Forecasting` (text: `Order Data: {{ JSON.stringify($json) }}`; system message as provided; enable output parser)
      - `AI Agent - Sentiment Analysis` (text: review/rating; system message as provided; enable output parser)
      - `AI Agent - Product Recommendations` (text: customer order; system message as provided; enable output parser)
   3. Connect **OpenAI Chat Model** to each agent via the **AI Language Model** connection.
   4. Connect each structured output parser to its corresponding agent via the **AI Output Parser** connection.
   5. Connect execution:
      - `Validate Orders` → `AI Agent - Demand Forecasting`
      - `Validate Orders` → `AI Agent - Product Recommendations`
      - `Validate Reviews` → `AI Agent - Sentiment Analysis`

11) **Merge AI outputs**
   1. Add **Merge** node `Merge Analysis Results` with `Number of Inputs = 3`
   2. Connect:
      - Sentiment agent → merge input 0
      - Forecast agent → merge input 1
      - Recommendations agent → merge input 2

12) **Deduplicate**
   1. Add **Remove Duplicates** node `Remove Duplicate Records`
   2. Compare: `Selected fields`
   3. Fields: `orderId, customerId`
   4. Connect: `Merge Analysis Results` → `Remove Duplicate Records`

13) **Pricing / promotion / replenishment code nodes**
   1. Add **Code** node `Apply Pricing Rules` and paste pricing JS (as in workflow).
   2. Add **Code** node `Apply Promotion Rules` and paste promotion JS.
   3. Add **Code** node `Apply Replenishment Rules` and paste replenishment JS.
   4. Connect: `Remove Duplicate Records` → `Apply Pricing Rules` → `Apply Promotion Rules` → `Apply Replenishment Rules`

14) **Exception routing + manual review**
   1. Add **IF** node `Check for Exceptions`
   2. Add OR conditions:
      - `={{ $json.currentStock }}` `<` `={{ $('Workflow Configuration').first().json.lowStockThreshold }}`
      - `={{ $json.sentimentScore }}` `<` `={{ $('Workflow Configuration').first().json.negativeSentimentThreshold }}`
      - `={{ $json.actionRequired }}` equals `"true"` (replicate original; recommended to switch to boolean)
   3. Add **Wait** node `Wait for Manual Review`
      - Resume: `webhook`
      - Limit wait time: enabled
      - Resume amount: 24 (configure unit per UI)
   4. Connect:
      - `Apply Replenishment Rules` → `Check for Exceptions`
      - `Check for Exceptions (true)` → `Wait for Manual Review` → `Store in CRM Database`
      - `Check for Exceptions (false)` → `Store in CRM Database`

15) **Postgres: CRM storage**
   1. Create **Postgres** credentials in n8n.
   2. Add **Postgres** node `Store in CRM Database`
   3. Set schema: `public`
   4. Set table: your CRM table name (replace placeholder)
   5. Map columns:
      - `status`, `orderId`, `orderDate`, `customerId`, `totalAmount`, `sentimentScore`, `recommendations`
   6. Connect to `Prepare Campaign Data`.

16) **Email campaign**
   1. Add **Set** node `Prepare Campaign Data`:
      - `customerEmail`, `customerName`, `recommendationsList`, `campaignId` as per workflow
   2. Add **Email Send** node `Send Email Campaign`
      - Configure email credentials (SMTP/Gmail/SendGrid depending on your n8n setup)
      - `toEmail = {{$json.customerEmail}}`
      - `fromEmail` replace placeholder
      - Subject + HTML as provided
   3. Connect: `Store in CRM Database` → `Prepare Campaign Data` → `Send Email Campaign`

17) **Postgres: Inventory storage + Slack supply chain**
   1. Add **Postgres** node `Store in Inventory Database`
      - Operation: `update`
      - Matching column: `productId`
      - Map: `productId, lastUpdated, currentStock, reorderPoint, forecastedDemand`
      - Replace placeholder table name
   2. Add **Slack** node `Notify Supply Chain Team`
      - OAuth2 Slack credentials
      - Channel ID: replace placeholder
      - Text template as provided
   3. Connect: `Validate Inventory` → `Store in Inventory Database` → `Notify Supply Chain Team`

18) **Slack: customer support for social**
   1. Add **Slack** node `Notify Customer Support Team`
      - OAuth2 Slack credentials
      - Channel ID: replace placeholder
      - Text template as provided
   2. Connect: `Validate Social Media` → `Notify Customer Support Team`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “How It Works” sticky note describing end-to-end automation and benefits | In-workflow note: explains parallel streams + AI analysis + Slack/email distribution |
| “Setup Steps” sticky note listing configuration tasks (webhooks, AI creds, CRM DB, email provider, Slack/Teams) | In-workflow note |
| “Prerequisites / Use Cases / Customization / Benefits” sticky note | In-workflow note (claims like “reduces overhead by 70%” are informational, not enforced by the workflow) |
| “AI-Powered Intelligence Analysis” sticky note | In-workflow note |
| “Automated Engagement Distribution” sticky note | In-workflow note |

