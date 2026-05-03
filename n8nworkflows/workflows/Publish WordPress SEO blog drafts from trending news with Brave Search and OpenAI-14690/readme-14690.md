Publish WordPress SEO blog drafts from trending news with Brave Search and OpenAI

https://n8nworkflows.xyz/workflows/publish-wordpress-seo-blog-drafts-from-trending-news-with-brave-search-and-openai-14690


# Publish WordPress SEO blog drafts from trending news with Brave Search and OpenAI

# 1. Workflow Overview

This workflow automatically creates SEO-oriented WordPress blog drafts from current trending news. It starts with a scheduled trigger, searches Brave Search for recent news, summarizes and evaluates the top results, chooses one story, builds a full article plan with image prompts, generates body images, writes the article in sections, assembles and cleans the final HTML, publishes the draft to WordPress, enriches it with an excerpt and featured image, then sends a Telegram notification.

Its main use cases are:
- Automated content production for news-reactive SEO blogs
- Draft generation for editorial review
- Scalable AI-assisted article production with image creation
- Branded publishing pipelines where tone, SEO goals, and WordPress metadata are controlled centrally

## 1.1 Trigger and Global Configuration
The workflow begins on a schedule and loads all configurable publishing, SEO, image, and WordPress settings from a single Set node.

## 1.2 News Retrieval and Preprocessing
It queries Brave Search for recent news, limits the result set, and formats the data into an AI-friendly summary input.

## 1.3 Story Selection and Master Plan Generation
The workflow summarizes results, loops through candidate data, and uses an AI agent to select a story and create a structured article blueprint including article sections and image prompts.

## 1.4 Image Generation and Upload Preparation
The image prompts are split out, sent to OpenAI image generation, uploaded to WordPress, then re-aggregated into an image array for downstream article writing.

## 1.5 Article Planning and Section Writing
The master plan and generated image list are merged, transformed into section-level briefs, split into individual parts, and written sequentially by an LLM.

## 1.6 HTML Assembly and Publishing
Written sections are aggregated, transformed into a single article, cleaned, and published to WordPress. The post is then updated with an excerpt and image metadata.

## 1.7 Featured Image and Final Notification
A separate OpenAI image step prepares a featured image, uploads it to WordPress, assigns it to the post, stores final result data, and sends a Telegram notification.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger and Configuration

### Overview
This block starts the workflow automatically and centralizes all editable parameters. It is the operational control center for search criteria, brand voice, image behavior, and WordPress publishing settings.

### Nodes Involved
- Schedule Trigger
- Set Config

### Node Details

#### Schedule Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; workflow entry point.
- **Configuration choices:** Trigger timing is defined in the node, though the exported JSON does not expose its exact schedule values here.
- **Key expressions or variables used:** None shown directly; it simply starts execution.
- **Input and output connections:** No input. Outputs to **Set Config**.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:** Misconfigured schedule, disabled workflow, timezone mismatch.
- **Sub-workflow reference:** None.

#### Set Config
- **Type and technical role:** `n8n-nodes-base.set`; central runtime configuration node.
- **Configuration choices:** Intended to hold all user-editable constants:
  - `search_query`
  - `search_country`
  - `search_language`
  - `top_results_count`
  - `brand_brief`
  - `target_word_count`
  - `keywords`
  - `additional_instructions`
  - `use_header_image`
  - `header_image_source`
  - `image_count`
  - `image_source`
  - `cloudinary_cloud_name`
  - `cloudinary_upload_preset`
  - `wp_post_status`
  - `wp_author_id`
  - `wp_category_id`
- **Key expressions or variables used:** Downstream nodes are expected to reference these fields with expressions like `$json.search_query`, `$json.brand_brief`, etc.
- **Input and output connections:** Input from **Schedule Trigger**. Output to **Brave Search1**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**
  - Missing required values causes downstream expression failures
  - Invalid WordPress IDs can break post creation
  - Unsupported search locale values may affect Brave Search results
  - Inconsistent image settings can break image branches logically
- **Sub-workflow reference:** None.

---

## 2.2 News Retrieval and Preprocessing

### Overview
This block fetches current news from Brave Search and formats the result set into a compact structure suitable for AI evaluation. It reduces noisy search output before LLM processing.

### Nodes Involved
- Brave Search1
- Format Search Results1
- Make sumarry
- OpenAI Chat Model
- Code in JavaScript2

### Node Details

#### Brave Search1
- **Type and technical role:** `@brave/n8n-nodes-brave-search.braveSearch`; external news search provider.
- **Configuration choices:** Expected to use values from **Set Config** for query, country, and language; notes indicate it can return up to 20 results.
- **Key expressions or variables used:** Likely expressions such as:
  - query from `search_query`
  - country from `search_country`
  - language from `search_language`
- **Input and output connections:** Input from **Set Config**. Output to **Format Search Results1**.
- **Version-specific requirements:** Type version `1`; requires Brave Search node package and credentials/API setup if applicable.
- **Edge cases or potential failure types:**
  - API auth issues
  - empty or low-quality result sets
  - rate limits
  - regional result inconsistency
- **Sub-workflow reference:** None.

#### Format Search Results1
- **Type and technical role:** `n8n-nodes-base.code`; transforms raw search response.
- **Configuration choices:** Filters the top N results and formats them into a structured string while preserving raw array data.
- **Key expressions or variables used:** Likely references:
  - `top_results_count`
  - Brave result fields such as title, URL, snippet, date
- **Input and output connections:** Input from **Brave Search1**. Output to **Make sumarry**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - unexpected Brave response structure
  - missing titles/URLs/snippets
  - code errors if arrays are empty or nested differently than expected
