Convert POML to AI-Ready Prompts & Chat Messages with Zero Dependencies

https://n8nworkflows.xyz/workflows/convert-poml-to-ai-ready-prompts---chat-messages-with-zero-dependencies-7609


# Convert POML to AI-Ready Prompts & Chat Messages with Zero Dependencies

---

## 1. Workflow Overview

This n8n workflow titled **"Convert POML to AI-Ready Prompts & Chat Messages with Zero Dependencies"** is designed to transform POML (Project Object Markup Language) — a structured XML-like markup format — into AI-consumable text prompts or chat message arrays without relying on external libraries. It supports contextual variable substitution, basic content structuring (headings, lists, code blocks, media, tables), and optional schema-driven validation of tags and attributes.

The workflow’s main use cases include:

- Converting structured authoring content (POML) into ready-to-use prompts for AI models.
- Generating chat message arrays for conversational AI interfaces.
- Supporting zero-dependency operation within n8n for portability and template-library friendliness.

### Logical Blocks

**1.1 Input Initialization**  
- Prepares and sets all necessary input variables including POML markup, context for variable substitution, speaker mode flag, list styling, component attribute specifications, and attribute type specifications.

**1.2 POML Parsing and Compilation**  
- Parses the POML string into an abstract syntax tree (AST) using a custom XML-ish tokenizer.  
- Recursively compiles the AST into either a Markdown prompt or structured chat messages, applying context variable substitution and respecting component and attribute schemas.

**1.3 AI Processing**  
- Feeds the generated prompt or messages into an AI agent node for further processing or response generation.  
- Uses an OpenAI GPT-4.1 mini model as the underlying language model for the AI agent.

**1.4 Manual Trigger**  
- Entry point for manual execution of the workflow.

**1.5 Documentation and Notes**  
- Multiple sticky notes providing detailed instructions, usage examples, limitations, troubleshooting tips, and credits.

---

## 2. Block-by-Block Analysis

### 2.1 Input Initialization

**Overview:**  
This block sets all input data and parameters required for POML parsing and compilation. It defines the POML markup string, context object for variable substitution, speaker mode toggle, list style preference, and specifications for allowed components and attributes.

**Nodes Involved:**  
- Set_Variables

**Node Details:**

- **Set_Variables**  
  - Type: Set node  
  - Role: Initializes workflow inputs by defining static variables and objects.  
  - Configuration:  
    - `poml`: Contains a sample POML markup string with `<task>` and `<stepwise-instructions>` tags with nested lists.  
    - `context`: JSON object describing project metadata (name, audience, timeframe, methods, success criteria) for variable substitution.  
    - `speakerMode`: Boolean flag set to `true` indicating output should be chat messages rather than plain prompt text.  
    - `listStyle`: String set to `dash` indicating the default bullet style for lists.  
    - `componentSpec`: Object defining allowed component tags and their allowed attributes, supporting case-insensitivity and detailed schema-driven validation.  
    - `attributeSpec`: Array specifying attribute types, descriptions, and applicable components (e.g., `boolean`, `integer`, `string`, `ai/human/system` roles).  
  - Input connections: Receives trigger from manual trigger node.  
  - Output connections: Passes data to the `Parse_POML` code node.  
  - Edge cases: Incorrect or missing inputs here will propagate errors downstream; static assignment reduces dynamic failure risk.

---

### 2.2 POML Parsing and Compilation

**Overview:**  
This block translates the POML markup string into a Markdown prompt or structured chat messages, applying context substitutions and respecting component and attribute specifications. It uses a zero-dependency custom JavaScript parser and compiler implemented in an n8n Code node.

**Nodes Involved:**  
- Parse_POML

**Node Details:**

- **Parse_POML**  
  - Type: Code node (Function)  
  - Role: Parses POML markup to AST, compiles into prompt text or chat message array.  
  - Configuration highlights:  
    - Implements a minimal XML-ish tokenizer to convert POML string to AST.  
    - Performs substitution of `{{dot.path}}` style variables using provided context.  
    - Normalizes component and attribute specifications for validation and coercion.  
    - Supports a subset of tags: headings (`<h>`), text blocks (`<p>`, `<text>`), emphasis (`<b>`, `<i>`), lists (`<list>`, `<item>`), code (`<code>`), media (`<img>`, `<audio>`), line breaks (`<br>`), tables (`<table>`), and chat messages (`<system-msg>`, `<user-msg>`, `<ai-msg>`).  
    - Handles speaker mode: when enabled, produces an array of message objects `{ role, content }` for AI chat consumption. If disabled, produces a single Markdown prompt string.  
    - Unknown tags render children without failure, ensuring graceful degradation.  
  - Key expressions/logic:  
    - `parsePoml(input)`: Tokenizes POML string to AST.  
    - `compileNode(node, ctx, roleCtx, options)`: Recursively compiles AST nodes into output strings or messages, with role context for speaker mode.  
    - `substitute(str, ctx)`: Performs variable substitution in text nodes.  
    - `normalizeComponentSpec` and `normalizeAttributeSpec` for schema validation.  
  - Input connections: Receives variables from `Set_Variables`.  
  - Output connections: Passes compiled prompt/messages to `AI Agent`.  
  - Version-specific requirements: Requires n8n supporting Function node with modern JavaScript (ES6+).  
  - Edge cases/potential failure:  
    - Missing or invalid `poml` string input yields error message `{ error: 'json.poml (string) is required' }`.  
    - Malformed POML syntax could cause parsing errors, caught and returned as error messages.  
    - Attribute substitution errors are handled gracefully by fallback defaults.  
    - Speaker mode enabled but no message tags present defaults to wrapping entire prompt as a single user message.  
  - Sub-workflow references: None.

