Create Research-Based LinkedIn Posts with OpenAI, Perplexity and Human Review

https://n8nworkflows.xyz/workflows/create-research-based-linkedin-posts-with-openai--perplexity-and-human-review-6030


# Create Research-Based LinkedIn Posts with OpenAI, Perplexity and Human Review

---

### 1. Workflow Overview

This workflow automates the creation of research-based LinkedIn posts, combining AI-powered research, content generation, human review, and image creation to produce polished, authentic posts ready for publishing. It targets professionals, content creators, and consultants who want to leverage AI insights while maintaining personal voice and manual quality control.

The workflow is logically divided into these functional blocks:

- **1.1 Trigger & Research Initiation:** Starts the workflow on schedule or manual trigger and queries Perplexity API for recent, factual insights on a chosen topic.
- **1.2 Human Topic Selection:** Sends an email listing top 3 researched topics for human selection via Gmail interactive form.
- **1.3 AI Content Drafting:** Generates a first draft LinkedIn post based on the selected topic and Perplexity insights using OpenAI.
- **1.4 Content Review & Refinement:** Sends draft to the user for approval or suggestions, routes accordingly, and optionally revises the post using OpenAI.
- **1.5 Image Prompt Generation & Creation:** Produces an abstract, conceptual infographic prompt from the post content and generates a matching image via OpenAI's DALL·E.
- **1.6 Final Delivery:** Emails the finalized LinkedIn post and the generated image to the user.

Each block involves multiple nodes orchestrated to ensure smooth data flow, error handling, and user interaction.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Research Initiation

**Overview:**  
This block triggers the workflow either on a weekly schedule or manually and initiates topic research by querying the Perplexity API for recent verified information on AI-related trends and use cases.

**Nodes Involved:**  
- Schedule Trigger  
- Click to start (Manual Trigger)  
- 🔍 Research the Trends (Perplexity API)  
- Sticky Note3 (Contextual instructions)

**Node Details:**  

- **Schedule Trigger**  
  - Type: Scheduled trigger  
  - Configured to trigger weekly on Monday  
  - Inputs: None  
  - Outputs: Triggers research node  
  - Edge cases: Potential trigger misconfiguration or delayed execution  
  - Version: 1.2

- **Click to start**  
  - Type: Manual trigger  
  - Allows user-initiated workflow start  
  - Inputs: None  
  - Outputs: Triggers research node  
  - Edge cases: None significant  
  - Version: 1

- **🔍 Research the Trends**  
  - Type: Perplexity API node  
  - Role: Queries Perplexity with a detailed prompt requesting 3 key verified insights on the given topic, focusing on recent news, business use cases, expert opinions, and source citations.  
  - Inputs: Trigger node output; expects "Topic" and "Additional_Context" inputs in JSON body for prompt personalization.  
  - Outputs: JSON with 3 insights, source URLs, dates, and citations.  
  - Credentials: Requires Perplexity API Key  
  - Edge cases: API errors, rate limits, or no verifiable data found (explicitly handled in prompt)  
  - Version: 1

- **Sticky Note3**  
  - Provides explanation of the block's purpose and setup instructions (Perplexity API key, prompt customization)

---

#### 1.2 Human Topic Selection

**Overview:**  
After gathering 3 topic ideas from Perplexity, this block emails the list to the user and collects their choice via an interactive Gmail form, enabling human-in-the-loop topic selection.

**Nodes Involved:**  
- Select the Best Topic (Gmail node)  
- Sticky Note1 (Contextual instructions)

**Node Details:**  

- **Select the Best Topic**  
  - Type: Gmail node with sendAndWait operation  
  - Sends email listing 3 topic titles extracted from Perplexity's output  
  - Includes a dropdown form field for user to select which topic to pursue (options labeled 1, 2, or 3 with topic titles)  
  - Inputs: JSON from Perplexity node containing “search_results” array  
  - Outputs: User's selected topic choice captured for next nodes  
  - Credentials: Requires Gmail OAuth2 account  
  - Edge cases: Email delivery failure, user non-response, invalid input handling  
  - Version: 2.1