- **Sub-workflow reference:** None.

#### Make sumarry
- **Type and technical role:** `@n8n/n8n-nodes-langchain.chainLlm`; LLM chain for search-result synthesis.
- **Configuration choices:** Uses **OpenAI Chat Model** as its language model. Based on placement and naming, it likely creates a concise comparative summary of candidate articles for downstream selection.
- **Key expressions or variables used:** Likely consumes formatted search summary text from previous node.
- **Input and output connections:** Main input from **Format Search Results1**; AI model input from **OpenAI Chat Model**. Main output to **Code in JavaScript2**.
- **Version-specific requirements:** Type version `1.9`.
- **Edge cases or potential failure types:**
  - malformed prompts
  - token overrun if too many results are passed
  - model/API auth or quota failures
- **Sub-workflow reference:** None.

#### OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; chat LLM provider.
- **Configuration choices:** Serves as the model backend for **Make sumarry**.
- **Key expressions or variables used:** None visible in JSON, but model choice and credentials are required.
- **Input and output connections:** AI language model connection into **Make sumarry** only.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**
  - invalid OpenAI credentials
  - model deprecation
  - timeout/rate limit
- **Sub-workflow reference:** None.

#### Code in JavaScript2
- **Type and technical role:** `n8n-nodes-base.code`; reshapes summary output into loop-ready items.
- **Configuration choices:** Likely prepares one or more candidate records for iteration.
- **Key expressions or variables used:** References prior summary and perhaps original search result metadata.
- **Input and output connections:** Input from **Make sumarry**. Output to **Loop Over Items**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - missing summary field
  - code assumptions about JSON structure
- **Sub-workflow reference:** None.

---

## 2.3 Story Selection, Notification Drafting, and Master Plan Creation

### Overview
This block iterates through prepared candidate items, aggregates context, drafts a notification message, and uses an AI agent to create a complete article plan. The output becomes the main structured blueprint for article and image generation.

### Nodes Involved
- Loop Over Items
- Aggregate
- Create Notification
- OpenAI Chat Model2
- Send a text message
- Set Data in Loop
- Idea and Plan Agent1
- OpenAI — Plan Agent1
- Create JSON1

### Node Details

#### Loop Over Items
- **Type and technical role:** `n8n-nodes-base.splitInBatches`; iteration controller.
- **Configuration choices:** Processes incoming candidate items one batch at a time.
- **Key expressions or variables used:** Typically batch size is set to 1 in this pattern, though not visible here.
- **Input and output connections:** Input from **Code in JavaScript2** and later feedback from **Set results data**. First output goes to **Aggregate**; loop output goes to **Set Data in Loop**.
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**
  - infinite-loop behavior if loop-closing logic is wrong
  - empty input causes no downstream execution
- **Sub-workflow reference:** None.

#### Aggregate
- **Type and technical role:** `n8n-nodes-base.aggregate`; combines loop data.
- **Configuration choices:** Likely gathers current result set into a single notification context.
- **Key expressions or variables used:** Not visible.
- **Input and output connections:** Input from **Loop Over Items**. Output to **Create Notification**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** Unexpected aggregation schema if inputs differ.
- **Sub-workflow reference:** None.

#### Create Notification
- **Type and technical role:** `@n8n/n8n-nodes-langchain.chainLlm`; drafts a Telegram message or status summary.
- **Configuration choices:** Uses **OpenAI Chat Model2**.
- **Key expressions or variables used:** Likely includes article topic, selected story, or execution status.
- **Input and output connections:** Main input from **Aggregate**; AI model from **OpenAI Chat Model2**. Main output to **Send a text message**.
- **Version-specific requirements:** Type version `1.9`.
- **Edge cases or potential failure types:**
  - malformed prompt input
  - OpenAI failures
  - output not matching Telegram formatting expectations
- **Sub-workflow reference:** None.

#### OpenAI Chat Model2
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; LLM backend.
- **Configuration choices:** Powers **Create Notification**.
- **Key expressions or variables used:** None visible.
- **Input and output connections:** AI connection into **Create Notification**.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:** standard OpenAI auth/rate-limit/model issues.
- **Sub-workflow reference:** None.

#### Send a text message
- **Type and technical role:** `n8n-nodes-base.telegram`; sends Telegram message.
- **Configuration choices:** Requires Telegram bot credentials and a target chat.
- **Key expressions or variables used:** Likely uses generated notification text from previous node.
- **Input and output connections:** Input from **Create Notification**. No downstream connection shown.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**
  - invalid bot token
  - unauthorized chat
  - message formatting issues
- **Sub-workflow reference:** None.

#### Set Data in Loop
- **Type and technical role:** `n8n-nodes-base.set`; prepares per-item payload for planning.
- **Configuration choices:** Likely selects the active news item and relevant config fields for the agent.
- **Key expressions or variables used:** Likely merges current loop item with values from **Set Config**.
- **Input and output connections:** Input from **Loop Over Items**. Output to **Idea and Plan Agent1**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:** Missing fields from loop item or config.
- **Sub-workflow reference:** None.

#### Idea and Plan Agent1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; main planning agent.
- **Configuration choices:** Generates:
  - `master_plan`
  - `image_prompts` array
  - `header_image_prompt`
  It is marked `retryOnFail: true`, which is appropriate for LLM transient failures.
- **Key expressions or variables used:** Likely consumes search summaries, selected article source URL, brand brief, keyword hints, target length, and image count.
- **Input and output connections:** Main input from **Set Data in Loop**; AI model from **OpenAI — Plan Agent1**. Main output to **Create JSON1**.
- **Version-specific requirements:** Type version `3.1`.
- **Edge cases or potential failure types:**
  - non-JSON output despite expecting structured JSON
  - omitted required keys
  - hallucinated or invalid URLs
  - token length issues if prompts are too long