---

### 2.3 AI Processing

**Overview:**  
This block consumes the compiled prompt or message array and invokes an AI language model agent to process or generate output based on the input.

**Nodes Involved:**  
- AI Agent  
- OpenAI Chat Model

**Node Details:**

- **AI Agent**  
  - Type: LangChain Agent node  
  - Role: Acts as an interface for AI language model interaction, using the compiled prompt/messages to produce AI responses.  
  - Configuration:  
    - Text input set dynamically as the compiled prompt from `Parse_POML` (`={{ $json.prompt }}`).  
    - Prompt type: `define` (custom prompt input).  
    - No additional options set.  
  - Input connections: Receives prompt/messages from `Parse_POML`.  
  - Output connections: None configured (end node).  
  - Edge cases: Invalid or empty prompt may produce empty or error responses.  
  - Version-specific requirements: Requires n8n version with LangChain integration.

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model node  
  - Role: Provides underlying GPT-4.1 mini language model to `AI Agent`.  
  - Configuration:  
    - Model set to `gpt-4.1-mini`.  
    - No additional options specified.  
  - Credentials: Uses configured OpenAI API credentials named "SNPT - OpenAi account 3".  
  - Input connections: Connected as AI language model provider to `AI Agent`.  
  - Output connections: None (passes output to `AI Agent`).  
  - Edge cases: API key issues, rate limits, or model unavailability can cause runtime errors.

---

### 2.4 Manual Trigger

**Overview:**  
Entry point node for manual execution of the workflow.

**Nodes Involved:**  
- ‘Execute workflow’ (Manual Trigger)

**Node Details:**

- **‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Allows user to start the workflow manually from the n8n editor interface.  
  - Configuration: Default, no parameters.  
  - Input connections: None.  
  - Output connections: Triggers `Set_Variables`.  
  - Edge cases: None significant.

---

### 2.5 Documentation and Notes

**Overview:**  
Provides rich contextual information, usage instructions, examples, limitations, troubleshooting tips, and credits via sticky notes.

**Nodes Involved:**  
- Sticky Note (README)  
- Sticky Note1 (Quick Start)  
- Sticky Note2 (More Examples)  
- Sticky Note3 (Limitations & Troubleshooting)  
- Sticky Note4 (Credits)

**Node Details:**

- **Sticky Note (README)**  
  - Type: Sticky Note  
  - Content: Detailed overview of POML → Prompt/Messages conversion, input/output expectations, supported tags, and usage tips.  
  - Positioned off-main canvas; serves as embedded documentation.

- **Sticky Note1 (Quick Start)**  
  - Provides a copy-paste quick start guide including example `context` JSON, supported tags, and behavior tips.

- **Sticky Note2 (More Examples)**  
  - Shows example input and output JSON for typical usage with sample POML markup and message output.

- **Sticky Note3 (Limitations & Troubleshooting)**  
  - Lists known limitations by design, common error messages, and troubleshooting advice.

- **Sticky Note4 (Credits)**  
  - Contains author credits, branding, and links for additional templates and support.

---

## 3. Summary Table

| Node Name       | Node Type                         | Functional Role                              | Input Node(s)          | Output Node(s)   | Sticky Note                                                                                  |
|-----------------|----------------------------------|----------------------------------------------|-----------------------|------------------|----------------------------------------------------------------------------------------------|
| ‘Execute workflow’ | Manual Trigger                   | Entry point for manual workflow execution    | -                     | Set_Variables    |                                                                                              |
| Set_Variables   | Set                              | Initializes all input variables & specs      | ‘Execute workflow’     | Parse_POML       | Quick start guide and example context in Sticky Note1                                       |
| Parse_POML      | Code (Function)                  | Parses POML, compiles prompt/messages        | Set_Variables          | AI Agent         | Main README and detailed notes in Sticky Note, plus limitations and troubleshooting in Sticky Note3 |
| AI Agent        | LangChain Agent                  | Sends prompt/messages to AI model            | Parse_POML, OpenAI Chat Model | -            |                                                                                              |
| OpenAI Chat Model | LangChain OpenAI Chat Model      | Provides GPT-4.1 mini model for AI Agent     | -                     | AI Agent         |                                                                                              |
| Sticky Note     | Sticky Note                     | README and detailed workflow explanation     | -                     | -                | Contains full README and usage instructions                                                 |
| Sticky Note1    | Sticky Note                     | Quick start instructions                      | -                     | -                | Quick start guide with examples                                                            |
| Sticky Note2    | Sticky Note                     | Additional examples                           | -                     | -                | More examples of input/output POML conversion                                              |
| Sticky Note3    | Sticky Note                     | Limitations and troubleshooting               | -                     | -                | Known limitations and troubleshooting tips                                                 |
| Sticky Note4    | Sticky Note                     | Credits and branding                          | -                     | -                | Project credits and external resource links                                                |

