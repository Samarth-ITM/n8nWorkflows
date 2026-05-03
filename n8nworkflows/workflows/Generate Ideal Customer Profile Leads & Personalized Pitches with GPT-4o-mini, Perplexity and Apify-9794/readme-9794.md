Generate Ideal Customer Profile Leads & Personalized Pitches with GPT-4o-mini, Perplexity and Apify

https://n8nworkflows.xyz/workflows/generate-ideal-customer-profile-leads---personalized-pitches-with-gpt-4o-mini--perplexity-and-apify-9794


# Generate Ideal Customer Profile Leads & Personalized Pitches with GPT-4o-mini, Perplexity and Apify

### 1. Workflow Overview

This workflow automates the generation of Ideal Customer Profile (ICP) leads and personalized sales pitches for businesses using a combination of AI models and external APIs. It is designed to intake business information through a JotForm form, deeply analyze the company, define the best-fit customer profile, identify the relevant industry, find local business leads matching the ICP, and finally create tailored email pitches for outreach.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures business data submitted via JotForm.
- **1.2 Company Analysis & ICP Generation:** Uses Perplexity AI and GPT-4o-mini to analyze the company and generate an Ideal Customer Profile.
- **1.3 ICP Industry Identification:** Determines the most appropriate industry sector for the ICP.
- **1.4 Lead Generation:** Fetches local business leads matching the ICP and industry.
- **1.5 Personalized Pitch Creation:** Crafts customized email pitches for each lead found.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Receives business information via a JotForm submission, serving as the workflow entry point.

**Nodes Involved:**  
- JotForm Trigger

**Node Details:**  

- **JotForm Trigger**  
  - Type: Trigger node for JotForm submissions  
  - Configuration: Listens for submissions on form ID `252802732808054`  
  - Credentials: Uses JotForm API credentials (named "JotForm account 2")  
  - Inputs: None (triggered by external event)  
  - Outputs: Business data JSON object containing at least a `website` field  
  - Edge Cases: Potential form API authentication errors, webhook misconfigurations, or missing form data  
  - Notes: Sticky note instructs users to fill the form about the business for lead generation  

#### 1.2 Company Analysis & ICP Generation

**Overview:**  
Analyzes the submitted company in detail using Perplexity AI, then uses GPT-4o-mini (via n8n’s LangChain OpenAI node) to generate the Ideal Customer Profile based on the analysis.

**Nodes Involved:**  
- perplexity (HTTP Request)  
- ICP finder (OpenAI LangChain node)  
- Edit Fields (Set node)

**Node Details:**  

- **perplexity**  
  - Type: HTTP Request  
  - Configuration: POST request to Perplexity AI search API with the company’s website URL as the query  
  - Headers: Authorization header required (value not shown)  
  - Inputs: Business data from JotForm Trigger  
  - Outputs: JSON with analytical snippets about the company  
  - Edge Cases: API auth failure, rate limits, malformed body, or no relevant results  
  - Notes: Sticky note emphasizes detailed company analysis  

- **ICP finder**  
  - Type: OpenAI GPT-4o-mini via LangChain node  
  - Configuration: System prompt defines role as expert client acquisition manager; user prompt requests best ICP based on Perplexity snippet  
  - Inputs: Output from Perplexity node, specifically `results[0].snippet`  
  - Outputs: Text content with ICP description  
  - Credentials: OpenAI API with GPT-4o-mini model  
  - Edge Cases: OpenAI API errors, incomplete or ambiguous outputs, token limits  
  - Version: Node version 1.8  

- **Edit Fields**  
  - Type: Set node  
  - Configuration: Assigns a new field `ICP` with the content from `ICP finder` output’s `message.content`  
  - Inputs: Output from ICP finder  
  - Outputs: JSON enriched with ICP string for downstream use  
  - Edge Cases: Expression failures if ICP content missing or malformed  
  - Version: Node version 3.4  
  - Notes: Sticky note indicates this block creates ICP suited for the company  

#### 1.3 ICP Industry Identification

**Overview:**  
Determines the specific industry sector best aligned with the ICP, filtering out irrelevant or broad sectors.

**Nodes Involved:**  
- ICP industry finder (OpenAI LangChain node)

**Node Details:**  

- **ICP industry finder**  
  - Type: OpenAI GPT-4o-mini via LangChain node  
  - Configuration: System prompt sets role as ICP analyzer, restricting output to a single professional industry term excluding SaaS and broad categories; user prompt includes ICP text  
  - Inputs: JSON with ICP text field from Edit Fields node  
  - Outputs: Single-sector industry string  
  - Credentials: OpenAI API with GPT-4o-mini model  
  - Edge Cases: Ambiguous or multiple outputs, empty responses, token limits  
  - Version: Node version 1.8  
  - Notes: Sticky note clarifies the requirement for a single, clean industry term  

