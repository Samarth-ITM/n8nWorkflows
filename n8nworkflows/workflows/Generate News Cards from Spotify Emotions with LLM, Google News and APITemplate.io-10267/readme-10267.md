Generate News Cards from Spotify Emotions with LLM, Google News and APITemplate.io

https://n8nworkflows.xyz/workflows/generate-news-cards-from-spotify-emotions-with-llm--google-news-and-apitemplate-io-10267


# Generate News Cards from Spotify Emotions with LLM, Google News and APITemplate.io

### 1. Workflow Overview

This workflow, titled **Spotify Emotion-to-News Card Generator (APITemplate.io + Slack)**, automates the generation and sharing of personalized news cards based on the emotional tone of your recently played Spotify tracks. It is designed for music enthusiasts, marketers, or developers who want to create daily visual digests or Slack updates that reflect their current listening mood.

**Logical blocks:**

- **1.1 Scheduled Trigger and Spotify Data Fetch**  
  Periodically triggers the workflow and retrieves the user’s recently played Spotify tracks.

- **1.2 Emotion Analysis using LLM**  
  Uses a large language model (LLM) via OpenRouter to infer the predominant emotion conveyed by each track.

- **1.3 Per-Track Processing (Batch)**  
  Processes each track individually to generate a related news card.

- **1.4 Google News RSS Feed Query and Fetch**  
  Constructs a Google News RSS feed URL based on the inferred emotion, fetches top news articles.

- **1.5 News Selection and Formatting**  
  Selects the top news article and prepares data for card generation.

- **1.6 News Card Generation via APITemplate.io**  
  Generates a visual news card image using APITemplate.io, embedding emotion and news information.

- **1.7 Slack Posting**  
  Posts the news card, along with the news title and link, into a designated Slack channel.


---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger and Spotify Data Fetch

- **Overview:**  
  This block initiates the workflow periodically and retrieves the user’s recently played tracks from Spotify.

- **Nodes Involved:**  
  - Start on Schedule (Cron)  
  - Fetch Spotify Recently Played

