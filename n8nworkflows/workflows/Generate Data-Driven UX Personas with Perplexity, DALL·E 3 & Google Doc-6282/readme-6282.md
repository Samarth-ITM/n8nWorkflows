Generate Data-Driven UX Personas with Perplexity, DALL·E 3 & Google Doc

https://n8nworkflows.xyz/workflows/generate-data-driven-ux-personas-with-perplexity--dall-e-3---google-doc-6282


# Generate Data-Driven UX Personas with Perplexity, DALL·E 3 & Google Doc

### 1. Workflow Overview

This n8n workflow automates the creation of data-driven UX personas by combining market research, AI-based persona generation, image creation, and document publishing. It is designed for UX researchers, product managers, and marketing teams who want to generate detailed user personas based on real market data, enriched with AI-generated images, and consolidated into a Google Doc for easy sharing and collaboration.

**Target Use Cases:**
- Creating validated UX personas based on actual market research
- Enriching personas with AI-generated realistic images
- Publishing persona descriptions and images in Google Docs and Google Drive
- Enabling non-technical users to trigger the process via a web form

**Logical Blocks:**

- **1.1 Input Reception:** Captures user input from a web form (website URL, region, industry).
- **1.2 Market Research via Perplexity:** Performs deep research to gather market insights and segment data.
- **1.3 Persona Description Generation:** Uses OpenAI to transform research data into structured persona profiles.
- **1.4 Image Prompt Creation:** Generates prompts for image creation based on persona usage stories.
- **1.5 Image Generation and Upload:** Creates persona images with OpenAI's DALL·E 3 and uploads them to Google Drive.
- **1.6 Document Update:** Updates a Google Doc with the finalized persona descriptions.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block triggers the workflow via a web form submission, collecting essential inputs to define the scope for persona creation.

- **Nodes Involved:**  
  - On form submission1

- **Node Details:**

  - **On form submission1**  
    - **Type & Role:** Form Trigger node; initiates workflow on web form submission.  
    - **Configuration:**  
      - Form titled "Creating Personas"  
      - Fields: Website URL (required), Region, Industry  
      - Form description explains its purpose for persona creation based on market research.  
    - **Expressions/Variables:** Captures form fields as JSON for downstream use.  
    - **Input:** HTTP POST form submission.  
    - **Output:** JSON object with form data.  
    - **Failure modes:** Missing required fields, webhook connection issues.  
    - **Version:** 2.2

#### 1.2 Market Research via Perplexity

- **Overview:**  
  Conducts automated deep market research and segmentation using Perplexity’s AI, creating a data foundation for personas.

- **Nodes Involved:**  
  - 🔍 Research  
  - Sticky Note (Research on user segments and their market size)

- **Node Details:**

  - **🔍 Research**  
    - **Type & Role:** Perplexity node; performs research queries.  
    - **Configuration:**  
      - Model: "sonar-deep-research" for thorough analysis  
      - Prompt content dynamically composed from form inputs: website URL, region, industry  
      - Research prompt structured to extract business overview, segment names, descriptions, market size metrics (TAM, CAGR, etc.), user needs, pain points, and usage stories, all with citations.  
    - **Input:** JSON from form submission node.  
    - **Output:** Research results including citations and structured data.  
    - **Credential:** Requires Perplexity API credentials.  
    - **Failure modes:** API rate limits, invalid input URLs, network errors, malformed prompt.  
    - **Version:** 1

  - **Sticky Note (Research on user segments and their market size)**  
    - Provides contextual guidance on this block’s purpose.

#### 1.3 Persona Description Generation

- **Overview:**  
  Transforms raw research data into polished, structured persona descriptions formatted for business use.

- **Nodes Involved:**  
  - OpenAI Chat Model2  
  - Personas descriptions (LangChain Agent)  
  - Sticky Note (Persona description creation)

