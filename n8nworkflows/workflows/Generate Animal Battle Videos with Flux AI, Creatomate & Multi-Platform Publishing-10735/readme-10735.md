Generate Animal Battle Videos with Flux AI, Creatomate & Multi-Platform Publishing

https://n8nworkflows.xyz/workflows/generate-animal-battle-videos-with-flux-ai--creatomate---multi-platform-publishing-10735


# Generate Animal Battle Videos with Flux AI, Creatomate & Multi-Platform Publishing

### 1. Workflow Overview

This workflow, titled **"Generate Animal Battle Videos with Flux AI, Creatomate & Multi-Platform Publishing"**, automates the creation of cinematic animal battle videos. It generates fight matchups between a main animal character and various opponents, creates photorealistic close-up images of the combatants and the battle winners, compiles these assets into a video using Creatomate, and publishes the video on multiple social media platforms (Instagram, TikTok, YouTube) via Blotato.

The workflow is composed of the following logical blocks:

- **1.1 Input Reception and Initialization:** Triggered on schedule, it fetches the main character from a Google Sheet.
- **1.2 Opponent Generation and Scene Creation:** Uses AI to generate a set of eight opponents suitable for the main character and parses the output.
- **1.3 Image Prompt Generation:** Creates photorealistic close-up image prompts for each animal (main character and opponents).
- **1.4 Image Generation and Collection:** Calls image generation APIs to produce the close-up images, then aggregates and writes results back to Google Sheets.
- **1.5 Winner Image Prompt Creation:** Generates a prompt depicting the aftermath of the battle showing the winner and loser.
- **1.6 Final Image Generation:** Produces the winner battle scene image.
- **1.7 Video Rendering:** Sends all images and metadata to Creatomate to render the final video.
- **1.8 Multi-Platform Publishing:** Uploads the final video to Blotato and posts it to Instagram, TikTok, and YouTube.

The workflow relies heavily on AI-driven nodes (Langchain agents) for content generation, HTTP nodes for API calls to Flux AI (PiAPi), Creatomate, and Blotato, and Google Sheets nodes for input/output data storage.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

**Overview:**  
This block triggers the workflow on a schedule and retrieves the main character data from a Google Sheet where tasks are marked "To Do."

**Nodes Involved:**  
- Schedule Trigger  
- Get Main Character  
- Main Character (Set node)

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow periodically (default interval, every minute)  
  - Config: No complex parameters, default interval used  
  - Inputs: None  
  - Outputs: Connects to Get Main Character  
  - Failures: None expected, but misconfiguration can cause no trigger

- **Get Main Character**  
  - Type: Google Sheets  
  - Role: Reads from a Google Sheet to find a row with Status = "To Do"  
  - Config: Sheet named "Sheet1", document ID linked to a Viral Shorts spreadsheet  
  - Filters: Lookup column "Status" must equal "To Do"  
  - Returns: First matching row only  
  - Inputs: From Schedule Trigger  
  - Outputs: To Main Character node  
  - Failures: API failure, authentication errors, no matching rows

- **Main Character**  
  - Type: Set  
  - Role: Extracts and assigns the main character name from the Google Sheet data to the variable `mainCharacter`  
  - Config: Sets `mainCharacter` with expression referencing Get Main Character's output JSON  
  - Inputs: From Get Main Character  
  - Outputs: To Scene Creator and Merge node  
  - Failures: Expression failures if input missing or malformed

---

#### 2.2 Opponent Generation and Scene Creation

**Overview:**  
Generates a list of eight opponents suitable for the main character, formatted as eight distinct scenes. Parses the output into structured JSON.

**Nodes Involved:**  
- Scene Creator (Langchain Agent)  
- Scenes (Output Parser Structured)  
- Opponents (Set)  
- Split Out  
- Merge  

**Node Details:**

- **Scene Creator**  
  - Type: Langchain Agent (OpenRouter GPT)  
  - Role: Given main character and opponent category, AI generates a list of eight scene opponents, one per scene  
  - Config: Uses system message instructing to output only eight opponent names labeled "Scene 1" to "Scene 8"  
  - Inputs: From Main Character and Get Main Character (opponents input)  
  - Outputs: To Scenes node and Opponents node  
  - Failures: API errors, malformed output, rate limits

- **Scenes**  
  - Type: Langchain Output Parser Structured  
  - Role: Parses raw AI output into JSON object with keys `scene1` to `scene8`  
  - Inputs: From Scene Creator  
  - Outputs: To Opponents node (via expression referencing parsed output)  
  - Failures: Parsing errors if output is malformed

