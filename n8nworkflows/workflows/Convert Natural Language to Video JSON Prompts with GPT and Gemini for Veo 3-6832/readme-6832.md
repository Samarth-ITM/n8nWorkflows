Convert Natural Language to Video JSON Prompts with GPT and Gemini for Veo 3

https://n8nworkflows.xyz/workflows/convert-natural-language-to-video-json-prompts-with-gpt-and-gemini-for-veo-3-6832


# Convert Natural Language to Video JSON Prompts with GPT and Gemini for Veo 3

### 1. Workflow Overview

This workflow converts natural language prompts describing video scenes into detailed, structured JSON prompts optimized for Veo 3 video generation, then uses the Google Gemini model to generate videos based on those JSON prompts. It is designed for users who want to input video scene descriptions in plain language and receive high-quality video outputs via Veo 3’s generation engine.

The workflow is logically divided into these blocks:

- **1.1 Input Reception**: Receives user input via a chat trigger.
- **1.2 Prompt Conversion**: Converts the user’s natural language input into a structured JSON prompt adhering to Veo 3’s schema using GPT-4 via Azure OpenAI.
- **1.3 JSON Parsing**: Parses and validates the generated JSON to ensure compliance and structure.
- **1.4 Video Generation**: Sends the validated JSON prompt to the Google Gemini Veo 3 video generation model to produce video outputs.

This linear flow ensures natural language input is accurately transformed into the required technical format and then used to generate videos.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Accepts user natural language descriptions via a chat webhook to start the workflow.

- **Nodes Involved:**  
  - Prompt Input

- **Node Details:**  
  - **Prompt Input**  
    - Type: `chatTrigger` (n8n LangChain Chat Trigger node)  
    - Purpose: Entry point that listens for chat messages/webhook calls from users.  
    - Configuration: Uses a webhook ID to receive input asynchronously. No specific options configured beyond default.  
    - Inputs: External user input (chat message)  
    - Outputs: Sends raw user text to the next node (“Prompt converter”)  
    - Failure cases: Webhook errors, connectivity issues, or malformed incoming messages could cause failure.  
    - Version: 1.1  
    - Notes: This node is the user’s interface to the workflow.

#### 2.2 Prompt Conversion

- **Overview:**  
  Converts the free-form user input into a detailed JSON prompt aligned with Veo 3 video generation specifications using Azure OpenAI GPT-4.

- **Nodes Involved:**  
  - Alternative (OpenRouter fallback)  
  - Openai (Azure OpenAI GPT-4)  
  - Prompt converter

- **Node Details:**  

  - **Alternative**  
    - Type: `lmChatOpenRouter` (OpenRouter language model chat node)  
    - Role: Alternative AI chat model for fallback or comparative processing.  
    - Configuration: Uses OpenRouter API credentials. No special options.  
    - Inputs: No direct connection in the active workflow; possibly reserved for alternative processing paths.  
    - Outputs: None connected in this workflow.  
    - Edge cases: API rate limits, authentication errors.  
    - Version: 1

  - **Openai**  
    - Type: `lmChatAzureOpenAi` (Azure OpenAI GPT language model chat node)  
    - Role: Primary AI model to interpret and convert natural language prompts.  
    - Configuration: Model set to “gpt” (GPT-4.1) via Azure credentials. No additional options set.  
    - Inputs: Receives user prompt from “Prompt Input” node.  
    - Outputs: Sends AI-generated JSON text to “Prompt converter” node for further processing.  
    - Edge cases: Authentication issues, timeouts, or malformed prompts causing AI errors.  
    - Version: 1

  - **Prompt converter**  
    - Type: `chainLlm` (Language Model Chain node)  
    - Role: Core node that formats the input prompt using a detailed system prompt instructing the AI to output strictly valid JSON conforming to Veo 3 schema.  
    - Configuration:  
      - Uses a custom system prompt detailing JSON keys, defaults, expected values, and processing rules.  
      - Ensures output contains only valid JSON without explanation text.  
      - Includes sample JSON schema and parameter guidelines directly embedded in the prompt.  
    - Inputs: Receives raw user input from “Prompt Input” and AI chat output from “Openai”.  
    - Outputs: Sends structured JSON text to the “Json parser” node.  
    - Key expressions: Uses an extensive system prompt text that governs output format and content.  
    - Edge cases: Possible AI output that is not valid JSON or incomplete; expression evaluation failures if input unexpected.  
    - Version: 1.7

#### 2.3 JSON Parsing

- **Overview:**  
  Parses and validates the JSON output from the previous node, ensuring it matches the expected schema before usage.

- **Nodes Involved:**  
  - Json parser

- **Node Details:**  
  - **Json parser**  
    - Type: `outputParserStructured` (LangChain Output Parser Structured node)  
    - Role: Validates and parses the AI-generated JSON prompt against an example schema.  
    - Configuration: Uses a JSON schema example defining all expected fields such as prompt, aspect_ratio, style, etc.  
    - Inputs: Receives AI JSON output from “Prompt converter”.  
    - Outputs: Sends parsed and validated JSON objects to “Generate a video” node.  
    - Edge cases: Parsing failures if the AI output is invalid JSON or schema mismatch.  
    - Version: 1.3