- **Node Details:**

  - **OpenAI Chat Model2**  
    - **Type & Role:** OpenAI Chat Model node (GPT-4.1-mini); processes raw research output.  
    - **Configuration:** Model set to GPT-4.1-mini for balanced power and cost.  
    - **Input:** Research output JSON passed as prompt content.  
    - **Output:** Text containing persona descriptions with citations.  
    - **Credential:** OpenAI API key required.  
    - **Failure modes:** API quota limits, invalid or incomplete input, timeout.  
    - **Version:** 1.2

  - **Personas descriptions**  
    - **Type & Role:** LangChain Agent node; formats and structures persona text for clarity and business context.  
    - **Configuration:**  
      - System message instructs formatting using Markdown-style headings for Google Docs rendering.  
      - Output includes sections: User Segment, Market Size, User Needs, Pain Points, Usage Story, and Reference links.  
    - **Input:** Output from OpenAI Chat Model2.  
    - **Output:** Formatted persona descriptions.  
    - **Failure modes:** Formatting errors, empty input, API errors.  
    - **Version:** 2

  - **Sticky Note (Persona description creation)**  
    - Context note explaining this block’s role.

#### 1.4 Image Prompt Creation

- **Overview:**  
  Generates detailed image generation prompts for each persona based on the Usage Story section.

- **Nodes Involved:**  
  - Generate Prompts for Images (LangChain Agent)  
  - Parse Prompts into JSON Array  
  - Sticky Note (Create a prompt for generating an image for each persona)

- **Node Details:**

  - **Generate Prompts for Images**  
    - **Type & Role:** LangChain Agent node; creates image prompts for AI generation.  
    - **Configuration:**  
      - System message instructs to create a JSON array with prompts for 400x400 pixel square images, no props or text, filenames based on persona title.  
    - **Input:** Persona descriptions from previous block.  
    - **Output:** JSON array of image prompt objects.  
    - **Failure modes:** Improper JSON output, incomplete prompt generation, API failures.  
    - **Version:** 1.7

  - **Parse Prompts into JSON Array**  
    - **Type & Role:** LangChain Output Parser; ensures output is valid JSON array for downstream splitting.  
    - **Configuration:** Uses example JSON schema to validate structure.  
    - **Input:** Raw JSON string from Generate Prompts for Images.  
    - **Output:** Parsed JSON array.  
    - **Failure modes:** Parsing errors if output is malformed.  
    - **Version:** 1.2

  - **Sticky Note (Create a prompt for generating an image for each persona)**  
    - Clarifies the functionality of this block.

#### 1.5 Image Generation and Upload

- **Overview:**  
  Generates persona images using OpenAI's image generation capability and uploads them to a specified Google Drive folder.

- **Nodes Involved:**  
  - Split Prompts  
  - Generate images (OpenAI Image Generation)  
  - Upload images (Google Drive)  
  - Sticky Note (Generate images and upload to Google Drive)

- **Node Details:**

  - **Split Prompts**  
    - **Type & Role:** Split Out node; splits the JSON array of image prompts into individual items for parallel image generation.  
    - **Configuration:** Splits on the "output" field.  
    - **Input:** Parsed JSON array of prompts.  
    - **Output:** Individual prompt objects.  
    - **Failure modes:** Empty input, incorrect field name.  
    - **Version:** 1

  - **Generate images**  
    - **Type & Role:** OpenAI resource node for image generation (DALL·E 3).  
    - **Configuration:**  
      - Uses prompt from split item.  
      - Generates 400x400 pixel images as specified.  
    - **Input:** Single prompt per execution.  
    - **Output:** Binary image data.  
    - **Credential:** OpenAI API key required.  
    - **Failure modes:** API quota, invalid prompt, generation timeout.  
    - **Version:** 1.8

  - **Upload images**  
    - **Type & Role:** Google Drive node; uploads generated images to a specific folder in Google Drive.  
    - **Configuration:**  
      - Target folder ID: "1kv3Yo3cKK6Vsd2sBQYUN2RsBL4E1KOW8" (configurable).  
      - Filename based on timestamp for uniqueness.  
    - **Input:** Binary image data from Generate images.  
    - **Output:** Google Drive file metadata.  
    - **Credential:** Google Drive OAuth2 credentials required.  
    - **Failure modes:** Permission errors, invalid folder ID, upload failures.  
    - **Version:** 3

  - **Sticky Note (Generate images and upload to Google Drive)**  
    - Provides context on this block’s purpose.

#### 1.6 Document Update

- **Overview:**  
  Inserts the finalized persona descriptions into an existing Google Doc for easy access and sharing.

- **Nodes Involved:**  
  - Update a document1