- **Opponents**  
  - Type: Set  
  - Role: Creates an array `opponents` listing the eight opponents extracted from the parsed scenes  
  - Config: Assigns array using expressions referencing `scene1` to `scene8` from Scenes output  
  - Inputs: From Scenes  
  - Outputs: To Split Out node  
  - Failures: Expression or data availability issues

- **Split Out**  
  - Type: Split Out  
  - Role: Splits the `opponents` array into individual items for parallel processing  
  - Inputs: From Opponents  
  - Outputs: To Merge node and subsequently to Image Prompt Generator  
  - Failures: Empty arrays, invalid data types

- **Merge**  
  - Type: Merge (Combine)  
  - Role: Combines main character data and opponents back into a single data item for subsequent processing  
  - Inputs: From Main Character (mainCharacter) and Split Out (opponents)  
  - Outputs: To Image Prompt Generator and Winner Image Prompt nodes  
  - Failures: Mismatched data, connection issues

---

#### 2.3 Image Prompt Generation

**Overview:**  
Generates vivid text prompts for photorealistic close-up images of both the main character and each opponent using AI.

**Nodes Involved:**  
- Image Prompt Generator (Langchain Agent)  
- Split Out1

**Node Details:**

- **Image Prompt Generator**  
  - Type: Langchain Agent (OpenRouter GPT)  
  - Role: Creates two detailed text-to-image prompts per animal pair (main animal and opponent) emphasizing fierce, photorealistic close-ups  
  - Config: System message instructs to focus on hyper-realistic, aggressive, and detailed animal imagery without background stories or camera details  
  - Inputs: From Merge node (contains mainCharacter and individual opponent)  
  - Outputs: To Split Out1 node  
  - Failures: AI model errors, malformed output, rate limiting

- **Split Out1**  
  - Type: Split Out  
  - Role: Splits the AI output containing two prompts into separate items for individual image generation API calls  
  - Inputs: From Image Prompt Generator  
  - Outputs: To Generate Close Ups node  
  - Failures: Empty or invalid output

---

#### 2.4 Image Generation and Collection

**Overview:**  
Calls Flux AI (PiAPi) API to generate images based on the prompts, waits for completion, aggregates results, updates Google Sheets with image URLs.

**Nodes Involved:**  
- Generate Close Ups (HTTP Request)  
- 90 seconds (Wait)  
- Get Close Ups (HTTP Request)  
- Aggregate  
- Add Close Ups (Google Sheets)  
- Merge1  
- Get Elements (Google Sheets)  
- Render Video (HTTP Request)  
- 90_Seconds (Wait)  
- Get Video (HTTP Request)  
- Google Sheets (Update)  
- Upload to Blotato (HTTP Request)

**Node Details:**

- **Generate Close Ups**  
  - Type: HTTP Request  
  - Role: Sends text-to-image generation requests to Flux AI (PiAPi) with prompts, specifying dimensions 1024x1024  
  - Config: Authenticated with generic HTTP header containing API key  
  - Inputs: From Split Out1 (individual prompts)  
  - Outputs: To 90 seconds wait node  
  - Failures: API errors, auth failure, rate limits

- **90 seconds**  
  - Type: Wait  
  - Role: Delays workflow for 90 seconds to allow image generation to complete  
  - Inputs: From Generate Close Ups  
  - Outputs: To Get Close Ups  
  - Failures: None expected

- **Get Close Ups**  
  - Type: HTTP Request  
  - Role: Polls Flux AI API for generated image results using task ID from previous node  
  - Inputs: From 90 seconds  
  - Outputs: To Aggregate  
  - Failures: API errors, timeout, invalid task ID

- **Aggregate**  
  - Type: Aggregate  
  - Role: Combines multiple image results into a single data set for Google Sheets update  
  - Inputs: From Get Close Ups  
  - Outputs: To Add Close Ups  
  - Failures: Empty data, aggregation issues

- **Add Close Ups**  
  - Type: Google Sheets  
  - Role: Updates the Google Sheet rows for the main character with generated close-up image URLs indexed by scene and position  
  - Config: Matches by "Main Character" column, updates fields like "1.1", "1.2", etc.  
  - Inputs: From Aggregate  
  - Outputs: To Merge1  
  - Failures: API errors, authentication, sheet locks, data mismatches

