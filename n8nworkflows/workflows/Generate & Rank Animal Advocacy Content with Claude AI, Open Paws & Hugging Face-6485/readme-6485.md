Generate & Rank Animal Advocacy Content with Claude AI, Open Paws & Hugging Face

https://n8nworkflows.xyz/workflows/generate---rank-animal-advocacy-content-with-claude-ai--open-paws---hugging-face-6485


# Generate & Rank Animal Advocacy Content with Claude AI, Open Paws & Hugging Face

### 1. Workflow Overview

This workflow, titled **"Multi-Version Content Generator with AI Scoring & Advocacy Preference Ranking"**, is designed to generate multiple versions of animal advocacy-focused content, perform AI-driven scoring and ranking based on advocacy preferences, and aggregate the results for optimized outreach use. It targets organizations or individuals creating persuasive, research-based content for animal welfare campaigns, enabling data-driven selection of the most effective messaging variants.

The workflow is logically grouped into the following blocks:

- **1.1 Input Reception:** Receives user inputs defining the content generation parameters.
- **1.2 Content Generation & Research:** Uses a multi-tool research subworkflow and Claude AI (via OpenRouter) to create content variants.
- **1.3 Content Extraction & Splitting:** Parses AI output JSON and splits content into individual variants.
- **1.4 AI Scoring & Evaluation:** Invokes an evaluation subworkflow for scoring each content variant using Hugging Face and Open Paws models.
- **1.5 Aggregation & Final Output:** Aggregates scored content variants and prepares them for downstream use.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block captures all necessary user inputs to customize the content generation process, including content type, tone, style, topic, and poster profile description.

**Nodes Involved:**  
- When Executed by Another Workflow  
- Sticky Note (User Input Configuration)

**Node Details:**

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Entry point that accepts input parameters from an external workflow or trigger.  
  - Configuration: Accepts inputs for `contentType`, `Poster’s profile description`, `Tone (e.g., formal, casual, friendly)`, `Style (e.g., concise, detailed, persuasive)`, and `Topic or Instructions`.  
  - Connections: Outputs data to "Create Content" node.  
  - Edge Cases: Missing or malformed inputs may cause downstream failures or incorrect content generation.

- **Sticky Note**  
  - Type: Sticky Note (Documentation)  
  - Role: Provides user instructions on filling input fields to ensure effective content generation.  
  - Content Notes: Emphasizes accuracy and clarity of inputs, lists required fields and their purpose.

---

#### 1.2 Content Generation & Research

**Overview:**  
This block performs focused research and generates multiple full-length content versions based on the user inputs, leveraging AI language models and a research subworkflow.

**Nodes Involved:**  
- Create Content  
- OpenRouter Chat Model2  
- Extract Content  
- Sticky Note1 (Research Agent Usage)

**Node Details:**

- **Create Content**  
  - Type: Execute Workflow  
  - Role: Calls the "Multi-tool Research Agent" subworkflow that researches the topic via Serper (Google Search) and the Open Paws database, then generates content.  
  - Configuration: Passes user input parameters as `chatInput` along with a session ID for context.  
  - Connections: Output feeds into "Extract Content".  
  - Edge Cases: Subworkflow failures or API timeouts; data inconsistency if research sources are unavailable.  
  - Sub-workflow: Referenced workflow ID `k053fXGjIF7dUIQZ` (Multi-tool Research Agent).

- **OpenRouter Chat Model2**  
  - Type: LangChain OpenRouter Chat Model  
  - Role: Uses the Claude Sonnet 4 model to generate natural language content based on instructions.  
  - Configuration: Model set to `anthropic/claude-sonnet-4`, credentials configured for OpenRouter API.  
  - Connections: Outputs generated text to "Extract Content" node (via ai_languageModel connection).  
  - Edge Cases: API quota limits, authentication failures, or model unavailability.

- **Extract Content**  
  - Type: LangChain Information Extractor  
  - Role: Parses the JSON-formatted AI-generated content, ensuring exactly 10 variations are extracted as per instructions.  
  - Configuration: Uses system prompt to enforce strict JSON output without extraneous formatting. Parses input from the `output` field.  
  - Connections: Outputs parsed JSON to "Split Out Content".  
  - Edge Cases: Parsing errors if AI output is malformed or incomplete.

- **Sticky Note1**  
  - Type: Sticky Note  
  - Role: Documents the usage of the research agent subworkflow, highlighting research sources and generation process.

---

#### 1.3 Content Extraction & Splitting

**Overview:**  
Separates the JSON array of content variants into individual items for subsequent scoring.