- **Node Details:**

  - **Update a document1**  
    - **Type & Role:** Google Docs node; inserts text into a Google Doc.  
    - **Configuration:**  
      - Operation: "update" (inserts text into the document).  
      - Inserts the persona description output as text.  
    - **Input:** Persona descriptions from LangChain Agent.  
    - **Credential:** Google Docs OAuth2 credentials required.  
    - **Failure modes:** Document access permission errors, invalid document ID, API quota limits.  
    - **Version:** 2

---

### 3. Summary Table

| Node Name               | Node Type                                 | Functional Role                       | Input Node(s)           | Output Node(s)                 | Sticky Note                                                                                                      |
|-------------------------|-------------------------------------------|------------------------------------|------------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------|
| On form submission1      | Form Trigger                             | Input reception from web form       | —                      | 🔍 Research                   |                                                                                                                  |
| 🔍 Research             | Perplexity                               | Market research and segmentation   | On form submission1    | Personas descriptions          | Research on user segments and their market size                                                                 |
| Sticky Note             | Sticky Note                             | Explanation of research block      | —                      | —                             | Research on user segments and their market size                                                                 |
| OpenAI Chat Model2      | OpenAI Chat Model (GPT)                  | Processes raw research output      | 🔍 Research             | Personas descriptions          |                                                                                                                  |
| Personas descriptions   | LangChain Agent                         | Formats persona descriptions       | 🔍 Research, OpenAI Chat Model2 | Update a document1, Generate Prompts for Images | Persona description creation                                                                                     |
| Sticky Note2            | Sticky Note                             | Explanation of persona description | —                      | —                             | Persona description creation                                                                                     |
| Generate Prompts for Images | LangChain Agent                     | Creates image generation prompts   | Personas descriptions   | Split Prompts                 | Create a prompt for generating an image for each persona                                                        |
| Parse Prompts into JSON Array | LangChain Output Parser            | Validates and parses image prompts | Generate Prompts for Images | Generate Prompts for Images    |                                                                                                                  |
| Sticky Note3            | Sticky Note                             | Explanation of image prompt creation | —                      | —                             | Create a prompt for generating an image for each persona                                                        |
| Split Prompts           | Split Out                               | Splits prompt array for images     | Generate Prompts for Images | Generate images                |                                                                                                                  |
| Generate images         | OpenAI Image Generation                 | Generates persona images           | Split Prompts           | Upload images                 | Generate images and upload to Google Drive                                                                       |
| Upload images           | Google Drive                           | Uploads images to Google Drive     | Generate images         | —                             | Generate images and upload to Google Drive                                                                       |
| Sticky Note4            | Sticky Note                             | Explanation of image generation and upload | —                      | —                             | Generate images and upload to Google Drive                                                                       |
| Update a document1      | Google Docs                            | Inserts persona descriptions into Google Doc | Personas descriptions   | —                             |                                                                                                                  |
| Sticky Note1            | Sticky Note                             | Overview and instructions for entire workflow | —                      | —                             | ***Automated Data-Driven UX Persona Creation – Try It Out!*** ... [Detailed workflow description and requirements] |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node:**  
   - Type: Form Trigger  
   - Name: "On form submission1"  
   - Configure form: Title "Creating Personas"  
   - Add fields:  
     - Website url (required, placeholder "Enter website url")  
     - Region (optional)  
     - Industry (optional)  
   - Set form description to explain persona creation purpose.

2. **Add Perplexity Node:**  
   - Type: Perplexity  
   - Name: "🔍 Research"  
   - Model: "sonar-deep-research"  
   - Message content: build a prompt incorporating form inputs (website URL, region, industry) requesting detailed market research and segmentation with citations.  
   - Connect "On form submission1" output to this node input.  
   - Set Perplexity API credentials.

3. **Add OpenAI Chat Model Node:**  
   - Type: LangChain OpenAI Chat Model  
   - Name: "OpenAI Chat Model2"  
   - Model: "gpt-4.1-mini" (or latest GPT-4 variant)  
   - Input: research output from Perplexity node  
   - Set OpenAI API credentials.

