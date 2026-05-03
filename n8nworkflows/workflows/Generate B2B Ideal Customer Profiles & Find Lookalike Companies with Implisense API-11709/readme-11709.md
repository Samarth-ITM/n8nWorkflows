Generate B2B Ideal Customer Profiles & Find Lookalike Companies with Implisense API

https://n8nworkflows.xyz/workflows/generate-b2b-ideal-customer-profiles---find-lookalike-companies-with-implisense-api-11709


# Generate B2B Ideal Customer Profiles & Find Lookalike Companies with Implisense API

### 1. Workflow Overview

This workflow, titled **"Generate B2B Ideal Customer Profiles & Find Lookalike Companies with Implisense API"**, is designed to identify and analyze ideal customer profiles (ICPs) for B2B companies and discover similar companies (lookalikes) using the Implisense API. The workflow specifically targets companies in Germany and generates both a detailed ICP report and a curated list of lookalike companies enriched with CRM-ready data fields.

The workflow is logically divided into the following blocks:

- **1.1 Input Initialization**: Receives manual trigger and sets up API authorization and configuration parameters.
- **1.2 Base Company IDs Preparation**: Provides a mock list of base company Implisense IDs and serializes them for API consumption.
- **1.3 Lookalike Companies Retrieval**: Queries Implisense API to retrieve recommended lookalike companies based on the base companies.
- **1.4 Conditional Branch for Explanation Mode**: Depending on the `explain` flag, either processes term features for ICP report generation or directly outputs lookalike companies.
- **1.5 Term Feature Processing and ICP Generation**: Extracts, filters, sorts, and digests term features in English and German; then uses OpenAI GPT to generate a detailed ICP report.
- **1.6 Output Preparation**: Formats both the list of lookalike companies and the ICP report for downstream consumption or export.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Initialization

- **Overview:** This block sets the starting point of the workflow, enabling manual execution and configuring API credentials and initial request parameters.
- **Nodes Involved:**  
  - Manual Trigger  
  - Authorization (Set node)  
  - Configuration (Set node)

- **Node Details:**

  - **Manual Trigger**
    - Type: Manual Trigger node  
    - Role: Starts the workflow manually.  
    - Configuration: Default, no parameters.  
    - Input: None  
    - Output: Triggers next node (Authorization).  
    - Edge cases: None significant.

  - **Authorization**
    - Type: Set node  
    - Role: Sets the Implisense API key as HTTP header parameter `x-api-key`.  
    - Configuration: API key must be set here (placeholder "XXXX" in template).  
    - Input: Manual Trigger  
    - Output: Configuration node  
    - Notes: API key must be obtained from https://implisense.com/de/contact  
    - Edge cases: Missing or invalid API key will cause authentication errors in downstream HTTP requests.

  - **Configuration**
    - Type: Set node  
    - Role: Sets two parameters for the Implisense API request:  
      - `explain` (boolean): Whether to enable explanation mode (false by default).  
      - `size` (number): Number of lookalike companies requested (default 3).  
    - Input: Authorization  
    - Output: Mock ICP Companies node  
    - Edge cases: Invalid values may cause unexpected API behavior or empty responses.

---

#### Block 1.2: Base Company IDs Preparation

- **Overview:** Provides a predefined list of base companies by Implisense ID (mock data), serializes the IDs into a string suitable for API requests.
- **Nodes Involved:**  
  - Mock ICP Companies based on Implisense-ID (Code node)  
  - serialize_ids (Function node)  
  - trim (Set node)

