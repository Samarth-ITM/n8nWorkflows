Humanize AI-generated text using Rephrasy by detecting and rewriting only flagged sentences

https://n8nworkflows.xyz/workflows/humanize-ai-generated-text-using-rephrasy-by-detecting-and-rewriting-only-flagged-sentences-14545


# Humanize AI-generated text using Rephrasy by detecting and rewriting only flagged sentences

# 1. Workflow Overview

This workflow detects whether a text appears AI-generated at the sentence level, then rewrites only the sentences that are flagged as likely AI-written using Rephrasy’s Humanizer API. Sentences considered already human-like are preserved unchanged. At the end, all processed sentences are merged back into one final output.

Typical use cases include:

- improving AI-generated drafts before publishing,
- selectively humanizing text instead of rewriting everything,
- reducing cost by only sending flagged sentences to the rewriting service,
- building a reusable text-cleanup component inside a larger n8n automation.

The workflow is organized into three main logical blocks plus one entry point:

## 1.1 Manual Input Reception

A manual trigger starts the workflow and a Set node provides the source text to analyze.

## 1.2 AI Detection and Sentence Extraction

The text is sent to Rephrasy’s detector API in `depth` mode, which returns sentence-level scoring. A Code node converts the detector response into an array of sentence objects, and a Split Out node prepares them for per-sentence processing.

## 1.3 Conditional Humanization Loop

The workflow iterates sentence by sentence. An IF node checks whether the sentence score is above 50. If yes, the sentence is sent to the Rephrasy Humanizer API and replaced with the rewritten version. If not, the original sentence is kept as-is.

## 1.4 Reassembly of Final Text

Once all loop iterations are complete, the workflow aggregates the resulting sentence texts into one collected output representing the final processed text.

---

# 2. Block-by-Block Analysis

## Block 1 — Manual Input Reception

### Overview
This block initializes execution and defines the text to process. In the provided version, the text is hardcoded in a Set node, but the design allows easy replacement with another input source such as a webhook, form, or upstream workflow.

### Nodes Involved
- When clicking ‘Execute workflow’
- Set text

### Node Details

#### 1) When clicking ‘Execute workflow’
- **Type and technical role:** `n8n-nodes-base.manualTrigger`  
  Manual entry point used for test or ad hoc execution inside the n8n editor.
- **Configuration choices:** No parameters are configured.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: none
  - Output: `Set text`
- **Version-specific requirements:** Type version 1; no special version dependency.
- **Edge cases or potential failure types:**
  - No runtime failure expected.
  - Only usable manually from the editor; not suitable for unattended production entry.
- **Sub-workflow reference:** None.

#### 2) Set text
- **Type and technical role:** `n8n-nodes-base.set`  
  Defines the source payload by assigning a `text` field.
- **Configuration choices:**
  - Creates one string field named `text`
  - Current value is hardcoded to `XXX`
- **Key expressions or variables used:**
  - Field created: `text`
- **Input and output connections:**
  - Input: `When clicking ‘Execute workflow’`
  - Output: `AI Detector`
- **Version-specific requirements:** Type version 3.4.
- **Edge cases or potential failure types:**
  - If `text` remains `XXX`, the workflow will analyze placeholder text rather than real input.
  - If replaced with an expression from another node, expression resolution may fail if the referenced field is missing.
  - Empty text may produce weak or invalid detector results depending on API behavior.
- **Sub-workflow reference:** None.

---

## Block 2 — AI Detection and Sentence Extraction

### Overview
This block sends the original text to Rephrasy’s AI detector, retrieves sentence-level scoring, then reshapes the response into a list of sentence objects suitable for iteration. It is the core analysis stage that determines which sentences should be rewritten later.

### Nodes Involved
- AI Detector
- Get sentences
- Split Out

### Node Details

#### 3) AI Detector
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the Rephrasy detection API to classify the text and score its sentences.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://detector.rephrasy.ai/detect_api`
  - Authentication: HTTP Bearer Auth credential named `Rephrasy`
  - Sends JSON body
  - Header `Content-Type: application/json`
  - Request body includes:
    - `text`: taken from `$json.text`
    - `mode`: `"depth"`
- **Key expressions or variables used:**
  - `{{ $json.text }}`
- **Input and output connections:**
  - Input: `Set text`
  - Output: `Get sentences`
