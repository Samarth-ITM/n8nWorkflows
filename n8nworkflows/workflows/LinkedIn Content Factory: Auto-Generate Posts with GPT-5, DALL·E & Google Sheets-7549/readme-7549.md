LinkedIn Content Factory: Auto-Generate Posts with GPT-5, DALL·E & Google Sheets

https://n8nworkflows.xyz/workflows/linkedin-content-factory--auto-generate-posts-with-gpt-5--dall-e---google-sheets-7549


# LinkedIn Content Factory: Auto-Generate Posts with GPT-5, DALL·E & Google Sheets

---

### 1. Workflow Overview

This workflow automates the generation of LinkedIn posts tailored for Product Managers, founders, and AI enthusiasts by leveraging GPT-5, DALL·E image generation, and Google Sheets for persistent idea management. Its primary use case is to create engaging, data-backed social media content regularly, minimizing duplication and maximizing relevance.

The workflow is logically divided into four major blocks:

- **1.1 Idea Generation and Deduplication:** Automatically triggers and builds a content brief, generates LinkedIn post ideas using GPT-5, extracts and cleans these ideas, and deduplicates them against previously published ideas stored in Google Sheets.

- **1.2 Post Draft Generation:** Selects the best post idea using AI editorial judgment, generates a detailed LinkedIn post draft from the chosen idea, and extracts structured components such as hook, body, and CTA.

- **1.3 Post Polishing and CTA/Hashtag Enrichment:** Refines the draft by adding specificity and voice conformity, enhances engagement elements including CTAs and hashtags, and ensures compliance with LinkedIn's formatting and character limits.

- **1.4 Image Generation and Publishing:** Creates a LinkedIn-style social graphic using the post's hook and themes, packages the post with the image, uploads the image to Google Drive, and logs the completed post data into Google Sheets.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Idea Generation and Deduplication

**Overview:**  
This block initiates the workflow on a schedule, constructs a detailed content brief, generates multiple raw LinkedIn post ideas using GPT-5, extracts and cleans these ideas, then compares them against past published ideas stored in Google Sheets for exact and fuzzy deduplication to avoid repetition.

**Nodes Involved:**  
- 01_AutoStart  
- 02_BuildBrief  
- 03_GenerateIdea  
- 04_ExtractIdeasList  
- 02_ReadPastIdeas  
- 03_NormalizePastIdeas  
- 04.5_MergeIdeasPastIdeas  
- 05_ExactDedupeCheck  
- 06_FuzzyDeduplication  
- Merge (combine nodes where outputs combine)

**Node Details:**

- **01_AutoStart**  
  - Type: Schedule Trigger  
  - Role: Triggers the workflow daily at 10 AM.  
  - Configuration: Cron set to trigger at hour 10.  
  - Inputs: None  
  - Outputs: Starts the workflow chain.  
  - Failure Modes: Missed trigger if n8n instance is down or time misconfigured.

- **02_BuildBrief**  
  - Type: Code  
  - Role: Constructs a JSON brief containing audience, geography, goals, CTAs, topics, angles, tone, and constraints to guide AI generation.  
  - Key variables: Static brief object embedded in JavaScript code.  
  - Inputs: Trigger from 01_AutoStart  
  - Outputs: JSON brief for downstream AI nodes.  
  - Failure Modes: Code errors unlikely unless manual edit introduces issues.

- **03_GenerateIdea**  
  - Type: OpenAI GPT-5 Chat (Langchain Node)  
  - Role: Generates 5 raw LinkedIn post ideas based on the brief.  
  - Configuration: Uses GPT-5-chat-latest with a system prompt defining role and task.  
  - Expressions: Uses dynamic expressions to insert brief fields into prompt.  
  - Inputs: JSON brief from 02_BuildBrief  
  - Outputs: Raw AI-generated text with ideas.  
  - Potential Failures: API auth errors, timeouts, malformed prompts.

- **04_ExtractIdeasList**  
  - Type: Code  
  - Role: Parses raw AI output to extract a clean array of idea titles or short descriptions.  
  - Logic: Uses regex and string operations to remove numbering, bullets, quotes, and duplicates.  
  - Inputs: Raw AI output from 03_GenerateIdea  
  - Outputs: JSON array of cleaned ideas.  
  - Failure Modes: Format changes in AI output may cause parsing issues.

