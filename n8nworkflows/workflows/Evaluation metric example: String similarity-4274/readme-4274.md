Evaluation metric example: String similarity

https://n8nworkflows.xyz/workflows/evaluation-metric-example--string-similarity-4274


# Evaluation metric example: String similarity

### 1. Workflow Overview

This workflow demonstrates how to evaluate a handwritten code recognition system using a string similarity metric. Its main purpose is to compare the extracted handwritten code from images against expected codes from a dataset, producing a similarity score between 0 and 1, where 1 indicates a perfect match.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives input either via webhook calls or dataset rows from Google Sheets.
- **1.2 Data Preparation & Download:** Prepares webhook input into the expected format and downloads the image from a URL.
- **1.3 Code Extraction:** Uses an AI model (OpenAI GPT-4o) to extract the handwritten code string from the downloaded image.
- **1.4 Evaluation Check & Metric Calculation:** Checks if the workflow is in evaluation mode, calculates the string similarity score using Levenshtein distance, and sets the evaluation metric.
- **1.5 Response Handling:** Sends back the evaluation result when invoked via webhook.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block handles the initial input to the workflow. It supports two modes: receiving a dataset row from Google Sheets or receiving data via a webhook. This enables flexibility for batch evaluation or real-time evaluation requests.

**Nodes Involved:**  
- When fetching a dataset row  
- Webhook

**Node Details:**

- **When fetching a dataset row**  
  - Type: Evaluation Trigger (Google Sheets integration)  
  - Configuration: Reads rows from a Google Sheets document, using OAuth2 credentials. The sheet URL and document ID are specified.  
  - Key expressions: None in parameters, but outputs JSON containing expected outputs and image file URLs.  
  - Inputs: None (trigger node).  
  - Outputs: Passes row data to "Match webhook format".  
  - Edge cases: Google Sheets API auth errors, rate limits, network failures.

- **Webhook**  
  - Type: Webhook Trigger  
  - Configuration: Listens for HTTP requests on a unique path (UUID-based).  
  - Key expressions: None.  
  - Inputs: None (trigger node).  
  - Outputs: Passes received data to "Download image".  
  - Edge cases: Unauthorized access, malformed requests, missing parameters.

---

#### 2.2 Data Preparation & Download

**Overview:**  
This block formats incoming dataset rows into the expected webhook format and downloads the handwritten image from the provided URL.

**Nodes Involved:**  
- Match webhook format  
- Download image

**Node Details:**

- **Match webhook format**  
  - Type: Set  
  - Configuration: Transforms input JSON into a structure imitating the webhook payload format, specifically placing the image URL under `query.url`.  
  - Key expressions:  
    ```javascript
    {
      "headers": {},
      "params": {},
      "query": { "url": {{ $json.file_url.toJsonString() }} },
      "body": {},
      "executionMode": "test"
    }
    ```  
  - Inputs: From "When fetching a dataset row".  
  - Outputs: To "Download image".  
  - Edge cases: Missing or invalid `file_url` field causing malformed URL.

- **Download image**  
  - Type: HTTP Request  
  - Configuration: Downloads image from the URL extracted from `query.url`.  
  - Key expressions: URL set dynamically from input JSON.  
  - Inputs: From "Webhook" or "Match webhook format".  
  - Outputs: Base64 image data passed to "Extract code from image".  
  - Edge cases: HTTP errors (404, 500), invalid URL, network timeouts.

---

#### 2.3 Code Extraction

**Overview:**  
This block extracts the handwritten code from the image using an AI model, enforcing strict formatting rules for the code.

**Nodes Involved:**  
- Extract code from image

**Node Details:**

- **Extract code from image**  
  - Type: OpenAI (Langchain integration)  
  - Configuration: Uses GPT-4o model to analyze the base64 image input.  
  - Prompt: Instructs the model to extract ONLY the handwritten code in the top-right corner, enforcing a strict regex-like format for the code (BT/ED/... pattern). The prompt forbids any extra text or explanations and requires validation of the format.  
  - Inputs: Base64 image from "Download image".  
  - Outputs: Extracted code string to "Evaluating?".  
  - Credentials: OpenAI API key configured.  
  - Edge cases: Model fails to extract code, returns no output if format mismatch, API rate limits or errors, prompt misinterpretation.

---

#### 2.4 Evaluation Check & Metric Calculation

**Overview:**  
This block verifies if the workflow is running in evaluation mode, computes the similarity score between expected and extracted codes using Levenshtein distance, and sets the metric for evaluation tracking.

**Nodes Involved:**  
- Evaluating?  
- Calc string distance  
- Set metrics

**Node Details:**

