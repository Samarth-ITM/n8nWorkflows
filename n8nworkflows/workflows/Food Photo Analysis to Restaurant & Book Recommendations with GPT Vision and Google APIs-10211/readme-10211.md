Food Photo Analysis to Restaurant & Book Recommendations with GPT Vision and Google APIs

https://n8nworkflows.xyz/workflows/food-photo-analysis-to-restaurant---book-recommendations-with-gpt-vision-and-google-apis-10211


# Food Photo Analysis to Restaurant & Book Recommendations with GPT Vision and Google APIs

### 1. Workflow Overview

This workflow automatically analyzes a newly uploaded food photo to recommend a dining spot and a related book. It targets food enthusiasts, bloggers, and teams seeking a seamless integration that turns a single food photo into a restaurant suggestion plus a themed book recommendation.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Detects new food photos uploaded to a specific Google Drive folder.
- **1.2 Food Image Analysis:** Downloads the photo and uses an AI vision model to classify the dish and extract nutritional info as structured JSON.
- **1.3 Classification Normalization:** Parses and maps the AI output into a restaurant category and relevant search keywords.
- **1.4 Restaurant Search & Selection:** Uses Google Places API to find nearby restaurants matching the category, summarizes the results, and selects the best one via an AI strict evaluator.
- **1.5 Book Recommendation:** Infers a thematic book related to the chosen restaurant using AI, searches Google Books for matching titles, and formats detailed book information.
- **1.6 Output Posting:** Posts a formatted summary of the dish, restaurant, and book recommendation to a Slack channel.

The workflow integrates multiple APIs (Google Drive, Google Places, Google Books, Slack) and leverages advanced AI models with strict JSON outputs for reliability.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Watches a specific Google Drive folder and triggers when a new file (food photo) is added.
- **Nodes Involved:** Google Drive Trigger, Fetch Image (Drive)
- **Node Details:**

  - **Google Drive Trigger**
    - Type: Trigger node for Google Drive file events.
    - Configuration: Watches for 'fileCreated' events in a specified folder (ID: `1uoky4OAFYULqTrWkkfm8JFGBGNcLaLT0`), polling every minute.
    - Credentials: Google Drive OAuth2.
    - Inputs: None (trigger).
    - Outputs: Emits metadata of the new file, including file ID.
    - Potential failures: OAuth token expiry, permission issues on folder access, polling delays.

  - **Fetch Image (Drive)**
    - Type: HTTP Request.
    - Configuration: Downloads the image from Google Drive using the file ID from the trigger (`https://drive.google.com/uc?id={{$json.id}}&export=download`).
    - Inputs: Google Drive Trigger output.
    - Outputs: Binary image data passed to the next node.
    - Potential failures: Broken links, insufficient permissions, large file timeout.

#### 2.2 Food Image Analysis

- **Overview:** Uses an AI vision language model to identify the dish name, category, and nutrition info from the food photo, returning strictly formatted JSON.
- **Nodes Involved:** LLM (Vision/JSON), Dish Classifier
- **Node Details:**

  - **LLM (Vision/JSON)**
    - Type: Language Model Chat node using OpenRouter with GPT-5 model.
    - Configuration: No special prompt here; likely a routing node for AI input.
    - Inputs: Fetch Image outputs.
    - Outputs: Passes image and prompt to the Agent node.
    - Credentials: OpenRouter API.
    - Failure modes: Model API limits, network timeout.

  - **Dish Classifier**
    - Type: Langchain Agent node.
    - Configuration: Custom prompt instructing the AI to strictly output JSON with keys: dish_name, category, calories_kcal, protein_g, fat_g, carbs_g. Input includes the image URL from Google Drive.
    - Inputs: Receives image URL from Fetch Image (Drive).
    - Outputs: JSON describing the dish.
    - Failure modes: AI returning non-JSON output, parsing errors, poor image quality leading to misclassification.

#### 2.3 Classification Normalization

