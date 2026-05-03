Website Scam Risk Detector with GPT-4o and SerpAPI

https://n8nworkflows.xyz/workflows/website-scam-risk-detector-with-gpt-4o-and-serpapi-5614


# Website Scam Risk Detector with GPT-4o and SerpAPI

---

### 1. Workflow Overview

This workflow, titled **"Website Scam Risk Detector with GPT-4o and SerpAPI"**, is designed to evaluate the likelihood that a given website URL is a scam or fraudulent. It leverages multiple specialized AI agents powered by OpenAI’s GPT-4o models, combined with real-time data retrieval via SerpAPI, a Google Search API. The workflow accepts a website URL via a form submission, performs multifaceted analysis through domain and technical details checks, search engine signals, product/pricing pattern examinations, and content analysis. The results from these analyses are aggregated and interpreted by a final evaluation agent to produce a scam risk score (1-10 scale) along with detailed reasoning.

The key logical blocks are:

- **1.1 Input Reception:** Receives the website URL via a form trigger.
- **1.2 Domain & Technical Details Analysis:** Checks domain age, TLD trustworthiness, and SSL certificate presence.
- **1.3 Search Engine Signals Analysis:** Investigates scam reports, forum warnings, ratings, and public concern.
- **1.4 Product & Pricing Patterns Analysis:** Looks for suspicious pricing, exaggerated discounts, and fake endorsements.
- **1.5 Content Analysis:** Examines website reviews for duplicates, grammar, company info, and address authenticity.
- **1.6 Data Aggregation and Merging:** Combines all agents’ outputs into a single consolidated dataset.
- **1.7 Final Evaluation:** Uses a GPT-4o agent to synthesize all information and generate a risk score and summary.
- **1.8 Documentation and Guidance:** Includes sticky notes with setup instructions, disclaimers, and usage notes.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Captures user input URL via an interactive form to initiate the scam evaluation process.
- **Nodes Involved:**  
  - `On form submission`
- **Node Details:**  
  - Type: Form Trigger  
  - Role: Webhook-based trigger node for receiving website URLs from users.  
  - Configuration: Single form field named "Website URL" with a descriptive example URL ("https://www.scam.com/").  
  - Input: User-submitted URL string.  
  - Output: JSON with key `"Website URL"` passed downstream.  
  - Version: 2.2  
  - Edge Cases: No URL input, malformed URL, or injection attempts may cause failures or incorrect downstream processing. Proper validation outside this workflow recommended.

#### 1.2 Domain & Technical Details Analysis

- **Overview:** Evaluates domain age, suspicious TLDs, and SSL certificate status using a GPT-4o agent with SerpAPI support.
- **Nodes Involved:**  
  - `Domain & Technical Details`  
  - `SerpAPI` (credential connected)  
  - `OpenAI Chat Model2` (GPT-4o language model)
- **Node Details:**  
  - `Domain & Technical Details`  
    - Type: LangChain Agent  
    - Role: Executes a prompt-driven AI task focused on domain and technical checks.  
    - Configuration: Uses system message defining role, constraints, and tasks (e.g., domain age <6 months as red flag, suspicious TLDs, SSL presence).  
    - Input: `Website URL` from form node.  
    - Output: Textual analysis passed downstream.  
    - Integration: Uses `SerpAPI` node as AI tool for real-time search data to inform answers.  
    - Edge Cases: API quota exhaustion, slow SerpAPI responses, or malformed URLs can cause errors or incomplete data.
  - `SerpAPI`  
    - Type: LangChain Tool SerpApi  
    - Role: Provides Google Shopping and search data to the agent.  
    - Credentials: Requires valid SerpAPI key.  
    - Edge Cases: API key invalid/expired, rate limits.
  - `OpenAI Chat Model2`  
    - Type: GPT-4o Language Model Node  
    - Role: Processes prompts for the agent's AI language understanding.  
    - Credentials: OpenAI API key required.  
    - Edge Cases: API rate limits, model unavailability, network issues.

#### 1.3 Search Engine Signals Analysis

- **Overview:** Investigates community feedback and reputation by checking scam reports and ratings on platforms like Reddit, Trustpilot, and others.
- **Nodes Involved:**  
  - `Search Engine Signals`  
  - `SerpAPI2`  
  - `OpenAI Chat Model3`
- **Node Details:**  
  - `Search Engine Signals`  
    - Type: LangChain Agent  
    - Role: Collects and synthesizes signals of scam reports or complaints about the URL.  
    - Configuration: Prompt instructs checking forums and ratings, searching for multiple user queries about scam status.  
    - Input: URL from form.  
    - Output: Text analysis on reputation and complaints.  
    - Tools: Uses `SerpAPI2` for search data.  
    - Edge Cases: Forum or site data may be sparse or outdated; API failures.
  - `SerpAPI2` and `OpenAI Chat Model3` mirror the setup and potential issues of their counterparts in block 1.2.

