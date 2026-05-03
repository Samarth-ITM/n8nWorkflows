Auto-generate sticky notes and rename nodes

https://n8nworkflows.xyz/workflows/auto-generate-sticky-notes-and-rename-nodes-13868


# Auto-generate sticky notes and rename nodes

This document provides a technical analysis of the **Auto-generate sticky notes and rename nodes** workflow, designed to automate the documentation and formatting of n8n templates according to official guidelines.

### 1. Workflow Overview
The primary purpose of this workflow is to transform a "raw" n8n workflow JSON into a production-ready template. It groups nodes into logical clusters, creates descriptive sticky notes, ensures no visual overlapping, and optionally renames nodes to follow action-oriented naming conventions.

#### Logical Blocks:
1.  **Initialization & Parsing:** Receives the target JSON and extracts node positions, types, and connection data.
2.  **AI Logical Grouping:** Uses GPT-4o to analyze node purposes and spatial proximity to define clusters.
3.  **Spatial Calculation:** Computes bounding boxes for clusters and resolves collisions between sticky notes.
4.  **Refinement Loop:** An iterative process that detects overlaps and re-adjusts positions up to a defined limit to ensure a clean layout.
5.  **Node Renaming (Optional):** A second AI pass that analyzes node parameters to suggest specific, verb-driven names.
6.  **Final Export:** Merges the new sticky notes and renamed nodes back into a valid n8n workflow structure.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization
This block captures the input data and establishes the configuration for the process.
*   **Nodes Involved:** `Start`, `Set Workflow Variables`.
*   **Configuration:** 
    *   `workflow`: The JSON of the workflow to be processed.
    *   `MAX_RETRIES`: Number of iterations for the collision fixer (default: 3).
    *   `renameNodes`: Boolean toggle to enable/disable the renaming logic.
*   **Edge Cases:** Invalid JSON input or missing `renameNodes` flag will cause downstream failures in the AI logic.

#### 2.2 Pre-processing & Parsing
Prepares the data for AI analysis by removing existing stickies and calculating actual node dimensions.
*   **Nodes Involved:** `Strip & Prepare`, `Parse Nodes`.
*   **Technical Role:** 
    *   `Strip & Prepare`: Maps AI connections (e.g., tools to agents) and cleans existing documentation.
    *   `Parse Nodes`: Calculates the height/width of nodes based on the number of input/output slots (16px grid) and identifies "entry point" nodes.
*   **Key Expressions:** Uses `ai_languageModel`, `ai_tool`, `ai_memory`, and `ai_outputParser` connection types to identify sub-node relationships.

#### 2.3 AI Grouping & Bounding Boxes
Determines how nodes should be grouped based on logic and spatial distance.
*   **Nodes Involved:** `AI Groups Logically`, `OpenAI Chat Model`, `Structured Output Parser`, `Compute Bounding Boxes`.
*   **AI Logic:** The agent is instructed to group nodes that are within 800px of each other and logically related.
*   **Geometry:** `Compute Bounding Boxes` calculates the smallest rectangle that can encompass all nodes in a group, including a buffer for the sticky note title.

#### 2.4 Collision Resolution & Generation
Handles the visual layout to prevent overlapping sticky notes.
*   **Nodes Involved:** `Collision Resolution`, `Generate Stickies`, `Merge & Export`, `Collision Detector`.
*   **Logic:** If two sticky notes overlap, the lower-priority group (typically later in the execution flow) is shifted along the axis of least overlap.
*   **Output:** Generates new `n8n-nodes-base.stickyNote` objects with formatted Markdown content.

#### 2.5 The Refinement Loop
Controls the quality of the spatial output.
*   **Nodes Involved:** `Loop Controller (Automatic Fixer)`, `If`, `Pick Best Result`.
*   **Function:** If `Collision Detector` finds overlaps, the workflow restarts the positioning logic. It uses static data to remember the "best" version (fewest collisions) across attempts.

#### 2.6 Conditional Node Renaming
Applies descriptive names to nodes based on their specific configuration.
*   **Nodes Involved:** `Should Rename Nodes`, `Parse for Renaming`, `AI Rename`, `OpenAI Chat Model1`, `Structured Output Parser1`, `Apply Renames`, `Format for Export`.
*   **Renaming Logic:** Transforms generic names (e.g., "HTTP Request") into action-oriented ones (e.g., "Fetch YouTube Videos") by inspecting parameters like URLs, methods, or JS code hints.
*   **Safety:** `Apply Renames` updates expression references (e.g., `$('Node Name')`) globally in the JSON to ensure the workflow doesn't break.