- **02_ReadPastIdeas**  
  - Type: Google Sheets  
  - Role: Reads previously logged ideas from a Google Sheet ("Idea_Log" document).  
  - Configuration: Reads sheet with gid=0 using OAuth2 credentials.  
  - Inputs: Triggered after 04_ExtractIdeasList.  
  - Outputs: Rows with past ideas.  
  - Failure Modes: Authentication or API quota issues, sheet availability.

- **03_NormalizePastIdeas**  
  - Type: Code  
  - Role: Extracts and normalizes past ideas into a simple array for deduplication.  
  - Inputs: Google Sheets rows from 02_ReadPastIdeas  
  - Outputs: Array of past ideas (strings).  
  - Failure Modes: Empty or malformed input from Sheets.

- **04.5_MergeIdeasPastIdeas**  
  - Type: Merge (Combine)  
  - Role: Combines arrays of new ideas and past ideas into one data structure for dedupe.  
  - Inputs: Ideas from 04_ExtractIdeasList and past ideas from 03_NormalizePastIdeas  
  - Outputs: Combined data for deduplication nodes.  
  - Failure Modes: Data mismatch or empty inputs.

- **05_ExactDedupeCheck**  
  - Type: Code  
  - Role: Performs exact text matching deduplication comparing new ideas against past ideas.  
  - Logic: Case-insensitive comparison; separates kept and rejected ideas.  
  - Inputs: Combined ideas and past ideas from Merge node.  
  - Outputs: JSON listing kept_exact and rejected_exact ideas.  
  - Edge Cases: Minor text differences may not be caught; only exact matches.

- **06_FuzzyDeduplication**  
  - Type: OpenAI GPT-5 (Langchain)  
  - Role: Uses semantic similarity to remove near-duplicate ideas conservatively.  
  - Input: kept_exact ideas and past ideas for context.  
  - Output: JSON with kept_fuzzy (safe ideas) and rejected_fuzzy (near duplicates with reasons).  
  - Failure Modes: AI misclassification, response parsing errors.

- **Merge Nodes (Merge, Merge1, etc.)**  
  - Type: Merge  
  - Role: Combine outputs from parallel streams for unified downstream processing.

---

#### 1.2 Post Draft Generation

**Overview:**  
This block selects the best LinkedIn post idea using AI editorial judgment, generates a full LinkedIn post draft based on that idea and brief, and extracts structured components such as hook, body, and CTA from the draft.

**Nodes Involved:**  
- 05_PickIdeaAI  
- 05b_MergeBriefAndPick  
- 05a_ParsePickedIdea  
- 06_GeneratePost  
- 10_ExtractPublishPack  
- 11_ParsePublishPack  
- Merge1, Merge2

**Node Details:**

- **05_PickIdeaAI**  
  - Type: OpenAI GPT-5 (Langchain)  
  - Role: Acts as editorial board to pick one best idea from 3–8 candidates, explain choice, and rank all ideas.  
  - Input: JSON brief and deduplicated ideas.  
  - Output: JSON strictly formatted with chosenIdea, why, and ranked list.  
  - Failure Modes: Parsing errors if AI response deviates from strict JSON.

- **05b_MergeBriefAndPick**  
  - Type: Merge (Combine)  
  - Role: Combines brief and chosen idea data for the next steps.  
  - Inputs: Output of 05_PickIdeaAI and 02_BuildBrief.  
  - Outputs: Combined data for parsing node.

- **05a_ParsePickedIdea**  
  - Type: Code  
  - Role: Parses the strict JSON response from 05_PickIdeaAI, extracting chosen idea and metadata.  
  - Input: Result of 05b_MergeBriefAndPick  
  - Outputs: Parsed JSON with chosenIdea, why, ranked, and brief.  
  - Failure Modes: JSON parse errors if AI output malformed.

- **06_GeneratePost**  
  - Type: OpenAI GPT-5 (Langchain)  
  - Role: Generates a full LinkedIn post draft based on the chosen idea and brief.  
  - Input: Parsed chosenIdea, brief, and rationale.  
  - Output: JSON with post draft text.  
  - Failure Modes: API issues, prompt misinterpretation.