---

## 4. Reproducing the Workflow from Scratch

**Step 1:** Create a **Manual Trigger** node.  
- Name: ‘Execute workflow’.  
- No special parameters needed.

**Step 2:** Add a **Set** node to initialize variables.  
- Name: Set_Variables.  
- Configure the following fields as static assignments:  
  - `poml` (string): Paste your POML markup string, e.g. the provided example with `<task>` and `<stepwise-instructions>` containing nested lists.  
  - `context` (object): Paste or define a JSON object for variable substitution context (e.g., project metadata).  
  - `speakerMode` (boolean): Set to `true` to enable chat message output; `false` for plain prompt.  
  - `listStyle` (string): Set preferred list bullet style, e.g. `dash`.  
  - `componentSpec` (object): Define allowed components and their attributes as a JSON object with tag names as keys and arrays of attribute names. Use the example provided.  
  - `attributeSpec` (array): Define attribute type specifications as an array of objects with `attribute`, `applies_to`, `type`, and `description` fields as per example.

- Connect **Manual Trigger** → **Set_Variables**.

**Step 3:** Add a **Code** node (Function) to parse and compile POML.  
- Name: Parse_POML.  
- Paste the complete JavaScript code provided (the zero-dependency POML parser and compiler).  
- Configure no credentials needed.  
- Connect **Set_Variables** → **Parse_POML**.

**Step 4:** Add a **LangChain OpenAI Chat Model** node.  
- Name: OpenAI Chat Model.  
- Set Model to `gpt-4.1-mini`.  
- Provide valid OpenAI API credentials in n8n (create or select credential named accordingly).  
- Connect no input nodes, but connect output to **AI Agent** as AI language model provider.

**Step 5:** Add a **LangChain Agent** node.  
- Name: AI Agent.  
- Set Text input to expression: `={{ $json.prompt }}`.  
- Set Prompt Type to `define`.  
- Connect **Parse_POML** (main output) → **AI Agent** (text input).  
- Connect **OpenAI Chat Model** → **AI Agent** (AI language model input).

**Step 6:** (Optional) Add Sticky Notes for documentation.  
- Add Sticky Notes containing README, Quick Start, Examples, Limitations, and Credits content as per provided texts.  
- Position for readability, not connected.

**Step 7:** Save and activate the workflow.  
- Trigger manually via `Execute workflow` node to test.

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                               |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Turns **POML** markup into Markdown prompts or chat messages using a zero-dependency JavaScript parser within n8n. Supports variable substitution and schema-driven validation.                                                                                                                                         | See Sticky Note (README) in workflow.                                                                        |
| Quick start example context JSON for variable substitution: project name, audience, timeframe, methods, success criteria.                                                                                                                                                                                             | See Sticky Note1 (Quick Start).                                                                               |
| Supported POML tags include headings, text, emphasis, lists, code, images, audio, line breaks, tables, and chat message roles (system, user, AI).                                                                                                                                                                    | See Sticky Note1 and Sticky Note2.                                                                            |
| Limitations: no external libraries, no file includes or stylesheets, attribute substitutions not enabled by default, pragmatic parser not full POML spec, input syntax must be valid. Troubleshooting tips included.                                                                                                    | See Sticky Note3 (Limitations & Troubleshooting).                                                             |
| Authored by Real Simple Solutions as an n8n template-library-friendly POML compiler with no dependencies. For full POML feature parity, contact for workflows using Microsoft’s official SDK.                                                                                                                        | See Sticky Note4 (Credits).                                                                                   |
| More templates and resources available at [https://joeperes.gumroad.com/](https://joeperes.gumroad.com/) and [https://realsimple.dev](https://realsimple.dev).                                                                                                                                                       | See Sticky Note4 (Credits).                                                                                   |
| POML logo image used in README note: ![Poml Logo](https://poml-team.gallerycdn.vsassets.io/extensions/poml-team/poml/0.0.7/1753436631886/Microsoft.VisualStudio.Services.Icons.Default)                                                                                                                                   | Branding in README sticky note.                                                                                |

---

*Disclaimer: The text provided originates exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly respects content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.*