#### 2.4 Video Generation

- **Overview:**  
  Uses the structured JSON prompt to generate a video via Google Gemini’s Veo 3 model.

- **Nodes Involved:**  
  - Generate a video

- **Node Details:**  
  - **Generate a video**  
    - Type: `googleGemini` (LangChain Google Gemini node)  
    - Role: Calls Google Gemini API with the JSON prompt to generate video content.  
    - Configuration:  
      - Prompt parameter is set dynamically to the stringified JSON output from the “Json parser”.  
      - Model ID is explicitly set to “models/veo-2.0-generate-001” which corresponds to Veo 3 video generation.  
      - Options include aspect ratio “16:9” and sample count 1 (single video).  
      - Resource type set to “video”.  
    - Credentials: Uses Google Palm API credentials configured with GCP project named “n8n-khmuhtadin”.  
    - Inputs: Receives validated JSON prompt from the “Json parser”.  
    - Outputs: Produces video generation results (likely metadata or video URL).  
    - Edge cases: API limits, authentication failures, invalid prompt causing generation errors, network timeouts.  
    - Version: 1

---

### 3. Summary Table

| Node Name       | Node Type                       | Functional Role                     | Input Node(s)    | Output Node(s)       | Sticky Note                         |
|-----------------|--------------------------------|-----------------------------------|------------------|----------------------|-----------------------------------|
| Prompt Input    | chatTrigger                    | Receives user natural language input | —                | Prompt converter     |                                   |
| Alternative     | lmChatOpenRouter               | Alternative AI language model (unused) | —                | —                    |                                   |
| Openai          | lmChatAzureOpenAi              | Primary AI model for prompt conversion | Prompt Input     | Prompt converter     |                                   |
| Prompt converter| chainLlm                      | Converts input text to Veo 3 JSON prompt | Prompt Input, Openai | Json parser          | Detailed system prompt for JSON format and defaults |
| Json parser     | outputParserStructured         | Parses and validates JSON output   | Prompt converter | Generate a video      |                                   |
| Generate a video| googleGemini                   | Generates video from JSON prompt  | Json parser      | —                    | Uses Veo 3 model "veo-2.0-generate-001" |

---

### 4. Reproducing the Workflow from Scratch

1. **Create “Prompt Input” node**  
   - Type: LangChain Chat Trigger (`chatTrigger`)  
   - Configure webhook ID to listen for incoming chat messages.  
   - No special options.  
   - This node is the entry point.

2. **Create “Openai” node**  
   - Type: LangChain Azure OpenAI Chat (`lmChatAzureOpenAi`)  
   - Set model to “gpt” (GPT-4.1).  
   - Configure Azure OpenAI API credentials.  
   - Connect input from “Prompt Input” node’s output.

3. **Create “Prompt converter” node**  
   - Type: LangChain Chain LLM (`chainLlm`)  
   - Paste the detailed system prompt instructing the AI to produce strict Veo 3 JSON output including the JSON schema example and parameter guidelines.  
   - Enable output parser for structured JSON.  
   - Connect two inputs: the raw user input (from “Prompt Input”) and AI chat output (from “Openai”).  
   - Configure batching as default.

4. **Create “Json parser” node**  
   - Type: LangChain Output Parser Structured (`outputParserStructured`)  
   - Paste the JSON schema example matching Veo 3 prompt fields (prompt, negative_prompt, aspect_ratio, etc.).  
   - Connect input from “Prompt converter” output.

5. **Create “Generate a video” node**  
   - Type: LangChain Google Gemini (`googleGemini`)  
   - Set model ID to “models/veo-2.0-generate-001” (Veo 3 video generation).  
   - Set prompt parameter to `={{ JSON.stringify($json.output) }}` to send parsed JSON as string.  
   - Set options with aspectRatio “16:9” and sampleCount 1.  
   - Set resource to “video”.  
   - Configure Google Palm API credentials with appropriate GCP project.  
   - Connect input from “Json parser”.

6. **Connect nodes in this order:**  
   Prompt Input → Openai → Prompt converter → Json parser → Generate a video

7. **Credential Setup:**  
   - Azure OpenAI API credential with GPT-4 access for “Openai” node.  
   - Google Palm API credential for “Generate a video” node.

8. **Test the workflow by sending a natural language video description to the webhook** and verify the output video generation.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                           |
|----------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| The system prompt in the “Prompt converter” node is detailed and critical for proper JSON output from GPT-4.         | Embedded in “Prompt converter” node configuration.                                                       |
| Google Gemini Veo 3 video generation model ID: “models/veo-2.0-generate-001”                                        | Used in “Generate a video” node.                                                                         |
| Default video generation parameters (aspect ratio 16:9, sample count 1) are set explicitly in the Gemini node.      | “Generate a video” node options.                                                                         |
| Workflow disabled by default (`active: false`), enable before use.                                                   | Workflow metadata.                                                                                        |
| Workflow uses multiple AI models: Azure OpenAI GPT-4 for prompt conversion and Google Gemini for video generation.   | Integration details in node configurations.                                                              |

---

**Disclaimer:** This document is generated exclusively from an n8n automated workflow JSON. It complies with current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.