#### 1.4 Product & Pricing Patterns Analysis

- **Overview:** Detects suspicious pricing tactics such as exaggerated discounts, fake endorsements, or unrealistic brand offers.
- **Nodes Involved:**  
  - `Product & Pricing Patterns`  
  - `SerpAPI3`  
  - `OpenAI Chat Model4`
- **Node Details:**  
  - `Product & Pricing Patterns`  
    - Type: LangChain Agent  
    - Role: Examines product pricing data via search results to identify scam-like marketing patterns.  
    - Configuration: Prompt focuses on discounts, brand pricing anomalies.  
    - Input: URL.  
    - Output: Analytical text on product/pricing risks.  
    - Uses `SerpAPI3` for search data.  
    - Edge Cases: Limited product data, API rate limits.
  - `SerpAPI3` and `OpenAI Chat Model4` configured similarly to earlier SerpAPI and OpenAI nodes.

#### 1.5 Content Analysis

- **Overview:** Analyzes website content for review authenticity, grammar quality, and company info legitimacy.
- **Nodes Involved:**  
  - `Content Analysis`  
  - `SerpAPI4`  
  - `OpenAI Chat Model5`
- **Node Details:**  
  - `Content Analysis`  
    - Type: LangChain Agent  
    - Role: Reviews website content and public feedback for signs of scam-like characteristics.  
    - Configuration: Prompt directs evaluation of review duplication, poor grammar, and missing/fake company data.  
    - Input: URL.  
    - Output: Text detailing content-based risk factors.  
    - Uses `SerpAPI4` for data retrieval.  
    - Edge Cases: Sparse content or reviews, API issues.  
  - `SerpAPI4` and `OpenAI Chat Model5` follow the same pattern as other SerpAPI/OpenAI nodes.

#### 1.6 Data Aggregation and Merging

- **Overview:** Consolidates all four individual agents’ output into a single structured dataset for final evaluation.
- **Nodes Involved:**  
  - `Merge1`  
  - `Aggregate1`
- **Node Details:**  
  - `Merge1`  
    - Type: Merge Node  
    - Role: Combines outputs from all four analysis agents into one data stream.  
    - Configuration: Set to accept 5 inputs (4 from agents + possibly one empty or control input).  
    - Inputs: From `Domain & Technical Details`, `Search Engine Signals`, `Product & Pricing Patterns`, and `Content Analysis`.  
    - Outputs: Merged data passed to aggregator.  
    - Edge Cases: Missing or failed inputs might break merge or cause incomplete data.
  - `Aggregate1`  
    - Type: Aggregate Node  
    - Role: Aggregates merged data fields, renaming output fields to `Output 1` through `Output 5`.  
    - Inputs: From `Merge1`.  
    - Outputs: Aggregated data is sent to the final evaluator.  
    - Edge Cases: Aggregation errors if data is malformed or missing.

#### 1.7 Final Evaluation

- **Overview:** Produces a scam risk score and organized summary based on aggregated analysis results.
- **Nodes Involved:**  
  - `Evaluator`  
  - `OpenAI Chat Model`
- **Node Details:**  
  - `Evaluator`  
    - Type: LangChain Agent  
    - Role: Synthesizes all prior analysis outputs into a final risk rating and detailed explanation with disclaimer.  
    - Configuration: Uses system prompt with instructions to rate scam likelihood (1-10 scale) and include a disclaimer.  
    - Input: Aggregated output from `Aggregate1`.  
    - Output: Final textual assessment with risk score and organized sections.  
    - Uses `OpenAI Chat Model` (GPT-4o-mini) as language model.  
    - Edge Cases: API limits, incomplete data, or parsing errors.
  - `OpenAI Chat Model`  
    - Type: GPT-4o-mini Language Model Node  
    - Role: Processes final evaluation prompt.  
    - Credentials: OpenAI API key required.  
    - Edge Cases: Network or API issues.

#### 1.8 Documentation and Guidance

- **Overview:** Provides setup instructions, disclaimers, and usage guidance for users.
- **Nodes Involved:**  
  - `Sticky Note` (Setup & Usage Instructions)  
  - `Sticky Note1` (Legal Disclaimer and Warranties)
- **Node Details:**  
  - `Sticky Note`  
    - Type: Sticky Note  
    - Role: Contains detailed instructions on how to create OpenAI and SerpAPI API keys, link credentials, cost warnings, and usage instructions.  
  - `Sticky Note1`  
    - Type: Sticky Note  
    - Role: Contains legal warranty disclaimers and limitation of liability statements.  