- **Sub-workflow reference:** None.

#### OpenAI — Plan Agent1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; model backend for the planning agent.
- **Configuration choices:** Notes indicate `gpt-4.1-mini` for planning, with `gpt-4.1` as an optional higher-quality substitute.
- **Key expressions or variables used:** None visible.
- **Input and output connections:** AI language model connection to **Idea and Plan Agent1**.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:** model access restrictions, pricing/quota constraints, timeouts.
- **Sub-workflow reference:** None.

#### Create JSON1
- **Type and technical role:** `n8n-nodes-base.code`; parser and repair layer for agent output.
- **Configuration choices:** Parses raw JSON string from the planning agent, attempts cleanup of:
  - smart quotes
  - code fences
  - trailing commas
  It uses `onError: continueRegularOutput`, so even error-handling is designed not to hard-stop the main branch immediately.
- **Key expressions or variables used:** Expected parsed fields include `master_plan`, `image_prompts`, `header_image_prompt`.
- **Input and output connections:** Input from **Idea and Plan Agent1**. Outputs to:
  - **Split Out1**
  - **Merge**
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - unrecoverable malformed JSON
  - missing expected arrays or keys
  - downstream mismatch if parsing produces partial objects
- **Sub-workflow reference:** None.

---

## 2.4 Body Image Generation and Aggregation

### Overview
This block turns the image prompts from the article plan into generated images, uploads them, and reassembles the resulting image references into a single array for article composition.

### Nodes Involved
- Split Out1
- Generate an image3
- Upload Image to Wordpress
- Set Output Image1
- Aggregate1
- set images
- Merge

### Node Details

#### Split Out1
- **Type and technical role:** `n8n-nodes-base.splitOut`; explodes the `image_prompts` array into one item per prompt.
- **Configuration choices:** Expected to split the array created in **Create JSON1**.
- **Key expressions or variables used:** Likely targets the `image_prompts` field.
- **Input and output connections:** Input from **Create JSON1**. Output to **Generate an image3**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - missing or empty `image_prompts`
  - field type not array
- **Sub-workflow reference:** None.