- **Node Details:**

  - **Mock ICP Companies based on Implisense-ID**
    - Type: Code node (JavaScript)  
    - Role: Supplies a hardcoded list of Implisense company IDs as input data.  
    - Configuration: Returns an array with one JSON object containing an `id` array of Implisense IDs.  
    - Input: Configuration node  
    - Output: serialize_ids node  
    - Edge cases: Should be replaced with real matched companies in production.

  - **serialize_ids**
    - Type: Function node  
    - Role: Converts the array of company IDs into a serialized comma-separated string.  
    - Configuration: Iterates over input items, concatenates `id` values with commas.  
    - Key expression: Builds a string `serialization` by concatenating all IDs separated by commas.  
    - Input: Mock ICP Companies node  
    - Output: trim node  
    - Edge cases: Empty or malformed input array results in empty string or errors.

  - **trim**
    - Type: Set node  
    - Role: Transforms the serialized string by replacing commas with "," and trims whitespace to fit API body format.  
    - Configuration: Uses expression to replace commas with `","` and trim spaces.  
    - Input: serialize_ids node  
    - Output: get_lookalikes node  
    - Edge cases: Expression failure if input is null or unexpected format.

---

#### Block 1.3: Lookalike Companies Retrieval

- **Overview:** Sends a POST request to Implisense API to get recommended lookalike companies based on the base company IDs and configured filters.
- **Nodes Involved:**  
  - get_lookalikes (HTTP Request node)  
  - lookalikes_or_icp (Switch node)

- **Node Details:**

  - **get_lookalikes**
    - Type: HTTP Request node  
    - Role: Makes POST request to Implisense `/recommend` endpoint with serialized base company IDs and location filter (hardcoded to Berlin state `de-be`).  
    - Configuration:  
      - URL dynamically constructed including `explain` and `size` from Configuration node.  
      - Request body includes `baseCompanies` as array of IDs and `locationsFilter` for state `de-be`.  
      - Headers include API key from Authorization node.  
      - Retries on failure up to 2 times with 5 seconds delay.  
      - Continues workflow on failure to allow fallback.  
    - Input: trim node  
    - Output: lookalikes_or_icp node  
    - Edge cases: API errors, network timeouts, invalid or empty company IDs, authentication failure.

  - **lookalikes_or_icp**
    - Type: Switch node  
    - Role: Routes workflow depending on `explain` flag from Configuration node.  
      - If `explain` is true: route to ICP generation pipeline.  
      - Else: route to output lookalike companies.  
    - Configuration: Uses boolean comparison on `explain` parameter.  
    - Input: get_lookalikes node  
    - Output: Two outputs - one for ICP generation, one for lookalike companies.  
    - Edge cases: Incorrect or missing `explain` parameter may cause misrouting.

---

#### Block 1.4: Conditional Branch for Explanation Mode

- **Overview:** Based on the switch, either processes term features for ICP report or extracts lookalike companies.
- **Nodes Involved:**  
  - parse_term_features (Function node) [ICP branch]  
  - get_companies (SplitOut node) [Lookalike branch]

- **Node Details:**

  - **parse_term_features**
    - Type: Function node  
    - Role: Extracts the array of `features` from `targetProfile` in API response for ICP analysis.  
    - Configuration: Iterates over `items[0].json.targetProfile.features` and returns each feature as a separate item.  
    - Input: lookalikes_or_icp node (ICP output)  
    - Output: Filter term.en and Filter term.de nodes  
    - Edge cases: Missing or malformed `targetProfile` or `features` array causes empty output or errors.  
    - On error: Configured to continue workflow despite errors.

  - **get_companies**
    - Type: SplitOut node  
    - Role: Extracts `companies` array from the API response for lookalike companies output.  
    - Input: lookalikes_or_icp node (lookalike output)  
    - Output: list_of_companies node  
    - Edge cases: Missing or empty `companies` array results in no output.

---

#### Block 1.5: Term Feature Processing and ICP Generation

- **Overview:** Processes the term features to create digestible summaries in English and German, combines them, and triggers an OpenAI GPT model to generate a detailed ICP report.
- **Nodes Involved:**  
  - Filter term.en (Function node)  
  - Filter term.de (Function node)  
  - sort_desc (Function node)  
  - sort_desc2 (Function node)  
  - digest_terms_en (Function node)  
  - digest_terms_de (Function node)  
  - merge_terms (Merge node)  
  - set_f2 (Set node)  
  - generate_icp_report (OpenAI node)  
  - icp_report (Set node)