- **Evaluating?**  
  - Type: Evaluation  
  - Configuration: Checks the evaluation state (operation: checkIfEvaluating).  
  - Inputs: From "Extract code from image".  
  - Outputs: Two branches: one to "Calc string distance" (if evaluating), another to "Respond to Webhook" (if not).  
  - Edge cases: Evaluation state not set properly, causing incorrect branching.

- **Calc string distance**  
  - Type: Code (JavaScript)  
  - Configuration: Runs custom JS code once per item to compute Levenshtein distance between expected code (from dataset) and actual extracted code.  
  - Key expressions:  
    - Retrieves expected code from "When fetching a dataset row".  
    - Uses a standard dynamic programming algorithm to compute edit distance.  
    - Calculates score as `1 - (distance / max_length)`.  
    - Adds score to item JSON under `score`.  
  - Inputs: From "Evaluating?".  
  - Outputs: Passes scored data to "Set metrics".  
  - Edge cases: Missing expected or actual code causing errors, division by zero if both codes empty.

- **Set metrics**  
  - Type: Evaluation  
  - Configuration: Sets the evaluation metric named `score` with the calculated similarity score value.  
  - Inputs: From "Calc string distance".  
  - Outputs: None explicitly connected.  
  - Edge cases: Metric assignment failures, invalid score values.

---

#### 2.5 Response Handling

**Overview:**  
This block handles the HTTP response when the workflow is invoked via webhook, sending back the evaluation results or status.

**Nodes Involved:**  
- Respond to Webhook

**Node Details:**

- **Respond to Webhook**  
  - Type: Respond to Webhook  
  - Configuration: Returns response to the webhook caller, typically after finishing evaluation or if not evaluating.  
  - Inputs: From "Evaluating?" fallback branch.  
  - Outputs: Ends workflow execution.  
  - Edge cases: Response timeout, malformed response payloads.

---

### 3. Summary Table