- **Version-specific requirements:** Type version 4.4.
- **Edge cases or potential failure types:**
  - Missing or invalid bearer token causes authentication failure.
  - API downtime, rate limits, or timeout can stop execution.
  - If `text` is empty, malformed, or too large, the remote API may reject the request.
  - Because `mode` is `depth`, the response must include sentence scores in a structure expected by the next Code node.
- **Sub-workflow reference:** None.

#### 4) Get sentences
- **Type and technical role:** `n8n-nodes-base.code`  
  Transforms the detector response into a normalized array of sentence objects containing `text`, `score`, and a derived `type`.
- **Configuration choices:**
  - JavaScript code iterates over all incoming items.
  - It reads `data.sentences || {}`.
  - It converts each sentence-score pair into:
    - `text`
    - `score`
    - `type`: `"ai"` if score > 50, otherwise `"human"`
  - It returns a single item with `json.sentences = merged`
- **Key expressions or variables used:**
  - `item.json`
  - `data.sentences`
  - `Object.entries(sentences)`
  - Derived threshold logic: `score > 50 ? "ai" : "human"`
- **Input and output connections:**
  - Input: `AI Detector`
  - Output: `Split Out`
- **Version-specific requirements:** Type version 2 for the Code node.
- **Edge cases or potential failure types:**
  - If the detector response structure changes and `sentences` is absent or not an object, the node will produce an empty `sentences` array.
  - If scores are returned in a non-numeric format, later numeric comparisons may fail or behave unexpectedly.
  - If sentence text keys contain unusual characters, they still remain valid object keys here, but downstream assumptions should be checked.
- **Sub-workflow reference:** None.

#### 5) Split Out
- **Type and technical role:** `n8n-nodes-base.splitOut`  
  Expands the `sentences` array into individual items so each sentence can be processed independently.
- **Configuration choices:**
  - `fieldToSplitOut`: `sentences`
  - `include`: `allOtherFields`
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: `Get sentences`
  - Output: `Loop Over Items`
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:**
  - If `sentences` is missing or not an array, no useful per-sentence output is produced.
  - Including all other fields is harmless here, but can increase payload size if upstream data grows.
- **Sub-workflow reference:** None.

---

## Block 3 — Conditional Humanization Loop

### Overview
This block iterates through each sentence and decides whether it needs rewriting. Sentences scoring above 50 are routed through the Humanizer API; other sentences bypass rewriting and are preserved exactly as they were.

### Nodes Involved
- Loop Over Items
- > 50?
- AI Humanizer
- Text
- Text 2

### Node Details

#### 6) Loop Over Items
- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Implements loop behavior over the split sentence items.
- **Configuration choices:**
  - Default options only
  - Used as an item-by-item iterator
- **Key expressions or variables used:** None directly.
- **Input and output connections:**
  - Input: `Split Out`, `Text`, `Text 2`
  - Output 0: `Aggregate Text` when loop completes
  - Output 1: `> 50?` for each iteration item
- **Version-specific requirements:** Type version 3.
- **Edge cases or potential failure types:**
  - If no items are received, the loop body may not execute and aggregation output may be empty.
  - Incorrect loop-back wiring would break iteration, but the current configuration is valid.
- **Sub-workflow reference:** None.

#### 7) > 50?
- **Type and technical role:** `n8n-nodes-base.if`  
  Applies the sentence score threshold to decide whether to rewrite.
- **Configuration choices:**
  - Numeric condition
  - Left value: `{{$json.sentences.score}}`
  - Operation: greater than
  - Right value: `50`
  - Strict type validation enabled through node options
- **Key expressions or variables used:**
  - `={{ $json.sentences.score }}`
- **Input and output connections:**
  - Input: `Loop Over Items`
  - True output: `AI Humanizer`
  - False output: `Text`
- **Version-specific requirements:** Type version 2.3.
- **Edge cases or potential failure types:**
  - If `score` is missing or non-numeric, the comparison can fail or always evaluate incorrectly.
  - Threshold 50 is a design assumption; changing detector semantics may require updating it.
- **Sub-workflow reference:** None.

#### 8) AI Humanizer
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends flagged sentences to Rephrasy’s Humanizer API for rewriting.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://v2-humanizer.rephrasy.ai/api`
  - Authentication: HTTP Bearer Auth credential named `Rephrasy`
  - Sends JSON body
  - Header `Content-Type: application/json`
  - Request body includes:
    - `text`: `{{$json.sentences.text}}`
    - `model`: `"v3"`
    - `words`: `true`
    - `costs`: `true`
    - `language`: `"English"`