#### 2.7 Finalization
Standardizes the final JSON for the user.
*   **Nodes Involved:** `Output Normalization`.
*   **Role:** Ensures the output is a single object containing the finalized `workflow` property.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Start | Manual Trigger | Entry Point | - | Set Workflow Variables | ## Initialize workflow Starts the workflow and sets initial variables. |
| Set Workflow Variables | Set | Configuration | Start | Strip & Prepare | ## Initialize workflow Starts the workflow and sets initial variables. |
| Strip & Prepare | Code | Data Cleaning | Set Workflow Variables | Parse Nodes | ## Prepare and parse nodes Prepares nodes and parses them for further processing. |
| Parse Nodes | Code | Dimension Calc | Strip & Prepare | AI Groups Logically | ## Prepare and parse nodes Prepares nodes and parses them for further processing. |
| AI Groups Logically | Agent | Logical Clustering | Parse Nodes, If | Compute Bounding Boxes | ## AI logical grouping Uses AI to logically group nodes. |
| Compute Bounding Boxes | Code | Spatial Mapping | AI Groups Logically | Collision Resolution | ## AI logical grouping Uses AI to logically group nodes. |
| Collision Resolution | Code | Overlap Prevention | Compute Bounding Boxes | Generate Stickies | ## AI logical grouping Uses AI to logically group nodes. |
| Generate Stickies | Code | Node Creation | Collision Resolution | Merge & Export | ## Collision handling and export Handles collisions and merges results for export. |
| Merge & Export | Code | JSON Assembly | Generate Stickies | Collision Detector | ## Collision handling and export Handles collisions and merges results for export. |
| Collision Detector | Code | Validation | Merge & Export | Loop Controller | ## Collision handling and export Handles collisions and merges results for export. |
| Loop Controller | Code | Iteration Manager | Collision Detector | If | ## Iterative adjustment Controls loop for iterative collision fixes and picks the best result. |
| If | If | Branching Logic | Loop Controller | AI Groups Logically, Pick Best Result | ## Iterative adjustment Controls loop for iterative collision fixes and picks the best result. |
| Pick Best Result | Code | Quality Selection | If | Should Rename Nodes | ## Iterative adjustment Controls loop for iterative collision fixes and picks the best result. |
| Should Rename Nodes | If | User Toggle | Pick Best Result | Parse for Renaming, Output Normalization | ## Conditional node renaming Decides and executes the renaming procedure with AI assistance. |
| Parse for Renaming | Code | Feature Extraction | Should Rename Nodes | AI Rename | ## Conditional node renaming Decides and executes the renaming procedure with AI assistance. |
| AI Rename | Agent | Naming Engine | Parse for Renaming | Apply Renames | ## Conditional node renaming Decides and executes the renaming procedure with AI assistance. |
| Apply Renames | Code | Global Ref Update | AI Rename | Format for Export | ## Conditional node renaming Decides and executes the renaming procedure with AI assistance. |
| Format for Export | Code | Data Structuring | Apply Renames | Output Normalization | ## Conditional node renaming Decides and executes the renaming procedure with AI assistance. |
| Output Normalization | Set | Final Output | Should Rename Nodes, Format for Export | - | ## Final output preparation Formats and normalizes the final workflow for output. |
| OpenAI Chat Model | LangChain Model | AI Engine | - | AI Groups Logically | |
| Structured Output Parser | LangChain Parser | JSON Enforcer | - | AI Groups Logically | |
| OpenAI Chat Model1 | LangChain Model | AI Engine | - | AI Rename | |
| Structured Output Parser1 | LangChain Parser | JSON Enforcer | - | AI Rename | |

---

### 4. Reproducing the Workflow from Scratch

1.  **Environment Setup:** Ensure you have the LangChain nodes installed in your n8n instance. Obtain an OpenAI API Key.
2.  **Trigger:** Add a **Manual Trigger** node.
3.  **Variable Configuration:** Add a **Set** node ("Set Workflow Variables"). Define three variables: `workflow` (Object), `MAX_RETRIES` (Number: 3), and `renameNodes` (Boolean: true).
4.  **Parsing Logic:**
    *   Create a **Code** node ("Strip & Prepare") to remove existing stickies and map AI sub-nodes.
    *   Create a second **Code** node ("Parse Nodes") to calculate X/Y coordinates and node dimensions.
5.  **AI Grouping Chain:**
    *   Add an **AI Agent** ("AI Groups Logically"). Set prompt type to "Define".
    *   Attach an **OpenAI Chat Model** node (configure with `gpt-4o`).
    *   Attach a **Structured Output Parser** node. Define a schema requiring `mainOverview` and a `groups` array (containing `title`, `description`, and `nodeIds`).
6.  **Spatial Logic:**
    *   Add a **Code** node ("Compute Bounding Boxes") to calculate the rects for each AI group.
    *   Add a **Code** node ("Collision Resolution") to shift group positions based on overlap detection.
7.  **Generation & Export:**
    *   Add a **Code** node ("Generate Stickies") to turn group bounds into `stickyNote` types.
    *   Add a **Code** node ("Merge & Export") to stitch nodes and stickies into one JSON.
    *   Add a **Code** node ("Collision Detector") to verify if any overlaps remain.
8.  **The Loop:**
    *   Add a **Code** node ("Loop Controller") using `$getWorkflowStaticData` to track the iteration count and best result.
    *   Add an **If** node to check if `_loopRetry` is true. Connect the "true" branch back to "AI Groups Logically" and the "false" branch to "Pick Best Result".
9.  **Renaming Module:**
    *   Add an **If** node ("Should Rename Nodes") checking the `renameNodes` variable.
    *   On the "true" branch: Create a **Code** node ("Parse for Renaming"), an **AI Agent** ("AI Rename" with GPT-4o), and a **Code** node ("Apply Renames") that uses Regex to update all internal references.
10. **Standardization:** End with a **Set** node ("Output Normalization") to return the final workflow object.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Official Sticky Note Guidelines | [n8n Naming & Sticky Guidelines](https://n8n.notion.site/Sticky-note-guidelines-for-templates-2aa5b6e0c94f8058b0aefddd02655887) |
| Video Tutorial | [YouTube Link (RScKsGfrhs4)](https://www.youtube.com/watch?v=RScKsGfrhs4) |
| Feedback Form | [Submit Feedback](https://templates.app.n8n.cloud/form/dbf8d22e-4b05-4328-b164-dcc5555623e0) |
| AI Model Recommendation | Uses `gpt-4o` for high-precision spatial reasoning. |