- **Overview:** Parses the JSON from the dish classifier, extracts the top-level food category, and maps it to a restaurant or cafe keyword for Places API.
- **Nodes Involved:** Normalize Classification
- **Node Details:**

  - **Normalize Classification**
    - Type: Code node (JavaScript).
    - Configuration: Parses AI output JSON safely; extracts `category` top-level keyword; applies a predefined mapping to set `type` (restaurant/cafe) and a keyword string for Google Places search.
    - Inputs: Dish Classifier output.
    - Outputs: Normalized data including dish_name, category, confidence, type, keyword.
    - Edge cases: Missing or malformed JSON, unknown categories default to generic 'restaurant' keyword.

#### 2.4 Restaurant Search & Selection

- **Overview:** Sets search origin and radius, queries Google Places API to find matching restaurants, summarizes results, and selects the best restaurant using AI evaluation.
- **Nodes Involved:** Set Origin & Radius, Pass the API, Search Google Places, Summarize Place List, Select Best Place (AI), Format Best Place JSON
- **Node Details:**

  - **Set Origin & Radius**
    - Type: Set node.
    - Configuration: Hardcoded values for search origin ("中野駅 東京都中野区") and coordinates (lat: 35.7073, lng: 139.6655), radius 1200 meters.
    - Inputs: Normalize Classification.
    - Outputs: Origin and radius parameters for the Places search.
    - Customization: User-editable for different locations.

  - **Pass the API**
    - Type: Set node (raw mode).
    - Configuration: Passes Google API key from credentials to the next node.
    - Inputs: Set Origin & Radius.
    - Outputs: API key injection.
    - Security: API key stored in credentials, not hardcoded.

  - **Search Google Places**
    - Type: HTTP Request.
    - Configuration: POST to `https://places.googleapis.com/v1/places:searchText` with JSON body containing textQuery (category from Normalize Classification), location bias circle from origin and radius, max 3 results.
    - Headers: Content-Type application/json, X-Goog-Api-Key from credentials, X-Goog-FieldMask specifying fields to retrieve.
    - Inputs: Pass the API.
    - Outputs: Raw Google Places response with places array.
    - Edge cases: Rate limits, empty results, wrong API key.

  - **Summarize Place List**
    - Type: Code node.
    - Configuration: Extracts name, rating, reviews, and short address from Places data; creates a concise multiline summary string for AI evaluation.
    - Inputs: Search Google Places.
    - Outputs: JSON with places array and summary string.
    - Edge cases: Missing fields in API response.

  - **Select Best Place (AI)**
    - Type: Langchain Agent node.
    - Configuration: AI prompt instructs strict evaluation to pick the best restaurant by highest rating, then by number of reviews as tiebreaker; returns JSON with best_place, rating, reviews, address, reason.
    - Inputs: Summarize Place List summary.
    - Outputs: AI JSON evaluation string.
    - Failure modes: AI returning prose instead of JSON; parsing needed downstream.

  - **Format Best Place JSON**
    - Type: Code node.
    - Configuration: Safely parses AI output JSON string; extracts best_place and reason; defaults to empty strings if invalid.
    - Inputs: Select Best Place (AI).
    - Outputs: Clean JSON with best_place and reason.
    - Edge cases: Malformed AI output.

#### 2.5 Book Recommendation

- **Overview:** Uses AI to recommend a book matching the best restaurant's vibe, searches Google Books for the title, and formats the book details for output.
- **Nodes Involved:** Recommend Book (AI), Normalize Book JSON, Search Google Books, Format Book Details, LLM (Books)
- **Node Details:**

  - **Recommend Book (AI)**
    - Type: Langchain Agent node.
    - Configuration: Creative prompt asking AI to infer restaurant theme and recommend a real Google Books title with author and reason; expects strict JSON output.
    - Inputs: Format Best Place JSON.best_place.
    - Outputs: JSON with restaurant, recommended_book (title, author), and reason.
    - Failure modes: AI generating incomplete or prose outputs.

  - **Normalize Book JSON**
    - Type: Code node.
    - Configuration: Strips any markdown JSON fences; ensures fields exist with defaults; outputs normalized JSON.
    - Inputs: Recommend Book (AI).
    - Outputs: Normalized book recommendation JSON.
    - Edge cases: AI output formatting issues.

  - **Search Google Books**
    - Type: Google Books node.
    - Configuration: Searches volumes using quoted recommended book title for precision.
    - Inputs: Normalize Book JSON.recommended_book.title.
    - Outputs: Array of matching book volumes.
    - Credentials: Google Books OAuth2.
    - Edge cases: No matches found, API limits.

  - **Format Book Details**
    - Type: Code node.
    - Configuration: Extracts title, authors, publisher, publishedDate, selfLink from first Google Books volume; defaults if missing.
    - Inputs: Search Google Books.
    - Outputs: Single formatted book record.
    - Edge cases: Empty search results.

  - **LLM (Books)**
    - Type: LLM Chat node (OpenRouter).
    - Configuration: Passes queries for book recommendation.
    - Inputs: Recommend Book (AI).
    - Outputs: Connected to Normalize Book JSON.
    - Credentials: OpenRouter API.