- **Key expressions or variables used:**
  - `{{ $json.sentences.text }}`
- **Input and output connections:**
  - Input: `> 50?` true branch
  - Output: `Text 2`
- **Version-specific requirements:** Type version 4.4.
- **Edge cases or potential failure types:**
  - Invalid bearer token, expired credentials, rate limiting, or API downtime.
  - If the sentence language does not match the `language` parameter, results may degrade.
  - If the API response format changes and no `output` field is returned, the next Set node will fail logically.
  - Long or malformed sentence strings may trigger remote validation issues.
- **Sub-workflow reference:** None.

#### 9) Text
- **Type and technical role:** `n8n-nodes-base.set`  
  Pass-through branch for sentences that are not rewritten. It renames the original sentence content into a unified field called `text`.
- **Configuration choices:**
  - Creates field `text`
  - Value: original sentence text from `sentences.text`
- **Key expressions or variables used:**
  - `={{ $json.sentences.text }}`
- **Input and output connections:**
  - Input: `> 50?` false branch
  - Output: `Loop Over Items`
- **Version-specific requirements:** Type version 3.4.
- **Edge cases or potential failure types:**
  - If `sentences.text` is missing, resulting `text` will be empty or invalid.
- **Sub-workflow reference:** None.

#### 10) Text 2
- **Type and technical role:** `n8n-nodes-base.set`  
  Normalizes the Humanizer API response so rewritten sentences use the same output field name (`text`) as unchanged sentences.
- **Configuration choices:**
  - Creates field `text`
  - Value: `{{$json.output}}`
- **Key expressions or variables used:**
  - `={{ $json.output }}`
- **Input and output connections:**
  - Input: `AI Humanizer`
  - Output: `Loop Over Items`
- **Version-specific requirements:** Type version 3.4.
- **Edge cases or potential failure types:**
  - If the Humanizer API returns a different property name than `output`, this node will not pass the rewritten text correctly.
  - Empty rewrite results will propagate silently unless explicitly validated.
- **Sub-workflow reference:** None.

---

## Block 4 — Reassembly of Final Text

### Overview
After all sentences have been processed, this block collects the unified `text` field from each iteration result. The final output is an aggregated structure containing all sentence texts in order of processing.

### Nodes Involved
- Aggregate Text
- Continue...

### Node Details

#### 11) Aggregate Text
- **Type and technical role:** `n8n-nodes-base.aggregate`  
  Collects the `text` field values from all loop iterations into a single aggregated output.
- **Configuration choices:**
  - Aggregates one field: `text`
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: `Loop Over Items` completion output
  - Output: `Continue...`
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:**
  - Depending on n8n version and aggregate behavior, output may be an array of texts rather than a single concatenated string.
  - If exact sentence spacing or punctuation joining matters, an additional formatting node may be needed after aggregation.
  - If loop results are empty, the aggregate result will also be empty.
- **Sub-workflow reference:** None.

#### 12) Continue...
- **Type and technical role:** `n8n-nodes-base.noOp`  
  Terminal placeholder node that simply exposes the aggregated result at the end of the workflow.
- **Configuration choices:** None.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: `Aggregate Text`
  - Output: none
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:**
  - No operational logic; used mainly as a visual endpoint.
- **Sub-workflow reference:** None.

---

## Non-executable Documentation Nodes

The workflow also contains three sticky notes used for explanation and setup guidance. These nodes do not affect runtime execution but are important for understanding intended behavior.

### 13) Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Purpose:** Documents Step 1, supported languages, score interpretation, and Flesch readability guidance.
- **Runtime behavior:** None.

### 14) Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Purpose:** Documents Step 2, the conditional humanization logic based on score threshold.
- **Runtime behavior:** None.

### 15) Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Purpose:** Explains the overall workflow, setup steps, and the role of Rephrasy credentials and language settings.
- **Runtime behavior:** None.