- **Sticky Note1**  
  - Explains purpose of Gmail for topic selection and setup instructions (Gmail account, recipient email, customization tips)

---

#### 1.3 AI Content Drafting

**Overview:**  
This block uses OpenAI's GPT-4o to draft a medium-length LinkedIn post personalized with the selected topic, combining Alberto Bordoni’s persona, research insights, and desired tone.

**Nodes Involved:**  
- ✍️ Content Creator (OpenAI)  
- Sticky Note4 (Contextual instructions)

**Node Details:**  

- **✍️ Content Creator**  
  - Type: OpenAI Chat Completion node (LangChain integration)  
  - Model: chatgpt-4o-latest  
  - Prompt: Detailed system and user prompt embedding Alberto’s persona, selected topic from Gmail, research insights from Perplexity, and output language (default Italian)  
  - Outputs: Draft LinkedIn post content in JSON  
  - Credentials: OpenAI API key  
  - Edge cases: API errors, prompt failures, rate limits  
  - Version: 1.8

- **Sticky Note4**  
  - Explains this is a core content generation node with setup and personalization guidance, including API key needs and prompt customization

---

#### 1.4 Content Review & Refinement

**Overview:**  
This block sends the draft post via Gmail for user review, routes the workflow based on approval response, and optionally generates a revised post with OpenAI if changes are requested.

**Nodes Involved:**  
- Content Aggregator (Set node)  
- Content Review & Approval (Gmail node)  
- IF – Content Approval Routing (If node)  
- ✍️ Content Reviewer (OpenAI)  
- Sticky Note5, Sticky Note6, Sticky Note7, Sticky Note10 (Contextual instructions)

**Node Details:**  

- **Content Aggregator**  
  - Type: Set node  
  - Role: Consolidates content from the AI draft or reviewer to a unified JSON property for downstream use  
  - Inputs: Draft or reviewed content  
  - Outputs: Sets “message.content” field  
  - Edge cases: Mapping errors if input missing or malformed  
  - Version: 3.4

- **Content Review & Approval**  
  - Type: Gmail node with sendAndWait operation  
  - Sends draft post content to user email for final review  
  - Includes a form with dropdown “Do you like the content?” (Yes/To review) and optional text input for suggestions  
  - Captures user response to route workflow  
  - Credentials: Gmail OAuth2  
  - Edge cases: Email delivery failure, no response, ambiguous replies  
  - Version: 2.1

- **IF – Content Approval Routing**  
  - Type: If node  
  - Checks if user replied with “Yes” (case sensitive) to proceed to image generation  
  - Otherwise, routes to content revision node  
  - Edge cases: Reply parsing errors, alternative affirmative inputs not handled by default  
  - Version: 2.2

- **✍️ Content Reviewer**  
  - Type: OpenAI Chat Completion (LangChain integration)  
  - Model: chatgpt-4o-latest  
  - Prompt: Combines original draft and user feedback to generate revised post maintaining original tone and style  
  - Outputs: Revised content JSON  
  - Credentials: OpenAI API key  
  - Edge cases: API failures, prompt misinterpretation, rate limiting  
  - Version: 1.8

- **Sticky Notes 5, 6, 7, 10**  
  - Provide detailed explanations on the content aggregation, review email, approval routing logic, and reviewer node usage/setup

---

#### 1.5 Image Prompt Generation & Creation

**Overview:**  
Generates a vector-style abstract infographic prompt based on the finalized LinkedIn post content and uses OpenAI’s DALL·E model to create a visual asset supporting the post.

**Nodes Involved:**  
- 🖼️ Image Prompt Generator (OpenAI)  
- Code (JavaScript)  
- Generate an image (OpenAI Image Generation)  
- Sticky Note11, Sticky Note12, Sticky Note13 (Contextual instructions)

