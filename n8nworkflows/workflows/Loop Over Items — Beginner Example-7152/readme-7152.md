Loop Over Items — Beginner Example

https://n8nworkflows.xyz/workflows/loop-over-items---beginner-example-7152


# Loop Over Items — Beginner Example

---
### 1. Workflow Overview

This workflow demonstrates a beginner-friendly example of iterating over multiple data items using n8n’s batch processing capabilities. It simulates content ideas, processes each idea individually through an AI language model to generate LinkedIn captions, and optionally shows how to chain additional creative AI tools.

**Target Use Cases:**  
- Learning how to loop over multiple items in n8n without custom code loops  
- Generating AI-driven content captions from ideas one-by-one  
- Integrating OpenAI’s GPT models via LangChain nodes  
- Demonstrating optional AI tool chaining for enhanced creativity  

**Logical Blocks:**  
- **1.1 Input Reception and Data Simulation:** Starts the workflow and generates sample data with multiple content ideas.  
- **1.2 Item Looping:** Splits the data array into single-item batches to process sequentially.  
- **1.3 Caption Generation with AI:** Uses a LangChain agent node connected to OpenAI GPT-4o Mini to create captions for each idea.  
- **1.4 Optional Creativity Injection:** An additional LangChain tool node that could modify or enhance AI outputs (not connected in this workflow).  
- **1.5 Output Preparation:** Sets output fields for downstream processing or review.  
- **1.6 User Guidance (Sticky Note):** Provides detailed instructions and contact info for customization and support.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Data Simulation

- **Overview:** Manually triggers the workflow and generates simulated data representing multiple content ideas to process.  
- **Nodes Involved:**  
  - Run Workflow (Manual Trigger)  
  - Create Random Data (Code)  

- **Node Details:**  
  - **Run Workflow (Manual Trigger)**  
    - Type: Manual Trigger  
    - Configuration: Default manual trigger, no parameters.  
    - Input/Output: Starts the workflow; output triggers next node.  
    - Edge Cases: None significant; manual start ensures controlled execution.  

  - **Create Random Data (Code)**  
    - Type: Code (JavaScript)  
    - Configuration: Returns an array of three JSON objects, each containing a `row_number`, `id`, `Date`, `idea`, `caption`, and `complete` field. The `idea` field holds strings like “n8n rises to the top”.  
    - Key Expressions: Static JavaScript array returned directly.  
    - Input: Triggered by Manual Trigger node.  
    - Output: Emits multiple items for looping.  
    - Edge Cases: Code errors could occur if modified incorrectly; current static code is stable.

#### 2.2 Item Looping

- **Overview:** Processes each data item individually by splitting the batch into single-item chunks. This simulates looping over items since n8n does not have traditional loops.  
- **Nodes Involved:**  
  - Loop Over Items (SplitInBatches)  

- **Node Details:**  
  - Type: SplitInBatches  
  - Configuration: Default options, splits incoming data into batches of one item by default (implicit).  
  - Input: Receives array of items from Create Random Data.  
  - Output: Emits one item at a time to connected nodes.  
  - Edge Cases: If input is empty, no batches are created; large data sets may affect performance if batch size is not configured.  
  - Version: typeVersion 3, latest SplitInBatches features assumed.

#### 2.3 Caption Generation with AI

- **Overview:** Uses LangChain’s agent node to generate captions for LinkedIn posts based on each idea, leveraging OpenAI GPT-4o Mini.  
- **Nodes Involved:**  
  - Create Captions (LangChain Agent)  
  - OpenAI Chat Model (Language Model Backend)  