### 16) Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Purpose:** Documents Step 3, final reassembly.
- **Runtime behavior:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point for testing/execution |  | Set text | ## Detect, Analyze and Humanize AI-generated text automatically<br>The workflow then sends this text to an AI detection service to evaluate how “AI-like” it is. The text is broken down into individual sentences, and each sentence is scored based on its likelihood of being AI-generated.<br><br>### How it works<br><br>The workflow takes an input text, sends it to the Rephrasy AI Detector, and receives sentence-level scores showing how likely each sentence is to be AI-generated. A code step labels each sentence as AI or human, then the workflow loops through them and only routes sentences with a score above 50 to the AI Humanizer for rewriting. Sentences already considered human are kept unchanged, and all results are merged back into one final text.<br><br>### Setup steps<br><br>Create and connect an HTTP Bearer Auth credential in n8n for both the AI Detector and AI Humanizer nodes using your Rephrasy API key. Then update the **Set text** node with the text you want to process, or map in text from another trigger such as a webhook or form. Check that the `language` field in the AI Humanizer node matches your input language, then run the workflow manually or activate it for ongoing use. |
| Set text | Set | Defines the source text payload | When clicking ‘Execute workflow’ | AI Detector | ## STEP 1 - Detect text AI<br>"Set Text" in the relative node<br>Supported Languages:<br> - English,  German, French, Spanish, Italian, Portuguese, Dutch, Polish, Japanese<br><br>Score Interpretation:<br> - Overall Score: Ranges from 0 (human) to 100 (AI).<br> - Sentence Scores (mode = "depth"): 0 is human, 100 is AI.<br> - Other modes: 100 indicates human, 0 indicates AI.<br><br>Flesch Score Interpretation:<br> - The Flesch score measures readability. Higher scores indicate easier-to-read text.<br> - Range: 0-100. A score of 60-70 is easily readable for 13- to 15-year-olds.<br> - A higher score often means simpler language. |
| AI Detector | HTTP Request | Calls Rephrasy detector API with the full text | Set text | Get sentences | ## STEP 1 - Detect text AI<br>"Set Text" in the relative node<br>Supported Languages:<br> - English,  German, French, Spanish, Italian, Portuguese, Dutch, Polish, Japanese<br><br>Score Interpretation:<br> - Overall Score: Ranges from 0 (human) to 100 (AI).<br> - Sentence Scores (mode = "depth"): 0 is human, 100 is AI.<br> - Other modes: 100 indicates human, 0 indicates AI.<br><br>Flesch Score Interpretation:<br> - The Flesch score measures readability. Higher scores indicate easier-to-read text.<br> - Range: 0-100. A score of 60-70 is easily readable for 13- to 15-year-olds.<br> - A higher score often means simpler language. |
| Get sentences | Code | Converts detector response into sentence objects with score/type | AI Detector | Split Out | ## STEP 1 - Detect text AI<br>"Set Text" in the relative node<br>Supported Languages:<br> - English,  German, French, Spanish, Italian, Portuguese, Dutch, Polish, Japanese<br><br>Score Interpretation:<br> - Overall Score: Ranges from 0 (human) to 100 (AI).<br> - Sentence Scores (mode = "depth"): 0 is human, 100 is AI.<br> - Other modes: 100 indicates human, 0 indicates AI.<br><br>Flesch Score Interpretation:<br> - The Flesch score measures readability. Higher scores indicate easier-to-read text.<br> - Range: 0-100. A score of 60-70 is easily readable for 13- to 15-year-olds.<br> - A higher score often means simpler language. |
| Split Out | Split Out | Expands sentence array into one item per sentence | Get sentences | Loop Over Items | ## STEP 1 - Detect text AI<br>"Set Text" in the relative node<br>Supported Languages:<br> - English,  German, French, Spanish, Italian, Portuguese, Dutch, Polish, Japanese<br><br>Score Interpretation:<br> - Overall Score: Ranges from 0 (human) to 100 (AI).<br> - Sentence Scores (mode = "depth"): 0 is human, 100 is AI.<br> - Other modes: 100 indicates human, 0 indicates AI.<br><br>Flesch Score Interpretation:<br> - The Flesch score measures readability. Higher scores indicate easier-to-read text.<br> - Range: 0-100. A score of 60-70 is easily readable for 13- to 15-year-olds.<br> - A higher score often means simpler language. |
| Loop Over Items | Split In Batches | Iterates through sentences and manages loop flow | Split Out, Text, Text 2 | Aggregate Text, > 50? | ## STEP 2 - Conditional Humanization<br><br>For each sentence, the "> 50?" node (IF condition) checks the AI score. If the score is greater than 50 (indicating AI), the workflow routes the sentence to the **AI Humanizer** node to be rewritten. If the score is 50 or less (human), the sentence is passed through unchanged.<br>## STEP 3 - Reassembly<br><br>Once the loop finishes, the “Aggregate Text” node collects all the sentences (both the rewritten AI sentences and the original human sentences) and reassembles them into a single, final text output. |
| > 50? | IF | Routes AI-like sentences to rewriting and human-like sentences to pass-through | Loop Over Items | AI Humanizer, Text | ## STEP 2 - Conditional Humanization<br><br>For each sentence, the "> 50?" node (IF condition) checks the AI score. If the score is greater than 50 (indicating AI), the workflow routes the sentence to the **AI Humanizer** node to be rewritten. If the score is 50 or less (human), the sentence is passed through unchanged. |
| AI Humanizer | HTTP Request | Rewrites flagged sentences via Rephrasy Humanizer | > 50? | Text 2 | ## STEP 2 - Conditional Humanization<br><br>For each sentence, the "> 50?" node (IF condition) checks the AI score. If the score is greater than 50 (indicating AI), the workflow routes the sentence to the **AI Humanizer** node to be rewritten. If the score is 50 or less (human), the sentence is passed through unchanged. |
| Text | Set | Keeps original sentence unchanged in unified output field | > 50? | Loop Over Items | ## STEP 2 - Conditional Humanization<br><br>For each sentence, the "> 50?" node (IF condition) checks the AI score. If the score is greater than 50 (indicating AI), the workflow routes the sentence to the **AI Humanizer** node to be rewritten. If the score is 50 or less (human), the sentence is passed through unchanged. |
| Text 2 | Set | Normalizes rewritten sentence into unified output field | AI Humanizer | Loop Over Items | ## STEP 2 - Conditional Humanization<br><br>For each sentence, the "> 50?" node (IF condition) checks the AI score. If the score is greater than 50 (indicating AI), the workflow routes the sentence to the **AI Humanizer** node to be rewritten. If the score is 50 or less (human), the sentence is passed through unchanged. |
| Aggregate Text | Aggregate | Collects all processed sentence texts after loop completion | Loop Over Items | Continue... | ## STEP 3 - Reassembly<br><br>Once the loop finishes, the “Aggregate Text” node collects all the sentences (both the rewritten AI sentences and the original human sentences) and reassembles them into a single, final text output. |
| Continue... | No Operation, do nothing | Final endpoint placeholder for aggregated result | Aggregate Text |  |  |
| Sticky Note | Sticky Note | Visual documentation for step 1 and scoring guidance |  |  | ## STEP 1 - Detect text AI<br>"Set Text" in the relative node<br>Supported Languages:<br> - English,  German, French, Spanish, Italian, Portuguese, Dutch, Polish, Japanese<br><br>Score Interpretation:<br> - Overall Score: Ranges from 0 (human) to 100 (AI).<br> - Sentence Scores (mode = "depth"): 0 is human, 100 is AI.<br> - Other modes: 100 indicates human, 0 indicates AI.<br><br>Flesch Score Interpretation:<br> - The Flesch score measures readability. Higher scores indicate easier-to-read text.<br> - Range: 0-100. A score of 60-70 is easily readable for 13- to 15-year-olds.<br> - A higher score often means simpler language. |
| Sticky Note1 | Sticky Note | Visual documentation for conditional humanization |  |  | ## STEP 2 - Conditional Humanization<br><br>For each sentence, the "> 50?" node (IF condition) checks the AI score. If the score is greater than 50 (indicating AI), the workflow routes the sentence to the **AI Humanizer** node to be rewritten. If the score is 50 or less (human), the sentence is passed through unchanged. |
| Sticky Note2 | Sticky Note | Visual overview and setup instructions |  |  | ## Detect, Analyze and Humanize AI-generated text automatically<br>The workflow then sends this text to an AI detection service to evaluate how “AI-like” it is. The text is broken down into individual sentences, and each sentence is scored based on its likelihood of being AI-generated.<br><br>### How it works<br><br>The workflow takes an input text, sends it to the Rephrasy AI Detector, and receives sentence-level scores showing how likely each sentence is to be AI-generated. A code step labels each sentence as AI or human, then the workflow loops through them and only routes sentences with a score above 50 to the AI Humanizer for rewriting. Sentences already considered human are kept unchanged, and all results are merged back into one final text.<br><br>### Setup steps<br><br>Create and connect an HTTP Bearer Auth credential in n8n for both the AI Detector and AI Humanizer nodes using your Rephrasy API key. Then update the **Set text** node with the text you want to process, or map in text from another trigger such as a webhook or form. Check that the `language` field in the AI Humanizer node matches your input language, then run the workflow manually or activate it for ongoing use. |
| Sticky Note3 | Sticky Note | Visual documentation for reassembly |  |  | ## STEP 3 - Reassembly<br><br>Once the loop finishes, the “Aggregate Text” node collects all the sentences (both the rewritten AI sentences and the original human sentences) and reassembles them into a single, final text output. |