**Node Details:**  

- **🖼️ Image Prompt Generator**  
  - Type: OpenAI Chat Completion (LangChain integration)  
  - Model: chatgpt-4o-latest  
  - Prompt: Detailed instructions to generate an image prompt for a conceptual infographic illustrating the LinkedIn post’s core message in an abstract, readable style  
  - Inputs: Final post content from Content Aggregator  
  - Outputs: Text prompt for image generation  
  - Credentials: OpenAI API key  
  - Edge cases: Prompt generation errors, ambiguous output  
  - Version: 1.8

- **Code**  
  - Type: Code node (JavaScript)  
  - Cleans the image prompt by escaping quotes, removing line breaks, and trimming whitespace for safe API consumption  
  - Inputs: Raw image prompt text  
  - Outputs: Cleaned prompt string as “clean_prompt”  
  - Edge cases: Unexpected characters or formatting issues in prompt text  
  - Version: 2

- **Generate an image**  
  - Type: OpenAI Image Generation node  
  - Model: gpt-image-1 (DALL·E 3 or equivalent)  
  - Uses cleaned prompt to generate a PNG conceptual infographic image  
  - Credentials: OpenAI API key (with image generation rights)  
  - Edge cases: API quota limits, generation failures  
  - Version: 1.8

- **Sticky Notes 11, 12, 13**  
  - Explain the image prompt generation concept, code cleaning step, and image generation setup, including API key and model requirements

---

#### 1.6 Final Delivery

**Overview:**  
Compiles the finalized LinkedIn post and generated image, and sends them via email to the user for easy copy-paste publishing on LinkedIn.

**Nodes Involved:**  
- Final Content Delivery (Gmail)  
- Sticky Note14 (Contextual instructions)

**Node Details:**  

- **Final Content Delivery**  
  - Type: Gmail node  
  - Sends an email with the final LinkedIn post text and the generated image attachment or URL  
  - Inputs: Content Aggregator content and image binary data from image generation  
  - Credentials: Gmail OAuth2  
  - Edge cases: Email sending failures, attachment size or format issues  
  - Version: 2.1

- **Sticky Note14**  
  - Describes final delivery node purpose and setup instructions (recipient address, mapping image correctly, personalization options)

---

### 3. Summary Table