- **10_ExtractPublishPack**  
  - Type: OpenAI GPT-5 (Langchain)  
  - Role: Extracts structured components from the LinkedIn post draft: hook, body, CTA, hashtags, first comment, and character count.  
  - Input: Raw post draft JSON from 06_GeneratePost and brief.  
  - Output: Strict JSON with extracted components.  
  - Failure Modes: Parsing errors, AI hallucination.

- **11_ParsePublishPack**  
  - Type: Code  
  - Role: Parses and normalizes the publish pack JSON from 10_ExtractPublishPack, recomputes character count, and attaches brief.  
  - Input: Output of 10_ExtractPublishPack and brief from earlier nodes.  
  - Output: Normalized JSON with all relevant post fields.  
  - Failure Modes: JSON parse errors.

- **Merge Nodes (Merge1, Merge2)**  
  - Combine outputs for downstream polishing.

---

#### 1.3 Post Polishing and CTA/Hashtag Enrichment

**Overview:**  
This block refines the LinkedIn post draft by adding crisp specificity, adjusting voice to match the brief’s tone, building CTA and hashtags, and ensuring formatting compliance and engagement hygiene suitable for LinkedIn.

**Nodes Involved:**  
- 12_SpecificityPass  
- 12a_ParseSpecificityJSON  
- 13_VoiceConformity  
- 13a_ParseVoiceJSON  
- 14_BuildCTAHashtags_LLM  
- Merge3, Merge4  
- 15_EngagementHygiene  
- 16_FormatCompliance

**Node Details:**

- **12_SpecificityPass**  
  - Type: OpenAI GPT-5 (Langchain)  
  - Role: Adds specific examples, concrete details, and brief citations to the post without changing length or core claims.  
  - Input: Parsed publish pack including hook, body, and brief.  
  - Output: JSON with revised hook, body, and notes describing clarifications.  
  - Failure Modes: Parsing errors, hallucination risk.

- **12a_ParseSpecificityJSON**  
  - Type: Code  
  - Role: Parses JSON output from 12_SpecificityPass, extracting updated text and notes.  
  - Failure Modes: JSON parsing errors.

- **13_VoiceConformity**  
  - Type: OpenAI GPT-5 (Langchain)  
  - Role: Adjusts language to a sharp, authoritative voice with punchy, scannable sentences while preserving facts.  
  - Input: Specificity pass output.  
  - Output: JSON with voice-refined hook, body, notes, and brief.  
  - Failure Modes: Parsing errors.

- **13a_ParseVoiceJSON**  
  - Type: Code  
  - Role: Parses JSON output from 13_VoiceConformity.  
  - Failure Modes: Parsing errors.

- **14_BuildCTAHashtags_LLM**  
  - Type: OpenAI GPT-4o Mini (Langchain)  
  - Role: Picks a single CTA from allowed options, generates up to 8 relevant hashtags, creates an optional first comment to invite discussion, and calculates character count.  
  - Input: Hook, body and brief.  
  - Output: Strict JSON with cta, hashtags, first_comment, char_count.  
  - Failure Modes: AI misinterpretation, parsing issues.

- **Merge3, Merge4**  
  - Combine outputs from parsing and CTA building for final polishing.

- **15_EngagementHygiene**  
  - Type: Code  
  - Role: Cleans spacing, punctuation, bullet formatting, ensures question presence if CTA is comments, normalizes hashtags with preferred casing, and sets first comment fallback.  
  - Input: Merged post JSON (hook, body, cta, hashtags, first_comment).  
  - Output: Cleaned and engagement-optimized post JSON.  
  - Failure Modes: Code exceptions on unexpected input.

- **16_FormatCompliance**  
  - Type: Code  
  - Role: Removes duplicate hook lines from the body, enforces LinkedIn’s 3000 character limit, trims body text smartly, deduplicates hashtags, and recomputes final character count.  
  - Inputs: Output from 15_EngagementHygiene.  
  - Outputs: Final compliant post JSON.  
  - Failure Modes: None expected unless input malformed.

---

#### 1.4 Image Generation and Publishing