---

# 4. Reproducing the Workflow from Scratch

Follow these steps to rebuild the workflow manually in n8n.

## Preparation

1. **Create a new workflow** in n8n.
2. **Create an HTTP Bearer Auth credential**:
   - Go to Credentials.
   - Create credential type: **HTTP Bearer Auth**.
   - Name it something like `Rephrasy`.
   - Paste your Rephrasy API key as the bearer token.
3. Keep in mind that both API nodes will use the same credential.

## Build the workflow

### Step 1 — Create the entry point

1. Add a **Manual Trigger** node.
2. Keep the default name or rename it to **When clicking ‘Execute workflow’**.

### Step 2 — Add the source text node

3. Add a **Set** node after the trigger.
4. Rename it to **Set text**.
5. Configure one field:
   - Field name: `text`
   - Type: `String`
   - Value: your sample text, or keep placeholder text during testing
6. Connect:
   - `When clicking ‘Execute workflow’` → `Set text`

### Step 3 — Add the detector API call

7. Add an **HTTP Request** node.
8. Rename it to **AI Detector**.
9. Configure:
   - Method: `POST`
   - URL: `https://detector.rephrasy.ai/detect_api`
   - Authentication: `Generic Credential Type`
   - Generic Auth Type: `HTTP Bearer Auth`
   - Select the `Rephrasy` credential