- **Merge1**  
  - Type: Merge (Combine by Position)  
  - Role: Combines image URLs with other data for next steps (winner image prompt generation)  
  - Inputs: From Add Close Ups and Aggregate1 (winner images)  
  - Outputs: To Get Elements  
  - Failures: Data mismatch, connection issues

- **Get Elements**  
  - Type: Google Sheets  
  - Role: Reads the updated row for the main character to collect all image URLs and metadata for video rendering  
  - Inputs: From Merge1  
  - Outputs: To Render Video  
  - Failures: API errors, missing data

- **Render Video**  
  - Type: HTTP Request  
  - Role: Sends a request to Creatomate API to render the video using a template and modifications with all gathered images and music  
  - Config: Uses JSON body with template ID, dynamic image URLs, and authorization header with API Key  
  - Inputs: From Get Elements  
  - Outputs: To 90_Seconds (Wait)  
  - Failures: API errors, invalid template ID, network issues, malformed JSON

- **90_Seconds**  
  - Type: Wait  
  - Role: Waits 90 seconds for the video rendering process to complete on Creatomate  
  - Inputs: From Render Video  
  - Outputs: To Get Video  
  - Failures: None expected

- **Get Video**  
  - Type: HTTP Request  
  - Role: Retrieves the rendered video URL from Creatomate using the URL returned from rendering step  
  - Inputs: From 90_Seconds  
  - Outputs: To Google Sheets node for final update  
  - Failures: API errors, video not ready

- **Google Sheets (Update Final Video)**  
  - Type: Google Sheets  
  - Role: Updates the Google Sheet row status to "Created" and writes the final video URL  
  - Inputs: From Get Video  
  - Outputs: To Upload to Blotato  
  - Failures: API errors, authentication

- **Upload to Blotato**  
  - Type: HTTP Request  
  - Role: Uploads the final video URL to Blotato's media endpoint to enable social media publishing  
  - Config: Authenticated with API key, sends video URL in POST body  
  - Inputs: From Google Sheets update  
  - Outputs: To Instagram, TikTok, YouTube nodes  
  - Failures: API errors, auth failure, network issues

---

#### 2.5 Winner Image Prompt Creation and Generation

**Overview:**  
Generates a photorealistic image prompt illustrating the winner of each battle standing over the loser, then generates the corresponding image.

**Nodes Involved:**  
- Winner Image Prompt (Langchain Agent)  
- Generate Scene (HTTP Request)  
- 90_seconds (Wait)  
- Get Winners (HTTP Request)  
- Aggregate1  
- Add Winner (Google Sheets)

**Node Details:**

- **Winner Image Prompt**  
  - Type: Langchain Agent (OpenRouter GPT)  
  - Role: Creates a detailed single paragraph prompt showing the aftermath of the fight with a winner standing dominantly over the loser, including battle scars and lifeless loser  
  - Inputs: From Merge node with main character and opponents  
  - Outputs: To Generate Scene  
  - Failures: API errors, malformed output

- **Generate Scene**  
  - Type: HTTP Request  
  - Role: Sends prompt to Flux AI (PiAPi) for text-to-image generation (540x960) of the winner scene  
  - Inputs: From Winner Image Prompt  
  - Outputs: To 90_seconds wait node  
  - Failures: API errors, auth failure

- **90_seconds**  
  - Type: Wait  
  - Role: Pauses 90 seconds to allow image generation to complete  
  - Inputs: From Generate Scene  
  - Outputs: To Get Winners  
  - Failures: None expected

- **Get Winners**  
  - Type: HTTP Request  
  - Role: Polls Flux AI API to retrieve winner image results by task ID  
  - Inputs: From 90_seconds  
  - Outputs: To Aggregate1  
  - Failures: API errors, invalid task ID

- **Aggregate1**  
  - Type: Aggregate  
  - Role: Aggregates winner images data for updating Google Sheets  
  - Inputs: From Get Winners  
  - Outputs: To Add Winner  
  - Failures: Empty data, aggregation errors

- **Add Winner**  
  - Type: Google Sheets  
  - Role: Updates Google Sheets with URLs of winner images by scene for the main character  
  - Inputs: From Aggregate1  
  - Outputs: To Merge1  
  - Failures: API errors, authentication, data conflicts

---

#### 2.6 Multi-Platform Publishing

**Overview:**  
Publishes the final rendered video on Instagram, TikTok, and YouTube using Blotato API.

