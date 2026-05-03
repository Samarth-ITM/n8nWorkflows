Automate social fan-out, marketplace outreach, and CRM alerts with Salesforce

https://n8nworkflows.xyz/workflows/automate-social-fan-out--marketplace-outreach--and-crm-alerts-with-salesforce-14984


# Automate social fan-out, marketplace outreach, and CRM alerts with Salesforce

Let me analyze this n8n workflow thoroughly. It's a satirical "Hell Yeah!" workflow that contains useful patterns underneath the jokes. Let me map out all nodes, their connections, and group them into logical blocks.

Let me trace through the workflow:

1. **Rise and Grind** (Cron trigger) → Multiple outputs:
   - Beer & Cig Farmer
   - FACEBOOK1
   - INSTAGRAM1
   - THREADS1
   - TIKTOK1
   - LINKEDIN1
   - TWITTER1
   - Wait2

2. **Daily Lifestyle Routine** (satirical branch):
   - Beer & Cig Farmer → Wait (20 min) → Uber Black to Casino, Bet365 API
   - Uber Black to Casino → Wait1 (35 min), FB Marketplace scraper
   - Wait1 → Tinder API → Wait3 (1 hour) → Steak & Monster Delivery → Book All Available Slots
   - Book All Available Slots → Find Facebook Group, Search eBay Deals
   - Wait2 → Play Man in the box (Spotify)

3. **Gambling → Flex Chain** (Arbitrage):
   - Bet365 API → Arbitrage Calculator → Flex Wins on Socials → INSTAGRAM, TIKTOK, TWITTER, LINKEDIN, THREADS, FACEBOOK

4. **Facebook Marketplace Lowball Engine**:
   - FB Marketplace scraper → Data Normalizer → Item filter → Lowball Algorithm → Flaw Detection → Message Blaster1 → Deal Analyzer → Relentless Negotiation Engine → Follow-up Nuke → Opportunity Recalculator → CRM Integration
   - CRM Integration → Deal Alert (Telegram), Value Calculator → Execute Purchase

5. **eBay Arbitrage Loop**:
   - Search eBay Deals → Find Profitable Items → Item Lists → Get Item Details → List Item for Sale → Log Transaction (Google Sheets)

6. **Facebook Group Engagement**:
   - Find Facebook Group → Filter Recent Discussions → Send WhatsApp Alert

Now let me organize this into logical blocks and document each node.

Let me also trace the connections more carefully from the JSON:

Connections:
- Wait → [Uber Black to Casino, Bet365 API]
- Wait1 → [Tinder API]
- Wait2 → [Play Man in the box]
- Wait3 → [Steak & Monster Delivery]
- Bet365 API → [Arbitrage Calculator]
- Item Lists → [Get Item Details]
- Tinder API → [Wait3]
- Item filter → [Lowball Algorithm]
- Deal Analyzer → [Relentless Negotiation Engine]
- Flaw Detection → [Message Blaster1]
- Follow-up Nuke → [Opportunity Recalculator]
- Rise and Grind → [Beer & Cig Farmer, FACEBOOK1, INSTAGRAM1, THREADS1, TIKTOK1, LINKEDIN1, TWITTER1, Wait2]
- CRM Integration → [Deal Alert, Value Calculator]
- Data Normalizer → [Item filter]
- Get Item Details → [List Item for Sale]
- Message Blaster1 → [Deal Analyzer]
- Value Calculator → [Execute Purchase]
- Beer & Cig Farmer → [Wait]
- Lowball Algorithm → [Flaw Detection]
- Search eBay Deals → [Find Profitable Items]
- List Item for Sale → [Log Transaction]
- Find Facebook Group → [Filter Recent Discussions]
- Arbitrage Calculator → [Flex Wins on Socials]
- Flex Wins on Socials → [INSTAGRAM, TIKTOK, TWITTER, LINKEDIN, THREADS, FACEBOOK]
- Uber Black to Casino → [Wait1, FB Marketplace scraper]
- Find Profitable Items → [Item Lists]
- Book All Available Slots → [Find Facebook Group, Search eBay Deals]
- Steak & Monster Delivery → [Book All Available Slots]
- Filter Recent Discussions → [Send WhatsApp Alert]
- Relentless Negotiation Engine → [Follow-up Nuke]

Now let me create a comprehensive document.

Block structure:
1. **Daily Trigger & Lifestyle Satire Chain** - Cron trigger through satirical lifestyle nodes
2. **Multi-Channel Social Fan-Out (Daily Posts)** - The six social media posting nodes triggered by Rise and Grind
3. **Gambling → Arbitrage → Social Flex** - Bet365 + Arbitrage + Social blast
4. **Facebook Marketplace Lowball Engine** - Scraper through CRM
5. **eBay Arbitrage Loop** - eBay search through Google Sheets logging
6. **Facebook Group Engagement & WhatsApp Alerts** - Group discovery through WhatsApp
7. **Satirical Lifestyle Branches** - Tinder, Uber, Steak, Spotify

Let me write this up properly. 1. Workflow Overview

This workflow is a satirical "Hell Yeah!" automation that parodies hustle-culture "automate your whole life" content while embedding genuinely reusable n8n patterns. Underneath the jokes, it demonstrates five production-grade skeletons:

- **Cron-triggered daily chain** – A scheduled trigger fans out to multiple parallel branches every morning.
- **Multi-channel social fan-out** – A single event posts content simultaneously to Instagram, TikTok, Facebook, Threads, LinkedIn, and Twitter via the Blotato API.
- **Marketplace scraper → filter → outreach → nurture** – A Facebook Marketplace scraping pipeline that normalizes listings, filters by price and recency, generates lowball offers with contextual excuses, sends messages, analyses deal probability, and loops through a follow-up negotiation engine before syncing to Salesforce CRM and alerting via Telegram.
- **eBay arbitrage loop** – Searches eBay for underpriced items, evaluates profitability, fetches details, relists them, and logs every transaction to Google Sheets.
- **CRM-sync-with-chat-notification** – After CRM upsert, a Telegram alert fires and a value calculator aggregates deal metrics.

The satirical branches (beer delivery, Uber to casino, Tinder auto-booking, steak delivery, Spotify playback) exist for comedic effect and can be deleted or replaced with real integrations.

---

### 2. Block-by-Block Analysis

---

#### Block 1 – Daily Trigger & Lifestyle Satire Chain

**Overview:** A cron node ("Rise and Grind") fires each morning and dispatches execution to eight parallel branches: six social-media posts, a satirical lifestyle chain (beer → wait → Uber → wait → Tinder → wait → steak → booking agent), and a Spotify music trigger.

**Nodes Involved:** Rise and Grind, Beer & Cig Farmer, Wait, Uber Black to Casino, Wait1, Tinder API, Wait3, Steak & Monster Delivery, Book All Available Slots, Wait2, Play Man in the box

| Node | Type | Configuration | Key Expressions / Variables | Input | Output | Edge Cases / Notes |
|---|---|---|---|---|---|---|
| **Rise and Grind** | Cron | Default cron schedule (no custom expression set) | None | None (trigger) | Beer & Cig Farmer, INSTAGRAM1, TIKTOK1, FACEBOOK1, THREADS1, LINKEDIN1, TWITTER1, Wait2 | No schedule configured – will default to n8n server timezone. Set a specific cron expression before activating. |
| **Beer & Cig Farmer** | HTTP Request | `GET https://api.doordash.com/v2/orders` | None | Rise and Grind | Wait | Fictional DoorDash endpoint. Replace with real food-delivery API or remove. Auth not configured. |
| **Wait** | Wait | 20 minutes | `amount: 20`, `unit: minutes` | Beer & Cig Farmer | Uber Black to Casino, Bet365 API | Uses webhook-based wait; the workflow will pause and resume. Ensure n8n instance allows webhook callbacks. |
| **Uber Black to Casino** | HTTP Request | `POST https://api.uber.com/v1/requests` with `genericCredentialType` auth | None | Wait | Wait1, FB Marketplace scraper | Fictional Uber endpoint. Generic credential type selected but no credential bound. Replace or remove. |
| **Wait1** | Wait | 35 minutes | `amount: 35`, `unit: minutes` | Uber Black to Casino | Tinder API | Same webhook-based wait mechanism. |
| **Tinder API** | HTTP Request | `GET https://www.tinder.com/api` | None | Wait1 | Wait3 | Fictional Tinder endpoint. No auth. Replace or delete. |
| **Wait3** | Wait | 1 hour | `unit: hours`, default amount 1 | Tinder API | Steak & Monster Delivery | Webhook wait. |
| **Steak & Monster Delivery** | HTTP Request | `GET https://api.doordash.com/v2/orders` | None | Wait3 | Book All Available Slots | Fictional DoorDash endpoint, same as Beer & Cig Farmer. |
| **Book All Available Slots** | HTTP Request | `GET https://api.rangebooking.com/v1/reservations` | None | Steak & Monster Delivery | Find Facebook Group, Search eBay Deals | Fictional range-booking API. This is the branching point from satire into real patterns. |
| **Wait2** | Wait | Default (1 unit, minutes) | None | Rise and Grind | Play Man in the box | Minimal wait. |
| **Play Man in the box** | HTTP Request | `PUT https://api.spotify.com/v1/me/player/play` | None | Wait2 | None (terminal) | Fictional Spotify endpoint. No auth bound. Replace or delete. |

---

#### Block 2 – Multi-Channel Social Fan-Out (Daily Posts)

**Overview:** Six parallel HTTP Request nodes post content (video + caption) to Instagram, TikTok, Facebook, Threads, LinkedIn, and Twitter via the Blotato multi-platform publishing API. All are triggered directly by the cron node.

**Nodes Involved:** INSTAGRAM1, TIKTOK1, FACEBOOK1, THREADS1, LINKEDIN1, TWITTER1