#### Generate an image3
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`; OpenAI image generation.
- **Configuration choices:** Used for creating body images from prompts. `retryOnFail: true`.
- **Key expressions or variables used:** Likely prompt text from split items.
- **Input and output connections:** Input from **Split Out1**. Output to **Upload Image to Wordpress**.
- **Version-specific requirements:** Type version `1.8`; requires OpenAI credentials with image model access.
- **Edge cases or potential failure types:**
  - safety-policy rejections
  - invalid prompt structure
  - model access limitations
  - image generation latency
- **Sub-workflow reference:** None.

#### Upload Image to Wordpress
- **Type and technical role:** `n8n-nodes-base.httpRequest`; uploads generated body image to WordPress media library.
- **Configuration choices:** Uses `onError: continueRegularOutput` and `retryOnFail: false`, so failures are tolerated and passed onward.
- **Key expressions or variables used:** Likely sends binary or URL/image payload to WordPress REST media endpoint.
- **Input and output connections:** Input from **Generate an image3**. Output to **Set Output Image1**.
- **Version-specific requirements:** Type version `4.2`; requires proper WordPress authentication and multipart/media endpoint formatting.
- **Edge cases or potential failure types:**
  - auth failure
  - incorrect content-type
  - unsupported file format
  - large image upload failures
- **Sub-workflow reference:** None.

#### Set Output Image1
- **Type and technical role:** `n8n-nodes-base.set`; normalizes uploaded image metadata.
- **Configuration choices:** Likely extracts media URL, media ID, alt text, or fallback values.
- **Key expressions or variables used:** Probably references upload response fields.
- **Input and output connections:** Input from **Upload Image to Wordpress**. Output to **Aggregate1**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:** Missing upload response when previous node continued on error.
- **Sub-workflow reference:** None.

#### Aggregate1
- **Type and technical role:** `n8n-nodes-base.aggregate`; combines all body images back into one array.
- **Configuration choices:** Re-forms individual image items into a single collection.
- **Key expressions or variables used:** Likely aggregates URLs or objects.
- **Input and output connections:** Input from **Set Output Image1**. Output to **set images**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** Empty aggregation if all uploads fail.
- **Sub-workflow reference:** None.

#### set images
- **Type and technical role:** `n8n-nodes-base.set`; final image payload packaging.
- **Configuration choices:** Likely stores the aggregated image array in a field expected by article planning/writing.
- **Key expressions or variables used:** Probably creates something like `images`.
- **Input and output connections:** Input from **Aggregate1**. Output to **Merge** on input index 1.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:** Incorrect field naming can break **Aggregate for Planner** and later nodes.
- **Sub-workflow reference:** None.

#### Merge
- **Type and technical role:** `n8n-nodes-base.merge`; combines plan JSON and image array stream.
- **Configuration choices:** Receives:
  - input 0 from **Create JSON1**
  - input 1 from **set images**
- **Key expressions or variables used:** Not visible; likely merge by position or combine mode.
- **Input and output connections:** Outputs to **Aggregate for Planner**.
- **Version-specific requirements:** Type version `3.2`.
- **Edge cases or potential failure types:**
  - branch timing mismatch
  - merge mode misconfiguration
  - missing image branch output
- **Sub-workflow reference:** None.

---

## 2.5 Article Planning and Section Writing

### Overview
This block transforms the full article plan into section-by-section writing briefs, writes each section with the LLM, and aggregates the resulting text into a complete article draft.

### Nodes Involved
- Aggregate for Planner
- Article Planner
- OpenAI — Article Planner
- Code in JavaScript
- Split Out2
- Article wrtiter
- OpenAI Chat Model1
- Aggregate2

### Node Details

#### Aggregate for Planner
- **Type and technical role:** `n8n-nodes-base.aggregate`; prepares a unified data object for the planner.
- **Configuration choices:** Notes state it aggregates merged streams so the planner receives both plan data and image array together.
- **Key expressions or variables used:** Likely produces one combined item containing `master_plan` and `images`.
- **Input and output connections:** Input from **Merge**. Output to **Article Planner**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** Aggregation schema issues if merge structure differs.
- **Sub-workflow reference:** None.

#### Article Planner
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; converts master plan into section briefs.
- **Configuration choices:** Produces sequential part briefs for the writer. Each part includes:
  - title
  - objectives
  - content outline
  - assigned image URL
  - word count target
  - source URL if relevant
  - brand voice instructions
  Uses `retryOnFail: true`.
- **Key expressions or variables used:** Consumes `master_plan`, image list, `brand_brief`, target length, and likely additional instructions.
- **Input and output connections:** Main input from **Aggregate for Planner**; AI model from **OpenAI — Article Planner**. Main output to **Code in JavaScript**.
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - invalid or incomplete plan object
  - non-structured output
  - image count mismatch with section count
- **Sub-workflow reference:** None.

#### OpenAI — Article Planner
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; planner model backend.
- **Configuration choices:** Notes recommend `gpt-4.1-mini` for cost efficiency.
- **Key expressions or variables used:** None visible.
- **Input and output connections:** AI connection into **Article Planner**.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:** standard OpenAI issues.
- **Sub-workflow reference:** None.

#### Code in JavaScript
- **Type and technical role:** `n8n-nodes-base.code`; reshapes planner output into an array suitable for splitting into individual section-writing tasks.
- **Configuration choices:** Likely extracts parts array and normalizes fields for the writer.
- **Key expressions or variables used:** Expected fields include section title, objectives, outline, image URL, target words.
- **Input and output connections:** Input from **Article Planner**. Output to **Split Out2**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:** Wrong assumptions about planner output schema.
- **Sub-workflow reference:** None.

#### Split Out2
- **Type and technical role:** `n8n-nodes-base.splitOut`; creates one item per article part.
- **Configuration choices:** Likely splits the section briefs array.
- **Key expressions or variables used:** Presumably targets `parts`.
- **Input and output connections:** Input from **Code in JavaScript**. Output to **Article wrtiter**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** empty parts array or malformed planner output.
- **Sub-workflow reference:** None.

#### Article wrtiter
- **Type and technical role:** `@n8n/n8n-nodes-langchain.chainLlm`; writes each article section.
- **Configuration choices:** Receives one section brief at a time and uses **OpenAI Chat Model1**. `retryOnFail: true`.
- **Key expressions or variables used:** Likely includes:
  - part title
  - objective
  - outline
  - assigned image URL
  - tone/brand brief
  - target word count
- **Input and output connections:** Main input from **Split Out2**; AI model from **OpenAI Chat Model1**. Output to **Aggregate2**.
- **Version-specific requirements:** Type version `1.7`.
- **Edge cases or potential failure types:**
  - repetitive or inconsistent section writing
  - token overrun on large prompts
  - HTML/Markdown mismatches
- **Sub-workflow reference:** None.

#### OpenAI Chat Model1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; writing model backend.
- **Configuration choices:** Used exclusively by **Article wrtiter**.
- **Key expressions or variables used:** None visible.
- **Input and output connections:** AI connection into **Article wrtiter**.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:** auth, quota, timeout, model selection drift.
- **Sub-workflow reference:** None.

#### Aggregate2
- **Type and technical role:** `n8n-nodes-base.aggregate`; recombines all written sections.
- **Configuration choices:** Collects section outputs into a single article structure.
- **Key expressions or variables used:** Likely aggregates body sections in original order.
- **Input and output connections:** Input from **Article wrtiter**. Output to **Code in JavaScript1**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** order loss if aggregation is not sequence-aware.
- **Sub-workflow reference:** None.

---

## 2.6 HTML Assembly and WordPress Publishing

### Overview
This block converts aggregated article sections into final post HTML, cleans it, creates the WordPress post, and then updates metadata such as the excerpt.

### Nodes Involved
- Code in JavaScript1
- clean html1
- Wordpress
- WordPress excerpt add

### Node Details

#### Code in JavaScript1
- **Type and technical role:** `n8n-nodes-base.code`; assembles final article content.
- **Configuration choices:** Likely concatenates article parts, inserts headings/images, and prepares title/content fields.
- **Key expressions or variables used:** Probably references aggregated section array and plan metadata.
- **Input and output connections:** Input from **Aggregate2**. Output to **clean html1**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - malformed concatenation
  - duplicate headings
  - invalid HTML construction
- **Sub-workflow reference:** None.

#### clean html1
- **Type and technical role:** `n8n-nodes-base.code`; post-processing/cleanup of final HTML.
- **Configuration choices:** Likely removes wrapping artifacts, bad tags, excessive markdown remnants, or unsafe formatting before publish.
- **Key expressions or variables used:** Operates on article HTML/content fields.
- **Input and output connections:** Input from **Code in JavaScript1**. Output to **Wordpress**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**
  - removing desired markup accidentally
  - breaking image tags/links
- **Sub-workflow reference:** None.

#### Wordpress
- **Type and technical role:** `n8n-nodes-base.wordpress`; creates the WordPress post.
- **Configuration choices:** Likely creates a post in status defined by `wp_post_status`, with author/category from config and generated title/content.
- **Key expressions or variables used:** Expected fields:
  - title
  - content
  - status
  - author ID
  - category ID
- **Input and output connections:** Input from **clean html1**. Output to **WordPress excerpt add**.
- **Version-specific requirements:** Type version `1`; requires WordPress credentials in n8n.
- **Edge cases or potential failure types:**
  - auth failure
  - invalid author/category IDs
  - HTML rejected or sanitized by WordPress
- **Sub-workflow reference:** None.

#### WordPress excerpt add
- **Type and technical role:** `n8n-nodes-base.httpRequest`; custom REST update for excerpt or post fields not handled directly.
- **Configuration choices:** Likely issues a PATCH/POST request to the WordPress REST API using the created post ID.
- **Key expressions or variables used:** Probably uses post ID from **Wordpress** and excerpt text derived from article content.
- **Input and output connections:** Input from **Wordpress**. Output to **OpenAI**.
- **Version-specific requirements:** Type version `4.2`; requires WordPress REST auth.
- **Edge cases or potential failure types:**
  - wrong endpoint
  - auth/nonce/application password issue
  - excerpt field not writable due to permissions/plugins
- **Sub-workflow reference:** None.

---

## 2.7 Featured Image Assignment and Result Capture

### Overview
This block creates or prepares the featured image, uploads it, assigns it to the WordPress post, and stores the final publication result for loop completion and notification handling.

### Nodes Involved
- OpenAI
- Upload Image to Wordpress2
- Set Image on Wordpress Post2
- Set results data

### Node Details

#### OpenAI
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`; second OpenAI image-generation or media-preparation step.
- **Configuration choices:** Since it sits after the excerpt update and before featured-image upload, it likely generates the header/featured image defined by `header_image_prompt`.
- **Key expressions or variables used:** Expected to use `header_image_prompt` and possibly image-source settings.
- **Input and output connections:** Input from **WordPress excerpt add**. Output to **Upload Image to Wordpress2**.
- **Version-specific requirements:** Type version `1.8`; OpenAI image permissions required.
- **Edge cases or potential failure types:**
  - missing header prompt
  - content-policy rejection
  - if header images are optional, this node may require guarding logic not visible here