- **Node Details:**

  - **Filter term.en**
    - Type: Function node  
    - Role: Filters term features for English language keys (`term.en`).  
    - Input: parse_term_features  
    - Output: sort_desc  
    - Edge cases: If no English terms found, outputs empty.

  - **Filter term.de**
    - Type: Function node  
    - Role: Filters term features for German language keys (`term.de`).  
    - Input: parse_term_features  
    - Output: sort_desc2  
    - Edge cases: Same as above for German terms.

  - **sort_desc** (English terms)
    - Type: Function node  
    - Role: Sorts English term features descending by `targetCount` (frequency).  
    - Input: Filter term.en  
    - Output: digest_terms_en  
    - Edge cases: Handles empty input gracefully.

  - **sort_desc2** (German terms)
    - Type: Function node  
    - Role: Sorts German term features descending by `targetCount`.  
    - Input: Filter term.de  
    - Output: digest_terms_de  
    - Edge cases: Same as above.

  - **digest_terms_en**
    - Type: Function node  
    - Role: Aggregates sorted English terms into a string digest with format `value: targetCount`.  
    - Input: sort_desc  
    - Output: merge_terms (first input)  
    - Edge cases: Empty input results in empty digest string.

  - **digest_terms_de**
    - Type: Function node  
    - Role: Aggregates sorted German terms into a similar digest string.  
    - Input: sort_desc2  
    - Output: merge_terms (second input)  
    - Edge cases: Same as above.

  - **merge_terms**
    - Type: Merge node  
    - Role: Combines the two digests into one output multiplexing both inputs.  
    - Input: digest_terms_en and digest_terms_de  
    - Output: set_f2  
    - Edge cases: Handles empty inputs by passing through non-empty side.

  - **set_f2**
    - Type: Set node  
    - Role: Extracts the digest string from JSON to a field named `text` for easy consumption by OpenAI node.  
    - Input: merge_terms  
    - Output: generate_icp_report  
    - Edge cases: Missing digest fields results in empty text.

  - **generate_icp_report**
    - Type: LangChain OpenAI node  
    - Role: Calls OpenAI GPT model (`gpt-5-mini`) to generate a detailed text ICP report based on the digested keyword statistics.  
    - Configuration:  
      - System message defines strict analyst role and detailed instructions to analyze keywords and produce the ICP report with multiple sections (technologies, products, needs, etc.).  
      - User message content comes from `text` field (digest).  
      - Uses OpenAI credentials stored in workflow.  
    - Input: set_f2  
    - Output: icp_report  
    - Edge cases: OpenAI API errors, rate limits, malformed input text.

  - **icp_report**
    - Type: Set node  
    - Role: Extracts the final generated ICP report text from OpenAI output for downstream use.  
    - Input: generate_icp_report  
    - Output: Terminal output (end of ICP branch)  
    - Edge cases: Missing or unexpected OpenAI response format.

---

#### Block 1.6: Output Preparation for Lookalikes

- **Overview:** Processes lookalike companies data by splitting, mapping CRM fields, and preparing a clean output list.
- **Nodes Involved:**  
  - get_companies (SplitOut node)  
  - list_of_companies (Set node)

- **Node Details:**

  - **get_companies**
    - (Described above in Block 1.4)  
    - Splits array of companies into individual items for processing.

  - **list_of_companies**
    - Type: Set node  
    - Role: Maps Implisense company data fields to CRM-friendly field names for easy integration.  
    - Configuration:  
      - Maps Implisense properties like `name`, `url`, `city`, `zip`, `street`, `id`, `profile`, `score`, `active`, and `explanation` to standardized fields prefixed with `crm`.  
      - Calculates `crmLeadScore` by multiplying Implisense `score` by 100 and rounding.  
      - Sets boolean for active company status.  
    - Input: get_companies  
    - Output: Terminal output (end of lookalike branch)  
    - Edge cases: Missing fields in input JSON may cause empty or incorrect CRM fields.

---

### 3. Summary Table