| Node | Type | Configuration | Key Expressions | Input | Output | Edge Cases / Notes |
|---|---|---|---|---|---|---|
| **INSTAGRAM1** | HTTP Request (v4.2) | `POST https://backend.blotato.com/v2/posts`, JSON body with `blotato-api-key` header | `accountId` from `Assign Social Media IDs.instagram_id`; `text` from `Get my video.DESCRIPTION`; `mediaUrls` from `$json.url`; `scheduledTime` from `Log Shorts & Schedule Info to Google Sheets.Schedule Time` | Rise and Grind | None (terminal) | Requires Blotato API key. Referenced nodes (Assign Social Media IDs, Get my video, Log Shorts & Schedule Info to Google Sheets) are not present in this workflow – they must be created or this node rewired. |
| **TIKTOK1** | HTTP Request (v4.2) | `POST https://backend.blotato.com/v2/posts`, JSON body with TikTok-specific target fields (privacy, duet, stitch, AI-generated flags) | `accountId` from `Assign Social Media IDs.tiktok_id`; `text` from `Log Shorts & Schedule Info to Google Sheets.Clip Caption`; `mediaUrls` from `$json.url` | Rise and Grind | None (terminal) | Same Blotato dependency. TikTok requires boolean-style string fields (e.g. `isAiGenerated`). |
| **FACEBOOK1** | HTTP Request (v4.2) | `POST https://backend.blotato.com/v2/posts`, JSON body with `pageId` | `accountId` from `Assign Social Media IDs.facebook_id`; `pageId` from `Assign Social Media IDs.facebook_page_id`; `text` from `Get my video.DESCRIPTION` | Rise and Grind | None (terminal) | Facebook posting requires both account and page IDs. |
| **THREADS1** | HTTP Request (v4.2) | `POST https://backend.blotato.com/v2/posts` | `accountId` from `Assign Social Media IDs.threads_id`; `text` from `Get my video.DESCRIPTION` | Rise and Grind | None (terminal) | Threads platform via Blotato. |
| **LINKEDIN1** | HTTP Request (v4.2) | `POST https://backend.blotato.com/v2/posts` | `accountId` from `Assign Social Media IDs.linkedin_id`; `text` from `Get my video.DESCRIPTION` | Rise and Grind | None (terminal) | LinkedIn via Blotato. No media URLs referenced (text-only or image attached upstream). |
| **TWITTER1** | HTTP Request (v4.2) | `POST https://backend.blotato.com/v2/posts` | `accountId` from `Assign Social Media IDs.twitter_id`; `text` from `Get my video.DESCRIPTION`; `mediaUrls` from `$json.url` | Rise and Grind | None (terminal) | Twitter via Blotato. |

---

#### Block 3 – Gambling → Arbitrage → Social Flex Chain

**Overview:** After the initial 20-minute wait, a fake Bet365 API fetches sports odds. The Arbitrage Calculator function node identifies arbitrage opportunities (>2% profit), and the results are posted to six social channels (second fan-out, nodes without the "1" suffix) via Blotato.

**Nodes Involved:** Bet365 API, Arbitrage Calculator, Flex Wins on Socials, INSTAGRAM, TIKTOK, TWITTER, LINKEDIN, THREADS, FACEBOOK

| Node | Type | Configuration | Key Expressions | Input | Output | Edge Cases / Notes |
|---|---|---|---|---|---|---|
| **Bet365 API** | HTTP Request | `GET https://api.bet365.com` | None | Wait | Arbitrage Calculator | Fictional endpoint. No auth. Replace with real odds provider. |
| **Arbitrage Calculator** | Function | Calculates implied probabilities from two sets of odds, identifies arbitrage opportunities with >2% profit, returns one item per opportunity with game name, profit %, and stake recommendations | Uses `$input.first().json` for site1 odds and `$input.all()[1].json` for site2 odds; calculates `impliedProb = 1/odds`, `totalImplied`, `profit = ((1/totalImplied)-1)*100` | Bet365 API (and a second implied input source) | Flex Wins on Socials | Expects two inputs (two odds sources). Currently only Bet365 API feeds it. A second odds source node is missing and must be added for the arbitrage logic to work. |
| **Flex Wins on Socials** | HTTP Request | `POST https://api.stocktwits.com/v2/messages` | None | Arbitrage Calculator | INSTAGRAM, TIKTOK, TWITTER, LINKEDIN, THREADS, FACEBOOK | Fictional Stocktwits API. In a real implementation, this would be a transformation step before the fan-out. |
| **INSTAGRAM** | HTTP Request (v4.2) | `POST https://backend.blotato.com/v2/posts` with JSON body | `accountId` from `Assign Social Media IDs.instagram_id`; `text` from `Get my video.DESCRIPTION`; `mediaUrls` from `$json.url`; `scheduledTime` from `Log Shorts & Schedule Info to Google Sheets.Schedule Time` | Flex Wins on Socials | None (terminal) | Same Blotato dependency as INSTAGRAM1. |
| **TIKTOK** | HTTP Request (v4.2) | Same pattern as TIKTOK1 with TikTok-specific target fields | `accountId` from `Assign Social Media IDs.tiktok_id`; `text` from `Log Shorts & Schedule Info to Google Sheets.Clip Caption` | Flex Wins on Socials | None (terminal) | Same notes as TIKTOK1. |
| **TWITTER** | HTTP Request (v4.2) | Same pattern as TWITTER1 | `accountId` from `Assign Social Media IDs.twitter_id`; `text` from `Get my video.DESCRIPTION` | Flex Wins on Socials | None (terminal) | Same notes as TWITTER1. |
| **LINKEDIN** | HTTP Request (v4.2) | Same pattern as LINKEDIN1 | `accountId` from `Assign Social Media IDs.linkedin_id` | Flex Wins on Socials | None (terminal) | Same notes as LINKEDIN1. |
| **THREADS** | HTTP Request (v4.2) | Same pattern as THREADS1 | `accountId` from `Assign Social Media IDs.threads_id` | Flex Wins on Socials | None (terminal) | Same notes as THREADS1. |
| **FACEBOOK** | HTTP Request (v4.2) | Same pattern as FACEBOOK1 | `accountId` from `Assign Social Media IDs.facebook_id`; `pageId` from `Assign Social Media IDs.facebook_page_id` | Flex Wins on Socials | None (terminal) | Same notes as FACEBOOK1. |

---

#### Block 4 – Facebook Marketplace Lowball Engine (Scraper → Filter → Outreach → Nurture → CRM)

**Overview:** After the Uber Black node, the workflow scrapes Facebook Marketplace, normalizes data, filters items, runs a lowball algorithm with contextual excuses, sends messages, analyses deal probability, runs a relentless negotiation loop, recalculates offers, syncs to Salesforce, alerts via Telegram, and calculates aggregate value.