- **Sub-workflow reference:** None.

#### Upload Image to Wordpress2
- **Type and technical role:** `n8n-nodes-base.httpRequest`; uploads the featured image to WordPress.
- **Configuration choices:** `onError: continueRegularOutput`; if upload fails, workflow continues.
- **Key expressions or variables used:** Likely media endpoint and binary/image payload from **OpenAI**.
- **Input and output connections:** Input from **OpenAI**. Output to **Set Image on Wordpress Post2**.
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - upload failure
  - missing image binary
  - invalid auth
  - continuing with incomplete response
- **Sub-workflow reference:** None.

#### Set Image on Wordpress Post2
- **Type and technical role:** `n8n-nodes-base.httpRequest`; sets featured media on the created post.
- **Configuration choices:** Uses `onError: continueRegularOutput`; probably updates the post with `featured_media`.
- **Key expressions or variables used:** Needs:
  - created post ID from **Wordpress**
  - media ID from **Upload Image to Wordpress2**
- **Input and output connections:** Input from **Upload Image to Wordpress2**. Output to **Set results data**.
- **Version-specific requirements:** Type version `4.2`.
- **Edge cases or potential failure types:**
  - missing media ID if upload failed
  - post ID/media ID mismatch
  - permission failure on post update
- **Sub-workflow reference:** None.

#### Set results data
- **Type and technical role:** `n8n-nodes-base.set`; stores final result payload.
- **Configuration choices:** Likely normalizes final post URL, post ID, image status, and completion state.
- **Key expressions or variables used:** Likely combines WordPress post info with image assignment result.
- **Input and output connections:** Input from **Set Image on Wordpress Post2**. Output to **Loop Over Items** to continue/complete loop cycle.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:** incomplete fields if prior HTTP nodes continued after errors.
- **Sub-workflow reference:** None.

---

## 2.8 Sticky Notes and Canvas Annotation Layer

### Overview
These nodes are visual annotations only. In this exported workflow, their content is empty, so they provide layout grouping but no retained explanatory text.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6
- Sticky Note7
- Sticky Note8
- Sticky Note9
- Sticky Note10

