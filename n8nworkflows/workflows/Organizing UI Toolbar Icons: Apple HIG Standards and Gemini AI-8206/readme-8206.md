Organizing UI Toolbar Icons: Apple HIG Standards and Gemini AI

https://n8nworkflows.xyz/workflows/organizing-ui-toolbar-icons--apple-hig-standards-and-gemini-ai-8206


# Organizing UI Toolbar Icons: Apple HIG Standards and Gemini AI

---

### 1. Workflow Overview

This n8n workflow, titled **"Organizing UI Toolbar Icons: Apple HIG Standards and Gemini AI"**, automates the categorization, grouping, and ordering of UI toolbar icons based on Apple Human Interface Guidelines (HIG). It is designed to assist UI/UX designers or developers by analyzing uploaded screen images and associated icon sets, then proposing an optimized toolbar icon layout according to established standards and typical usage patterns.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Receives user input via a form with uploaded screen image and icons.

- **1.2 AI Categorization:** Uses Google Gemini AI to analyze the screen context and icons, grouping icons according to Apple HIG roles and assigning placements.

- **1.3 Data Cleaning:** Processes the raw AI JSON output to extract structured data for further processing.

- **1.4 AI Reordering:** Reorders grouped icons based on typical usage frequency per toolbar placement.

- **1.5 Final Design Proposal:** Generates a full structured design recommendation for the UI toolbar icon arrangement following Apple HIG.

These blocks are interconnected in a linear flow from input to final proposal, with intermediate data cleaning steps ensuring accurate processing.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception

- **Overview:**  
  Captures user inputs through a web form including the target screen image and a set of icons to be organized.

- **Nodes Involved:**  
  - On form submission  
  - Sticky Note (Get the screen)

- **Node Details:**

  - **On form submission**  
    - *Type & Role:* Form Trigger node; initiates workflow upon form data submission.  
    - *Configuration:*  
      - Form titled "Icon submission" with two required file fields: "The screen" (single file) and "Icons" (multiple files allowed).  
      - Form description instructs user to upload the screen and icon set.  
    - *Inputs/Outputs:*  
      - Input: Webhook trigger from form submission.  
      - Output: JSON containing uploaded files metadata (filename, etc.).  
    - *Edge Cases:*  
      - Missing or invalid files (e.g., unsupported formats).  
      - Network or webhook errors.  
    - *Sticky Notes:*  
      - Covers "Get the screen" indicating this is the entry point for screen data.

#### Block 1.2: AI Categorization

- **Overview:**  
  Analyzes screen and icons to categorize icons by Apple HIG roles and assign placement groups using an AI agent powered by Google Gemini.

- **Nodes Involved:**  
  - AI Agent: Icon Categorizer  
  - Gemini (Google Gemini API node)  
  - Sticky Note1 (Analysing the screen and icons to categories them)  
  - Code: Clean Outcome

- **Node Details:**

  - **AI Agent: Icon Categorizer**  
    - *Type & Role:* Langchain Agent node using Google Gemini API for chat-based AI.  
    - *Configuration:*  
      - Prompt includes screen filename and JSON stringified icons.  
      - Task: Categorize icons into specific Apple HIG roles (e.g., primaryAction, navigation) and assign placements (Left, Right, Center, Bottom, System-decided).  
      - Output strictly in JSON format describing Screen, Groups, and Placement.  
    - *Credentials:* Google Gemini (PaLM) API.  
    - *Inputs/Outputs:*  
      - Input: Form submission output with files info.  
      - Output: JSON string wrapped in triple backticks (```json ... ```).  
    - *Edge Cases:*  
      - AI misunderstanding input or returning malformed JSON.  
      - API quota limits or authentication issues.  
    - *Sticky Notes:*  
      - Notes on analyzing screen and icon categorization.

  - **Gemini**  
    - *Type & Role:* AI language model node for Google Gemini calls used internally by the agent.  
    - *Configuration:* Empty, uses default call with credentials.  
    - *Inputs/Outputs:* Connected as AI LLM provider for the agent.

  - **Code: Clean Outcome**  
    - *Type & Role:* Code node in JavaScript cleaning raw AI output.  
    - *Configuration:*  
      - Extracts JSON string by removing ```json and ``` wrappers from AI output.  
      - Parses JSON to n8n output format separating Screen, Groups, and Placement.  
    - *Inputs/Outputs:*  
      - Input: Raw AI response JSON string.  
      - Output: Parsed JSON object.  
    - *Edge Cases:*  
      - Malformed JSON causing parse errors.

#### Block 1.3: AI Reordering

- **Overview:**  
  Reorders the grouped icons within each placement category based on typical usage frequency for UI toolbars, enhancing practical usability.

- **Nodes Involved:**  
  - Basic LLM Chain: Reordering  
  - Gemini: Reordering  
  - Code: Clean Outcomes  
  - Sticky Note2 (Reordering based on the usage)