**Overview:**  
This block generates a LinkedIn-style social graphic image based on the final post hook and themes, merges the image with text data, uploads the image to Google Drive, builds a row for Google Sheets logging, and appends this row to the log sheet.

**Nodes Involved:**  
- Generate an image  
- Merge5  
- 18_PublishPackager  
- Upload file  
- Merge6  
- 18b_BuildSheetRow  
- 19_Sheets Append

**Node Details:**

- **Generate an image**  
  - Type: OpenAI GPT-Image (DALL·E)  
  - Role: Creates a 1024x1024 LinkedIn-friendly social graphic with headline text and 2-3 inferred icons illustrating core themes from the post body.  
  - Configuration: Prompt includes styling instructions for a modern, minimalist look with specified color palette and content layout.  
  - Input: Final post hook and body from previous nodes.  
  - Output: Binary image data.  
  - Failure Modes: API errors, image generation timeouts.

- **Merge5**  
  - Type: Merge (Combine)  
  - Role: Combines final post JSON and binary image data into one item for packaging.  
  - Inputs: Image node and 16_FormatCompliance output.  
  - Outputs: Combined JSON + image binary.

- **18_PublishPackager**  
  - Type: Code  
  - Role: Validates presence of post text fields and image binary, constructs the final post string including hashtags and CTA, attaches metadata about the image.  
  - Inputs: Merged post JSON and image.  
  - Outputs: JSON with final_post string and image binary for upload.  
  - Failure Modes: Throws error if image or text missing.

- **Upload file**  
  - Type: Google Drive  
  - Role: Uploads the image binary to a specified Google Drive folder with filename based on timestamp and sanitized hook text.  
  - Credentials: Google Drive OAuth2.  
  - Output: File metadata including URL and webViewLink.  
  - Failure Modes: Auth errors, quota limits, folder permission issues.

- **Merge6**  
  - Type: Merge (Combine)  
  - Role: Combines uploaded file metadata with post data for logging.  
  - Inputs: Upload file node and 18_PublishPackager output.  
  - Outputs: Combined data for sheet logging.

- **18b_BuildSheetRow**  
  - Type: Code  
  - Role: Constructs a row object with timestamps (ISO and IST), idea text, slug, hash, post content, image metadata, and debug info, formatted for Google Sheets append.  
  - Inputs: Combined data from Merge6.  
  - Outputs: JSON rows for sheet insertion.  
  - Failure Modes: None expected; uses robust data extraction.

- **19_Sheets Append**  
  - Type: Google Sheets  
  - Role: Appends or updates the generated post data into the "Idea_Log" Google Sheet to maintain a record of published posts and ideas.  
  - Configuration: Uses OAuth2 credentials, documentId and sheet gid=0.  
  - Inputs: Rows from 18b_BuildSheetRow.  
  - Outputs: Confirmation of append operation.  
  - Failure Modes: Auth issues, API quota, sheet permission.

---

### 3. Summary Table