#### 1.4 Lead Generation

**Overview:**  
Uses the identified ICP industry and customer profile to find local business leads via an Apify actor.

**Nodes Involved:**  
- Leads (HTTP Request)  
- Loop Over Items (SplitInBatches)

**Node Details:**  

- **Leads**  
  - Type: HTTP Request  
  - Configuration: POST to Apify’s local business lead generator actor endpoint with JSON body specifying business types (from ICP), location as "United states", and max results 5  
  - Inputs: ICP industry from ICP industry finder node  
  - Outputs: List of local business leads matching criteria  
  - Edge Cases: API authentication failure, rate limits, empty results  
  - Notes: Sticky note states this block finds leads for the company  

- **Loop Over Items**  
  - Type: SplitInBatches node  
  - Configuration: Splits incoming list of leads into individual items for sequential processing  
  - Inputs: Leads array from Leads node  
  - Outputs: Single lead item per iteration for downstream processing  
  - Edge Cases: Empty input array, batch size configuration effects  

#### 1.5 Personalized Pitch Creation

**Overview:**  
Generates personalized email pitches for each lead based on lead data and predefined messaging parameters.

**Nodes Involved:**  
- personalized emails (HTTP Request)  
- Loop Over Items (SplitInBatches) — second output connection

**Node Details:**  

- **personalized emails**  
  - Type: HTTP Request  
  - Configuration: POST to Apify’s pitches-pro actor endpoint with JSON body containing parameters like additional notes, benefits, business name (from current lead), goal, language, tone, and promotion description  
  - Inputs: Individual lead item from Loop Over Items node (second output)  
  - Outputs: Generated personalized email pitches dataset  
  - Edge Cases: API auth error, malformed input data, rate limits, unexpected API response  
  - Notes: Sticky note highlights creation of personalized emails focusing on time-saving, avoiding jargon, soft call to action  

- **Loop Over Items (second output)**  
  - Feeds each generated pitch back into the Loop Over Items node to potentially repeat or process further (loop-back)  
  - Edge Cases: Potential infinite loop if not properly managed  

---

### 3. Summary Table

| Node Name          | Node Type                  | Functional Role                          | Input Node(s)         | Output Node(s)          | Sticky Note                                                   |
|--------------------|----------------------------|----------------------------------------|-----------------------|-------------------------|---------------------------------------------------------------|
| JotForm Trigger    | JotForm Trigger             | Receive form business input             | None                  | perplexity               | Fill the form about the business you want to generate leads for |
| perplexity         | HTTP Request                | Analyze company via Perplexity AI       | JotForm Trigger       | ICP finder               | Analyze the company in a detailed way                         |
| ICP finder         | OpenAI LangChain (GPT-4o-mini) | Generate Ideal Customer Profile (ICP) | perplexity            | Edit Fields              | Creates ICP that suited best for the company                  |
| Edit Fields        | Set                        | Assign ICP text to JSON field           | ICP finder            | ICP industry finder      | Creates ICP that suited best for the company                  |
| ICP industry finder| OpenAI LangChain (GPT-4o-mini) | Identify ICP industry sector            | Edit Fields           | Leads                   | Finds the industry for the ideal prospects                    |
| Leads              | HTTP Request               | Fetch local business leads               | ICP industry finder   | Loop Over Items          | Find leads for the company                                    |
| Loop Over Items    | SplitInBatches             | Iterate over each lead                   | Leads                 | personalized emails, (loop-back) |                                                              |
| personalized emails| HTTP Request               | Generate personalized email pitches     | Loop Over Items       | Loop Over Items          | Create personalized emails for them                           |
| Sticky Note        | Sticky Note                | Comments/Instruction                     | N/A                   | N/A                      | Fill the form about the business you want to generate leads for |
| Sticky Note1       | Sticky Note                | Comments/Instruction                     | N/A                   | N/A                      | Analyze the company in a detailed way                         |
| Sticky Note2       | Sticky Note                | Comments/Instruction                     | N/A                   | N/A                      | Creates ICP that suited best for the company                  |
| Sticky Note3       | Sticky Note                | Comments/Instruction                     | N/A                   | N/A                      | Finds the industry for the ideal prospects                    |
| Sticky Note4       | Sticky Note                | Comments/Instruction                     | N/A                   | N/A                      | Find leads for the company                                    |
| Sticky Note5       | Sticky Note                | Comments/Instruction                     | N/A                   | N/A                      | Create personalized emails for them                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the JotForm Trigger node:**  
   - Type: JotForm Trigger  
   - Configure to listen on form ID: `252802732808054`  
   - Set credentials to an authorized JotForm API account  
   - This node will trigger the workflow on form submission  