**Nodes Involved:** FB Marketplace scraper, Data Normalizer, Item filter, Lowball Algorithm, Flaw Detection, Message Blaster1, Deal Analyzer, Relentless Negotiation Engine, Follow-up Nuke, Opportunity Recalculator, CRM Integration, Deal Alert, Value Calculator, Execute Purchase

| Node | Type | Configuration | Key Expressions | Input | Output | Edge Cases / Notes |
|---|---|---|---|---|---|---|
| **FB Marketplace scraper** | Facebook Graph API | Edge and node set to `=` (placeholder); graphApiVersion `=` (placeholder) | None | Uber Black to Casino | Data Normalizer | Requires Facebook Graph API credential with marketplace/pages permissions. Current parameters are placeholder values (`=`). Must configure actual edge/node for marketplace listings. |
| **Data Normalizer** | Function | Returns hardcoded sample data (two items: iPhone 13 Pro and Gaming PC RTX 3080) as a stand-in for real normalized scrape output | Returns static objects with `id`, `title`, `price`, `description`, `seller`, `seller_id`, `image_url`, `location`, `posted` | FB Marketplace scraper | Item filter | Currently returns hardcoded stub data. Replace with normalization logic that maps real Facebook Graph API response fields. |
| **Item filter** | IF | Boolean conditions: `price` larger (no comparison value set), `posted` contains (no comparison value set) | `{{$json['price']}}` and `{{$json['posted']}}` | Data Normalizer | Lowball Algorithm | Conditions are incomplete (missing right-hand values). Configure actual threshold (e.g., price > 100, posted contains "hour" or "minute") before use. |
| **Lowball Algorithm** | Function | Determines strategy (aggressive/moderate/soft) based on item type demand mapping; calculates offer as percentage of price | `strategies` map (aggressive: 15%, moderate: 25%, soft: 35%); `marketDemand` map (iPhone: 0.85, Gaming: 0.65, etc.); detects item type from title; returns `lowballOffer`, `offerStrategy`, `marketDemand`, `opportunityValue` | Item filter | Flaw Detection | Logic is illustrative. Adjust percentages and demand keys for real use. |
| **Flaw Detection** | Function | Generates a contextual excuse message randomly selected from five templates and composes an outreach message | Random excuse selection via `Math.floor(Math.random() * excuses.length)`; message template: `Hi {seller}! I'm very interested in your {title}. Would you consider ${lowballOffer}? {excuse}` | Lowball Algorithm | Message Blaster1 | Excuses are satirical. Replace with professional outreach copy for production. |
| **Message Blaster1** | Facebook Graph API | Node and graphApiVersion set to `=` (placeholder) | None | Flaw Detection | Deal Analyzer | Requires Facebook Graph API credential with messaging permissions. Configure actual message-send endpoint (e.g., `/conversations` edge). |
| **Deal Analyzer** | Function | Calculates `successProbability` based on `marketDemand` and `offerStrategy`; sets initial negotiation stage and last offer | `successProbability = floor(marketDemand * 100) - (aggressive: 35, moderate: 20, soft: 10)`, clamped to minimum 5 | Message Blaster1 | Relentless Negotiation Engine | Simple heuristic. Enhance with real probability models for production. |
| **Relentless Negotiation Engine** | IF | Boolean condition: `successProbability` smaller (no comparison value) | `{{$json['successProbability']}}` | Deal Analyzer | Follow-up Nuke | Condition incomplete. Set a threshold (e.g., `successProbability < 50`) to determine whether to follow up. |
| **Follow-up Nuke** | Facebook Graph API | Edge and node `=`, graphApiVersion `=` (placeholder) | None | Relentless Negotiation Engine | Opportunity Recalculator | Same Facebook credential requirement. Configure actual follow-up message endpoint. |
| **Opportunity Recalculator** | Function | Reduces the last offer by 15%, updates stage to "Nuclear Offer", recalculates opportunity value and bumps success probability by 15 (capped at 90) | `newOffer = Math.round($json.lastOffer * 0.85)` | Follow-up Nuke | CRM Integration | Demonstrates an automated escalation loop. Tune discount rate for real negotiations. |
| **CRM Integration** | Salesforce | Operation: create contact (default); company and lastname set to `=` (placeholder); additionalFields empty | None | Opportunity Recalculator | Deal Alert, Value Calculator | Requires Salesforce OAuth2 credential. Configure `company`, `lastname`, and any additional fields with real data mappings. |
| **Deal Alert** | Telegram | Send message; text and chatId set to `=` (placeholder) | None | CRM Integration | None (terminal) | Requires Telegram Bot API credential. Set `chatId` and `text` with dynamic expressions referencing the deal data. |
| **Value Calculator** | Function | Aggregates `totalOpportunityValue`, `averageDiscount`, and `totalOffers` across all items | `items.reduce(...)` over all input items | CRM Integration | Execute Purchase | Works on the aggregate item set. Ensure all items reach this node for accurate totals. |
| **Execute Purchase** | Function | Generates a formatted summary report string | Template literal with emoji and data from `$json` | Value Calculator | None (terminal) | Purely informational output. Connect to a real purchase/payment API for production use. |

---

#### Block 5 – eBay Arbitrage Loop

**Overview:** After the satirical "Book All Available Slots" node, the workflow searches eBay for deals, filters profitable items, iterates over them, fetches details, relists them, and logs transactions to Google Sheets.

**Nodes Involved:** Search eBay Deals, Find Profitable Items, Item Lists, Get Item Details, List Item for Sale, Log Transaction