- **Node Details:**

  - **Basic LLM Chain: Reordering**  
    - *Type & Role:* Langchain Chain LLM node using Google Gemini.  
    - *Configuration:*  
      - Receives JSON object of icon filenames grouped by placement.  
      - Task: Reorder icons from most to least used per placement category, returning JSON with reordered arrays.  
      - No explanations or extra text in output to ensure clean JSON.  
    - *Inputs/Outputs:*  
      - Input: Parsed JSON from previous cleaning node.  
      - Output: JSON with icons reordered per placement.  
    - *Edge Cases:*  
      - Empty arrays for placements with no icons.  
      - AI model returning unexpected output format.

  - **Gemini: Reordering**  
    - *Type & Role:* AI language model node, backing the Basic LLM Chain.  
    - *Configuration:* Default with Google Gemini credentials.  
    - *Inputs/Outputs:* Connects to Basic LLM Chain.

  - **Code: Clean Outcomes**  
    - *Type & Role:* Cleans AI output by removing code block markers and parsing JSON.  
    - *Configuration:* Similar to previous cleaning node.  
    - *Inputs/Outputs:*  
      - Input: Raw AI output from reordering.  
      - Output: Parsed clean JSON for final use.  
    - *Edge Cases:* Malformed JSON or missing fields.

#### Block 1.4: Final Design Proposal

- **Overview:**  
  Uses AI to generate a comprehensive, structured design proposal for arranging icons in the UI toolbar, following Apple Human Interface Guidelines.

- **Nodes Involved:**  
  - UI Guidance  
  - Google Gemini Chat Model  
  - Sticky Note3 (Full structured design proposal)

- **Node Details:**

  - **UI Guidance**  
    - *Type & Role:* Chain LLM node using Google Gemini to create final text-based design instructions.  
    - *Configuration:*  
      - Input prompt includes JSON with icon filenames, groups, and placements from the categorizer.  
      - Task: Read Apple HIG toolbars guidelines and produce a full, structured design proposal for icon arrangement.  
    - *Credentials:* Google Gemini API.  
    - *Inputs/Outputs:*  
      - Input: JSON from cleaned AI categorization output.  
      - Output: Textual structured design recommendation.  
    - *Edge Cases:* AI may produce verbose, non-structured text if prompt is not followed strictly.

  - **Google Gemini Chat Model**  
    - *Type & Role:* AI language model node supporting UI Guidance.  
    - *Configuration:* Default Google Gemini credentials.  
    - *Inputs/Outputs:* Linked as AI provider for UI Guidance.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                              | Input Node(s)               | Output Node(s)                     | Sticky Note                                                                                             |
|-------------------------|----------------------------------|----------------------------------------------|-----------------------------|----------------------------------|-------------------------------------------------------------------------------------------------------|
| On form submission      | Form Trigger                     | Receives uploaded screen and icons            | (Webhook trigger)            | AI Agent: Icon Categorizer       | ## Get the screen                                                                                      |
| Sticky Note             | Sticky Note                      | Annotation: Entry point                        |                             |                                  | ## Get the screen                                                                                      |
| AI Agent: Icon Categorizer | Langchain Agent (Google Gemini) | Categorizes icons by Apple HIG roles and assigns placement | On form submission           | Code: Clean Outcome               | ## Analysing the screen and icons to categories them                                                  |
| Gemini                  | Langchain LM Chat (Google Gemini) | AI model for icon categorization              | AI Agent: Icon Categorizer   | AI Agent: Icon Categorizer (internal) | ## Analysing the screen and icons to categories them                                                  |
| Code: Clean Outcome     | Code                             | Cleans AI raw JSON output                      | AI Agent: Icon Categorizer   | Basic LLM Chain: Reordering      |                                                                                                       |
| Basic LLM Chain: Reordering | Chain LLM (Google Gemini)       | Reorders icons per placement by usage frequency | Code: Clean Outcome          | Code: Clean Outcomes             | ## Reordering based on the usage                                                                       |
| Gemini: Reordering      | Langchain LM Chat (Google Gemini) | AI model for reordering                        | Basic LLM Chain: Reordering  | Basic LLM Chain: Reordering (internal) | ## Reordering based on the usage                                                                       |
| Code: Clean Outcomes    | Code                             | Cleans AI reordering output                    | Basic LLM Chain: Reordering  | UI Guidance                     |                                                                                                       |
| UI Guidance             | Chain LLM (Google Gemini)         | Generates full structured design proposal     | Code: Clean Outcomes         | (End node)                      | ## Full structured design proposa                                                                     |
| Google Gemini Chat Model | Langchain LM Chat (Google Gemini) | AI model for final design guidance             | UI Guidance                 | UI Guidance (internal)           | ## Full structured design proposa                                                                     |
| Sticky Note1            | Sticky Note                      | Annotation for AI categorization block        |                             |                                  | ## Analysing the screen and icons to categories them                                                  |
| Sticky Note2            | Sticky Note                      | Annotation for reordering block                |                             |                                  | ## Reordering based on the usage                                                                       |
| Sticky Note3            | Sticky Note                      | Annotation for final design proposal           |                             |                                  | ## Full structured design proposa                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node: "On form submission"**  
   - Type: Form Trigger  
   - Configure form with title: "Icon submission"  
   - Add two fields:  
     - "The screen" (File input, single file, required)  
     - "Icons" (File input, multiple files allowed, required)  
   - Add form description instructing user to upload screen image and icons set.