- **Node Details:**

  - **Start on Schedule (Cron)**  
    - *Type:* Schedule Trigger  
    - *Role:* Initiates workflow execution on a fixed schedule (default: every minute)  
    - *Configuration:* Default interval trigger; adjustable in node parameters  
    - *Inputs:* None (trigger node)  
    - *Outputs:* To "Fetch Spotify Recently Played"  
    - *Edge Cases:* Scheduling misconfiguration may cause too frequent or infrequent runs.

  - **Fetch Spotify Recently Played**  
    - *Type:* Spotify Node  
    - *Role:* Retrieves recently played tracks from the authenticated Spotify account  
    - *Configuration:* Operation set to "recentlyPlayed"  
    - *Credentials:* Spotify OAuth2 (user's Spotify account)  
    - *Inputs:* Trigger from Cron  
    - *Outputs:* Track data to "Emotion Analyzer"  
    - *Edge Cases:* Spotify API rate limits, token expiration, no recent tracks available.

#### 2.2 Emotion Analysis using LLM

- **Overview:**  
  Analyzes each track’s title and artist to infer the main emotion conveyed using a language model.

- **Nodes Involved:**  
  - Emotion Analyzer  
  - LLM: Infer Emotion from Track

- **Node Details:**

  - **Emotion Analyzer**  
    - *Type:* LangChain Agent (Custom LLM prompt)  
    - *Role:* Formats input text for emotion inference and sends it to the LLM node  
    - *Configuration:* Prompt instructs to analyze song title and artist to output emotion JSON (e.g., joyful, nostalgic) with reason  
    - *Key Expression:* `{{ $json.track.name }} - {{ $json.track.album.artists[0].name }}` used as input text  
    - *Inputs:* From "Fetch Spotify Recently Played"  
    - *Outputs:* To "For Each Track (Batch)"  
    - *Edge Cases:* Unexpected input structure, LLM API errors, malformed JSON output.

  - **LLM: Infer Emotion from Track**  
    - *Type:* LangChain OpenRouter Chat Model  
    - *Role:* Executes the actual language model request using OpenRouter API  
    - *Credentials:* OpenRouter API key  
    - *Inputs:* Called internally by the "Emotion Analyzer" agent node  
    - *Outputs:* Emotion JSON back to "Emotion Analyzer"  
    - *Edge Cases:* API key invalid, rate limits, network timeouts.

#### 2.3 Per-Track Processing (Batch)

- **Overview:**  
  Processes each track’s emotion analysis output individually to proceed with news fetching and card generation.

- **Nodes Involved:**  
  - For Each Track (Batch)

- **Node Details:**

  - **For Each Track (Batch)**  
    - *Type:* Split In Batches  
    - *Role:* Iteratively processes each track one at a time downstream  
    - *Configuration:* Default batch size (1)  
    - *Inputs:* From "Emotion Analyzer"  
    - *Outputs:* Two outputs:  
      - Main (empty, no direct continuation)  
      - Second output to "Build Google News RSS Query"  
    - *Edge Cases:* Large batch size may cause performance issues; empty inputs.

#### 2.4 Google News RSS Feed Query and Fetch

- **Overview:**  
  Builds a Google News RSS query URL based on the inferred emotion keyword, then retrieves news articles matching that emotion.

- **Nodes Involved:**  
  - Build Google News RSS Query  
  - Fetch Google News RSS

- **Node Details:**

  - **Build Google News RSS Query**  
    - *Type:* Set Node  
    - *Role:* Constructs a Google News RSS feed URL using the emotion keyword  
    - *Configuration:*  
      - Extracts `emotion` from the current item JSON (or parsed LLM output)  
      - Cleans whitespace and encodes for URL query  
      - Constructs URL like `https://news.google.com/rss/search?hl=en-US&gl=US&ceid=US%3Aen&q=emotion`  
    - *Inputs:* From "For Each Track (Batch)"  
    - *Outputs:* To "Fetch Google News RSS"  
    - *Edge Cases:* Missing emotion field, malformed JSON, localization parameters (hl, gl) may be customized.

  - **Fetch Google News RSS**  
    - *Type:* RSS Feed Read  
    - *Role:* Reads RSS feed from the constructed URL, fetching news articles  
    - *Configuration:* URL dynamically set from previous node’s output  
    - *Inputs:* From "Build Google News RSS Query"  
    - *Outputs:* To "Pick Top News & Format"  
    - *Edge Cases:* Network failures, empty RSS feed, RSS format issues.

#### 2.5 News Selection and Formatting

- **Overview:**  
  Selects the top news article from the RSS feed and prepares its details for the news card generation.

- **Nodes Involved:**  
  - Pick Top News & Format

- **Node Details:**

  - **Pick Top News & Format**  
    - *Type:* Code (JavaScript) Node  
    - *Role:* Selects the first RSS feed item (top news)  
    - *Configuration:* Returns only the first item using `return items.slice(0, 1);`  
    - *Inputs:* From "Fetch Google News RSS"  
    - *Outputs:* To "Generate News Card (APITemplate)"  
    - *Edge Cases:* Empty input list, malformed RSS items.

#### 2.6 News Card Generation via APITemplate.io

- **Overview:**  
  Generates a visual news card image incorporating the inferred emotion and top news article details.

- **Nodes Involved:**  
  - Generate News Card (APITemplate)

- **Node Details:**

  - **Generate News Card (APITemplate)**  
    - *Type:* APITemplate.io Node  
    - *Role:* Sends data to APITemplate.io to render an image card  
    - *Configuration:*  
      - JSON parameters override with fields: `text_emotion`, `text_url`, `text_1` (news snippet), `text_2` (news title)  
      - Template ID is dynamically assigned (user should replace with their own)  
    - *Credentials:* APITemplate.io API key  
    - *Inputs:* From "Pick Top News & Format"  
    - *Outputs:* To "Post to Slack (Title + Link + Card URL)"  
    - *Edge Cases:* Invalid template ID, API errors, missing fields.

#### 2.7 Slack Posting

- **Overview:**  
  Posts the generated news card image, along with the news title and link, into a specified Slack channel.

- **Nodes Involved:**  
  - Post to Slack (Title + Link + Card URL)

- **Node Details:**

  - **Post to Slack (Title + Link + Card URL)**  
    - *Type:* Slack Node  
    - *Role:* Sends a message to Slack containing the news title, link, and generated card URL  
    - *Configuration:*  
      - Text message concatenates: news title, news URL, label "📰 ニュースカード", and card image URL (PNG or fallback)  
      - Posting channel set to a specific Slack channel (e.g., #general)  
      - Unfurling of links and media disabled for cleaner appearance  
    - *Credentials:* Slack OAuth2 credentials  
    - *Inputs:* From "Generate News Card (APITemplate)"  
    - *Outputs:* None (end node)  
    - *Edge Cases:* Slack API rate limits, invalid channel ID, OAuth token expiration.

---

### 3. Summary Table

| Node Name                     | Node Type                        | Functional Role                             | Input Node(s)               | Output Node(s)                   | Sticky Note                                                                                                                                                                     |
|-------------------------------|---------------------------------|---------------------------------------------|-----------------------------|---------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Start on Schedule (Cron)       | Schedule Trigger                | Triggers workflow on scheduled interval     | None                        | Fetch Spotify Recently Played    | “Triggers this workflow on a fixed schedule. You can adjust interval in node settings.”                                                                                        |
| Fetch Spotify Recently Played  | Spotify Node                   | Retrieves recently played Spotify tracks    | Start on Schedule (Cron)      | Emotion Analyzer                | “Retrieves your last played Spotify tracks. Requires Spotify OAuth2 credentials.”                                                                                              |
| Emotion Analyzer              | LangChain Agent                | Formats input and calls LLM to infer emotion | Fetch Spotify Recently Played | For Each Track (Batch)           | “Uses LLM (OpenRouter) to infer the dominant emotion from song title and artist.”                                                                                              |
| LLM: Infer Emotion from Track | LangChain OpenRouter Chat Model | Executes LLM API call for emotion inference | Called internally by Emotion Analyzer | Emotion Analyzer (returns result) |                                                                                                                                                                                 |
| For Each Track (Batch)         | Split In Batches               | Processes each track individually            | Emotion Analyzer             | Build Google News RSS Query      | “Processes each track individually for emotion-to-news mapping.”                                                                                                              |
| Build Google News RSS Query    | Set Node                      | Constructs Google News RSS feed URL          | For Each Track (Batch)       | Fetch Google News RSS            | “Converts emotion into a Google News RSS query string. Modify hl and gl for language/country.”                                                                                  |
| Fetch Google News RSS          | RSS Feed Read                 | Reads news articles from Google News RSS     | Build Google News RSS Query  | Pick Top News & Format           | “Reads top articles matching the emotion keyword.”                                                                                                                             |
| Pick Top News & Format         | Code Node                    | Selects top news article from RSS feed       | Fetch Google News RSS        | Generate News Card (APITemplate) | “Selects the first article from RSS feed and prepares title/link/snippet.”                                                                                                      |
| Generate News Card (APITemplate) | APITemplate.io               | Generates news card image                      | Pick Top News & Format       | Post to Slack (Title + Link + Card URL) | “Creates a news card image using APITemplate.io. Replace template ID with yours.”                                                                                               |
| Post to Slack (Title + Link + Card URL) | Slack Node           | Posts news card and text message to Slack    | Generate News Card (APITemplate) | None                        | “Posts the generated card, title, and link to Slack. Unfurling disabled for clean appearance.”                                                                                  |
| Sticky Note                   | Sticky Note                   | Workflow overview and general information    | None                        | None                           | See detailed content in block 5 below                                                                                                                                            |
| Sticky Note1                  | Sticky Note                   | Comment on Cron node                          | None                        | None                           | “Triggers this workflow on a fixed schedule. You can adjust interval in node settings.”                                                                                        |
| Sticky Note2                  | Sticky Note                   | Comment on Spotify node                       | None                        | None                           | “Retrieves your last played Spotify tracks. Requires Spotify OAuth2 credentials.”                                                                                              |
| Sticky Note3                  | Sticky Note                   | Comment on Emotion Analyzer node              | None                        | None                           | “Uses LLM (OpenRouter) to infer the dominant emotion from song title and artist.”                                                                                              |
| Sticky Note4                  | Sticky Note                   | Comment on For Each Track node                | None                        | None                           | “Processes each track individually for emotion-to-news mapping.”                                                                                                              |
| Sticky Note5                  | Sticky Note                   | Comment on Build RSS Query node               | None                        | None                           | “Converts emotion into a Google News RSS query string. Modify hl and gl for language/country.”                                                                                  |
| Sticky Note6                  | Sticky Note                   | Comment on Fetch Google News RSS node         | None                        | None                           | “Reads top articles matching the emotion keyword.”                                                                                                                             |
| Sticky Note7                  | Sticky Note                   | Comment on Pick Top News & Format node        | None                        | None                           | “Selects the first article from RSS feed and prepares title/link/snippet.”                                                                                                      |
| Sticky Note8                  | Sticky Note                   | Comment on Generate News Card node             | None                        | None                           | “Creates a news card image using APITemplate.io. Replace template ID with yours.”                                                                                               |
| Sticky Note9                  | Sticky Note                   | Comment on Slack Posting node                  | None                        | None                           | “Posts the generated card, title, and link to Slack. Unfurling disabled for clean appearance.”                                                                                  |
| Sticky Note10                 | Sticky Note                   | Setup instructions and prerequisites          | None                        | None                           | Detailed setup and credential creation steps (see section 5 below)                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Credentials:**
   - Spotify OAuth2: Obtain client ID/secret from Spotify Developer Portal, create OAuth2 credential in n8n.
   - OpenRouter API: Obtain API key from OpenRouter, create credential in n8n.
   - APITemplate.io API: Get API key from APITemplate.io, create credential in n8n.
   - Slack OAuth2: Create Slack app with appropriate scopes, generate OAuth2 credentials in n8n.

2. **Create node: _Start on Schedule (Cron)_**
   - Type: Schedule Trigger  
   - Set interval (e.g., every hour or minute as preferred).

3. **Create node: _Fetch Spotify Recently Played_**
   - Type: Spotify Node  
   - Operation: recentlyPlayed  
   - Credentials: Assign Spotify OAuth2 credential  
   - Connect output of Cron node to this node.

4. **Create node: _Emotion Analyzer_**
   - Type: LangChain Agent (Community node)  
   - Parameters:  
     - Prompt:  
       ```
       You are an emotion analyst.
       Given the following song title and artist, infer the main emotion it conveys (e.g., joyful, nostalgic, melancholic, energetic, angry, romantic).
       Return JSON.
       
       Input:
       "{{ $json.track.name }} - {{ $json.track.album.artists[0].name }}"
       
       Output:
       {
         "emotion": "nostalgic",
         "reason": "The song's tone and lyrics evoke reflection and memory."
       }
       ```
   - Connect output of Spotify node to this node.

5. **Create node: _LLM: Infer Emotion from Track_**
   - Type: LangChain OpenRouter Chat Model  
   - Credentials: Assign OpenRouter API key  
   - No direct connection required; invoked by Emotion Analyzer internally.

6. **Create node: _For Each Track (Batch)_**
   - Type: Split In Batches  
   - Default batch size (1)  
   - Connect output of Emotion Analyzer to this node.

7. **Create node: _Build Google News RSS Query_**
   - Type: Set Node  
   - Parameters: Set JSON output with code:  
     ```js
     return {
       feedUrl: "https://news.google.com/rss/search?hl=en-US&gl=US&ceid=US%3Aen&q=" + encodeURIComponent(
         $json.emotion || (JSON.parse($json.output || '{}').emotion) || 'music'
       )
     };
     ```
   - Connect second output of For Each Track node to this.

8. **Create node: _Fetch Google News RSS_**
   - Type: RSS Feed Read  
   - Parameters:  
     - URL: `{{$json.feedUrl}}` (from previous node)  
   - Connect output of Set node to this.

9. **Create node: _Pick Top News & Format_**
   - Type: Code Node  
   - Parameters:  
     ```js
     return items.slice(0, 1);
     ```
   - Connect output of RSS feed node to this.

10. **Create node: _Generate News Card (APITemplate)_**
    - Type: APITemplate.io Node  
    - Credentials: Assign APITemplate.io API key  
    - Parameters:  
      - JSON Parameters: true  
      - Override JSON:  
        ```json
        [
          { "name": "text_emotion", "text": "{{ $json.emotion ?? 'neutral' }}" },
          { "name": "text_url", "text": "{{ $json.link ?? 'https://example.com' }}" },
          { "name": "text_1", "text": "{{ $json.contentSnippet ?? 'Unknown source' }}" },
          { "name": "text_2", "text": "{{ $json.title ?? 'No title' }}" }
        ]
        ```  
      - Image Template ID: Replace with your APITemplate.io template ID  
    - Connect output of Pick Top News node to this.

11. **Create node: _Post to Slack (Title + Link + Card URL)_**
    - Type: Slack Node  
    - Credentials: Assign Slack OAuth2 credentials  
    - Parameters:  
      - Channel: Select target Slack channel (e.g., #general)  
      - Text:  
        ```
        {{$node["Pick Top News & Format"].json.title}}
        {{$node["Pick Top News & Format"].json.link}}
        📰 ニュースカード
        {{$node["Generate News Card (APITemplate)"].json.download_url_png || $node["Generate News Card (APITemplate)"].json.download_url}}
        ```
      - Unfurl Links: Disabled  
      - Unfurl Media: Disabled  
    - Connect output of Generate News Card node to this.

12. **Activate Workflow and Test**  
    - Run the workflow manually or wait for scheduled trigger.  
    - Verify Slack posts appear as expected with news cards.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                 | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| 📄 **Workflow Overview**: This workflow analyzes your recent Spotify listening, infers emotional tone via LLM, fetches related Google News articles, generates news cards through APITemplate.io, and posts to Slack. | Sticky Note (Node: Sticky Note, position: -1072,-352)                                              |
| Setup details: Create credentials for Spotify, Slack, APITemplate.io, and OpenRouter LLM. Assign each to the respective nodes. Adjust RSS feed language parameters and Slack message as needed.                 | Sticky Note10 (Node: Sticky Note10)                                                                |
| Replace APITemplate.io template ID with your own to customize the news card design.                                                                                                                                  | Sticky Note8                                                                                       |
| Slack messages disable link/media unfurling for clean display.                                                                                                                                                     | Sticky Note9                                                                                       |
| Google News RSS feed localization can be customized by modifying `hl` (language) and `gl` (country) query parameters in the RSS URL builder node.                                                                | Sticky Note5                                                                                       |
| The LangChain nodes used require n8n self-hosted environment due to community node dependencies.                                                                                                                   | Workflow disclaimer in overview note                                                              |
| Video and branding info are not included in this workflow.                                                                                                                                                        | N/A                                                                                                |

---

**Disclaimer:**  
The text provided is extracted exclusively from an automated n8n workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.