| Node | Type | Configuration | Key Expressions | Input | Output | Edge Cases / Notes |
|---|---|---|---|---|---|---|
| **Search eBay Deals** | HTTP Request | `GET https://svcs.ebay.com/services/search/FindingService/v1` | None | Book All Available Slots | Find Profitable Items | Fictional eBay Finding API URL. Must add query parameters (keywords, pagination) and eBay App ID via header. |
| **Find Profitable Items** | Function | Parses eBay Finding API response, calculates `productValue` (placeholder: `productId * 3` or `currentPrice * 4`), filters items where `productValue > currentPrice * 3`, returns array of deals | `$json["findItemsByKeywordsResponse"][0]["searchResult"][0]["item"]` | Search eBay Deals | Item Lists | Relies on specific eBay response structure. The product value formula is a placeholder – replace with real margin calculation. |
| **Item Lists** | Item Lists | Splits items by field `Tactical Gear` (placeholder field name) | None | Find Profitable Items | Get Item Details | The `fieldToSplitOut` value "Tactical Gear" is a stub. Set it to the actual array field containing deal items (e.g., `deals`). |
| **Get Item Details** | HTTP Request | `GET https://api.ebay.com/buy/browse/v1/item/{{$json["itemId"]}}` | `{{$json["itemId"]}}` from each split item | Item Lists | List Item for Sale | Requires eBay OAuth2 credential and proper scopes (Browse API). |
| **List Item for Sale** | HTTP Request | `POST https://api.ebay.com/sell/inventory/v1/offer` with `jsonParameters: true` | None | Get Item Details | Log Transaction | Requires eBay Sell API credential. Must construct offer payload from item details. |
| **Log Transaction** | Google Sheets | Operation: append; authentication: OAuth2; sheetId `=` (placeholder) | None | List Item for Sale | None (terminal) | Requires Google Sheets OAuth2 credential. Set real `sheetId` and column mappings. |

---

#### Block 6 – Facebook Group Engagement & WhatsApp Alerts

**Overview:** After the booking node, the workflow searches for a Facebook Group, filters recent discussions from the past 7 days, and sends a WhatsApp alert with the results.

**Nodes Involved:** Find Facebook Group, Filter Recent Discussions, Send WhatsApp Alert