**Nodes Involved:**  
- Instagram (HTTP Request)  
- TikTok (HTTP Request)  
- YouTube (HTTP Request)

**Node Details:**

- **Instagram**  
  - Type: HTTP Request  
  - Role: Posts video to Instagram via Blotato API with caption including main character vs opponents  
  - Config: Authenticated with API key, platform-specific JSON body  
  - Inputs: From Upload to Blotato  
  - Outputs: None  
  - Failures: API errors, auth failure, rate limits

- **TikTok**  
  - Type: HTTP Request  
  - Role: Posts video to TikTok via Blotato API with detailed settings (public, AI generated, no duet/stitch/comments disabled false)  
  - Config: Similar to Instagram node, with TikTok-specific parameters  
  - Inputs: From Upload to Blotato  
  - Outputs: None  
  - Failures: API errors, auth failure

- **YouTube**  
  - Type: HTTP Request  
  - Role: Posts video to YouTube via Blotato API, unlisted privacy, no notification to subscribers  
  - Inputs: From Upload to Blotato  
  - Outputs: None  
  - Failures: API errors, auth failure

---

### 3. Summary Table

| Node Name            | Node Type                              | Functional Role                               | Input Node(s)               | Output Node(s)                       | Sticky Note                                                                                                                   |
|----------------------|--------------------------------------|-----------------------------------------------|-----------------------------|------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger      | Schedule Trigger                     | Initiates workflow on schedule                 | None                        | Get Main Character                 |                                                                                                                              |
| Get Main Character    | Google Sheets                       | Retrieves main character with Status "To Do"  | Schedule Trigger            | Main Character, Scene Creator      |                                                                                                                              |
| Main Character       | Set                                 | Sets mainCharacter variable                     | Get Main Character          | Merge, Scene Creator               |                                                                                                                              |
| Scene Creator        | Langchain Agent (OpenRouter GPT)    | Generates 8 opponents for main character       | Get Main Character, Main Character | Scenes, Opponents                 |                                                                                                                              |
| Scenes               | Output Parser Structured             | Parses AI output into structured JSON          | Scene Creator               | Opponents                        |                                                                                                                              |
| Opponents            | Set                                 | Creates array of opponent names                 | Scenes                      | Split Out                       |                                                                                                                              |
| Split Out            | Split Out                          | Splits opponents array into single items       | Opponents                   | Merge                           |                                                                                                                              |
| Merge                | Merge (combine)                    | Combines mainCharacter and opponents            | Main Character, Split Out   | Image Prompt Generator, Winner Image Prompt |                                                                                                                              |
| Image Prompt Generator | Langchain Agent (OpenRouter GPT)  | Generates text-to-image prompts for each animal | Merge                       | Split Out1                      |                                                                                                                              |
| Split Out1           | Split Out                          | Splits text-to-image prompt outputs             | Image Prompt Generator      | Generate Close Ups              |                                                                                                                              |
| Generate Close Ups   | HTTP Request                       | Requests AI image generation (close-ups)        | Split Out1                  | 90 seconds                     |                                                                                                                              |
| 90 seconds           | Wait                              | Waits 90 seconds for image generation           | Generate Close Ups          | Get Close Ups                  |                                                                                                                              |
| Get Close Ups        | HTTP Request                       | Retrieves generated close-up images              | 90 seconds                 | Aggregate                      |                                                                                                                              |
| Aggregate            | Aggregate                         | Aggregates close-up images                        | Get Close Ups               | Add Close Ups                 |                                                                                                                              |
| Add Close Ups        | Google Sheets                     | Updates sheet with close-up image URLs           | Aggregate                   | Merge1                        |                                                                                                                              |
| Winner Image Prompt  | Langchain Agent (OpenRouter GPT)  | Generates prompt for winner battle scene         | Merge                      | Generate Scene                |                                                                                                                              |
| Generate Scene       | HTTP Request                     | Requests AI image generation (winner scene)      | Winner Image Prompt         | 90_seconds                   |                                                                                                                              |
| 90_seconds           | Wait                            | Waits 90 seconds for winner image generation     | Generate Scene              | Get Winners                  |                                                                                                                              |
| Get Winners          | HTTP Request                     | Retrieves generated winner images                 | 90_seconds                 | Aggregate1                   |                                                                                                                              |
| Aggregate1           | Aggregate                       | Aggregates winner images                          | Get Winners                 | Add Winner                   |                                                                                                                              |
| Add Winner           | Google Sheets                   | Updates sheet with winner image URLs              | Aggregate1                 | Merge1                      |                                                                                                                              |
| Merge1               | Merge (combine by position)      | Combines all images and data for video rendering | Add Close Ups, Add Winner  | Get Elements                 |                                                                                                                              |
| Get Elements         | Google Sheets                   | Reads updated sheet row with all images           | Merge1                      | Render Video                 |                                                                                                                              |
| Render Video         | HTTP Request                   | Sends rendering request to Creatomate API        | Get Elements                | 90_Seconds                  |                                                                                                                              |
| 90_Seconds           | Wait                          | Waits 90 seconds for video rendering              | Render Video                | Get Video                   |                                                                                                                              |
| Get Video            | HTTP Request                   | Retrieves rendered video URL                       | 90_Seconds                 | Google Sheets (update final) |                                                                                                                              |
| Google Sheets        | Google Sheets (update)         | Updates sheet with final video URL and status    | Get Video                   | Upload to Blotato            |                                                                                                                              |
| Upload to Blotato    | HTTP Request                   | Uploads final video for multi-platform posting    | Google Sheets               | Instagram, TikTok, YouTube   |                                                                                                                              |
| Instagram            | HTTP Request                   | Posts video to Instagram via Blotato               | Upload to Blotato           | None                        | # Instagram                                                                                                                  |
| TikTok               | HTTP Request                   | Posts video to TikTok via Blotato                  | Upload to Blotato           | None                        | # TikTok                                                                                                                     |
| YouTube              | HTTP Request                   | Posts video to YouTube via Blotato                 | Upload to Blotato           | None                        | # YouTube                                                                                                                    |
| GPT 4.1-mini         | Langchain LM Chat Open Router  | Supports AI models for Scene Creator and Prompt   |                              | Scene Creator, Image Prompt Generator | # Output Parser & Chat Models                                                                                                |
| GPT 4.1              | Langchain LM Chat Open Router  | Supports AI models for Winner Image Prompt         |                              | Winner Image Prompt           | # Output Parser & Chat Models                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure with default interval (e.g., every minute) to start the workflow periodically.

