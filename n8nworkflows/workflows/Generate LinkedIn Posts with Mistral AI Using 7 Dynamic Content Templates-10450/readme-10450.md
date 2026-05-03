Generate LinkedIn Posts with Mistral AI Using 7 Dynamic Content Templates

https://n8nworkflows.xyz/workflows/generate-linkedin-posts-with-mistral-ai-using-7-dynamic-content-templates-10450


# Generate LinkedIn Posts with Mistral AI Using 7 Dynamic Content Templates

### 1. Workflow Overview

This workflow automates the generation of polished LinkedIn posts using Mistral AI, leveraging seven dynamic content templates tailored to different post types. It is designed to support marketers, content creators, and AI/automation businesses targeting SMBs and non-technical founders, helping them create engaging, professional posts quickly and consistently.

The workflow is logically divided into these blocks:

- **1.1 Input Reception & User Interaction**  
  Handles chat triggers or external workflow executions that provide the post topic and template choice.

- **1.2 Template Routing**  
  Routes the workflow to one of seven post template prompt generators based on the user-selected template.

- **1.3 Post Template Prompt Construction**  
  Each template node builds a carefully structured prompt with examples and style rules for different LinkedIn post types (Educational, Promotional, Discussion, Case Study, News, Personal, General).

- **1.4 Post Generation & Quality Validation**  
  An AI Agent uses the selected prompt and topic to generate the LinkedIn post text via Mistral Cloud, performing quality checks and ensuring tone and format compliance.

- **1.5 Memory & Think Tools**  
  Implements short-term chat memory for contextual continuity and "Think" nodes for validating logic and structure before finalizing output.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & User Interaction

- **Overview:**  
  Accepts input either through a chat interface or from another workflow. The user is prompted to select a post template type and provide a topic, which initiates the post generation process.

- **Nodes Involved:**  
  - When chat message received  
  - LinkedIn Agent  
  - When Executed by Another Workflow

- **Node Details:**

  - **When chat message received**  
    - Type: Chat Trigger  
    - Role: Starts the workflow upon receiving a chat message.  
    - Configuration: Default webhook trigger to catch user chat inputs.  
    - Inputs: Incoming chat message.  
    - Outputs: Passes chat content to LinkedIn Agent.  
    - Edge cases: Webhook failures, invalid inputs.

  - **LinkedIn Agent**  
    - Type: Langchain AI Agent  
    - Role: Converses with the user, asks for template choice and topic, guides the workflow.  
    - Configuration: Uses system message defining role as LinkedIn content expert, offers seven templates, validates inputs, and directs the next steps.  
    - Inputs: Chat messages or template/topic from sub-workflows.  
    - Outputs: Sends template choice and topic to the Switch node or template tool.  
    - Edge cases: Misinterpretation of user input, conversation dead-ends.

  - **When Executed by Another Workflow**  
    - Type: Execute Workflow Trigger  
    - Role: Allows external workflows to invoke this workflow with provided "template" and "topic" inputs.  
    - Configuration: Expects numeric template ID and string topic.  
    - Inputs: External workflow trigger.  
    - Outputs: Passes data to Switch node.  
    - Edge cases: Missing or invalid inputs.

---

#### 2.2 Template Routing

- **Overview:**  
  Routes the incoming request to the correct LinkedIn post prompt generator node based on the selected template number.

- **Nodes Involved:**  
  - Switch between templates

- **Node Details:**

  - **Switch between templates**  
    - Type: Switch Node  
    - Role: Routes flow to one of seven template nodes (Knowledge & Educational, Promotion, Discussion & Engagement, Case Study & Testimonial, News based post, Personal, General).  
    - Configuration: Matches input "template" number 1–7 to corresponding output.  
    - Inputs: JSON with template number.  
    - Outputs: Connects to one of the seven "Set" nodes for prompt construction.  
    - Edge cases: Invalid template number, no matching route.

---

#### 2.3 Post Template Prompt Construction

- **Overview:**  
  Each node constructs a detailed JSON prompt tailored for a specific LinkedIn post style, including tone, format, structure, example hooks, insights, CTAs, and a sample post.

- **Nodes Involved:**  
  - Knowledge & Educational  
  - Promotion  
  - Discussion & Engagement  
  - Case Study & Testimonial  
  - News based post  
  - personal (Personal Growth & Milestone)  
  - general