| Node | Type | Configuration | Key Expressions | Input | Output | Edge Cases / Notes |
|---|---|---|---|---|---|---|
| **Find Facebook Group** | Facebook Graph API | Node `=`, graphApiVersion `=` (placeholder) | None | Book All Available Slots | Filter Recent Discussions | Requires Facebook Graph API credential with group access. Configure actual group ID or search query. |
| **Filter Recent Discussions** | Function | Filters posts where `created_time` is within the last 7 days; maps to `post_id`, `topic`, `engagement` | `$input.all().filter(...)` comparing `created_time` to `new Date(Date.now() - 7 * 24 * 60 * 60 * 1000).toISOString()` | Find Facebook Group | Send WhatsApp Alert | Assumes input items have `created_time`, `id`, `message`, `comments.data.length`. Adapt to actual Facebook Graph API response structure. |
| **Send WhatsApp Alert** | WhatsApp | Operation: send; textBody `=`, phoneNumberId `=`, recipientPhoneNumber `=` (all placeholders) | None | Filter Recent Discussions | None (terminal) | Requires WhatsApp Business API credential. Set `phoneNumberId`, `recipientPhoneNumber`, and `textBody` with dynamic content. |

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Rise and Grind | Cron | Daily schedule trigger | — | Beer & Cig Farmer, INSTAGRAM1, TIKTOK1, FACEBOOK1, THREADS1, LINKEDIN1, TWITTER1, Wait2 | |
| Beer & Cig Farmer | HTTP Request | Fictional DoorDash order (satire) | Rise and Grind | Wait | |
| Wait | Wait | Pause 20 minutes before casino/odds | Beer & Cig Farmer | Uber Black to Casino, Bet365 API | |
| Uber Black to Casino | HTTP Request | Fictional Uber ride (satire) | Wait | Wait1, FB Marketplace scraper | |
| Wait1 | Wait | Pause 35 minutes before Tinder | Uber Black to Casino | Tinder API | |
| Wait2 | Wait | Pause before Spotify (satire) | Rise and Grind | Play Man in the box | |
| Play Man in the box | HTTP Request | Fictional Spotify playback (satire) | Wait2 | — | |
| Bet365 API | HTTP Request | Fetch sports betting odds (fictional) | Wait | Arbitrage Calculator | |
| Arbitrage Calculator | Function | Calculate arbitrage opportunities from two odds sources | Bet365 API (and implied second source) | Flex Wins on Socials | |
| Flex Wins on Socials | HTTP Request | Post arbitrage wins to Stocktwits (fictional) | Arbitrage Calculator | INSTAGRAM, TIKTOK, TWITTER, LINKEDIN, THREADS, FACEBOOK | |
| INSTAGRAM | HTTP Request (v4.2) | Post to Instagram via Blotato | Flex Wins on Socials | — | |
| TIKTOK | HTTP Request (v4.2) | Post to TikTok via Blotato | Flex Wins on Socials | — | |
| TWITTER | HTTP Request (v4.2) | Post to Twitter via Blotato | Flex Wins on Socials | — | |
| LINKEDIN | HTTP Request (v4.2) | Post to LinkedIn via Blotato | Flex Wins on Socials | — | |
| THREADS | HTTP Request (v4.2) | Post to Threads via Blotato | Flex Wins on Socials | — | |
| FACEBOOK | HTTP Request (v4.2) | Post to Facebook via Blotato | Flex Wins on Socials | — | |
| INSTAGRAM1 | HTTP Request (v4.2) | Post to Instagram via Blotato (daily) | Rise and Grind | — | |
| TIKTOK1 | HTTP Request (v4.2) | Post to TikTok via Blotato (daily) | Rise and Grind | — | |
| FACEBOOK1 | HTTP Request (v4.2) | Post to Facebook via Blotato (daily) | Rise and Grind | — | |
| THREADS1 | HTTP Request (v4.2) | Post to Threads via Blotato (daily) | Rise and Grind | — | |
| LINKEDIN1 | HTTP Request (v4.2) | Post to LinkedIn via Blotato (daily) | Rise and Grind | — | |
| TWITTER1 | HTTP Request (v4.2) | Post to Twitter via Blotato (daily) | Rise and Grind | — | |
| Tinder API | HTTP Request | Fictional Tinder auto-swipe (satire) | Wait1 | Wait3 | |
| Wait3 | Wait | Pause 1 hour before steak delivery | Tinder API | Steak & Monster Delivery | |
| Steak & Monster Delivery | HTTP Request | Fictional DoorDash order (satire) | Wait3 | Book All Available Slots | |
| Book All Available Slots | HTTP Request | Fictional range booking (satire) | Steak & Monster Delivery | Find Facebook Group, Search eBay Deals | |
| FB Marketplace scraper | Facebook Graph API | Scrape Facebook Marketplace listings | Uber Black to Casino | Data Normalizer | |
| Data Normalizer | Function | Normalize and stub marketplace data | FB Marketplace scraper | Item filter | |
| Item filter | IF | Filter items by price and recency | Data Normalizer | Lowball Algorithm | |
| Lowball Algorithm | Function | Calculate lowball offer and strategy | Item filter | Flaw Detection | |
| Flaw Detection | Function | Generate contextual excuse message | Lowball Algorithm | Message Blaster1 | |
| Message Blaster1 | Facebook Graph API | Send lowball offer message to seller | Flaw Detection | Deal Analyzer | |
| Deal Analyzer | Function | Calculate success probability of deal | Message Blaster1 | Relentless Negotiation Engine | |
| Relentless Negotiation Engine | IF | Decide whether to follow up based on probability | Deal Analyzer | Follow-up Nuke | |
| Follow-up Nuke | Facebook Graph API | Send follow-up message | Relentless Negotiation Engine | Opportunity Recalculator | |
| Opportunity Recalculator | Function | Reduce offer by 15% and recalculate | Follow-up Nuke | CRM Integration | |
| CRM Integration | Salesforce | Upsert deal/contact to Salesforce | Opportunity Recalculator | Deal Alert, Value Calculator | |
| Deal Alert | Telegram | Send deal notification to Telegram | CRM Integration | — | |
| Value Calculator | Function | Aggregate total opportunity value across deals | CRM Integration | Execute Purchase | |
| Execute Purchase | Function | Generate summary report | Value Calculator | — | |
| Search eBay Deals | HTTP Request | Search eBay for underpriced items | Book All Available Slots | Find Profitable Items | |
| Find Profitable Items | Function | Filter eBay items by profitability margin | Search eBay Deals | Item Lists | |
| Item Lists | Item Lists | Split deal array into individual items | Find Profitable Items | Get Item Details | |
| Get Item Details | HTTP Request | Fetch eBay item details | Item Lists | List Item for Sale | |
| List Item for Sale | HTTP Request | Relist item on eBay | Get Item Details | Log Transaction | |
| Log Transaction | Google Sheets | Append transaction row to spreadsheet | List Item for Sale | — | |
| Find Facebook Group | Facebook Graph API | Search for relevant Facebook Group | Book All Available Slots | Filter Recent Discussions | |
| Filter Recent Discussions | Function | Filter posts from last 7 days and extract engagement | Find Facebook Group | Send WhatsApp Alert | |
| Send WhatsApp Alert | WhatsApp | Send WhatsApp message with filtered discussions | Filter Recent Discussions | — | |

---

### 4. Reproducing the Workflow from Scratch

**Prerequisites:**
- n8n instance (self-hosted or cloud) with the following community or built-in nodes available: Cron, HTTP Request, Wait, Function, IF, Item Lists, Facebook Graph API, Salesforce, Telegram, Google Sheets, WhatsApp
- Credentials (to be created before activation):
  - Blotato API key (or replace with your own social publishing platform)
  - Facebook Graph API (with Marketplace, Groups, and Messaging permissions)
  - Salesforce OAuth2
  - Telegram Bot API
  - Google Sheets OAuth2
  - WhatsApp Business API
  - eBay Browse API / Sell API OAuth2
  - Generic HTTP credential (for any real endpoints replacing fictional ones)

**Step-by-step reconstruction:**

1. **Create Cron Trigger**
   - Add a **Cron** node named `Rise and Grind`.
   - Set schedule to desired daily time (e.g., `0 7 * * *` for 7 AM).
   - No additional parameters.

2. **Create Social Fan-Out (Daily Posts) – 6 nodes**
   - Add six **HTTP Request** nodes (v4.2): `INSTAGRAM1`, `TIKTOK1`, `FACEBOOK1`, `THREADS1`, `LINKEDIN1`, `TWITTER1`.
   - For each:
     - Method: `POST`
     - URL: `https://backend.blotato.com/v2/posts`
     - Send Body: enabled, specify body as JSON
     - Send Headers: enabled, add header `blotato-api-key` with your API key
     - JSON body varies per platform (see Block 2 details for exact structure per node)
     - Key dynamic references: `Assign Social Media IDs` (accountId fields), `Get my video.DESCRIPTION`, `Log Shorts & Schedule Info to Google Sheets.Schedule Time`, `$json.url` for media URLs
   - Connect `Rise and Grind` output to each of these six nodes.