### Node Details
All sticky notes:
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual documentation only.
- **Configuration choices:** All have empty `content`.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None; they do not execute.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Scheduled workflow entry point |  | Set Config |  |
| Set Config | n8n-nodes-base.set | Central configuration for search, SEO, images, and WordPress | Schedule Trigger | Brave Search1 |  |
| Brave Search1 | @brave/n8n-nodes-brave-search.braveSearch | Fetch trending/news search results | Set Config | Format Search Results1 |  |
| Format Search Results1 | n8n-nodes-base.code | Limit and format search results for AI | Brave Search1 | Make sumarry |  |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for search summarization |  | Make sumarry |  |
| Make sumarry | @n8n/n8n-nodes-langchain.chainLlm | Summarize and structure candidate news context | Format Search Results1, OpenAI Chat Model | Code in JavaScript2 |  |
| Code in JavaScript2 | n8n-nodes-base.code | Convert summary into loop-ready items | Make sumarry | Loop Over Items |  |
| Loop Over Items | n8n-nodes-base.splitInBatches | Iterate through candidate items / control loop | Code in JavaScript2, Set results data | Aggregate, Set Data in Loop |  |
| Aggregate | n8n-nodes-base.aggregate | Aggregate loop context for notification | Loop Over Items | Create Notification |  |
| OpenAI Chat Model2 | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for Telegram notification text |  | Create Notification |  |
| Create Notification | @n8n/n8n-nodes-langchain.chainLlm | Generate notification message | Aggregate, OpenAI Chat Model2 | Send a text message |  |
| Send a text message | n8n-nodes-base.telegram | Send Telegram notification | Create Notification |  |  |
| Set Data in Loop | n8n-nodes-base.set | Prepare selected item/config payload for planning | Loop Over Items | Idea and Plan Agent1 |  |
| OpenAI — Plan Agent1 | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for article plan generation |  | Idea and Plan Agent1 |  |
| Idea and Plan Agent1 | @n8n/n8n-nodes-langchain.agent | Select story and generate master article plan plus image prompts | Set Data in Loop, OpenAI — Plan Agent1 | Create JSON1 |  |
| Create JSON1 | n8n-nodes-base.code | Parse and repair JSON output from planning agent | Idea and Plan Agent1 | Split Out1, Merge |  |
| Split Out1 | n8n-nodes-base.splitOut | Split image prompts into individual items | Create JSON1 | Generate an image3 |  |
| Generate an image3 | @n8n/n8n-nodes-langchain.openAi | Generate body images from prompts | Split Out1 | Upload Image to Wordpress |  |
| Upload Image to Wordpress | n8n-nodes-base.httpRequest | Upload generated body image to WordPress media library | Generate an image3 | Set Output Image1 |  |
| Set Output Image1 | n8n-nodes-base.set | Normalize body image upload output | Upload Image to Wordpress | Aggregate1 |  |
| Aggregate1 | n8n-nodes-base.aggregate | Aggregate uploaded body images into array | Set Output Image1 | set images |  |
| set images | n8n-nodes-base.set | Package body image array for merge | Aggregate1 | Merge |  |
| Merge | n8n-nodes-base.merge | Combine parsed plan and body image array | Create JSON1, set images | Aggregate for Planner |  |
| Aggregate for Planner | n8n-nodes-base.aggregate | Build unified planner payload with plan and images | Merge | Article Planner |  |
| OpenAI — Article Planner | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for section-brief planning |  | Article Planner |  |
| Article Planner | @n8n/n8n-nodes-langchain.agent | Divide master plan into sequential section briefs | Aggregate for Planner, OpenAI — Article Planner | Code in JavaScript |  |
| Code in JavaScript | n8n-nodes-base.code | Normalize section briefs into array | Article Planner | Split Out2 |  |
| Split Out2 | n8n-nodes-base.splitOut | Split section briefs into one item per part | Code in JavaScript | Article wrtiter |  |
| OpenAI Chat Model1 | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM backend for section writing |  | Article wrtiter |  |
| Article wrtiter | @n8n/n8n-nodes-langchain.chainLlm | Write each article section | Split Out2, OpenAI Chat Model1 | Aggregate2 |  |
| Aggregate2 | n8n-nodes-base.aggregate | Collect written sections into full draft structure | Article wrtiter | Code in JavaScript1 |  |
| Code in JavaScript1 | n8n-nodes-base.code | Assemble final article HTML/content | Aggregate2 | clean html1 |  |
| clean html1 | n8n-nodes-base.code | Clean and normalize final HTML before publishing | Code in JavaScript1 | Wordpress |  |
| Wordpress | n8n-nodes-base.wordpress | Create WordPress post | clean html1 | WordPress excerpt add |  |
| WordPress excerpt add | n8n-nodes-base.httpRequest | Update excerpt or post metadata via REST API | Wordpress | OpenAI |  |
| OpenAI | @n8n/n8n-nodes-langchain.openAi | Generate featured image | WordPress excerpt add | Upload Image to Wordpress2 |  |
| Upload Image to Wordpress2 | n8n-nodes-base.httpRequest | Upload featured image to WordPress media library | OpenAI | Set Image on Wordpress Post2 |  |
| Set Image on Wordpress Post2 | n8n-nodes-base.httpRequest | Assign featured image to created post | Upload Image to Wordpress2 | Set results data |  |
| Set results data | n8n-nodes-base.set | Store final post result and loop data | Set Image on Wordpress Post2 | Loop Over Items |  |
| Sticky Note | n8n-nodes-base.stickyNote | Visual annotation |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual annotation |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual annotation |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual annotation |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Visual annotation |  |  |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Visual annotation |  |  |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Visual annotation |  |  |  |
| Sticky Note7 | n8n-nodes-base.stickyNote | Visual annotation |  |  |  |
| Sticky Note8 | n8n-nodes-base.stickyNote | Visual annotation |  |  |  |
| Sticky Note9 | n8n-nodes-base.stickyNote | Visual annotation |  |  |  |
| Sticky Note10 | n8n-nodes-base.stickyNote | Visual annotation |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Schedule Trigger** node.
   - Type: `Schedule Trigger`
   - Configure the desired cadence, e.g. hourly, daily, or specific weekdays.
   - Ensure timezone matches your publishing schedule.

3. **Add a Set node named `Set Config`** after the trigger.
   - Populate all central fields:
     - `search_query` — e.g. “latest AI news” or your niche news query
     - `search_country` — e.g. `US`
     - `search_language` — e.g. `en`
     - `top_results_count` — e.g. `5`
     - `brand_brief` — your audience, style, brand tone, positioning
     - `target_word_count` — e.g. `1800`
     - `keywords` — optional comma-separated SEO keywords
     - `additional_instructions` — optional writer constraints
     - `use_header_image` — `true` or `false`
     - `header_image_source` — `ai`, `pexels`, or `auto`
     - `image_count` — e.g. `3`
     - `image_source` — `ai`, `pexels`, or `auto`
     - `cloudinary_cloud_name`
     - `cloudinary_upload_preset`
     - `wp_post_status` — `draft` or `publish`
     - `wp_author_id`
     - `wp_category_id`