**Nodes Involved:**  
- Split Out Content

**Node Details:**

- **Split Out Content**  
  - Type: Split Out  
  - Role: Splits the `output` array into singular content objects for processing each variant independently.  
  - Configuration: Splitting on the `output` field.  
  - Connections: Outputs individual content items to "Score Text".  
  - Edge Cases: Empty or malformed array input may cause node to fail or produce no output.

---

#### 1.4 AI Scoring & Evaluation

**Overview:**  
Each content variant is scored individually by invoking an evaluation subworkflow that applies AI models to assess effectiveness and advocacy preference.

**Nodes Involved:**  
- Score Text  
- Sticky Note2 (Text Scoring and Evaluation)

**Node Details:**

- **Score Text**  
  - Type: Execute Workflow  
  - Role: Invokes the "Evaluate Animal Advocacy Text" subworkflow once per content item to score text quality and advocacy impact.  
  - Configuration: Runs in "each" mode, passing the `text` property of each content variant.  
  - Connections: Outputs scored results to "Aggregate".  
  - Edge Cases: Subworkflow or API failures, slow execution for large batch sizes.  
  - Sub-workflow: Referenced workflow ID `RyxYPLmF5sWDhC2Z` (Evaluate Animal Advocacy Text).

- **Sticky Note2**  
  - Type: Sticky Note  
  - Role: Describes the scoring process including AI models, metrics such as impact, persuasiveness, emotional resonance, and clarity used for evaluation.

---

#### 1.5 Aggregation & Final Output

**Overview:**  
Aggregates all scored content variants into a single dataset that combines generated content with performance and prediction scores for downstream utilization.

**Nodes Involved:**  
- Aggregate  
- Sticky Note3 (Final Aggregation & Output)

**Node Details:**

- **Aggregate**  
  - Type: Aggregate  
  - Role: Combines all scored content items into a single output array for final delivery.  
  - Configuration: Uses `aggregateAllItemData` option to collect all incoming data.  
  - Connections: Output serves as the workflow’s final output or input for further processing workflows.  
  - Edge Cases: Large data size may affect performance or memory; partial inputs could cause incomplete aggregation.

- **Sticky Note3**  
  - Type: Sticky Note  
  - Role: Explains the purpose of aggregation and how the combined data can be used for automated posting, AI decision-making, and campaign management.

---

### 3. Summary Table

| Node Name                 | Node Type                             | Functional Role                                  | Input Node(s)                | Output Node(s)            | Sticky Note                                                                                   |
|---------------------------|-------------------------------------|-------------------------------------------------|-----------------------------|---------------------------|----------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger            | Entry point for receiving user content parameters | —                           | Create Content             |                                                                                              |
| Sticky Note               | Sticky Note                         | User input configuration instructions            | —                           | —                         | # 📝 User Input Configuration - Explains fields to customize content generation               |
| Create Content            | Execute Workflow                    | Calls research subworkflow and generates content | When Executed by Another Workflow | Extract Content            | # 🔍 Research Agent Usage - Describes research tools and AI content generation                |
| OpenRouter Chat Model2    | LangChain OpenRouter Chat Model    | AI content generation using Claude Sonnet 4      | — (used inside subworkflow) | Extract Content (ai_languageModel) |                                                                                              |
| Extract Content           | LangChain Information Extractor    | Parses JSON content output from AI model          | Create Content, OpenRouter Chat Model2 | Split Out Content          |                                                                                              |
| Split Out Content         | Split Out                          | Splits JSON array into individual content items  | Extract Content             | Score Text                 |                                                                                              |
| Score Text                | Execute Workflow                   | Invokes scoring subworkflow per content variant  | Split Out Content           | Aggregate                  | # 📊 Text Scoring and Evaluation - Details on scoring methods and metrics used                |
| Aggregate                 | Aggregate                         | Combines scored content variants into final array | Score Text                  | —                         | # 📦 Final Aggregation & Output - Explains aggregation and downstream integration             |
| Sticky Note1              | Sticky Note                       | Research agent usage documentation                 | —                           | —                         | # 🔍 Research Agent Usage - Details on research tools and subworkflow                         |
| Sticky Note2              | Sticky Note                       | Text scoring and evaluation documentation          | —                           | —                         | # 📊 Text Scoring and Evaluation - Explains AI scoring and advocacy ranking                   |
| Sticky Note3              | Sticky Note                       | Final aggregation and output documentation          | —                           | —                         | # 📦 Final Aggregation & Output - Describes final data usage and integration                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add an **Execute Workflow Trigger** node named `"When Executed by Another Workflow"`.  
   - Configure inputs to accept:  
     - `contentType` (string)  
     - `Poster’s profile description` (string)  
     - `Tone (e.g., formal, casual, friendly)` (string)  
     - `Style (e.g., concise, detailed, persuasive)` (string)  
     - `Topic or Instructions` (string)