#### 2.6 Output Posting

- **Overview:** Posts a message to Slack summarizing the dish info, best restaurant, and recommended book details.
- **Nodes Involved:** Post to Slack
- **Node Details:**

  - **Post to Slack**
    - Type: Slack node.
    - Configuration: Posts to a specified Slack channel with a formatted message combining dish name, category, restaurant name, and book details including a Google Books link.
    - Inputs: Format Book Details.
    - Outputs: Slack post confirmation.
    - Credentials: Slack OAuth2.
    - Edge cases: Slack rate limits, invalid channel, missing fields causing "undefined" in message.

---

### 3. Summary Table

| Node Name               | Node Type                        | Functional Role                              | Input Node(s)             | Output Node(s)               | Sticky Note                                                                                             |
|-------------------------|---------------------------------|----------------------------------------------|---------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------|
| Google Drive Trigger     | GoogleDriveTrigger               | Detects new food photo upload                 | None                      | Fetch Image (Drive)          | Setup & Credentials: Connect OAuth for Google Drive. Configure folder and polling.                     |
| Fetch Image (Drive)      | HTTP Request                    | Downloads photo from Google Drive             | Google Drive Trigger      | Dish Classifier             | Photo → Category: downloads image for AI processing.                                                  |
| LLM (Vision/JSON)        | LLM Chat (OpenRouter GPT-5)    | Routes image to AI for vision classification  | Fetch Image (Drive)       | Dish Classifier             |                                                                                                       |
| Dish Classifier          | Langchain Agent                 | AI vision classification to JSON              | LLM (Vision/JSON)         | Normalize Classification    | Photo → Category: strict JSON output with dish info.                                                  |
| Normalize Classification | Code                           | Parses AI JSON; maps category to keywords     | Dish Classifier           | Set Origin & Radius         | Photo → Category: maps category to restaurant/cafe keywords.                                          |
| Set Origin & Radius      | Set                            | Sets search origin and radius parameters      | Normalize Classification  | Pass the API                | Find & Select Restaurant: centralized location config.                                                |
| Pass the API             | Set                            | Passes Google API key securely                 | Set Origin & Radius       | Search Google Places        | Find & Select Restaurant: keep API key in credentials, not hardcoded.                                |
| Search Google Places     | HTTP Request                   | Calls Google Places API to find restaurants   | Pass the API              | Summarize Place List        | Find & Select Restaurant: Google Places search with category keyword.                                |
| Summarize Place List     | Code                           | Extracts key info and summarizes places list  | Search Google Places      | Select Best Place (AI)      | Find & Select Restaurant: flattens API response for AI evaluation.                                   |
| Select Best Place (AI)   | Langchain Agent                | AI selects best restaurant by rating/reviews | Summarize Place List      | Format Best Place JSON      | Find & Select Restaurant: strict JSON output for best choice.                                        |
| Format Best Place JSON   | Code                           | Parses AI output JSON safely                    | Select Best Place (AI)    | Recommend Book (AI)         | Find & Select Restaurant: ensures clean best place data.                                             |
| Recommend Book (AI)      | Langchain Agent                | AI recommends book matching restaurant vibe   | Format Best Place JSON    | Normalize Book JSON         | Book Recommendation: creative AI output with strict JSON.                                           |
| Normalize Book JSON      | Code                           | Cleans AI output, ensures all fields present  | Recommend Book (AI)       | Search Google Books         | Book Recommendation: strips markdown fences, fills defaults.                                        |
| Search Google Books      | Google Books                  | Searches for recommended book on Google Books | Normalize Book JSON       | Format Book Details         | Book Recommendation: precise title search with OAuth.                                               |
| Format Book Details      | Code                           | Extracts and formats book metadata             | Search Google Books       | Post to Slack              | Book Recommendation: single book record formatting.                                                 |
| LLM (Books)              | LLM Chat (OpenRouter GPT-5)    | Facilitates AI book recommendation             | Recommend Book (AI)       | Normalize Book JSON         | Book Recommendation: connected AI node.                                                             |
| Post to Slack            | Slack                         | Posts combined dish, restaurant, and book info| Format Book Details       | None                       | Output: posts summary message to Slack channel.                                                     |
| Sticky Note              | Sticky Note                   | Documentation and overview                      | None                      | None                       | Overview and description of workflow purpose and usage.                                             |
| Sticky Note1             | Sticky Note                   | Setup and credentials instructions              | None                      | None                       | Setup & Credentials: OAuth connections, API key security, config pointers.                          |
| Sticky Note2             | Sticky Note                   | Photo classification explanation                 | None                      | None                       | Photo → Category: explains image processing nodes.                                                  |
| Sticky Note3             | Sticky Note                   | Restaurant search and selection explanation      | None                      | None                       | Find & Select Restaurant: details block nodes.                                                      |
| Sticky Note4             | Sticky Note                   | Book recommendation explanation                   | None                      | None                       | Book Recommendation: AI and Google Books nodes.                                                    |
| Sticky Note5             | Sticky Note                   | Output posting explanation                         | None                      | None                       | Output: Slack posting node info.                                                                    |
| Sticky Note6             | Sticky Note                   | Troubleshooting tips                               | None                      | None                       | Troubleshooting: common errors and fixes.                                                          |
| Sticky Note7             | Sticky Note                   | Security and review checklist                      | None                      | None                       | Security checklist: API keys, naming, privacy.                                                     |
| Sticky Note8             | Sticky Note                   | Google Books & Slack output quality tips           | None                      | None                       | Output formatting and search accuracy tips.                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Drive Trigger:**
   - Node type: Google Drive Trigger.
   - Configure to watch "fileCreated" events in a specific folder (set folder ID).
   - Set polling to every minute.
   - Connect Google Drive OAuth2 credentials.
   