4. **Add a Brave Search node named `Brave Search1`.**
   - Connect `Set Config -> Brave Search1`.
   - Configure it to search news/trending content using expressions from the Set node.
   - Use query from `{{$json.search_query}}`.
   - If supported, also map locale/country/language fields from config.
   - Set result count high enough to allow filtering later.

5. **Add a Code node named `Format Search Results1`.**
   - Connect `Brave Search1 -> Format Search Results1`.
   - Write code to:
     - take returned results
     - keep only the first `top_results_count`
     - build a structured text block for the LLM
     - preserve raw result objects for later reference
   - Include title, URL, snippet, date if available.

6. **Add a Chat Model node named `OpenAI Chat Model`.**
   - Type: `OpenAI Chat Model`
   - Configure OpenAI credentials.
   - Choose a model appropriate for summarization, e.g. `gpt-4.1-mini`.

7. **Add an LLM Chain node named `Make sumarry`.**
   - Connect `Format Search Results1 -> Make sumarry`.
   - Connect `OpenAI Chat Model` to the AI model input of `Make sumarry`.
   - Prompt it to summarize the top search results, identify promising article angles, and output a concise structured overview.

8. **Add a Code node named `Code in JavaScript2`.**
   - Connect `Make sumarry -> Code in JavaScript2`.
   - Transform the summary into the item structure expected by your loop logic.
   - If only one candidate should proceed, still normalize it as an array.

9. **Add a `Split In Batches` node named `Loop Over Items`.**
   - Connect `Code in JavaScript2 -> Loop Over Items`.
   - Batch size is typically `1`.

10. **Add an Aggregate node named `Aggregate`.**
    - Connect the first output of `Loop Over Items -> Aggregate`.
    - Aggregate loop context into a single item for notification text generation.

11. **Add another OpenAI Chat Model node named `OpenAI Chat Model2`.**
    - Configure OpenAI credentials and model.

12. **Add an LLM Chain node named `Create Notification`.**
    - Connect `Aggregate -> Create Notification`.
    - Connect `OpenAI Chat Model2` as its AI model.
    - Prompt it to produce a short Telegram-friendly summary of the selected topic or workflow status.

13. **Add a Telegram node named `Send a text message`.**
    - Connect `Create Notification -> Send a text message`.
    - Configure Telegram bot credentials.
    - Set the chat ID and map the message text from the chain output.

14. **Add a Set node named `Set Data in Loop`.**
    - Connect the second/loop path of `Loop Over Items -> Set Data in Loop`.
    - Use it to prepare the planning payload:
      - selected article data
      - source URL
      - search summary
      - `brand_brief`
      - `target_word_count`
      - `keywords`
      - `additional_instructions`
      - image settings

15. **Add an OpenAI Chat Model node named `OpenAI — Plan Agent1`.**
    - Configure OpenAI credentials.
    - Use `gpt-4.1-mini` or equivalent.

16. **Add an AI Agent node named `Idea and Plan Agent1`.**
    - Connect `Set Data in Loop -> Idea and Plan Agent1`.
    - Connect `OpenAI — Plan Agent1` to its language model input.
    - Enable retry on fail.
    - Prompt the agent to:
      - choose the best story/angle
      - generate `master_plan`
      - generate `image_prompts` array
      - generate `header_image_prompt`
      - return strict JSON

17. **Add a Code node named `Create JSON1`.**
    - Connect `Idea and Plan Agent1 -> Create JSON1`.
    - Set error mode to continue regular output.
    - Implement parsing logic to:
      - strip code fences
      - replace smart quotes
      - remove trailing commas
      - parse JSON
      - throw descriptive errors if final parse fails
    - Output clean fields such as:
      - `master_plan`
      - `image_prompts`
      - `header_image_prompt`

18. **Add a Split Out node named `Split Out1`.**
    - Connect `Create JSON1 -> Split Out1`.
    - Configure it to split the `image_prompts` array.

19. **Add an OpenAI node named `Generate an image3`.**
    - Connect `Split Out1 -> Generate an image3`.
    - Configure image generation with OpenAI credentials.
    - Pass each prompt into the image request.
    - Enable retry on fail.

20. **Add an HTTP Request node named `Upload Image to Wordpress`.**
    - Connect `Generate an image3 -> Upload Image to Wordpress`.
    - Configure WordPress media upload:
      - Method: usually `POST`
      - Endpoint: `/wp-json/wp/v2/media`
      - Auth: WordPress application password, basic auth, or other supported method
      - Content-Type: multipart/form-data or correct media payload
    - Enable “Continue on Fail”.

21. **Add a Set node named `Set Output Image1`.**
    - Connect `Upload Image to Wordpress -> Set Output Image1`.
    - Map the response into a normalized structure:
      - uploaded media ID
      - source URL
      - original prompt
      - optional alt text

22. **Add an Aggregate node named `Aggregate1`.**
    - Connect `Set Output Image1 -> Aggregate1`.
    - Aggregate all generated image objects into a single array.

23. **Add a Set node named `set images`.**
    - Connect `Aggregate1 -> set images`.
    - Store the aggregated body images in a field like `images`.

24. **Add a Merge node named `Merge`.**
    - Connect `Create JSON1 -> Merge` as input 1.
    - Connect `set images -> Merge` as input 2.
    - Use a combine mode that merges both branches into one item.

25. **Add an Aggregate node named `Aggregate for Planner`.**
    - Connect `Merge -> Aggregate for Planner`.
    - Ensure the output is a single object containing:
      - `master_plan`
      - `images`
      - any metadata needed downstream