10. Enable JSON body sending.
11. Add header:
   - `Content-Type` = `application/json`
12. Set the JSON body to:
   ```json
   {
     "text": "{{$json.text}}",
     "mode": "depth"
   }
   ```
13. Connect:
   - `Set text` → `AI Detector`

### Step 4 — Convert detector output into sentence objects

14. Add a **Code** node.
15. Rename it to **Get sentences**.
16. Paste this JavaScript logic:
   ```javascript
   const output = [];

   for (const item of $input.all()) {
     const input = item.json;
     const data = Array.isArray(input) ? input[0] : input;
     const sentences = data.sentences || {};
     const merged = [];

     for (const [text, score] of Object.entries(sentences)) {
       merged.push({
         text,
         score,
         type: score > 50 ? "ai" : "human"
       });
     }

     output.push({
       json: {
         sentences: merged
       }
     });
   }

   return output;
   ```
17. Connect:
   - `AI Detector` → `Get sentences`

### Step 5 — Split the sentence array into one item per sentence

18. Add a **Split Out** node.
19. Rename it to **Split Out**.
20. Configure:
   - Field to split out: `sentences`
   - Include: `All Other Fields`
21. Connect:
   - `Get sentences` → `Split Out`

### Step 6 — Add the loop controller

22. Add a **Loop Over Items** node using **Split In Batches**.
23. Rename it to **Loop Over Items**.
24. Keep default options.
25. Connect:
   - `Split Out` → `Loop Over Items`

### Step 7 — Add the score condition

26. Add an **IF** node.
27. Rename it to **> 50?**
28. Configure a numeric condition:
   - Left value: `{{$json.sentences.score}}`
   - Operation: `Greater Than`
   - Right value: `50`
29. Connect:
   - `Loop Over Items` second/main iteration output → `> 50?`

### Step 8 — Add the humanizer branch for flagged sentences

30. Add an **HTTP Request** node.
31. Rename it to **AI Humanizer**.
32. Configure:
   - Method: `POST`
   - URL: `https://v2-humanizer.rephrasy.ai/api`
   - Authentication: `Generic Credential Type`
   - Generic Auth Type: `HTTP Bearer Auth`
   - Select the same `Rephrasy` credential
33. Enable JSON body sending.
34. Add header:
   - `Content-Type` = `application/json`