| Node Name             | Node Type                      | Functional Role                               | Input Node(s)                   | Output Node(s)                  | Sticky Note                                                                                     |
|-----------------------|--------------------------------|----------------------------------------------|---------------------------------|---------------------------------|------------------------------------------------------------------------------------------------|
| 01_AutoStart          | Schedule Trigger               | Workflow trigger at 10 AM daily               | -                               | 02_BuildBrief                   |                                                                                                |
| 02_BuildBrief         | Code                          | Builds static content brief                    | 01_AutoStart                   | 03_GenerateIdea, Merge, 05b_MergeBriefAndPick, Merge1, Merge2, Merge3 |                                                                                                |
| 03_GenerateIdea       | OpenAI GPT-5 Chat (Langchain) | Generates 5 raw LinkedIn post ideas            | 02_BuildBrief                  | 04_ExtractIdeasList             |                                                                                                |
| 04_ExtractIdeasList   | Code                          | Extracts and cleans list of AI-generated ideas | 03_GenerateIdea                | 02_ReadPastIdeas, 04.5_MergeIdeasPastIdeas | Covered by Sticky Note: "Idea Generation and Dedupe with previous published posts"             |
| 02_ReadPastIdeas      | Google Sheets                  | Reads previously published ideas from sheet   | 04_ExtractIdeasList            | 03_NormalizePastIdeas           |                                                                                                |
| 03_NormalizePastIdeas | Code                          | Normalizes past ideas into simple list         | 02_ReadPastIdeas               | 04.5_MergeIdeasPastIdeas        |                                                                                                |
| 04.5_MergeIdeasPastIdeas | Merge                        | Combines new and past ideas for dedupe         | 04_ExtractIdeasList, 03_NormalizePastIdeas | 05_ExactDedupeCheck             |                                                                                                |
| 05_ExactDedupeCheck   | Code                          | Filters out exact duplicates                    | 04.5_MergeIdeasPastIdeas       | 06_FuzzyDeduplication           |                                                                                                |
| 06_FuzzyDeduplication | OpenAI GPT-5 (Langchain)       | Performs semantic similarity fuzzy dedupe      | 05_ExactDedupeCheck            | Merge                          |                                                                                                |
| Merge                 | Merge                         | Combines dedupe results                         | 03_GenerateIdea, 06_FuzzyDeduplication | 05_PickIdeaAI                 |                                                                                                |
| 05_PickIdeaAI         | OpenAI GPT-5 (Langchain)       | Selects best idea with explanation and ranking | Merge                         | 05b_MergeBriefAndPick           |                                                                                                |
| 05b_MergeBriefAndPick | Merge                         | Combines brief with chosen idea                 | 05_PickIdeaAI, 02_BuildBrief   | 05a_ParsePickedIdea             |                                                                                                |
| 05a_ParsePickedIdea   | Code                          | Parses chosen idea JSON                          | 05b_MergeBriefAndPick          | 06_GeneratePost                |                                                                                                |
| 06_GeneratePost       | OpenAI GPT-5 (Langchain)       | Generates full LinkedIn post draft               | 05a_ParsePickedIdea            | Merge1                         | Covered by Sticky Note: "Post generation basis the idea"                                       |
| Merge1                | Merge                         | Combines generated post with brief              | 06_GeneratePost, 02_BuildBrief | 10_ExtractPublishPack           |                                                                                                |
| 10_ExtractPublishPack | OpenAI GPT-5 (Langchain)       | Extracts structured post components              | Merge1                        | Merge2                         |                                                                                                |
| Merge2                | Merge                         | Combines extracted components                    | 10_ExtractPublishPack, 02_BuildBrief | 11_ParsePublishPack           |                                                                                                |
| 11_ParsePublishPack   | Code                          | Parses and normalizes extracted publish pack      | Merge2                        | 12_SpecificityPass             |                                                                                                |
| 12_SpecificityPass    | OpenAI GPT-5 (Langchain)       | Adds crisp specifics without bloating length    | 11_ParsePublishPack            | Merge3                         | Covered by Sticky Note: "Idea polishing, adding CTA and hashtags"                              |
| Merge3                | Merge                         | Combines specificity pass output                  | 12_SpecificityPass, 02_BuildBrief | 13_VoiceConformity            |                                                                                                |
| 13_VoiceConformity    | OpenAI GPT-5 (Langchain)       | Refines post voice to match brief tone           | Merge3                        | 13a_ParseVoiceJSON             |                                                                                                |
| 13a_ParseVoiceJSON    | Code                          | Parses voice conformity JSON                       | 13_VoiceConformity             | 14_BuildCTAHashtags_LLM, Merge4 |                                                                                                |
| 14_BuildCTAHashtags_LLM | OpenAI GPT-4o Mini (Langchain) | Generates CTA, hashtags, and first comment        | 13a_ParseVoiceJSON             | Merge4                         |                                                                                                |
| Merge4                | Merge                         | Combines voice conformity and CTA/hashtags        | 14_BuildCTAHashtags_LLM, 13a_ParseVoiceJSON | 15_EngagementHygiene          |                                                                                                |
| 15_EngagementHygiene  | Code                          | Cleans formatting, bullets, hashtags, adds CTA question | Merge4                        | 16_FormatCompliance            |                                                                                                |
| 16_FormatCompliance   | Code                          | Enforces LinkedIn limits and final cleaning        | 15_EngagementHygiene           | Generate an image, Merge5       | Covered by Sticky Note: "Image Generation, final post ready and updating the google sheet and google drive" |
| Generate an image     | OpenAI GPT-Image (DALL·E)      | Creates LinkedIn-style social graphic               | 16_FormatCompliance            | Merge5                         |                                                                                                |
| Merge5                | Merge                         | Combines final compliant post JSON and image binary | Generate an image, 16_FormatCompliance | 18_PublishPackager           |                                                                                                |
| 18_PublishPackager    | Code                          | Packages post text with image metadata               | Merge5                        | Upload file, Merge6            |                                                                                                |
| Upload file           | Google Drive                  | Uploads image to Google Drive                         | 18_PublishPackager            | Merge6                         |                                                                                                |
| Merge6                | Merge                         | Combines uploaded file metadata and post data        | Upload file, 18_PublishPackager | 18b_BuildSheetRow             |                                                                                                |
| 18b_BuildSheetRow     | Code                          | Builds rows for Google Sheets append                   | Merge6                        | 19_Sheets Append              |                                                                                                |
| 19_Sheets Append      | Google Sheets                 | Logs post and image metadata to Idea_Log sheet         | 18b_BuildSheetRow             | -                             |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Name: `01_AutoStart`  
   - Type: Schedule Trigger  
   - Set to trigger daily at 10:00 AM (hour 10).  
   - No credentials needed.