2. **Add HTTP Request node to Fetch Image:**
   - Type: HTTP Request.
   - URL: `https://drive.google.com/uc?id={{$json.id}}&export=download` (replace `{{$json.id}}` with the Drive file ID).
   - Input: Connect from Google Drive Trigger.
   
3. **Add LLM Chat node (Vision/JSON):**
   - Type: Langchain LLM Chat OpenRouter.
   - Model: OpenAI GPT-5 (or similar).
   - Credentials: OpenRouter API.
   - Input: Connect from Fetch Image node.
   
4. **Add Langchain Agent node (Dish Classifier):**
   - Use prompt:
     ```
     You are a strict JSON generator. Always return a JSON object that includes only the following keys: dish_name, category, calories_kcal, protein_g, fat_g, carbs_g. Do not include any explanations.
     
     Input:
     - type: image_url with URL from Drive
     - Request dish name, category, nutrition info in JSON only.
     ```
   - Connect input from LLM (Vision/JSON).
   
5. **Add Code node (Normalize Classification):**
   - JavaScript code to parse AI JSON, extract category top-level, map to keywords/types:
     ```
     // parse output JSON safely
     // map Japanese categories to restaurant/cafe keywords for Google Places
     ```
   - Input: Dish Classifier.
   
6. **Add Set node (Set Origin & Radius):**
   - Assign variables:
     - originLabel: e.g. "中野駅 東京都中野区"
     - originLat: 35.7073
     - originLng: 139.6655
     - radiusM: 1200
   - Input: Normalize Classification.
   
7. **Add Set node (Pass the API):**
   - Mode: Raw JSON.
   - Set key `"GOOGLE_API_KEY"` with empty string placeholder (to be set in credentials).
   - Input: Set Origin & Radius.
   