2. **Add a Google Sheets Node "Get Main Character"**  
   - Purpose: Read rows with `Status` = "To Do" from a specific sheet ("Sheet1") in a Google Sheet document (provide document ID)  
   - Configure filters to lookup column `Status` with value `To Do`  
   - Set to return first match only.

3. **Add a Set Node "Main Character"**  
   - Assign a string field `mainCharacter` with expression referencing `Get Main Character` output field `Main Character`.

4. **Add a Langchain Agent Node "Scene Creator"**  
   - Use the `OpenRouter` credential with an API key.  
   - Set prompt to use `mainCharacter` and opponent category inputs.  
   - Use system message instructing generation of eight opponent names, one per scene, labeled clearly.  
   - Configure output parser to parse JSON with keys `scene1` to `scene8`.

5. **Add an Output Parser Structured Node "Scenes"**  
   - Set JSON schema example with keys `scene1` through `scene8`.  
   - Connect from "Scene Creator" AI output parser.

6. **Add a Set Node "Opponents"**  
   - Assign array field `opponents` with values from `Scenes` output keys `scene1` to `scene8`.

7. **Add a Split Out Node "Split Out"**  
   - Set to split the `opponents` array into individual items.

8. **Add a Merge Node "Merge"**  
   - Combine mode: combine all  
   - Inputs: mainCharacter from "Main Character" node and individual opponents from "Split Out" node.

9. **Add a Langchain Agent Node "Image Prompt Generator"**  
   - Use OpenRouter API.  
   - System message instructs creation of two photorealistic, fierce animal close-up text-to-image prompts for main character and opponent.  
   - Connect input from "Merge" node.

10. **Add a Split Out Node "Split Out1"**  
    - Split the two prompts generated by the agent.

11. **Add an HTTP Request Node "Generate Close Ups"**  
    - POST to `https://api.piapi.ai/api/v1/task`  
    - Body: JSON with model `Qubico/flux1-dev`, task_type `txt2img`, input prompt from previous node, size 1024x1024  
    - Authenticate with PiAPi API key using HTTP Header Auth.

12. **Add a Wait Node "90 seconds"**  
    - Wait 90 seconds for image generation.

13. **Add an HTTP Request Node "Get Close Ups"**  
    - GET request polling `https://api.piapi.ai/api/v1/task/{{ task_id }}` for image generation results.