2. **Create a Code Node to Build the Content Brief**  
   - Name: `02_BuildBrief`  
   - Type: Code  
   - Paste the JavaScript code that returns a JSON object with fields: audience, geography, goals, CTA options, topics, angles, tone, constraints, trigger.  
   - Connect output of `01_AutoStart` to this node.

3. **Create an OpenAI GPT-5 Chat Node to Generate Ideas**  
   - Name: `03_GenerateIdea`  
   - Type: OpenAI (Langchain)  
   - Model: `gpt-5-chat-latest`  
   - System prompt: "You are an AI that generates LinkedIn post ideas based on a given brief."  
   - User prompt: Inject brief fields dynamically using expressions to build task requesting 5 raw LinkedIn post ideas (titles or one-line descriptions).  
   - Connect output of `02_BuildBrief` to this node.  
   - Use OpenAI API credentials.

4. **Create a Code Node to Extract Ideas List**  
   - Name: `04_ExtractIdeasList`  
   - Type: Code  
   - Paste provided JS code to parse raw AI output, clean formatting, remove numbering, bolding, and duplicates.  
   - Connect output of `03_GenerateIdea` to this node.

5. **Create a Google Sheets Node to Read Past Ideas**  
   - Name: `02_ReadPastIdeas`  
   - Type: Google Sheets  
   - Configure with OAuth2 credentials for Google Sheets.  
   - Document ID: the ID of your Google Sheet where past ideas are logged.  
   - Sheet GID: 0 (or appropriate sheet).  
   - Connect output of `04_ExtractIdeasList` to this node.

6. **Create a Code Node to Normalize Past Ideas**  
   - Name: `03_NormalizePastIdeas`  
   - Type: Code  
   - Extract the "idea" field from each row and return a simple array.  
   - Connect output of `02_ReadPastIdeas` to this node.

7. **Create a Merge Node to Combine Ideas and Past Ideas**  
   - Name: `04.5_MergeIdeasPastIdeas`  
   - Type: Merge (Combine)  
   - Connect outputs of `04_ExtractIdeasList` and `03_NormalizePastIdeas` to this node's inputs.

8. **Create a Code Node for Exact Deduplication**  
   - Name: `05_ExactDedupeCheck`  
   - Type: Code  
   - Implement exact text comparison logic to separate kept and rejected ideas.  
   - Connect output of `04.5_MergeIdeasPastIdeas` to this node.

9. **Create OpenAI GPT-5 Node for Fuzzy Deduplication**  
   - Name: `06_FuzzyDeduplication`  
   - Type: OpenAI (Langchain)  
   - Model: GPT-5  
   - System prompt to remove near-duplicate ideas based on semantic similarity conservatively, returning strict JSON.  
   - Connect output of `05_ExactDedupeCheck` to this node.  
   - Use OpenAI API credentials.

10. **Create a Merge Node to Combine Deduplication Results**  
    - Name: `Merge`  
    - Type: Merge (Combine)  
    - Connect outputs of `03_GenerateIdea` and `06_FuzzyDeduplication`.