| Node Name               | Node Type                  | Functional Role                          | Input Node(s)              | Output Node(s)            | Sticky Note                                                                                     |
|-------------------------|----------------------------|----------------------------------------|----------------------------|---------------------------|------------------------------------------------------------------------------------------------|
| Sticky Note1            | Sticky Note                | Informational note about string score  |                            |                           | Check how far apart the actual code is from the expected code (a score of 1 is a perfect match) |
| Sticky Note3            | Sticky Note                | Introductory explanation of workflow   |                            |                           | Explains workflow purpose, shows example image, and links to evaluation docs                   |
| Sticky Note4            | Sticky Note                | Dataset info note                       |                            |                           | Read in [this test dataset](https://docs.google.com/spreadsheets/d/1uuPS5cHtSNZ6HNLOi75A2m8nVWZrdBZ_Ivf58osDAS8/edit?gid=1786963566#gid=1786963566) of images |
| When fetching a dataset row | Evaluation Trigger       | Reads dataset rows from Google Sheets  |                            | Match webhook format       |                                                                                                |
| Webhook                 | Webhook                    | Receives real-time input requests      |                            | Download image             |                                                                                                |
| Match webhook format    | Set                        | Formats dataset row into webhook format| When fetching a dataset row | Download image             |                                                                                                |
| Download image          | HTTP Request               | Downloads image from URL                | Webhook, Match webhook format | Extract code from image    |                                                                                                |
| Extract code from image | OpenAI (Langchain)         | Extracts handwritten code from image   | Download image              | Evaluating?                |                                                                                                |
| Evaluating?             | Evaluation                 | Checks evaluation mode flag             | Extract code from image     | Calc string distance, Respond to Webhook |                                                                                                |
| Calc string distance    | Code                       | Calculates similarity score             | Evaluating?                 | Set metrics                |                                                                                                |
| Set metrics             | Evaluation                 | Sets the evaluation metric              | Calc string distance        |                           |                                                                                                |
| Respond to Webhook      | Respond to Webhook         | Sends HTTP response to webhook caller  | Evaluating?                 |                           |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Create a **Webhook** node.
     - Path: Use a unique UUID or string (e.g., `7ceb775c-b961-44f0-acfe-682a67612332`).
     - No credentials needed.
   - Create an **Evaluation Trigger** node named "When fetching a dataset row".
     - Configure with Google Sheets OAuth2 credentials.
     - Set Document ID and Sheet Name to the Google Sheets dataset URL:
       `https://docs.google.com/spreadsheets/d/1uuPS5cHtSNZ6HNLOi75A2m8nVWZrdBZ_Ivf58osDAS8/edit?gid=1786963566#gid=1786963566`.

2. **Data Preparation:**
   - Create a **Set** node named "Match webhook format".
     - Mode: Raw JSON.
     - Set JSON output to:
       ```javascript
       {
         "headers": {},
         "params": {},
         "query": { "url": {{ $json.file_url.toJsonString() }} },
         "body": {},
         "executionMode": "test"
       }
       ```
     - Connect "When fetching a dataset row" → "Match webhook format".

3. **Download Image:**
   - Create an **HTTP Request** node named "Download image".
     - HTTP Method: GET (default).
     - URL: Set dynamically from input JSON path `query.url`.
     - Connect both "Webhook" and "Match webhook format" nodes to "Download image".

4. **Extract Code from Image:**
   - Create an **OpenAI (Langchain)** node named "Extract code from image".
     - Resource: Image.
     - Operation: Analyze.
     - Input Type: Base64.
     - Model: GPT-4o (or equivalent with image analysis capability).
     - Text prompt:
       ```
       Extract ONLY the handwritten code in the top-right corner of this image.

       The code MUST follow this EXACT format:
       BT/ED/[1-3 capital letters]/[1-3 capital letters]/[1-3 capital letters]/[1-3 capital letters or empty]/[single letter + number (2-4 chars total)]

       Examples of correct format:
       BT/ED/ABC/DE/F/G/H1
       BT/ED/A/BC/DEF/GH/I23
       BT/ED/AB/CD/EF/GH/I234

       DO NOT include any explanations, notes, or other text.
       DO NOT return anything if the code doesn't match the required format.
       VERIFY the extracted code matches the format before returning it.
       Return ONLY the extracted code - nothing else.
       ```
     - Connect "Download image" → "Extract code from image".
     - Add OpenAI API credentials.

5. **Evaluation Mode Check:**
   - Create an **Evaluation** node named "Evaluating?".
     - Operation: checkIfEvaluating.
     - Connect "Extract code from image" → "Evaluating?".

6. **Calculate String Distance:**
   - Create a **Code** node named "Calc string distance".
     - Mode: Run once for each item.
     - JavaScript code:
       ```javascript
       const expected_code = $('When fetching a dataset row').item.json.expected_output
       const actual_code = $json.content

       function levenshteinDistance(str1, str2) {
         const m = str1.length;
         const n = str2.length;
         const dp = Array(m + 1).fill().map(() => Array(n + 1).fill(0));

         for (let i = 0; i <= m; i++) dp[i][0] = i;
         for (let j = 0; j <= n; j++) dp[0][j] = j;

         for (let i = 1; i <= m; i++) {
           for (let j = 1; j <= n; j++) {
             if (str1[i - 1] === str2[j - 1]) dp[i][j] = dp[i - 1][j - 1];
             else dp[i][j] = 1 + Math.min(dp[i - 1][j], dp[i][j - 1], dp[i - 1][j - 1]);
           }
         }
         return dp[m][n];
       }

       const dist = levenshteinDistance(expected_code, actual_code);
       const max_dist = Math.max(expected_code.length, actual_code.length);

       $input.item.json.score = 1 - (dist / max_dist);
       return $input.item;
       ```
     - Connect "Evaluating?" success output → "Calc string distance".

7. **Set Evaluation Metric:**
   - Create an **Evaluation** node named "Set metrics".
     - Operation: setMetrics.
     - Metrics configuration:
       - Assign metric `score` as a number, value from expression `{{ $json.score }}`.
     - Connect "Calc string distance" → "Set metrics".

8. **Respond to Webhook:**
   - Create a **Respond to Webhook** node.
     - Default configuration.
     - Connect "Evaluating?" failure/no-evaluation output → "Respond to Webhook".

9. **Connect the Webhook trigger output to "Download image"** for real-time requests.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                          | Context or Link                                                                                                             |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| This workflow evaluates handwritten code recognition by comparing extracted codes against expected answers using character-by-character similarity.                                                                                | Workflow purpose                                                                                                            |
| More about workflow evaluation and metric calculation examples are available at: https://docs.n8n.io/advanced-ai/evaluations/overview and https://docs.n8n.io/advanced-ai/evaluations/metric-based-evaluations/#2-calculate-metrics    | Official n8n documentation                                                                                                  |
| Example handwritten image used in this workflow: ![image](https://storage.googleapis.com/n8n_template_data/handwriting_scans/doc20250302_08223946_001.jpg)                                                                         | Visual example                                                                                                              |
| Dataset source for test images and expected outputs: https://docs.google.com/spreadsheets/d/1uuPS5cHtSNZ6HNLOi75A2m8nVWZrdBZ_Ivf58osDAS8/edit?gid=1786963566#gid=1786963566                                                         | Google Sheets dataset                                                                                                       |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All processed data are legal and public.