3. **Create Satirical Lifestyle Chain**
   - Add **HTTP Request** node `Beer & Cig Farmer`: URL `https://api.doordash.com/v2/orders`, method GET.
   - Add **Wait** node: 20 minutes.
   - Connect `Rise and Grind` → `Beer & Cig Farmer` → `Wait`.
   - Add **HTTP Request** node `Uber Black to Casino`: URL `https://api.uber.com/v1/requests`, method POST, authentication `genericCredentialType`.
   - Connect `Wait` → `Uber Black to Casino`.
   - Add **Wait** node `Wait1`: 35 minutes.
   - Connect `Uber Black to Casino` → `Wait1`.
   - Add **HTTP Request** node `Tinder API`: URL `https://www.tinder.com/api`, method GET.
   - Connect `Wait1` → `Tinder API`.
   - Add **Wait** node `Wait3`: 1 hour.
   - Connect `Tinder API` → `Wait3`.
   - Add **HTTP Request** node `Steak & Monster Delivery`: URL `https://api.doordash.com/v2/orders`, method GET.
   - Connect `Wait3` → `Steak & Monster Delivery`.
   - Add **HTTP Request** node `Book All Available Slots`: URL `https://api.rangebooking.com/v1/reservations`, method GET.
   - Connect `Steak & Monster Delivery` → `Book All Available Slots`.
   - Add **Wait** node `Wait2`: default 1 minute.
   - Add **HTTP Request** node `Play Man in the box`: URL `https://api.spotify.com/v1/me/player/play`, method PUT.
   - Connect `Rise and Grind` → `Wait2` → `Play Man in the box`.

4. **Create Gambling → Arbitrage → Social Flex Chain**
   - Add **HTTP Request** node `Bet365 API`: URL `https://api.bet365.com`, method GET.
   - Connect `Wait` → `Bet365 API`.
   - Add **Function** node `Arbitrage Calculator`: paste the provided function code (calculates implied probabilities, filters for >2% arbitrage profit).
   - Connect `Bet365 API` → `Arbitrage Calculator`.
   - Add **HTTP Request** node `Flex Wins on Socials`: URL `https://api.stocktwits.com/v2/messages`, method POST.
   - Connect `Arbitrage Calculator` → `Flex Wins on Socials`.
   - Add six **HTTP Request** nodes (v4.2): `INSTAGRAM`, `TIKTOK`, `TWITTER`, `LINKEDIN`, `THREADS`, `FACEBOOK`. Configure identically to the `*1` versions in step 2, referencing `Assign Social Media IDs` and the arbitrage result data.
   - Connect `Flex Wins on Socials` → all six nodes.

5. **Create Facebook Marketplace Lowball Engine**
   - Add **Facebook Graph API** node `FB Marketplace scraper`: configure with your Facebook credential; set appropriate edge/node for marketplace listing queries.
   - Connect `Uber Black to Casino` → `FB Marketplace scraper`.
   - Add **Function** node `Data Normalizer`: paste the provided stub code (returns two sample items). Replace with real normalization for production.
   - Connect `FB Marketplace scraper` → `Data Normalizer`.
   - Add **IF** node `Item filter`: set conditions for `price` larger than a threshold and `posted` contains a freshness indicator (configure values).
   - Connect `Data Normalizer` → `Item filter`.
   - Add **Function** node `Lowball Algorithm`: paste the provided code (strategy selection, offer calculation).
   - Connect `Item filter` (true branch) → `Lowball Algorithm`.
   - Add **Function** node `Flaw Detection`: paste the provided code (excuse generator and message composer).
   - Connect `Lowball Algorithm` → `Flaw Detection`.
   - Add **Facebook Graph API** node `Message Blaster1`: configure for sending messages (conversations edge).
   - Connect `Flaw Detection` → `Message Blaster1`.
   - Add **Function** node `Deal Analyzer`: paste the provided code (success probability calculation).
   - Connect `Message Blaster1` → `Deal Analyzer`.
   - Add **IF** node `Relentless Negotiation Engine`: condition on `successProbability` below a threshold.
   - Connect `Deal Analyzer` → `Relentless Negotiation Engine`.
   - Add **Facebook Graph API** node `Follow-up Nuke`: configure for follow-up message.
   - Connect `Relentless Negotiation Engine` (true branch) → `Follow-up Nuke`.
   - Add **Function** node `Opportunity Recalculator`: paste the provided code (15% offer reduction, stage update).
   - Connect `Follow-up Nuke` → `Opportunity Recalculator`.
   - Add **Salesforce** node `CRM Integration`: set operation to create contact; map `company` and `lastname` from deal data; configure Salesforce OAuth2 credential.
   - Connect `Opportunity Recalculator` → `CRM Integration`.
   - Add **Telegram** node `Deal Alert`: set `chatId` and `text` with expressions referencing deal fields; configure Telegram Bot credential.
   - Connect `CRM Integration` → `Deal Alert`.
   - Add **Function** node `Value Calculator`: paste the provided code (aggregate totals).
   - Connect `CRM Integration` → `Value Calculator`.
   - Add **Function** node `Execute Purchase`: paste the provided code (summary report).
   - Connect `Value Calculator` → `Execute Purchase`.