| Node Name                          | Node Type              | Functional Role                                      | Input Node(s)                        | Output Node(s)                      | Sticky Note                                                                                                                      |
|-----------------------------------|------------------------|-----------------------------------------------------|------------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Manual Trigger                    | Manual Trigger         | Starts workflow manually                             | -                                  | Authorization                     | ## Input                                                                                                                         |
| Authorization                    | Set                    | Sets API key for Implisense API                      | Manual Trigger                     | Configuration                    | Get API key here: https://implisense.com/de/contact                                                                             |
| Configuration                   | Set                    | Sets API request parameters (explain, size)         | Authorization                     | Mock ICP Companies based on Implisense-ID |                                                                                                                                |
| Mock ICP Companies based on Implisense-ID | Code                   | Provides mock base company IDs                       | Configuration                    | serialize_ids                   |                                                                                                                                |
| serialize_ids                   | Function                | Serializes array of company IDs to comma-separated string | Mock ICP Companies based on Implisense-ID | trim                           |                                                                                                                                |
| trim                            | Set                     | Formats serialized string for API request            | serialize_ids                    | get_lookalikes                 |                                                                                                                                |
| get_lookalikes                 | HTTP Request            | Calls Implisense API to get lookalike companies      | trim                            | lookalikes_or_icp             | Get your API key here: https://implisense.com/de/contact                                                                        |
| lookalikes_or_icp              | Switch                  | Routes based on explain flag to ICP report or lookalikes | get_lookalikes                 | parse_term_features, get_companies |                                                                                                                                |
| parse_term_features            | Function                | Extracts term features from API response             | lookalikes_or_icp (ICP branch)   | Filter term.en, Filter term.de  |                                                                                                                                |
| Filter term.en                 | Function                | Filters English language term features                | parse_term_features             | sort_desc                      |                                                                                                                                |
| Filter term.de                 | Function                | Filters German language term features                 | parse_term_features             | sort_desc2                     |                                                                                                                                |
| sort_desc                     | Function                | Sorts English terms descending by frequency           | Filter term.en                  | digest_terms_en                |                                                                                                                                |
| sort_desc2                    | Function                | Sorts German terms descending by frequency            | Filter term.de                  | digest_terms_de                |                                                                                                                                |
| digest_terms_en               | Function                | Aggregates English terms into digest string           | sort_desc                      | merge_terms                   |                                                                                                                                |
| digest_terms_de               | Function                | Aggregates German terms into digest string            | sort_desc2                     | merge_terms                   |                                                                                                                                |
| merge_terms                  | Merge                   | Combines English and German digests                   | digest_terms_en, digest_terms_de | set_f2                        |                                                                                                                                |
| set_f2                       | Set                     | Prepares text field from digest for OpenAI input      | merge_terms                   | generate_icp_report          |                                                                                                                                |
| generate_icp_report          | OpenAI (LangChain)      | Generates detailed ICP report using GPT model         | set_f2                        | icp_report                   |                                                                                                                                |
| icp_report                  | Set                     | Extracts ICP report text from OpenAI response         | generate_icp_report           | (end)                        |                                                                                                                                |
| get_companies               | SplitOut                | Splits lookalike companies array                      | lookalikes_or_icp (lookalike branch) | list_of_companies            |                                                                                                                                |
| list_of_companies           | Set                     | Maps company data to CRM-ready fields                  | get_companies                 | (end)                        |                                                                                                                                |
| Sticky Note1                | Sticky Note             | Labels Output section                                  | -                            | -                            | ## Output                                                                                                                      |
| Sticky Note2                | Sticky Note             | Labels get_lookalikes section                          | -                            | -                            | ## get_lookalikes                                                                                                              |
| Sticky Note3                | Sticky Note             | Labels get_icp section                                 | -                            | -                            | ## get_icp                                                                                                                    |
| Sticky Note4                | Sticky Note             | Labels Input section                                   | -                            | -                            | ## Input                                                                                                                      |
| 🏗️ Architecture Notes2     | Sticky Note             | Detailed architectural notes and setup instructions  | -                            | -                            | # B2B ICP Report & Lookalike Companies in Germany ... (full multi-paragraph content)                                            |