- **Node Details (example for one node, applies similarly to others):**

  - **Knowledge & Educational**  
    - Type: Set Node  
    - Role: Defines prompt JSON for educational posts, emphasizing professional tone, knowledge sharing, and structured flow (hook, main body, insight, takeaway, brand mention, CTA).  
    - Configuration: Raw JSON with keys for prompt text, description, writing rules, example hooks, and quality checks.  
    - Inputs: Receives data from Switch node, including topic and summary.  
    - Outputs: Passes prompt data to post generator agent.  
    - Edge cases: JSON formatting errors, missing fields.

  - (Similar details apply to the other six template nodes, each with uniquely tailored prompts and instructions.)

---

#### 2.4 Post Generation & Quality Validation

- **Overview:**  
  Generates the final LinkedIn post text using the selected prompt and topic. It uses the Mistral Cloud AI model with an AI Agent that performs quality validations and ensures compliance with tone and style requirements.

- **Nodes Involved:**  
  - post generator  
  - Mistral Cloud Chat Model

- **Node Details:**

  - **Mistral Cloud Chat Model**  
    - Type: Langchain Mistral Cloud Language Model  
    - Role: Performs AI text generation based on the prompt supplied.  
    - Configuration: Uses model "mistral-small-latest" with Mistral Cloud credentials.  
    - Inputs: Prompt and context from the "post generator" agent node.  
    - Outputs: AI-generated LinkedIn post text.  
    - Edge cases: API authentication errors, timeouts, rate limits.

  - **post generator**  
    - Type: Langchain AI Agent  
    - Role: Receives structured prompt and topic, generates post text, performs logic validation with "think" tools, applies quality checklist, and calls Mistral model.  
    - Configuration: System message defines voice, tone, style, forbidden phrases, output format, and use of think tools for validation.  
    - Inputs: JSON prompt from selected template node.  
    - Outputs: Final LinkedIn post text.  
    - Edge cases: Validation failures, prompt errors, unexpected AI outputs.

---

#### 2.5 Memory & Think Tools

- **Overview:**  
  Provides short-term memory for context retention during chat sessions and uses think nodes to validate workflow logic and quality before output.

- **Nodes Involved:**  
  - Simple Memory1  
  - Think  
  - Think1

- **Node Details:**

  - **Simple Memory1**  
    - Type: Memory Buffer Window  
    - Role: Stores last 10 chat messages to maintain conversation context.  
    - Inputs: Chat messages from user and agent.  
    - Outputs: Context to LinkedIn Agent.  
    - Edge cases: Memory overflow, context loss.

  - **Think** and **Think1**  
    - Type: Tool Think Nodes  
    - Role: Perform internal reasoning/validation steps to ensure quality and logic integrity before final output.  
    - Inputs: Intermediate data from LinkedIn Agent and post generator.  
    - Outputs: Validation results to guide generation.  
    - Edge cases: Logic or expression errors.

---

### 3. Summary Table