35. Set the JSON body to:
   ```json
   {
     "text": "{{$json.sentences.text}}",
     "model": "v3",
     "words": true,
     "costs": true,
     "language": "English"
   }
   ```
36. Important:
   - Change `"English"` to match the language of the text being processed.
   - Supported languages mentioned in the workflow notes: English, German, French, Spanish, Italian, Portuguese, Dutch, Polish, Japanese.
37. Connect:
   - `> 50?` true output → `AI Humanizer`

### Step 9 — Normalize rewritten output

38. Add a **Set** node after the humanizer.
39. Rename it to **Text 2**.
40. Create one field:
   - Field name: `text`
   - Type: `String`
   - Value: `{{$json.output}}`
41. Connect:
   - `AI Humanizer` → `Text 2`

### Step 10 — Add the unchanged pass-through branch

42. Add another **Set** node.
43. Rename it to **Text**.
44. Create one field:
   - Field name: `text`
   - Type: `String`
   - Value: `{{$json.sentences.text}}`
45. Connect:
   - `> 50?` false output → `Text`

### Step 11 — Close the loop

46. Connect the unchanged branch back to the loop:
   - `Text` → `Loop Over Items`
47. Connect the rewritten branch back to the loop:
   - `Text 2` → `Loop Over Items`

This allows each processed sentence to return to the loop controller before the next item is handled.

### Step 12 — Aggregate the final output

48. Add an **Aggregate** node.
49. Rename it to **Aggregate Text**.
50. Configure aggregation for field:
   - `text`
51. Connect:
   - `Loop Over Items` completion output → `Aggregate Text`

### Step 13 — Add a terminal endpoint

52. Add a **No Operation** node.
53. Rename it to **Continue...**
54. Connect:
   - `Aggregate Text` → `Continue...`

## Optional documentation nodes

55. Add **Sticky Note** nodes if you want the same visual guidance:
   - One for overall workflow description and setup
   - One for Step 1 detection details
   - One for Step 2 conditional humanization
   - One for Step 3 reassembly

## Expected input/output behavior

### Input expectation
- The workflow expects a field named `text` before the detector step.

### Detector response expectation
- The detector should return a `sentences` object where:
  - keys are sentence texts,
  - values are numeric AI-likelihood scores.

### Humanizer response expectation
- The humanizer is expected to return a field named `output` containing the rewritten sentence.

### Final output shape
- The Aggregate node collects all `text` values.
- Depending on your n8n version/configuration, this may be:
  - an array of sentence strings, or
  - an aggregated object containing the list.
- If you need one final concatenated paragraph, add an extra Code or Set node after aggregation to join the array with spaces.

## Recommended hardening improvements

56. Add error handling for both HTTP Request nodes:
   - retry logic,
   - timeout settings,
   - branch-on-fail if desired.
57. Validate empty text before calling the detector.
58. Add a post-aggregation formatting step to join sentences cleanly.
59. If using a webhook or form instead of the Manual Trigger, ensure the incoming payload maps into `text`.
60. If processing multilingual input dynamically, make the Humanizer `language` parameter expression-based.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow is titled: **Detect, Analyze and Humanize AI-generated text** | Workflow metadata |
| Purpose: detect AI-generated text at sentence level and rewrite only flagged sentences | Overall workflow design |
| Rephrasy credential must be configured as **HTTP Bearer Auth** and used in both HTTP Request nodes | AI Detector and AI Humanizer |
| `mode = "depth"` is essential because the workflow expects sentence-level detector scores | AI Detector configuration |
| Score interpretation from the workflow notes: overall score ranges from 0 = human to 100 = AI; sentence scores in `depth` mode also use 0 = human and 100 = AI | Detection semantics |
| In other detector modes, the scoring direction may differ according to the note: 100 indicates human and 0 indicates AI | Detector note |
| Flesch readability guidance from the workflow note: higher score means easier-to-read text; 60–70 is generally readable for ages 13–15 | Readability interpretation |
| Supported languages listed in the workflow note: English, German, French, Spanish, Italian, Portuguese, Dutch, Polish, Japanese | Language compatibility |
| The Humanizer node is hardcoded to `language = "English"` and should be changed if the input text is in another supported language | AI Humanizer configuration |
| The workflow uses selective rewriting, which can reduce rewriting volume and cost compared with rewriting the full document | Operational design benefit |