2. **Add Sticky Note Node: "Sticky Note"**  
   - Content: "## Get the screen"  
   - Position near the form trigger node for annotation.

3. **Add Langchain Agent Node: "AI Agent: Icon Categorizer"**  
   - Type: Langchain Agent (Google Gemini)  
   - Credentials: Configure with Google Gemini (PaLM) API credentials  
   - Prompt:  
     - Include screen filename and JSON stringified icons from form input.  
     - Task: Categorize each icon according to Apple HIG roles (list roles explicitly).  
     - Assign placements (Left, Right, Center, Bottom, System-decided) based on roles.  
     - Output strictly in JSON format with keys: Screen, Groups, Placement.  
   - Connect output of "On form submission" to this node’s input.

4. **Add Sticky Note Node: "Sticky Note1"**  
   - Content: "## Analysing the screen and icons to categories them"  
   - Position near the AI Agent node.

5. **Add Langchain LM Chat Node: "Gemini"**  
   - Type: Langchain LM Chat (Google Gemini)  
   - Credentials: Same Google Gemini API credentials  
   - No additional parameters (default)  
   - Connect as AI language provider to the agent node.

6. **Add Code Node: "Code: Clean Outcome"**  
   - Type: Code  
   - Add JavaScript code to:  
     - Extract AI output string from previous node’s `output` field.  
     - Remove triple backticks and `json` markers.  
     - Parse JSON string and return separated JSON with `screen`, `groups`, and `placement`.  
   - Connect output of AI Agent to this node.

7. **Add Chain LLM Node: "Basic LLM Chain: Reordering"**  
   - Type: Chain LLM (Google Gemini)  
   - Credentials: Google Gemini API  
   - Prompt:  
     - Input JSON with icon filenames grouped by placement (Left, Right, Center, Bottom, System-decided).  
     - Task: Reorder icons within each placement by typical usage frequency (most used to least).  
     - Output JSON with same keys and ordered arrays, no explanations.  
   - Connect output of "Code: Clean Outcome" to this node.

8. **Add Langchain LM Chat Node: "Gemini: Reordering"**  
   - Type: Langchain LM Chat (Google Gemini)  
   - Credentials: Google Gemini API  
   - Connect as AI provider for "Basic LLM Chain: Reordering".

9. **Add Code Node: "Code: Clean Outcomes"**  
   - Type: Code  
   - JavaScript to:  
     - Clean AI output by removing triple backticks.  
     - Parse JSON and return to n8n format.  
   - Connect output of "Basic LLM Chain: Reordering" to this node.

10. **Add Sticky Note Node: "Sticky Note2"**  
    - Content: "## Reordering based on the usage"  
    - Position near the reordering nodes.

11. **Add Chain LLM Node: "UI Guidance"**  
    - Type: Chain LLM (Google Gemini)  
    - Credentials: Google Gemini API  
    - Prompt:  
      - Input JSON with icon filenames, groups, and placements from categorizer output.  
      - Task: Read Apple HIG toolbar guidelines and generate a full structured design proposal for icon arrangement.  
    - Connect output of "Code: Clean Outcomes" to this node.

12. **Add Langchain LM Chat Node: "Google Gemini Chat Model"**  
    - Type: Langchain LM Chat (Google Gemini)  
    - Credentials: Google Gemini API  
    - Connect as AI provider for "UI Guidance".

13. **Add Sticky Note Node: "Sticky Note3"**  
    - Content: "## Full structured design proposa"  
    - Position near "UI Guidance".

14. **Connect all nodes in order:**  
    - On form submission → AI Agent: Icon Categorizer → Code: Clean Outcome → Basic LLM Chain: Reordering → Code: Clean Outcomes → UI Guidance.

15. **Ensure all Google Gemini API credentials are configured and tested.**

---

### 5. General Notes & Resources

| Note Content                                                                                                            | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow strictly follows Apple Human Interface Guidelines for toolbar icons.                                       | https://developer.apple.com/design/human-interface-guidelines/toolbars                          |
| The AI agent uses Google Gemini (PaLM API) for natural language understanding and JSON generation.                      | Google Gemini API credentials required                                                        |
| Sticky notes provide contextual explanations and grouping for the workflow steps.                                        | Internal annotations in workflow UI                                                           |
| The workflow output is designed to be consumed by UI/UX teams for improved toolbar icon organization and design layout. |                                                                                               |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.

---