| Node Name                   | Node Type                         | Functional Role                        | Input Node(s)               | Output Node(s)                    | Sticky Note                                                                                   |
|-----------------------------|----------------------------------|-------------------------------------|-----------------------------|----------------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger            | scheduleTrigger                  | Automatic weekly workflow start      | None                        | 🔍 Research the Trends            | 🕒 WORKFLOW STARTER: Setup preferred trigger interval, automated or manual                    |
| Click to start              | manualTrigger                   | Manual workflow start                | None                        | 🔍 Research the Trends            | 🕒 WORKFLOW STARTER: Alternative manual trigger for testing or ad hoc runs                    |
| 🔍 Research the Trends       | perplexity                      | Research recent AI trends & use cases| Schedule Trigger, Click to start | Select the Best Topic           | 🧠 Perplexity Research – Trends & Use Cases: Setup Perplexity API key, customize prompt      |
| Select the Best Topic       | gmail                          | Email 3 topics and collect user choice| 🔍 Research the Trends      | ✍️ Content Creator               | 📨 Gmail – Select the Best Topic (Human-in-the-loop): Setup Gmail account and recipient       |
| ✍️ Content Creator           | openAi (LangChain Chat Completion) | Generate LinkedIn post draft         | Select the Best Topic        | Content Aggregator               | 🧠 OpenAI – Content Creation Support: Setup OpenAI API, personalize prompt and tone          |
| Content Aggregator          | set                            | Consolidate content for downstream use| ✍️ Content Creator, ✍️ Content Reviewer | Content Review & Approval      | 📝 Edit Fields – Content Aggregator & Finalizer: Strategic node for merging content versions  |
| Content Review & Approval   | gmail                          | Send draft for approval, collect feedback| Content Aggregator          | IF – Content Approval Routing    | ✅ Gmail – Content Review & Approval: Setup Gmail, personalize subject/message               |
| IF – Content Approval Routing| if                             | Route workflow based on approval     | Content Review & Approval    | 🖼️ Image Prompt Generator, ✍️ Content Reviewer | 🔀 IF – Content Approval Routing: Decision node based on user reply                           |
| ✍️ Content Reviewer          | openAi (LangChain Chat Completion) | Generate revised post based on feedback| IF – Content Approval Routing (No branch) | Content Aggregator           | 🧐 OpenAI – Content Reviewer: Setup OpenAI API, customize prompt for revision                |
| 🖼️ Image Prompt Generator    | openAi (LangChain Chat Completion) | Generate image prompt text            | IF – Content Approval Routing (Yes branch) | Code                       | 🖼️ OpenAI – Image Prompt Generator: Setup OpenAI API, customize image prompt                 |
| Code                        | code                           | Clean image prompt for API use       | 🖼️ Image Prompt Generator   | Generate an image               | 🧹 Code – Clean Image Prompt for API: Escapes quotes, removes line breaks                    |
| Generate an image           | openAi Image                   | Create conceptual infographic image  | Code                        | Final Content Delivery          | 🧠 OpenAI – Generate Image (DALL·E 3): Setup API key with image generation rights             |
| Final Content Delivery      | gmail                          | Send final LinkedIn post + image     | Generate an image           | None                           | 📬 Gmail – Final Delivery of Your LinkedIn Post: Setup Gmail, map image & content correctly   |
| Sticky Note(s)              | stickyNote                     | Documentation and instructions       | None                        | None                           | Various notes linked contextually to relevant nodes                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Set to trigger weekly on Monday (or desired interval).  
   - Name: "Schedule Trigger".

2. **Create a Manual Trigger node:**  
   - For manual workflow starts.  
   - Name: "Click to start".

3. **Create a Perplexity node:**  
   - Use the Perplexity API node with model "sonar".  
   - Configure prompt to research a topic (parameterize input for "Topic" and "Additional_Context").  
   - Set API key credentials.  
   - Name: "🔍 Research the Trends".

4. **Connect both triggers to the Perplexity node:**  
   - Schedule Trigger → 🔍 Research the Trends  
   - Click to start → 🔍 Research the Trends

5. **Create a Gmail node for topic selection:**  
   - Operation: sendAndWait  
   - Recipient: your email  
   - Message: List 3 topics from Perplexity results with a dropdown form to choose one (options labeled 1, 2, or 3).  
   - Set Gmail OAuth2 credentials.  
   - Name: "Select the Best Topic".

6. **Connect Perplexity node to Gmail node:**  
   - 🔍 Research the Trends → Select the Best Topic

7. **Create OpenAI Chat Completion node for content creation:**  
   - Model: chatgpt-4o-latest  
   - Prompt: Embed persona, topic from Gmail node, Perplexity insights, and language choice (default Italian).  
   - Set OpenAI API credentials.  
   - Name: "✍️ Content Creator".

8. **Connect Gmail topic selection node to content creator:**  
   - Select the Best Topic → ✍️ Content Creator

9. **Create a Set node to aggregate content:**  
   - Assign "message.content" to the generated content field from ✍️ Content Creator.  
   - Name: "Content Aggregator".

10. **Connect Content Creator to Content Aggregator:**  
    - ✍️ Content Creator → Content Aggregator

11. **Create Gmail node for content review:**  
    - Operation: sendAndWait  
    - Send draft post for manual approval with a dropdown: "Do you like the content?" (Yes/To review) and a text field for suggestions.  
    - Set Gmail OAuth2 credentials.  
    - Name: "Content Review & Approval".