- **Node Details:**  
  - **Create Captions**  
    - Type: LangChain Agent  
    - Configuration:  
      - Prompt text: `idea: {{ $json.idea }}` dynamically injects the current idea.  
      - System Message: “You are a helpful assistant creating captions for a linkedin post. Please create a linkedin caption for the idea.”  
      - Model: References OpenAI GPT-4o Mini via linked node.  
    - Input: Receives single item from Loop Over Items.  
    - Output: Caption text result in `output` field on JSON.  
    - Edge Cases:  
      - Expression errors if `idea` field missing or malformed.  
      - API errors like rate limits or auth failures from OpenAI.  
      - Latency or timeout issues from external API calls.  
    - Version: typeVersion 2, compatible with LangChain agent specification.  

  - **OpenAI Chat Model**  
    - Type: LangChain LLM OpenAI Chat Model  
    - Configuration:  
      - Model set to GPT-4o Mini (value “gpt-4o-mini”).  
      - Uses saved OpenAI API credentials named “OpenAi account”.  
    - Input: Acts as language model backend for Create Captions.  
    - Output: Provides AI-generated text responses.  
    - Edge Cases:  
      - Invalid API key or expired credentials cause authentication errors.  
      - Model unavailability or quota exceeded errors.  
      - Network connectivity issues.  
    - Version: typeVersion 1.2  

#### 2.4 Optional Creativity Injection

- **Overview:** Demonstrates how an additional LangChain tool node can be chained to enhance the AI output, though currently not connected downstream.  
- **Nodes Involved:**  
  - Tool: Inject Creativity (LangChain Tool Think)  

- **Node Details:**  
  - Type: LangChain Tool Think  
  - Configuration: No parameters set, placeholder to demonstrate chaining tools.  
  - Input: Connected as an AI tool for Create Captions node but not actively used in this flow.  
  - Edge Cases: No active usage; potential for misconfiguration if connected improperly.  
  - Version: typeVersion 1  

#### 2.5 Output Preparation

- **Overview:** Prepares final output by setting selected fields from the processed data for review or further use.  
- **Nodes Involved:**  
  - Output Table (Set)  

- **Node Details:**  
  - Type: Set  
  - Configuration:  
    - Assigns two fields:  
      - `idea` is copied from the original `idea` in “Create Random Data”.  
      - `output` is taken from the AI-generated `output` field.  
  - Input: Receives output from Loop Over Items (one item at a time).  
  - Output: Cleaned-up JSON with idea and caption output.  
  - Edge Cases: If input data missing expected fields, could result in empty values.

#### 2.6 User Guidance (Sticky Note)

- **Overview:** Provides comprehensive instructions for users to understand, run, and customize the workflow. Contains contact info for support and links for credentials setup.  
- **Nodes Involved:**  
  - Sticky Note3  

- **Node Details:**  
  - Type: Sticky Note  
  - Configuration: Large note with detailed step-by-step guidance, including nodes description and OpenAI API credential instructions.  
  - Content Highlights:  
    - How to run the workflow manually.  
    - Explanation of data creation, looping, AI caption generation.  
    - Optional creativity injection explanation.  
    - Contact information: robert@ynteractive.com and LinkedIn link.  
  - Edge Cases: None; purely informational.  

---

### 3. Summary Table

| Node Name           | Node Type                          | Functional Role                     | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                               |
|---------------------|----------------------------------|-----------------------------------|------------------------|-------------------------|----------------------------------------------------------------------------------------------------------|
| Run Workflow        | Manual Trigger                   | Starts workflow manually          | —                      | Create Random Data       | See full step-by-step setup and contact info in Sticky Note3                                             |
| Create Random Data  | Code                            | Simulates multiple input ideas    | Run Workflow            | Loop Over Items          | See full step-by-step setup and contact info in Sticky Note3                                             |
| Loop Over Items     | SplitInBatches                  | Splits input into single-item batches (loop simulation) | Create Random Data       | Output Table, Create Captions | See full step-by-step setup and contact info in Sticky Note3                                             |
| Create Captions     | LangChain Agent                 | Generates AI captions per idea    | Loop Over Items          | —                       | See full step-by-step setup and contact info in Sticky Note3                                             |
| OpenAI Chat Model   | LangChain LLM OpenAI Chat Model | Provides GPT-4o Mini AI backend   | — (AI Language Model for Create Captions) | Create Captions          | See full step-by-step setup and contact info in Sticky Note3                                             |
| Tool: Inject Creativity | LangChain Tool Think           | Optional AI creativity enhancer   | — (AI Tool for Create Captions) | Create Captions          | See full step-by-step setup and contact info in Sticky Note3                                             |
| Output Table        | Set                             | Prepares final output fields      | Loop Over Items          | —                       | See full step-by-step setup and contact info in Sticky Note3                                             |
| Sticky Note3        | Sticky Note                     | User instructions and contact info | —                      | —                       | ## 📬 Need Help or Want to Customize This?  Contact robert@ynteractive.com, LinkedIn: https://www.linkedin.com/in/robert-breen-29429625/ |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `Run Workflow`  
   - Type: Manual Trigger  
   - Purpose: To manually start the workflow for testing or demonstration.