---

### 3. Summary Table

| Node Name                | Node Type                          | Functional Role                          | Input Node(s)                               | Output Node(s)                | Sticky Note                                                                                             |
|--------------------------|----------------------------------|----------------------------------------|--------------------------------------------|------------------------------|-------------------------------------------------------------------------------------------------------|
| On form submission       | Form Trigger                     | Input Reception for Website URL        | -                                          | Domain & Technical Details, Search Engine Signals, Product & Pricing Patterns, Content Analysis |                                                                                                       |
| Domain & Technical Details | LangChain Agent                  | Domain age, TLD, SSL validation        | On form submission, OpenAI Chat Model2, SerpAPI | Merge1                       |                                                                                                       |
| SerpAPI                  | LangChain SerpApi Tool           | Provides search data for domain checks | Domain & Technical Details                  | Domain & Technical Details    |                                                                                                       |
| OpenAI Chat Model2       | OpenAI GPT-4o Language Model     | AI language processing for domain check| Domain & Technical Details                  | Domain & Technical Details    |                                                                                                       |
| Search Engine Signals     | LangChain Agent                  | Reputation & scam report analysis      | On form submission, OpenAI Chat Model3, SerpAPI2 | Merge1                       |                                                                                                       |
| SerpAPI2                 | LangChain SerpApi Tool           | Provides search data for reputation    | Search Engine Signals                       | Search Engine Signals         |                                                                                                       |
| OpenAI Chat Model3       | OpenAI GPT-4o Language Model     | AI language processing for reputation  | Search Engine Signals                       | Search Engine Signals         |                                                                                                       |
| Product & Pricing Patterns| LangChain Agent                  | Product pricing and discount analysis  | On form submission, OpenAI Chat Model4, SerpAPI3 | Merge1                       |                                                                                                       |
| SerpAPI3                 | LangChain SerpApi Tool           | Provides search data for product analysis | Product & Pricing Patterns                  | Product & Pricing Patterns    |                                                                                                       |
| OpenAI Chat Model4       | OpenAI GPT-4o Language Model     | AI language processing for product data| Product & Pricing Patterns                  | Product & Pricing Patterns    |                                                                                                       |
| Content Analysis         | LangChain Agent                  | Website content and reviews analysis   | On form submission, OpenAI Chat Model5, SerpAPI4 | Merge1                       |                                                                                                       |
| SerpAPI4                 | LangChain SerpApi Tool           | Provides search data for content analysis | Content Analysis                            | Content Analysis             |                                                                                                       |
| OpenAI Chat Model5       | OpenAI GPT-4o Language Model     | AI language processing for content     | Content Analysis                            | Content Analysis             |                                                                                                       |
| Merge1                   | Merge                           | Combines outputs from 4 analysis agents| Domain & Technical Details, Search Engine Signals, Product & Pricing Patterns, Content Analysis | Aggregate1                   |                                                                                                       |
| Aggregate1               | Aggregate                       | Aggregates merged outputs               | Merge1                                     | Evaluator                    |                                                                                                       |
| Evaluator                | LangChain Agent                  | Final scam risk scoring and summary    | Aggregate1, OpenAI Chat Model               | -                            |                                                                                                       |
| OpenAI Chat Model        | OpenAI GPT-4o-mini Language Model| Final evaluation language model        | Evaluator                                  | Evaluator                    |                                                                                                       |
| Sticky Note              | Sticky Note                     | Setup instructions and usage notes     | -                                          | -                            | Detailed API key creation, credential linking, cost warnings, usage instructions, disclaimer          |
| Sticky Note1             | Sticky Note                     | Legal disclaimers and limitation of liability | -                                          | -                            | Contains warranty disclaimers and limits on liability                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node:**
   - Name: `On form submission`
   - Type: Form Trigger  
   - Configure with a form titled "Website URL" and a single form field labeled "Website URL" (default placeholder example: https://www.scam.com/).
   - This node triggers the workflow on URL submission.

2. **Create LangChain Agent Nodes for Four Analysis Categories:**
   For each agent, add:
   - Node Type: LangChain Agent
   - Credential: Link to OpenAI API key credential (create if missing).
   - Tool: Link a corresponding SerpAPI node (described below).
   - Input Expression: `={{ $json["Website URL"] }}` to pass the submitted URL.
   - Prompt: Use the respective system messages (see below) defining the role, constraints, and tasks.
   
   Specific agents:

   - **`Domain & Technical Details`**
     - System Message: Instruct to check domain age (<6 months = red flag), suspicious TLDs (.xyz, .top, etc.), SSL certificate presence.
     - Link to `SerpAPI` node.

   - **`Search Engine Signals`**
     - System Message: Check scam reports on Reddit, Trustpilot, ScamAdviser, SiteJabber; look at website ratings and complaints.
     - Link to `SerpAPI2` node.

   - **`Product & Pricing Patterns`**
     - System Message: Look for exaggerated discounts, fake endorsements, unrealistic pricing of high-end brands.
     - Link to `SerpAPI3` node.

   - **`Content Analysis`**
     - System Message: Analyze reviews for duplicates, grammar issues, lack of company info or fake address.
     - Link to `SerpAPI4` node.

3. **Create Four SerpAPI Nodes:**
   - Types: LangChain SerpAPI Tool nodes.
   - Credentials: Connect to SerpAPI API keys (create new credentials with your SerpAPI keys).
   - Name them `SerpAPI`, `SerpAPI2`, `SerpAPI3`, `SerpAPI4` respectively.
   - These nodes provide search and shopping data for their linked agents.

4. **Create OpenAI Chat Model Nodes for Language Processing:**
   - Types: LangChain GPT-4o nodes.
   - Credentials: Connect to OpenAI API keys.
   - Name nodes `OpenAI Chat Model2`, `OpenAI Chat Model3`, `OpenAI Chat Model4`, and `OpenAI Chat Model5`.
   - Link each to their respective agents (`Domain & Technical Details`, `Search Engine Signals`, `Product & Pricing Patterns`, `Content Analysis`).

5. **Connect Each Agent to Merge Node:**
   - Create a Merge node named `Merge1`.
   - Set `numberInputs` parameter to 5.
   - Connect four agent nodes (`Domain & Technical Details`, `Search Engine Signals`, `Product & Pricing Patterns`, `Content Analysis`) outputs to the Merge node inputs.

6. **Create Aggregate Node:**
   - Name: `Aggregate1`
   - Configure field aggregation to rename aggregated fields to `Output 1` through `Output 5` mapping to the `output` field from merged data.
   - Connect `Merge1` output to `Aggregate1` input.

7. **Set Up Final Evaluation Agent:**
   - Create a LangChain Agent node named `Evaluator`.
   - Connect `Aggregate1` output to `Evaluator` input.
   - Use system prompt instructing to synthesize all inputs into a scam risk score (1-10) and summary with disclaimer.
   - Connect an OpenAI GPT-4o-mini Chat Model node named `OpenAI Chat Model` to the `Evaluator` for language processing.
   - Provide OpenAI API credentials.

8. **Add Sticky Notes for Documentation:**
   - Add two sticky note nodes:
     - One with detailed API key creation, linking instructions, usage, and disclaimers (`Sticky Note`).
     - One with legal disclaimers and limitations on liability (`Sticky Note1`).

9. **Connect Nodes as per the following flow:**
   - `On form submission` → Four agents in parallel.
   - Each agent → Corresponding SerpAPI and OpenAI Chat Model nodes.
   - Agents → `Merge1` → `Aggregate1` → `Evaluator` → `OpenAI Chat Model`.

10. **Credential Setup:**
    - Create and link OpenAI API credentials for all OpenAI Chat Model nodes.
    - Create and link SerpAPI API credentials for all SerpAPI nodes.
    - Ensure API keys are valid and have sufficient quota.

11. **Testing:**
    - Use the form trigger to submit a website URL.
    - Monitor execution logs on the `Evaluator` node to view the scam risk score and detailed report.
    - Review sticky notes for instructions and disclaimers.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Create an Open AI key: Create an Open AI account at https://platform.openai.com/api-keys, generate a secret key, and paste it into n8n credentials for OpenAI nodes. Use the same key across all OpenAI chat model nodes. The workflow uses GPT-4o and GPT-4o-mini, which cost approximately $0.01 per run. Add funds to your OpenAI billing account (recommended $5 minimum). Create SerpAPI account at https://serpapi.com/users/sign_up, copy API key, and connect it to all SerpAPI nodes. The workflow uses 5-15 of the 100 free monthly SerpAPI searches; for more usage, create additional accounts with new keys. | Setup instructions (Sticky Note)                    |
| How to use: Submit a URL via the form or test workflow button. Workflow runs automatically. View output in `Evaluator` node logs. Results include a scam risk score (1-10) with explanations. Save results externally for reference. Prior searches accessible via executions list and logs.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Usage instructions (Sticky Note)                     |
| Disclaimer: This tool provides AI-generated insights based on public data and does not guarantee accuracy or legitimacy of any website. Users should perform independent due diligence. The creators and affiliated services disclaim liability for any damages or losses resulting from use.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Legal disclaimer (Sticky Note1)                      |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.

---