2. **Add an HTTP Request node named "perplexity":**  
   - Method: POST  
   - URL: `https://api.perplexity.ai/search`  
   - Body Type: JSON  
   - Body Content: `{"query": ["{{ $json.website }}"]}` — pass the website URL from the JotForm form data  
   - Add Authorization header with valid Perplexity API token  
   - Connect output of JotForm Trigger to input of this node  

3. **Add an OpenAI LangChain node named "ICP finder":**  
   - Model: GPT-4o-mini  
   - Credentials: OpenAI API key with access to GPT-4o-mini  
   - Messages:  
     - System: `You are an expert client acquisition manager your task is to find icp for the company provided to you.`  
     - User: `Your task is to find the best customer type for the company details provided. Details: {{ $json.results[0].snippet }}`  
   - Connect output of "perplexity" node to this node  

4. **Add a Set node named "Edit Fields":**  
   - Create new field "ICP" with value: `={{ $('ICP finder').item.json.message.content }}`  
   - Connect output of "ICP finder" node to this node  

5. **Add another OpenAI LangChain node named "ICP industry finder":**  
   - Model: GPT-4o-mini  
   - Credentials: OpenAI API key  
   - Messages:  
     - System: Instructions to output a single professional ICP sector excluding SaaS and broad categories  
     - User: Pass the ICP field: `ICP: {{ $json.ICP }}`  
   - Connect output of "Edit Fields" node to this node  

6. **Add an HTTP Request node named "Leads":**  
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/james.logantech~local-business-lead-generator/run-sync-get-dataset-items?token=YOUR_APIFY_TOKEN`  
   - Body Type: JSON  
   - Body Content:  
     ```json
     {
       "buisness_types": ["{{ $json.message.content }}"],
       "location": "United states",
       "max_results": 5
     }
     ```  
   - Connect output of "ICP industry finder" node to this node  

7. **Add a SplitInBatches node named "Loop Over Items":**  
   - Purpose: Split the list of leads into single items to process individually  
   - Connect output of "Leads" node to this node  

8. **Add an HTTP Request node named "personalized emails":**  
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/onescales~pitches-pro/run-sync-get-dataset-items?token=YOUR_APIFY_TOKEN`  
   - Body Type: JSON  
   - Body Content:  
     ```json
     {
       "additionalNotes": "Focus on the time-saving aspect. Avoid technical jargon. Include a soft CTA.",
       "benefits": "Save 10+ hours per week by automating manual tasks ",
       "businessName": "{{ $json.name }}",
       "goal": "Partnership",
       "language": "English",
       "tone": "Professional yet approachable, confident but not pushy",
       "whatPromoting": "Our automation services."
     }
     ```  
   - Connect the first output of "Loop Over Items" node to this node (processing individual leads)  

9. **Connect the output of "personalized emails" node back to the second output of "Loop Over Items" node:**  
   - This creates a loop potentially for further processing or iterative pitch refinement  

10. **Add Sticky Notes at appropriate positions:**  
    - Near JotForm Trigger: "Fill the form about the business you want to generate leads for"  
    - Near "perplexity": "Analyze the company in a detailed way"  
    - Near "Edit Fields": "Creates ICP that suited best for the company"  
    - Near "ICP industry finder": "Finds the industry for the ideal prospects"  
    - Near "Leads": "Find leads for the company"  
    - Near "personalized emails": "Create personalized emails for them"  

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                 |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow integrates multiple AI services (Perplexity AI, OpenAI GPT-4o-mini, Apify actors)  | Ensures diverse AI capabilities for analysis, profiling, lead generation, and pitch creation  |
| OpenAI GPT-4o-mini model is used for cost-effective yet powerful language processing             | Model choice balances performance and cost                                                    |
| Apify actor tokens must be securely stored and managed in n8n credentials                       | Tokens are required for external API access                                                    |
| The workflow assumes US-based lead generation ("United States" hardcoded)                       | Modify location parameter in Leads node for other geographies                                  |
| Pitches emphasize time-saving and approachable tone with soft call to action                    | Reflects best practices in cold email outreach                                                |
| Sticky notes provide contextual instructions for user clarity during workflow maintenance       | Improve usability and onboarding for operators                                                |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated n8n workflow. It strictly adheres to content policies, containing no illegal, offensive, or protected elements. All data processed is legal and publicly available.