26. **Add an OpenAI Chat Model node named `OpenAI — Article Planner`.**
    - Configure OpenAI credentials.
    - Use a cost-efficient model such as `gpt-4.1-mini`.

27. **Add an AI Agent node named `Article Planner`.**
    - Connect `Aggregate for Planner -> Article Planner`.
    - Connect `OpenAI — Article Planner` to its model input.
    - Enable retry on fail.
    - Prompt it to divide the article into sequential parts with:
      - part title
      - objectives
      - outline
      - assigned image URL
      - source URL where relevant
      - target word count
      - consistent brand voice

28. **Add a Code node named `Code in JavaScript`.**
    - Connect `Article Planner -> Code in JavaScript`.
    - Normalize the planner output into an array like `parts`.

29. **Add a Split Out node named `Split Out2`.**
    - Connect `Code in JavaScript -> Split Out2`.
    - Split the `parts` array.

30. **Add an OpenAI Chat Model node named `OpenAI Chat Model1`.**
    - Configure OpenAI credentials for section writing.

31. **Add an LLM Chain node named `Article wrtiter`.**
    - Connect `Split Out2 -> Article wrtiter`.
    - Connect `OpenAI Chat Model1` to the AI model input.
    - Enable retry on fail.
    - Prompt it to write each article section in HTML or the format you want, respecting:
      - part title
      - outline
      - target words
      - tone
      - image placement rules
      - SEO guidance

32. **Add an Aggregate node named `Aggregate2`.**
    - Connect `Article wrtiter -> Aggregate2`.
    - Aggregate all written parts in order.

33. **Add a Code node named `Code in JavaScript1`.**
    - Connect `Aggregate2 -> Code in JavaScript1`.
    - Assemble final post fields:
      - title
      - full article content
      - optional excerpt draft
      - slug if needed

34. **Add a Code node named `clean html1`.**
    - Connect `Code in JavaScript1 -> clean html1`.
    - Clean the output:
      - remove markdown leftovers
      - normalize heading structure
      - fix malformed HTML
      - ensure image tags/paragraphs are valid

35. **Add a WordPress node named `Wordpress`.**
    - Connect `clean html1 -> Wordpress`.
    - Configure WordPress credentials.
    - Operation: create post.
    - Map:
      - title
      - content
      - status from `wp_post_status`
      - author from `wp_author_id`
      - category from `wp_category_id`

36. **Add an HTTP Request node named `WordPress excerpt add`.**
    - Connect `Wordpress -> WordPress excerpt add`.
    - Configure a request to update the just-created post with excerpt or additional fields.
    - Use the returned post ID from `Wordpress`.
    - Auth must match your WordPress REST API method.

37. **Add an OpenAI node named `OpenAI`.**
    - Connect `WordPress excerpt add -> OpenAI`.
    - Use it to generate the featured image from `header_image_prompt`.
    - This requires OpenAI image credentials.
    - If you want optional featured images, add conditional logic around this step when rebuilding.

38. **Add an HTTP Request node named `Upload Image to Wordpress2`.**
    - Connect `OpenAI -> Upload Image to Wordpress2`.
    - Upload the featured image to the WordPress media endpoint.
    - Enable continue on fail.

39. **Add an HTTP Request node named `Set Image on Wordpress Post2`.**
    - Connect `Upload Image to Wordpress2 -> Set Image on Wordpress Post2`.
    - Update the created post using:
      - post ID from `Wordpress`
      - media ID from `Upload Image to Wordpress2`
    - Set `featured_media`.

40. **Add a Set node named `Set results data`.**
    - Connect `Set Image on Wordpress Post2 -> Set results data`.
    - Normalize final result values such as:
      - post ID
      - post URL
      - featured image status
      - execution status

41. **Close the loop by connecting `Set results data -> Loop Over Items`.**
    - This completes the batch-processing cycle.

42. **Optionally add Sticky Notes** for maintainability.
    - In the original export, all sticky notes are empty.
    - You can add your own labels for:
      - Search
      - Planning
      - Image generation
      - Writing
      - Publishing
      - Notification

## Credential configuration required

- **OpenAI credentials**
  - Needed for:
    - OpenAI Chat Model
    - OpenAI Chat Model1
    - OpenAI Chat Model2
    - OpenAI — Plan Agent1
    - OpenAI — Article Planner
    - Generate an image3
    - OpenAI
- **WordPress credentials**
  - Needed for:
    - Wordpress node
    - HTTP Request media upload/update nodes if using Basic Auth or Application Password
- **Telegram bot credentials**
  - Needed for:
    - Send a text message
- **Brave Search credentials or node setup**
  - Needed for:
    - Brave Search1

## Sub-workflow setup
This workflow does **not** invoke any sub-workflows. No `Execute Workflow` node is present.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow includes a strong centralized configuration pattern via `Set Config`, which should be edited before first execution. | Configuration strategy |
| All sticky notes in the provided export are empty, so no visual documentation text was preserved from the canvas. | Annotation layer |
| The workflow tolerates some media and parsing failures by using continue-on-fail behavior in selected nodes such as `Create JSON1`, `Upload Image to Wordpress`, `Upload Image to Wordpress2`, and `Set Image on Wordpress Post2`. | Error-handling design |
| There is no explicit conditional branch shown for `image_source`, `header_image_source`, or `use_header_image`; if those settings must alter execution, additional IF/Switch logic may be required during implementation. | Architecture consideration |
| The workflow title suggests Brave Search + OpenAI + WordPress SEO draft publishing from trending news, and the actual graph matches that end-to-end publishing pattern. | Project context |