2. **Add Sticky Note:**  
   - Add a **Sticky Note** near the trigger with instructions on user input fields and their purpose.

3. **Add Content Generation Node:**  
   - Add an **Execute Workflow** node named `"Create Content"`.  
   - Set it to call the research subworkflow with ID `k053fXGjIF7dUIQZ` (Multi-tool Research Agent).  
   - Map inputs by defining the `chatInput` string with embedded user inputs, following the prompt instructions (include content type, tone, style, topic, and poster profile description).  
   - Generate a unique `sessionId` using an expression like `={{ (Math.random().toString(36).substring(2) + Date.now().toString(36)) }}`.  
   - Connect `"When Executed by Another Workflow"` output to this node.

4. **Add AI Chat Model Node (Inside Subworkflow):**  
   - In the referenced subworkflow, configure the **LangChain OpenRouter Chat Model** node to use model `anthropic/claude-sonnet-4` with OpenRouter API credentials.  
   - This node generates the content text.

5. **Add Content Extraction Node:**  
   - Add a **LangChain Information Extractor** node named `"Extract Content"`.  
   - Set input to the AI-generated `output` field.  
   - Use a system prompt template to enforce strict JSON output with exactly 10 content variations.  
   - Connect `"Create Content"` output to this node.

6. **Add Content Split Node:**  
   - Add **Split Out** node named `"Split Out Content"`.  
   - Configure to split on the `output` field, separating content variants.  
   - Connect `"Extract Content"` output to this node.

7. **Add Scoring Workflow Node:**  
   - Add an **Execute Workflow** node named `"Score Text"`.  
   - Set to run in mode `"each"` to process each content variant separately.  
   - Call subworkflow ID `RyxYPLmF5sWDhC2Z` (Evaluate Animal Advocacy Text).  
   - Pass the `text` field of each content variant as input.  
   - Connect `"Split Out Content"` output to this node.

8. **Add Aggregation Node:**  
   - Add an **Aggregate** node named `"Aggregate"`.  
   - Configure it with option `aggregateAllItemData` to collect all scored results into one output array.  
   - Connect `"Score Text"` output to this node.

9. **Add Sticky Notes:**  
   - Add three **Sticky Notes**:  
     - Near `"Create Content"` node explaining research agent usage and sources.  
     - Near `"Score Text"` node explaining scoring metrics and AI models used.  
     - Near `"Aggregate"` node explaining final aggregation and downstream usage.

10. **Configure Credentials:**  
    - Set up **OpenRouter API** credentials for Claude AI integration.  
    - Ensure OAuth2 or API key credentials are configured for the research subworkflow APIs (Serper, Open Paws).  
    - Ensure Hugging Face or Open Paws credentials are set for the evaluation subworkflow.

11. **Finalize Connections:**  
    - Connect nodes as per flow:  
      When Executed by Another Workflow → Create Content → Extract Content → Split Out Content → Score Text → Aggregate

12. **Testing & Validation:**  
    - Test with sample inputs for content type, tone, style, and topic.  
    - Verify generated content variants are properly extracted and scored.  
    - Confirm aggregation outputs all scored content variants with expected fields.

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                                       |
|----------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Multi-tool Research Agent subworkflow uses Serper (Google search) and Open Paws database for up-to-date animal advocacy research. | https://n8n.io/workflows/5588-multi-tool-research-agent-for-animal-advocacy-with-openrouter-serper-and-open-paws-db/ |
| Evaluate Animal Advocacy Text subworkflow applies Hugging Face models fine-tuned on advocacy datasets to score content effectiveness. | https://n8n.io/workflows/5587-evaluate-animal-advocacy-text-with-hugging-face-open-paws-ai-models/                   |
| Use BB-code formatting in generated content (not Markdown) to meet platform compatibility requirements.                          | Specified in user instructions and AI prompts                                                                       |
| Session ID generation uses a combination of random string and timestamp for uniqueness and session tracking.                     | Expression: `={{ (Math.random().toString(36).substring(2) + Date.now().toString(36)) }}`                               |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This process strictly complies with current content policies and does not include any illegal, offensive, or protected elements. All data processed is legal and public.