12. **Connect Content Aggregator to Content Review & Approval:**  
    - Content Aggregator → Content Review & Approval

13. **Create an If node to route based on approval:**  
    - Condition: Check if answer equals "Yes" (case sensitivity as needed).  
    - Name: "IF – Content Approval Routing".

14. **Connect Content Review & Approval to If node:**  
    - Content Review & Approval → IF – Content Approval Routing

15. **Create OpenAI Chat Completion node for content reviewer:**  
    - Model: chatgpt-4o-latest  
    - Prompt: Combine original draft + user feedback to generate revised post maintaining tone.  
    - Set OpenAI API credentials.  
    - Name: "✍️ Content Reviewer".

16. **Connect If node branches:**  
    - Yes branch → Image Prompt Generator (next step)  
    - No branch → ✍️ Content Reviewer

17. **Connect Content Reviewer to Content Aggregator:**  
    - ✍️ Content Reviewer → Content Aggregator (to update content)

18. **Create OpenAI Chat Completion node for image prompt generation:**  
    - Model: chatgpt-4o-latest  
    - Prompt: Generate a conceptual infographic prompt based on final post content.  
    - Set OpenAI API credentials.  
    - Name: "🖼️ Image Prompt Generator".

19. **Connect If node Yes branch to Image Prompt Generator:**  
    - IF – Content Approval Routing (Yes) → 🖼️ Image Prompt Generator

20. **Create Code node to clean image prompt:**  
    - JavaScript to escape quotes, remove line breaks, and trim whitespace.  
    - Name: "Code".

21. **Connect Image Prompt Generator to Code node:**  
    - 🖼️ Image Prompt Generator → Code

22. **Create OpenAI Image Generation node:**  
    - Model: gpt-image-1 (DALL·E 3 or equivalent)  
    - Use cleaned image prompt as input.  
    - Set OpenAI API credentials with image generation enabled.  
    - Name: "Generate an image".

23. **Connect Code to Generate an image:**  
    - Code → Generate an image

24. **Create Gmail node for final delivery:**  
    - Send email with finalized LinkedIn post text and attached/generated image.  
    - Set Gmail OAuth2 credentials.  
    - Name: "Final Content Delivery".

25. **Connect Generate an image node to final delivery:**  
    - Generate an image → Final Content Delivery

26. **Ensure all credentials are configured:**  
    - OpenAI API key with chat and image generation enabled  
    - Perplexity API key  
    - Gmail OAuth2 account with send and receive permissions

27. **Optional:** Add sticky notes for documentation, color-coded for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| 🧠 Perplexity chosen for research due to source-driven, factual responses ideal for verified content creation   | Sticky Note3                                                                                     |
| 📨 Gmail used for reliable, accessible human-in-the-loop topic and content review; alternative channels possible | Sticky Note1, Sticky Note6                                                                       |
| 🧠 OpenAI GPT-4o model selected for balanced quality and cost efficiency in content generation and revision    | Sticky Note4, Sticky Note10                                                                      |
| 🖼️ Image prompt designed for abstract conceptual infographic style, optimized for LinkedIn feed visual impact | Sticky Note11                                                                                   |
| 🧹 Code node sanitizes image prompt to ensure API compatibility and avoid errors                               | Sticky Note12                                                                                   |
| 🧠 OpenAI DALL·E 3 image generation requires image-capable OpenAI API key                                     | Sticky Note13                                                                                   |
| 📬 Final email consolidates text and image for easy copy-paste publishing                                     | Sticky Note14                                                                                   |
| 🗂️ Color-coded sticky notes provide visual documentation aid: Blue=Info, Green=Triggers, Yellow=Processing, Red=API calls | Sticky Note2                                                                                    |
| 🚀 Workflow ideal for professionals/creators balancing AI-powered research with personal storytelling          | Sticky Note8                                                                                   |
| 📋 Workflow process overview provided for clarity on step sequencing                                          | Sticky Note9                                                                                   |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---