6. **Create eBay Arbitrage Loop**
   - Add **HTTP Request** node `Search eBay Deals`: URL `https://svcs.ebay.com/services/search/FindingService/v1`, method GET; add query parameters (keywords, pagination) and eBay App ID header.
   - Connect `Book All Available Slots` → `Search eBay Deals`.
   - Add **Function** node `Find Profitable Items`: paste the provided code (parse eBay response, calculate product value, filter deals).
   - Connect `Search eBay Deals` → `Find Profitable Items`.
   - Add **Item Lists** node: set `fieldToSplitOut` to the array field containing deals (e.g., `deals`).
   - Connect `Find Profitable Items` → `Item Lists`.
   - Add **HTTP Request** node `Get Item Details`: URL `https://api.ebay.com/buy/browse/v1/item/{{$json["itemId"]}}`; configure eBay Browse API OAuth2.
   - Connect `Item Lists` → `Get Item Details`.
   - Add **HTTP Request** node `List Item for Sale`: URL `https://api.ebay.com/sell/inventory/v1/offer`, method POST, `jsonParameters: true`; configure eBay Sell API OAuth2.
   - Connect `Get Item Details` → `List Item for Sale`.
   - Add **Google Sheets** node `Log Transaction`: operation `append`; set `sheetId` to your spreadsheet ID; configure Google Sheets OAuth2 credential.
   - Connect `List Item for Sale` → `Log Transaction`.

7. **Create Facebook Group Engagement & WhatsApp Alerts**
   - Add **Facebook Graph API** node `Find Facebook Group`: configure with group ID or search query; set up Facebook credential.
   - Connect `Book All Available Slots` → `Find Facebook Group`.
   - Add **Function** node `Filter Recent Discussions`: paste the provided code (7-day filter, map to post_id/topic/engagement).
   - Connect `Find Facebook Group` → `Filter Recent Discussions`.
   - Add **WhatsApp** node `Send WhatsApp Alert`: operation `send`; set `phoneNumberId`, `recipientPhoneNumber`, and `textBody` with dynamic content; configure WhatsApp Business API credential.
   - Connect `Filter Recent Discussions` → `Send WhatsApp Alert`.

8. **Create Referenced Data Nodes (Missing from Import)**
   - Create a **Function** or **Set** node named `Assign Social Media IDs` that outputs an object with fields: `instagram_id`, `tiktok_id`, `facebook_id`, `facebook_page_id`, `threads_id`, `linkedin_id`, `twitter_id`. Wire it as an input reference for all Blotato posting nodes.
   - Create a **Function** or **Set** node named `Get my video` that outputs `DESCRIPTION` and `url` fields for media content. Wire accordingly.
   - Create a **Google Sheets** node named `Log Shorts & Schedule Info to Google Sheets` that outputs `Clip Caption` and `Schedule Time` fields. Wire accordingly.

9. **Final Checks**
   - Verify all Wait nodes have webhook IDs generated (they are auto-generated on creation).
   - Confirm all credential bindings (Blotato API key header, Facebook, Salesforce, Telegram, Google Sheets, WhatsApp, eBay).
   - Test each branch individually before activating the full workflow.
   - Remove or replace satirical nodes (Beer & Cig Farmer, Uber Black, Tinder, Steak, Play Man in the box) if not needed.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow is satire. The "Hell Yeah!" template parodies hustle-culture AI automation content. All fictional API endpoints (DoorDash, Uber, Bet365, Tinder, Spotify, Stocktwits, RangeBooking) must be replaced with real endpoints before activation. | Workflow description |
| The Blotato API (`backend.blotato.com/v2/posts`) is used for multi-platform social posting. You need a Blotato account and API key. Alternatively, replace each platform node with native social media API calls (Facebook Graph, Twitter API v2, LinkedIn Share API, etc.). | INSTAGRAM, TIKTOK, FACEBOOK, THREADS, LINKEDIN, TWITTER nodes |
| The `Assign Social Media IDs`, `Get my video`, and `Log Shorts & Schedule Info to Google Sheets` nodes are referenced but not included in the workflow JSON. They must be created manually or the expressions will fail at runtime. | All Blotato posting nodes |
| Facebook Graph API nodes require specific permissions: `pages_manage_posts` for posting, `pages_read_engagement` for group scraping, `pages_messaging` for sending messages. Review Facebook App Review requirements. | FB Marketplace scraper, Message Blaster1, Follow-up Nuke, Find Facebook Group |
| The Arbitrage Calculator expects two input streams (two odds sources). Only one (Bet365 API) is connected. Add a second odds source node (e.g., another bookmaker API) and connect both to the Function node for the arbitrage logic to function. | Arbitrage Calculator |
| The Item filter IF node has incomplete condition values (missing right-hand comparison). Populate these before activating (e.g., `price > 50`, `posted contains "hour"`). | Item filter |
| The Relentless Negotiation Engine IF node has an incomplete condition. Set a numeric threshold (e.g., `successProbability smaller than 50`). | Relentless Negotiation Engine |
| eBay Finding API requires an `SECURITY-APPNAME` header and proper query parameters (keywords, outputSelector, pagination). The current URL is a bare endpoint. | Search eBay Deals |
| All placeholder values set to `=` in node parameters (Salesforce fields, Telegram chatId/text, Google Sheets sheetId, WhatsApp fields) must be replaced with actual values or dynamic expressions. | CRM Integration, Deal Alert, Log Transaction, Send WhatsApp Alert |
| The Data Normalizer function currently returns hardcoded sample data. For production, map the real Facebook Graph API response fields to the expected schema (`id`, `title`, `price`, `description`, `seller`, `seller_id`, `image_url`, `location`, `posted`). | Data Normalizer |
| Item Lists node's `fieldToSplitOut` is set to "Tactical Gear" which is a placeholder. Change to the actual array field name from the Find Profitable Items output (e.g., `deals`). | Item Lists |
| The workflow uses n8n Wait nodes which rely on webhook callbacks. Ensure your n8n instance is publicly accessible or configured for test webhooks during development. | Wait, Wait1, Wait2, Wait3 |