4. **Add LangChain Agent Node for Persona Descriptions:**  
   - Type: LangChain Agent  
   - Name: "Personas descriptions"  
   - System prompt: instruct detailed persona formatting in Markdown style with sections: User Segment, Market Size, User Needs, Pain Points, Usage Story, Reference links.  
   - Input: output from OpenAI Chat Model2  
   - Output connections: to Google Docs update and image prompt generation.

5. **Add Google Docs Node:**  
   - Type: Google Docs  
   - Name: "Update a document1"  
   - Operation: "update" (insert text)  
   - Input: formatted persona descriptions  
   - Configure Google Docs OAuth2 credentials and select target document.

6. **Add LangChain Agent Node for Image Prompt Generation:**  
   - Type: LangChain Agent  
   - Name: "Generate Prompts for Images"  
   - System prompt: generate a JSON array of image prompts based on Usage Story sections from persona descriptions; each prompt for a 400x400 px square image, no props/text, filename based on persona title.  
   - Input: persona descriptions output.  
   - Enable output parser.

7. **Add LangChain Output Parser Node:**  
   - Type: LangChain Output Parser Structured  
   - Name: "Parse Prompts into JSON Array"  
   - Configure JSON schema example matching expected array of prompt objects.  
   - Connect output of "Generate Prompts for Images" to this parser node.

8. **Add Split Out Node:**  
   - Type: Split Out  
   - Name: "Split Prompts"  
   - Field to split: "output" (the parsed JSON array)  
   - Connect parsed prompts output.

9. **Add OpenAI Image Generation Node:**  
   - Type: LangChain OpenAI Image node  
   - Name: "Generate images"  
   - Resource: image generation (DALL·E 3)  
   - Prompt: use the split prompt item’s "prompt" field  
   - Set OpenAI API credentials.

10. **Add Google Drive Upload Node:**  
    - Type: Google Drive  
    - Name: "Upload images"  
    - Configuration:  
      - Folder ID: set to desired Google Drive folder (e.g., "1kv3Yo3cKK6Vsd2sBQYUN2RsBL4E1KOW8")  
      - Filename: use timestamp or persona title for uniqueness  
    - Input data field: binary image data from "Generate images" node  
    - Set Google Drive OAuth2 credentials.

11. **Connect nodes following the order:**  
    - "On form submission1" → "🔍 Research" → "OpenAI Chat Model2" → "Personas descriptions"  
    - From "Personas descriptions" → "Update a document1" and "Generate Prompts for Images"  
    - "Generate Prompts for Images" → "Parse Prompts into JSON Array" → "Split Prompts" → "Generate images" → "Upload images"

12. **Add Sticky Notes (Optional):**  
    - Add informational sticky notes for each block to aid users:  
      - Overall workflow explanation  
      - Market research block  
      - Persona description block  
      - Image prompt generation block  
      - Image generation and upload block

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| ***Automated Data-Driven UX Persona Creation – Try It Out!***<br><br>About: Create personas based on website, region, and industry inputs using data from Perplexity research, AI persona generation, and image creation. Personas help define user targets, align teams, and inspire product ideas.<br><br>How It Works: Form triggers workflow → Perplexity research → AI formats personas → DALL·E 3 generates images → Google Doc updated with descriptions and images.<br><br>Requirements: Perplexity API, OpenAI API, Google Docs and Drive credentials.<br><br>Instructions: Import workflow, configure credentials, and run. Outputs in Google Docs and Drive. | Embedded Sticky Note at workflow start (Sticky Note1)                                                           |
| Use Markdown-style headings (e.g., `## Executive Summary`) in persona descriptions to ensure proper rendering in Google Docs.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Instruction in "Personas descriptions" LangChain Agent system prompt                                             |
| Persona images are 400x400 pixels, square, with no props or text, named using the persona title.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Instruction in "Generate Prompts for Images" LangChain Agent system prompt                                       |
| Google Drive upload folder ID is configurable; ensure the folder exists and OAuth credentials have access permissions.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Configuration note in the "Upload images" node                                                                  |
| Perplexity research prompt is highly structured to ensure data-driven and cited persona generation, including market size and user behavior analysis.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Prompt content in "🔍 Research" node                                                                             |

---

**Disclaimer:** The provided text is exclusively from an n8n automated workflow. It complies with all content policies and contains no illegal or protected elements. All data processed is legal and public.