---

### 4. Reproducing the Workflow from Scratch

**Step 1:** Create a **Manual Trigger** node to start the workflow.

**Step 2:** Add a **Set** node named "Authorization" to set the Implisense API key header:  
- Add a string field named `x-api-key` with your API key value (get from https://implisense.com/de/contact).

**Step 3:** Connect "Manual Trigger" → "Authorization".

**Step 4:** Add a **Set** node named "Configuration" with fields:  
- `explain` (boolean), default `false`. Controls if ICP explanation is generated.  
- `size` (number), default `3`. Number of lookalike companies to request.

Connect "Authorization" → "Configuration".

**Step 5:** Add a **Code** node named "Mock ICP Companies based on Implisense-ID" that returns a JSON object with a key `id` containing an array of Implisense company IDs (replace with your base companies):  
```js
return [
  {
    "id": [
      "DEXEF64Q6Q57", "DEMHMCXXVX35", "DE94CIJM2285", /* etc. */
    ]
  }
];
```
Connect "Configuration" → "Mock ICP Companies based on Implisense-ID".

**Step 6:** Create a **Function** node "serialize_ids" to serialize the array of IDs into a comma-separated string:  
```js
let serialization = '';
for (const item of items) {
  serialization += `${item.json.id},`;
}
return [{ json: { serialization } }];
```
Connect "Mock ICP Companies based on Implisense-ID" → "serialize_ids".

**Step 7:** Create a **Set** node "trim" to transform the serialized string by replacing commas with `","` and trimming spaces:  
- Add string field `serialization` with expression:  
```js
{{ $json.serialization.replace(/[,]/g,'","').trim() }}
```
Connect "serialize_ids" → "trim".

**Step 8:** Add an **HTTP Request** node "get_lookalikes":  
- Request URL:  
```
https://api.implisense.com/recommend?explain={{ $('Configuration').item.json.explain }}&size={{ $('Configuration').item.json.size }}
```  
- Method: POST  
- Headers: Add header `x-api-key` with value from "Authorization" node  
- Body (JSON):  
```json
{
  "baseCompanies": [
    "{{ $json[\"serialization\"].slice(0, -3) }}"
  ],
  "locationsFilter": [
    {
      "type": "state",
      "code": "de-be"
    }
  ]
}
```
- Enable retry on failure (max 2 tries, 5 sec wait).  
Connect "trim" → "get_lookalikes".

**Step 9:** Add a **Switch** node "lookalikes_or_icp" to route based on `explain` field in Configuration:  
- Rule: If `explain` equals `true`, route to ICP branch.  
- Else route to lookalike companies branch.  
Connect "get_lookalikes" → "lookalikes_or_icp".

**Step 10:** ICP Branch - Add a **Function** node "parse_term_features":  
- Extracts `items[0].json.targetProfile.features` into individual items:  
```js
const newItems = [];
for (const item of items[0].json.targetProfile.features) {
  newItems.push({ json: item });
}
return newItems;
```
Connect "lookalikes_or_icp" ICP output → "parse_term_features".

**Step 11:** Add two **Function** nodes "Filter term.en" and "Filter term.de":  
- "Filter term.en" filters items where `json.key === 'term.en'`.  
- "Filter term.de" filters items where `json.key === 'term.de'`.  
Connect "parse_term_features" → both filter nodes in parallel.

**Step 12:** Add **Function** nodes "sort_desc" and "sort_desc2":  
- Sort by `json.targetCount` descending for English and German respectively.  
Connect "Filter term.en" → "sort_desc" → "digest_terms_en"  
Connect "Filter term.de" → "sort_desc2" → "digest_terms_de"

Code for sort_desc / sort_desc2:  
```js
return items.sort((a,b) => b.json.targetCount - a.json.targetCount);
```

**Step 13:** Add **Function** nodes "digest_terms_en" and "digest_terms_de":  
- Concatenate `value: targetCount` pairs into a newline-separated string.  
Connect "sort_desc" → "digest_terms_en"  
Connect "sort_desc2" → "digest_terms_de"

Example code:  
```js
let digest = '';
for (const item of items) {
  digest += `${item.json.value}: ${item.json.targetCount},\n`;
}
return [{ json: { digest } }];
```

**Step 14:** Add a **Merge** node "merge_terms" in Combine mode multiplexing both digests.  
Connect "digest_terms_en" and "digest_terms_de" → "merge_terms".

**Step 15:** Add a **Set** node "set_f2":  
- Maps `digest` field to `text` for input to OpenAI node.  
Connect "merge_terms" → "set_f2".

**Step 16:** Add an **OpenAI (LangChain)** node "generate_icp_report":  
- Model: `gpt-5-mini`  
- System prompt: Provides detailed instructions for generating ICP report analyzing terms and frequencies, including sections for technologies, products, needs, personas, etc.  
- User message: `={{ $json.text }}` (the digest from previous node)  
- Credentials: Connect your OpenAI API key credential.  
Connect "set_f2" → "generate_icp_report".

**Step 17:** Add a **Set** node "icp_report":  
- Extracts final ICP report text from OpenAI response:  
```js
{{ $json.output[0].content[0].text }}
```  
Connect "generate_icp_report" → "icp_report".

**Step 18:** Lookalike companies branch - Add a **SplitOut** node "get_companies":  
- Splits `companies` array from API response into individual items.  
Connect "lookalikes_or_icp" lookalike output → "get_companies".

**Step 19:** Add a **Set** node "list_of_companies":  
- Map Implisense company data fields to CRM-specific fields:  
  - crmLeadScore = rounded `score * 100`  
  - crmAccountName = `name`  
  - crmDomain = `url`  
  - crmCity, crmZip, crmStreet, crmImplisenseId, crmProfileUrl, crmIsActiveCompany, crmExplanation mapped accordingly.  
Connect "get_companies" → "list_of_companies".

**Step 20:** Finalize outputs as needed for further integrations or export.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Context or Link                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| This workflow is specifically designed for the German B2B market using Implisense’s company dataset and API. It generates an Ideal Customer Profile report based on keyword frequency analysis and retrieves lookalike companies limited by geographic and size filters. The ICP report is generated by a GPT language model using detailed analyst-style instructions. The workflow advises to use high-quality base companies and to carefully configure filters by region, industry, and company size for best accuracy. CRM field mappings are provided for direct integration into sales or marketing systems.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | See full architectural notes sticky (🏗️ Architecture Notes2 node) for detailed setup and best practices. |
| API Key for Implisense must be obtained from https://implisense.com/de/contact and set properly in the Authorization node.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Implisense API key documentation                  |
| The `explain` flag controls whether the ICP report generation is performed; set to `true` to enable detailed feature analysis and report generation, otherwise the workflow only outputs lookalike companies.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Configuration node parameter                      |
| The OpenAI GPT model used is `gpt-5-mini` with LangChain integration; ensure OpenAI credentials are configured properly in n8n for this node to function.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | OpenAI API documentation                           |
| Locations filter in the Implisense API request is currently hardcoded to Berlin state (`de-be`), but can be adjusted to other German states or regions to constrain recommendations.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Implisense API locations and filters              |
| Use only high-quality and homogeneous base companies for best lookalike recommendations, as mixing heterogeneous companies dilutes statistical signals in Implisense's recommendation engine.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Architectural best practices                        |
| The workflow includes multiple sticky notes that label the input section, output section, and major logical blocks (`## Input`, `## Output`, `## get_icp`, `## get_lookalikes`) for better readability and maintenance.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Visual labels within workflow                       |

---

**Disclaimer:**  
The provided description and analysis comes exclusively from an automation workflow created in n8n, adhering strictly to content policies. All data processed is legal and public. No illegal, offensive, or protected content is included.