| Node Name                  | Node Type                                | Functional Role                          | Input Node(s)                     | Output Node(s)                      | Sticky Note                                                                                          |
|----------------------------|----------------------------------------|----------------------------------------|----------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------------|
| When chat message received  | Chat Trigger                           | Starts workflow on chat message        | -                                | LinkedIn Agent                    | ## Chat & Trigger: Starts the flow when a chat message arrives or another workflow executes it.    |
| LinkedIn Agent             | Langchain AI Agent                     | User conversation, template selection  | When chat message received, Mistral Cloud Chat Model1, Simple Memory1 | Switch between templates, think      | ## LinkedIn Agent: Handles conversation with the user and decides which template to use.           |
| When Executed by Another Workflow | Execute Workflow Trigger            | Triggered externally with inputs       | -                                | Switch between templates          | ## When Executed by Another Workflow: Allows other workflows to call this one with custom inputs.  |
| Switch between templates    | Switch Node                           | Routes to selected post template       | When Executed by Another Workflow | Knowledge & Educational, Promotion, Discussion & Engagement, Case Study & Testimonial, News based post, personal, general | ## Template Switch: Routes the workflow to one of seven post templates based on user choice.        |
| Knowledge & Educational     | Set Node                             | Builds educational post prompt         | Switch between templates          | post generator                   | ## Post Templates: Each “Set” node stores a prompt for a specific type of post.                     |
| Promotion                  | Set Node                             | Builds promotional post prompt          | Switch between templates          | post generator                   | ## Post Templates                                                                                   |
| Discussion & Engagement    | Set Node                             | Builds discussion post prompt           | Switch between templates          | post generator                   | ## Post Templates                                                                                   |
| Case Study & Testimonial   | Set Node                             | Builds case study/testimonial prompt    | Switch between templates          | post generator                   | ## Post Templates                                                                                   |
| News based post            | Set Node                             | Builds news post prompt                  | Switch between templates          | post generator                   | ## Post Templates                                                                                   |
| personal                   | Set Node                             | Builds personal growth post prompt      | Switch between templates          | post generator                   | ## Post Templates                                                                                   |
| general                    | Set Node                             | Builds general post prompt               | Switch between templates          | post generator                   | ## Post Templates                                                                                   |
| post generator             | Langchain AI Agent                   | Generates final post text with quality checks | Knowledge & Educational, Promotion, Discussion & Engagement, Case Study & Testimonial, News based post, personal, general, Mistral Cloud Chat Model | -                               | ## Post Generator: Uses the selected template + topic to craft the final LinkedIn post.             |
| Mistral Cloud Chat Model   | Langchain Language Model (Mistral)  | AI text generation                       | post generator                   | post generator                   | ## Mistral Model + Think Tools: Mistral handles text generation.                                   |
| Mistral Cloud Chat Model1  | Langchain Language Model (Mistral)  | AI text generation for LinkedIn Agent   | LinkedIn Agent                  | LinkedIn Agent                  | ## Mistral Model + Think Tools                                                                     |
| Simple Memory1             | Memory Buffer Window                 | Stores chat context                      | LinkedIn Agent                   | LinkedIn Agent                   | ## Simple Memory: Stores short chat context so the Agent remembers recent messages.                |
| Think                      | Tool Think                         | Validates logic and structure           | LinkedIn Agent                  | LinkedIn Agent                  | ## Mistral Model + Think Tools                                                                     |
| Think1                     | Tool Think                         | Validates logic and structure           | post generator                  | post generator                  | ## Mistral Model + Think Tools                                                                     |
| Sticky Note                | Sticky Note                        | Comment on LinkedIn Agent                | -                                | -                               | ## LinkedIn Agent: Handles conversation with the user and decides which template to use.           |
| Sticky Note1               | Sticky Note                        | Comment on template tool (post generator) | -                                | -                               | ## template tool (post generator)                                                                  |
| Sticky Note3               | Sticky Note                        | Comment on external workflow trigger    | -                                | -                               | ## When Executed by Another Workflow: Allows other workflows to call this one with custom inputs.  |
| Sticky Note4               | Sticky Note                        | Comment on Switch node                    | -                                | -                               | ## Template Switch: Routes the workflow to one of seven post templates based on user choice.       |
| Sticky Note5               | Sticky Note                        | Comment on post template "Set" nodes    | -                                | -                               | ## Post Templates: Each “Set” node stores a prompt for a specific type of post.                     |
| Sticky Note6               | Sticky Note                        | Comment on post generator and Mistral model | -                                | -                               | ## Post Generator: Uses the selected template + topic to craft the final LinkedIn post.            |
| Sticky Note7               | Sticky Note                        | General workflow overview and setup     | -                                | -                               | ## How it works: Explains workflow summary, setup, and usage tips.                                 |
| Sticky Note8               | Sticky Note                        | Comment on chat & trigger nodes          | -                                | -                               | ## Chat & Trigger: Starts the flow when a chat message arrives or another workflow executes it.    |
| Sticky Note10              | Sticky Note                        | Comment on Simple Memory node            | -                                | -                               | ## Simple Memory: Stores short chat context so the Agent remembers recent messages.                |
| Sticky Note11              | Sticky Note                        | Comment on Mistral Model + Think tools  | -                                | -                               | ## Mistral Model + Think Tools: Mistral handles text generation.                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Credential for Mistral Cloud API:**  
   - Add Mistral Cloud account credentials in n8n credentials manager (API key).

2. **Add Trigger Nodes:**  
   - Add a **Chat Trigger** node named "When chat message received" to start on user chat.  
   - Add an **Execute Workflow Trigger** node named "When Executed by Another Workflow" to accept external calls with inputs "template" (number) and "topic" (string).

3. **Add LinkedIn Agent Node:**  
   - Add a **Langchain AI Agent** node "LinkedIn Agent".  
   - Configure system message to define role as LinkedIn content expert who converses with user to select post template and topic.  
   - Connect "When chat message received" and "Mistral Cloud Chat Model1" outputs to this node.  
   - Connect **Simple Memory** node to provide chat context to this agent.