2. **Create Code Node for Random Data**  
   - Name: `Create Random Data`  
   - Type: Code (JavaScript)  
   - Parameters: Use this JavaScript snippet to return multiple content ideas as JSON objects:  
     ```javascript
     return [
       { json: { row_number: 2, id: 1, Date: '2025-07-30', idea: 'n8n rises to the top', caption: '', complete: '' } },
       { json: { row_number: 3, id: 2, Date: '2025-07-31', idea: 'n8n nodes', caption: '', complete: '' } },
       { json: { row_number: 4, id: 3, Date: '2025-08-01', idea: 'n8n use cases for marketing', caption: '', complete: '' } }
     ];
     ```
   - Connect `Run Workflow` output to this node’s input.

3. **Add SplitInBatches Node for Looping**  
   - Name: `Loop Over Items`  
   - Type: SplitInBatches  
   - Parameters: Leave batch size default (1) to send one item at a time downstream.  
   - Connect `Create Random Data` output to this node’s input.

4. **Add LangChain Agent Node for Caption Generation**  
   - Name: `Create Captions`  
   - Type: LangChain Agent  
   - Parameters:  
     - Text prompt: `idea: {{ $json.idea }}` (dynamic expression)  
     - System Message: “You are a helpful assistant creating captions for a linkedin post. Please create a linkedin caption for the idea.”  
     - Prompt Type: Define (custom prompt)  
   - Connect `Loop Over Items` output to this node’s input.

5. **Add LangChain OpenAI Chat Model Node**  
   - Name: `OpenAI Chat Model`  
   - Type: LangChain LLM OpenAI Chat Model  
   - Parameters:  
     - Model selection: GPT-4o Mini (`gpt-4o-mini`)  
   - Credentials: Configure OpenAI API credentials named (e.g.) `OpenAi account` with your OpenAI API key.  
   - Connect this node as the AI Language Model for `Create Captions` node.

6. **Optionally Add LangChain Tool Node for Creativity Injection**  
   - Name: `Tool: Inject Creativity`  
   - Type: LangChain Tool Think  
   - Parameters: None needed initially; serves as an example.  
   - Optionally connect as AI tool input to `Create Captions` node if creativity injection desired.

7. **Add Set Node for Output Preparation**  
   - Name: `Output Table`  
   - Type: Set  
   - Parameters: Assign two fields:  
     - `idea` = `={{ $('Create Random Data').item.json.idea }}`  
     - `output` = `={{ $json.output }}` (from AI caption result)  
   - Connect `Loop Over Items` output to this node.

8. **Add Sticky Note with User Guidance**  
   - Name: `Sticky Note3`  
   - Type: Sticky Note  
   - Content: Include instructions on workflow purpose, running instructions, node descriptions, and contact info for support.  
   - Position visually for easy user reference.

9. **Connect Node Flow:**  
   - `Run Workflow` → `Create Random Data` → `Loop Over Items`  
   - `Loop Over Items` → `Create Captions` (LangChain Agent)  
   - `Create Captions` uses `OpenAI Chat Model` as its AI model backend.  
   - `Loop Over Items` also connects to `Output Table` for final output formatting.  
   - Optionally, `Tool: Inject Creativity` can be linked as an AI tool input to `Create Captions`.  

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Contact for help or customization: robert@ynteractive.com | Contact info in Sticky Note3 |
| LinkedIn profile of author: https://www.linkedin.com/in/robert-breen-29429625/ | Sticky Note3 |
| OpenAI API Key setup instructions: https://platform.openai.com/account/api-keys | Described in Sticky Note3 |
| Workflow demonstrates beginner-level looping via SplitInBatches in n8n | Educational use |
| Shows integration of LangChain nodes with OpenAI GPT models in n8n workflows | AI integration example |

---

**Disclaimer:** The provided text originates exclusively from an automated n8n workflow. It complies strictly with content policies and includes no illegal, offensive, or protected material. All data handled is legal and public.