11. **Create OpenAI GPT-5 Node to Pick the Best Idea**  
    - Name: `05_PickIdeaAI`  
    - Type: OpenAI (Langchain)  
    - Model: GPT-5  
    - System prompt: editorial board role, picking one best idea with explanation and ranking, returning strict JSON.  
    - Inputs: Combined deduplicated ideas and brief from previous merge.  
    - Connect output of `Merge` to this node.  
    - Use OpenAI API credentials.

12. **Create Merge Node to Combine Brief and Picked Idea**  
    - Name: `05b_MergeBriefAndPick`  
    - Type: Merge (Combine)  
    - Connect outputs of `05_PickIdeaAI` and `02_BuildBrief`.

13. **Create Code Node to Parse Picked Idea JSON**  
    - Name: `05a_ParsePickedIdea`  
    - Type: Code  
    - Parse strict JSON from AI, extract chosenIdea and metadata.  
    - Connect output of `05b_MergeBriefAndPick`.

14. **Create OpenAI GPT-5 Node to Generate Post Draft**  
    - Name: `06_GeneratePost`  
    - Type: OpenAI (Langchain)  
    - Model: GPT-5  
    - System prompt: expert LinkedIn content strategist, instructions to write sharp, engaging post with strong hooks, storytelling, data support, and CTA.  
    - Inputs: Parsed chosenIdea and brief from `05a_ParsePickedIdea`.  
    - Connect output of `05a_ParsePickedIdea`.

15. **Create Merge Node to Combine Post Draft and Brief**  
    - Name: `Merge1`  
    - Type: Merge (Combine)  
    - Connect outputs of `06_GeneratePost` and `02_BuildBrief`.

16. **Create OpenAI GPT-5 Node to Extract Publish Pack**  
    - Name: `10_ExtractPublishPack`  
    - Type: OpenAI (Langchain)  
    - Model: GPT-5  
    - Extract components: hook, body, CTA, hashtags, first comment, char count from post draft JSON.  
    - Connect output of `Merge1`.

17. **Create Merge Node to Combine Extracted Pack and Brief**  
    - Name: `Merge2`  
    - Type: Merge (Combine)  
    - Connect outputs of `10_ExtractPublishPack` and `02_BuildBrief`.

18. **Create Code Node to Parse Publish Pack**  
    - Name: `11_ParsePublishPack`  
    - Type: Code  
    - Parse JSON, normalize fields, recompute character count, attach brief.  
    - Connect output of `Merge2`.

19. **Create OpenAI GPT-5 Node for Specificity Pass**  
    - Name: `12_SpecificityPass`  
    - Type: OpenAI (Langchain)  
    - Model: GPT-5  
    - Adds specific examples and citations without changing length or core claims, returns strict JSON.  
    - Connect output of `11_ParsePublishPack`.

20. **Create Code Node to Parse Specificity JSON**  
    - Name: `12a_ParseSpecificityJSON`  
    - Type: Code  
    - Parses JSON output from `12_SpecificityPass`.  
    - Connect output of `12_SpecificityPass`.

21. **Create OpenAI GPT-5 Node for Voice Conformity**  
    - Name: `13_VoiceConformity`  
    - Type: OpenAI (Langchain)  
    - Model: GPT-5  
    - Adjusts language to sharp, authoritative tone with scannable sentences, preserves data.  
    - Connect output of `12a_ParseSpecificityJSON`.

22. **Create Code Node to Parse Voice JSON**  
    - Name: `13a_ParseVoiceJSON`  
    - Type: Code  
    - Parses output from `13_VoiceConformity`.  
    - Connect output of `13_VoiceConformity`.

23. **Create OpenAI GPT-4o Mini Node to Build CTA and Hashtags**  
    - Name: `14_BuildCTAHashtags_LLM`  
    - Type: OpenAI (Langchain)  
    - Model: GPT-4o Mini  
    - Picks one CTA, generates up to 8 hashtags, writes an optional first comment, calculates char count.  
    - Connect output of `13a_ParseVoiceJSON`.