4. **Add Simple Memory Node:**  
   - Add **Memory Buffer Window** node "Simple Memory1" with context window length 10.  
   - Connect output to "LinkedIn Agent" AI memory.

5. **Add Mistral Language Model Node for LinkedIn Agent:**  
   - Add **Langchain Mistral Cloud Chat Model** node "Mistral Cloud Chat Model1".  
   - Set model to "mistral-small-latest" and assign Mistral Cloud credentials.  
   - Connect output to "LinkedIn Agent" AI language model input.

6. **Add Switch Node for Template Routing:**  
   - Add **Switch** node "Switch between templates".  
   - Configure seven rules matching "template" field values 1 to 7.  
   - Connect output of "When Executed by Another Workflow" to this switch node.

7. **Create Seven Post Template Nodes:**  
   - Add seven **Set** nodes named:  
     - "Knowledge & Educational"  
     - "Promotion"  
     - "Discussion & Engagement"  
     - "Case Study & Testimonial"  
     - "news based post"  
     - "personal"  
     - "general"  
   - Each node contains a raw JSON object defining prompt structure, examples, tone, rules, and quality checks tailored to the post type.  
   - Connect each switch output to the corresponding Set node.

8. **Add Post Generator Agent Node:**  
   - Add **Langchain AI Agent** node "post generator".  
   - Configure system message with instructions to generate LinkedIn posts using the structured prompt and topic.  
   - Include detailed tone, style, forbidden phrases, quality checklist, and output format.  
   - Connect all seven Set nodes outputs to this node.

9. **Add Mistral Language Model Node for Post Generator:**  
   - Add **Langchain Mistral Cloud Chat Model** node "Mistral Cloud Chat Model" with same model and credentials as above.  
   - Connect "post generator" AI language model input to this node output.

10. **Add Think Nodes for Validation:**  
    - Add two **Tool Think** nodes named "Think" and "Think1".  
    - Connect "LinkedIn Agent" AI tool output to "Think".  
    - Connect "post generator" AI tool output to "Think1".  
    - Connect validation outputs back to their respective agents for logic checks.

11. **Connect Outputs:**  
    - Connect "Think" node output to "LinkedIn Agent" to aid conversation flow.  
    - Connect "Think1" node output to "post generator" for final output validation.  
    - Connect final "post generator" main output to the workflow output.

12. **Add Sticky Notes (optional):**  
    - Add sticky notes at key points for documentation, e.g., above LinkedIn Agent, Switch node, template nodes, and post generator to explain purpose.

13. **Test Workflow:**  
    - Trigger chat or execute via external workflow with JSON inputs for "template" and "topic".  
    - Confirm prompt routing, AI generation, quality validation, and final post output.

---

### 5. General Notes & Resources

| Note Content                                                                                                                              | Context or Link                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| The workflow leverages Mistral AI via the n8n Langchain integration for state-of-the-art text generation tailored to LinkedIn posts.     | Mistral Cloud API credentials required, model: mistral-small-latest                                   |
| Templates are fully customizable by editing the JSON in each Set node, allowing tone, structure, and examples to be adapted as needed.   | Templates include Educational, Promotional, Discussion, Case Study, News, Personal, and General posts |
| The LinkedIn Agent manages conversational logic, guiding users through template selection and topic input, enhancing UX with memory.     | Uses Simple Memory Buffer Window node for context retention                                           |
| The post generator AI agent includes a strong quality checklist to eliminate fluff, hype, and marketing jargon, focusing on authenticity.| Ensures posts are engaging, professional, and accessible to non-technical audiences                   |
| The workflow supports triggering from other workflows, enabling integration into broader automation pipelines or custom user interfaces.| External workflows can provide template and topic, triggering post generation automatically           |
| For details on n8n Langchain nodes and usage, see: https://docs.n8n.io/integrations/builtin/n8n-nodes-langchain/                          | Official n8n documentation on Langchain nodes                                                         |
| Branding and style adhere to Sastha AI’s messaging, focusing on AI and automation for SMBs and non-technical founders.                   | Internal project branding and messaging guidelines                                                    |

---

**Disclaimer:**  
This document is based exclusively on an automated workflow designed with n8n. All data used and processed comply with current content policies and legal standards.