14. **Add an Aggregate Node "Aggregate"**  
    - Aggregate all image generation results.

15. **Add a Google Sheets Node "Add Close Ups"**  
    - Update Google Sheet row for main character with generated image URLs in columns like `1.1`, `1.2`, ..., matching by `Main Character`.

16. **Add a Langchain Agent Node "Winner Image Prompt"**  
    - Use OpenRouter API.  
    - System message instructs creation of a vivid prompt depicting the winner standing dominantly over the loser, with battle details.

17. **Add an HTTP Request Node "Generate Scene"**  
    - POST to Flux AI `txt2img` endpoint with prompt from "Winner Image Prompt" node, size 540x960.

18. **Add a Wait Node "90_seconds"**  
    - Wait 90 seconds for winner image generation.

19. **Add an HTTP Request Node "Get Winners"**  
    - Poll Flux AI for winner image results.

20. **Add an Aggregate Node "Aggregate1"**  
    - Aggregate winner image results.

21. **Add a Google Sheets Node "Add Winner"**  
    - Update Google Sheet with winner image URLs in columns like `1.3`, `2.3`, etc., matching by `Main Character`.

22. **Add a Merge Node "Merge1"**  
    - Combine close-ups and winner images by position.

23. **Add a Google Sheets Node "Get Elements"**  
    - Read updated Google Sheet row for main character including all images.

24. **Add an HTTP Request Node "Render Video"**  
    - POST to Creatomate API `https://api.creatomate.com/v1/renders`  
    - Body: JSON with template ID, music URL, and image URLs dynamically filled from sheet data  
    - Authentication: Bearer token with Creatomate API key.

25. **Add a Wait Node "90_Seconds"**  
    - Wait 90 seconds for video rendering.

26. **Add an HTTP Request Node "Get Video"**  
    - GET request to fetch rendered video URL.

27. **Add a Google Sheets Node**  
    - Update Google Sheet row with final video URL and set `Status` to "Created".

28. **Add an HTTP Request Node "Upload to Blotato"**  
    - POST to Blotato media endpoint with final video URL  
    - Authenticated via API key.

29. **Add HTTP Request Nodes "Instagram", "TikTok", and "YouTube"**  
    - POST to Blotato post endpoint for each platform with appropriate JSON body including video URL, captions, and platform-specific settings.

30. **Connect all nodes following the logical flow described above.**

31. **Configure credentials:**  
    - Google Sheets OAuth2 for reading/updating sheets  
    - OpenRouter API keys for Langchain nodes  
    - PiAPi API key for Flux AI image generation  
    - Creatomate API key for video rendering  
    - Blotato API key for media upload and social publishing

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                                               |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| **Author and Setup Guide:** Workflow created by Jadai Kongolo. For setup, make a copy of the provided Google Sheet template and connect it to all Google Sheets nodes. API keys must be connected for OpenRouter (chat models), PiAPi (image generation), Creatomate (video rendering), and Blotato (multi-platform publishing).                                                              | [Author Instagram](https://www.instagram.com/jadai_ai_automation/) and [Google Sheet Template](https://docs.google.com/spreadsheets/d/1hBjE1LR5rmSlEkwkNIwf_NnbjBS_cALS5FJ0r6vh7nM/edit?usp=sharing) |
| **Creatomate Template Source:** The video template used in Creatomate can be duplicated from source code linked in the Skool post where the workflow was downloaded.                                                                                                                                                                                                                  | Refer to original workflow download source (not included here)                                                                |
| **API Rate Limits and Delays:** The workflow uses fixed 90-second wait nodes after image and video generation requests to accommodate processing time. Adjust wait durations if needed depending on API response times.                                                                                                                                                                 | Best practice to avoid premature polling                                                                                      |
| **Error Handling:** Not explicitly implemented in the workflow. Users should consider adding error workflows or retry mechanisms for API failures, authentication errors, and empty or malformed AI responses.                                                                                                                                                                        | Recommended for production use                                                                                                |
| **Sticky Notes:** Visible in the workflow provide section headers such as "Create Scenes," "Create Close-up Images," "Create Winner Images," "Output Parser & Chat Models," "Render Video," and platform names like "Instagram," "TikTok," and "YouTube" for clarity and organization.                                                                                                     | Helpful for maintenance and understanding flow blocks                                                                       |

---

_Disclaimer: This output is a comprehensive analysis and documentation of an n8n workflow based solely on the provided JSON export. All data processed complies with legal and ethical standards._