8. **Add HTTP Request node (Search Google Places):**
   - POST to `https://places.googleapis.com/v1/places:searchText`.
   - Body JSON:
     ```
     {
       "textQuery": <category keyword from Normalize Classification>,
       "languageCode": "ja",
       "maxResultCount": 3,
       "locationBias": {
         "circle": {
           "center": {
             "latitude": <originLat>,
             "longitude": <originLng>
           },
           "radius": <radiusM>
         }
       }
     }
     ```
   - Headers:
     - "Content-Type": "application/json"
     - "X-Goog-Api-Key": from Pass the API node
     - "X-Goog-FieldMask": "places.displayName,places.formattedAddress,places.rating,places.id"
   - Input: Pass the API.
   - Credential: Google API key stored securely.
   
9. **Add Code node (Summarize Place List):**
   - Extract relevant fields from Places response.
   - Create a string summary listing places with rating, reviews, address.
   - Input: Search Google Places.
   
10. **Add Langchain Agent node (Select Best Place AI):**
    - Prompt to strictly select best place by rating, then reviews.
    - Output only JSON including best_place, rating, reviews, address, reason.
    - Input: Summarize Place List.
    - Credentials: OpenRouter API.
    
11. **Add Code node (Format Best Place JSON):**
    - Safely parse AI output string to JSON.
    - Extract best_place and reason with default empty strings.
    - Input: Select Best Place (AI).
    
12. **Add Langchain Agent node (Recommend Book AI):**
    - Prompt: Given restaurant name, infer theme and recommend a real book from Google Books.
    - Output strict JSON with restaurant, recommended_book.title, recommended_book.author, reason.
    - Input: Format Best Place JSON.best_place.
    - Credentials: OpenRouter API.
    
13. **Add Code node (Normalize Book JSON):**
    - Strip markdown fences.
    - Ensure all fields exist.
    - Input: Recommend Book (AI).
    
14. **Add Google Books node (Search Google Books):**
    - Operation: getAll volumes.
    - Search Query: quoted recommended_book.title, e.g. `"{{recommended_book.title}}"`
    - Credential: Google Books OAuth2.
    - Input: Normalize Book JSON.recommended_book.title.
    
15. **Add Code node (Format Book Details):**
    - Extract first book volume info: title, authors (joined), publisher, publishedDate, selfLink.
    - Provide default values if missing.
    - Input: Search Google Books.
    
16. **Add Slack node (Post to Slack):**
    - Configure channel ID (e.g., general channel).
    - Message text template combining:
      - Dish name and category.
      - Best restaurant name.
      - Book title, author(s), publisher, published date, Google Books link.
    - Credential: Slack OAuth2.
    - Input: Format Book Details.
    
17. **Connect all nodes in described order and verify all credentials are configured.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Centralize all user-editable variables in the "Set Origin & Radius" node for easy customization of search location and radius.                                                                                                           | Sticky Note1                                                                                     |
| Store all API keys and OAuth credentials securely in n8n Credentials; never hardcode secrets in HTTP headers or bodies.                                                                                                                 | Sticky Note7                                                                                     |
| If AI nodes return prose instead of JSON, tighten prompts and enable strict output format in Langchain nodes; add code nodes to parse and handle errors gracefully.                                                                      | Sticky Note6                                                                                     |
| To improve book search accuracy, use quoted titles in Google Books search queries; optionally add title filters or language restrictions.                                                                                                | Sticky Note8                                                                                     |
| Slack messages can be formatted with markdown and link label syntax `<URL|Label>` to avoid noisy link previews; disable link unfurling if needed.                                                                                       | Sticky Note8                                                                                     |
| Recommended to watch a short Loom video or include a workflow image for onboarding new users.                                                                                                                                             | Sticky Note                                                                                      |
| Google Places API requires a valid API key with Places API enabled; ensure proper billing and quota management.                                                                                                                           | Setup section                                                                                     |
| Slack OAuth2 scope must include chat:write to allow posting messages to channels.                                                                                                                                                          | Setup section                                                                                     |
| Google Drive folder must be accessible by the OAuth user and the workflow; permission errors will prevent triggering.                                                                                                                    | Input Reception section                                                                           |

---

**Disclaimer:** The provided content is exclusively derived from an n8n automated workflow following all prevailing content policies. It contains no illegal, offensive, or protected elements. All handled data are legal and public.