24. **Create Merge Node to Combine CTA/Hashtags with Voice Output**  
    - Name: `Merge4`  
    - Type: Merge (Combine)  
    - Connect outputs of `14_BuildCTAHashtags_LLM` and `13a_ParseVoiceJSON`.

25. **Create Code Node for Engagement Hygiene**  
    - Name: `15_EngagementHygiene`  
    - Type: Code  
    - Cleans spacing, punctuation, bullet formatting, enforces question for comments CTA, normalizes hashtags, and sets first comment fallback.  
    - Connect output of `Merge4`.

26. **Create Code Node for Format Compliance**  
    - Name: `16_FormatCompliance`  
    - Type: Code  
    - Removes duplicate hook lines from body, trims post to LinkedIn max chars (3000), deduplicates hashtags, recalculates char count.  
    - Connect output of `15_EngagementHygiene`.

27. **Create OpenAI GPT-Image Node for Image Generation**  
    - Name: `Generate an image`  
    - Type: OpenAI Image Generation (DALL·E)  
    - Prompt: Uses hook as headline, infers 2-3 core concepts from body, creates minimalist LinkedIn-friendly image with specified styling.  
    - Output binary property: `image`  
    - Connect output of `16_FormatCompliance`.

28. **Create Merge Node to Combine Image and Post JSON**  
    - Name: `Merge5`  
    - Type: Merge (Combine)  
    - Connect outputs of `Generate an image` and `16_FormatCompliance`.

29. **Create Code Node to Package Post and Image**  
    - Name: `18_PublishPackager`  
    - Type: Code  
    - Validates presence of post text and image binary, assembles final post string with hashtags and CTA, attaches image metadata.  
    - Connect output of `Merge5`.

30. **Create Google Drive Node to Upload Image**  
    - Name: `Upload file`  
    - Type: Google Drive  
    - Credentials: Google Drive OAuth2  
    - Folder: Specific folder ID for LinkedIn Post Generator images.  
    - Filename: Timestamp + sanitized hook text + `.png`  
    - Input Data Field Name: `image` (binary)  
    - Connect output of `18_PublishPackager`.

31. **Create Merge Node to Combine Upload Metadata and Post JSON**  
    - Name: `Merge6`  
    - Type: Merge (Combine)  
    - Connect outputs of `Upload file` and `18_PublishPackager`.

32. **Create Code Node to Build Google Sheets Row**  
    - Name: `18b_BuildSheetRow`  
    - Type: Code  
    - Builds a structured row with timestamps, idea text, slug, hash, post metadata, image metadata, for logging.  
    - Connect output of `Merge6`.

33. **Create Google Sheets Node to Append Row**  
    - Name: `19_Sheets Append`  
    - Type: Google Sheets  
    - Credentials: Google Sheets OAuth2  
    - Document ID and Sheet GID: Same as in `02_ReadPastIdeas`.  
    - Operation: Append or update row.  
    - Connect output of `18b_BuildSheetRow`.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                                        |
|-----------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Workflow uses GPT-5 and GPT-4o Mini models for different tasks to optimize cost and performance.   | Model selection in OpenAI nodes.                                                                                       |
| Deduplication is two-tier: exact string match followed by semantic similarity with conservative AI logic. | Sticky note: "Idea Generation and Dedupe with previous published posts"                                               |
| Post generation emphasizes clarity, data-backed claims, and sharp tone aimed at Product Managers and AI enthusiasts. | Sticky note: "Post generation basis the idea"                                                                          |
| Final polishing includes voice conformity, engagement hygiene, and LinkedIn formatting compliance. | Sticky note: "Idea polishing, adding CTA and hashtags"                                                                 |
| Image generation uses a custom prompt to produce minimalist, LinkedIn-appropriate social graphics without branding or faces. | Sticky note: "Image Generation, final post ready and updating the google sheet and google drive"                       |
| Google Drive and Google Sheets OAuth2 credentials must be configured with appropriate folder and document access. | Google Drive folder ID and Google Sheets document ID are critical for uploads and logging.                             |
| The entire workflow is designed to run unattended on schedule, generating, polishing, and publishing LinkedIn posts automatically. | Workflow trigger and continuous execution design.                                                                       |

---

**Disclaimer:** The provided content is extracted exclusively from an automated n8n workflow. It adheres strictly to content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.

---