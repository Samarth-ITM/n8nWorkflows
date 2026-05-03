Enrich HubSpot Contacts with LinkedIn Profiles using SerpAPI, Google Docs and AI

https://n8nworkflows.xyz/workflows/enrich-hubspot-contacts-with-linkedin-profiles-using-serpapi--google-docs-and-ai-8434


# Enrich HubSpot Contacts with LinkedIn Profiles using SerpAPI, Google Docs and AI

### 1. Workflow Overview

This workflow, titled **"HubSpot Contact Refinement"**, automates the enrichment of HubSpot contact records by researching and appending LinkedIn profile URLs. It targets use cases involving sales, marketing, or CRM teams seeking to enhance contact data with social professional profiles for better engagement.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Triggering**: Detects new or updated contacts in HubSpot and allows manual execution for batch processing or testing.
- **1.2 Data Preparation and Cleaning**: Extracts and normalizes contact fields (first name, last name, email) for downstream AI processing.
- **1.3 AI-Driven LinkedIn Profile Research**: Uses a Google Docs-based research guide and SerpAPI-powered Google searches to find LinkedIn URLs via an AI agent.
- **1.4 Result Cleaning**: Removes extraneous AI-generated metadata or formatting artifacts to isolate the clean LinkedIn URL.
- **1.5 HubSpot Contact Update**: Writes back the found LinkedIn URL into the corresponding HubSpot contact record.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Triggering

**Overview:**  
This block receives contact data changes from HubSpot or manual execution and fetches recently created/updated contacts for batch processing.

**Nodes Involved:**  
- HubSpot Trigger  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Get recently created/updated contacts  

**Node Details:**

- **HubSpot Trigger**  
  - *Type:* HubSpot event webhook trigger  
  - *Role:* Automatically fires when contacts are created or updated in HubSpot  
  - *Configuration:* Subscribes to contact create/update events using HubSpot Developer OAuth credentials  
  - *Connections:* Outputs to "Edit Fields" node  
  - *Edge Cases:* Webhook misconfiguration, OAuth token expiry, no event data received  
  - *Version:* v1  

- **When clicking ‘Execute workflow’**  
  - *Type:* Manual trigger  
  - *Role:* Allows ad-hoc execution for testing or batch processing  
  - *Connections:* Starts "Get recently created/updated contacts" node  
  - *Edge Cases:* Manual execution with no data may cause empty downstream processing  

- **Get recently created/updated contacts**  
  - *Type:* HubSpot node (getRecentlyCreatedUpdated operation)  
  - *Role:* Retrieves a list of contacts recently created or updated in HubSpot  
  - *Authentication:* HubSpot Private App Token  
  - *Connections:* Outputs to "Edit Fields"  
  - *Edge Cases:* API rate limits, empty result sets, token expiration  
  - *Version:* v2.1  

---

#### 2.2 Data Preparation and Cleaning

**Overview:**  
This block maps and normalizes key contact fields (First Name, Last Name, Email) into a standardized format for AI consumption.

**Nodes Involved:**  
- Edit Fields  
- Sticky Note1 (contextual)  

**Node Details:**

- **Edit Fields**  
  - *Type:* Set node  
  - *Role:* Extracts and assigns contact properties firstname, lastname, and email into new fields: "First Name", "Last Name", "Email"  
  - *Expressions Used:*  
    - First Name: `{{$json.properties.firstname.value}}`  
    - Last Name: `{{$json.properties.lastname.value}}`  
    - Email: `{{$json['identity-profiles'][0].identities[0].value}}`  
  - *Connections:* Outputs to "AI Agent" node  
  - *Edge Cases:* Missing or malformed contact properties, index out-of-range errors if identities array is empty  
  - *Version:* v3.4  

---

#### 2.3 AI-Driven LinkedIn Profile Research

**Overview:**  
This block directs an AI agent to read a Google Docs document explaining the research approach, then uses SerpAPI to perform targeted Google searches to find LinkedIn profile URLs related to the contact.

**Nodes Involved:**  
- Read Google Docs  
- AI Agent  
- SerpAPI  
- OpenRouter Chat Model  
- Sticky Note2 (contextual)  

**Node Details:**

- **Read Google Docs**  
  - *Type:* Google Docs Tool node  
  - *Role:* Retrieves text content from a specified Google Doc containing research instructions and expected output format  
  - *Parameters:* Document URL to be replaced with actual doc URL; uses a Google service account for authentication  
  - *Connections:* Provides input to AI Agent’s ai_tool parameter  
  - *Edge Cases:* Document access denied, invalid URL, API quota exceeded  
  - *Version:* v2  

- **AI Agent**  
  - *Type:* Langchain agent node  
  - *Role:* Orchestrates AI research steps: first reads the Google Doc, then performs a Google search using SerpAPI with contact details (first name, last name, email)  
  - *Prompt:* Custom prompt instructing the agent to use the Google Docs content and then research the contact on Google  
  - *Connections:* Receives ai_tool input from "Read Google Docs" & "SerpAPI", uses OpenRouter Chat Model as language model  
  - *Edge Cases:* AI model latency or failure, malformed prompt results, API key issues  
  - *Version:* v2.2  

- **SerpAPI**  
  - *Type:* Langchain SerpAPI tool node  
  - *Role:* Executes Google searches programmatically for the AI agent to find LinkedIn URLs  
  - *Credentials:* SerpAPI account with valid API key  
  - *Connections:* Connected to AI Agent’s ai_tool input  
  - *Edge Cases:* Rate limits, invalid API key, search result inconsistencies  
  - *Version:* v1  

- **OpenRouter Chat Model**  
  - *Type:* Langchain OpenRouter chat language model node  
  - *Role:* Provides chat-based language model responses to the AI Agent for understanding instructions and generating search queries  
  - *Credentials:* OpenRouter API key  
  - *Connections:* Connected as ai_languageModel input to AI Agent  
  - *Edge Cases:* API latency, authentication errors, model capacity limits  
  - *Version:* v1  

---

#### 2.4 Result Cleaning

**Overview:**  
This block removes internal AI reasoning tags and formatting artifacts from the agent’s output to isolate a clean LinkedIn URL.

**Nodes Involved:**  
- Code - Remove Think part  
- Sticky Note3 (contextual)  

**Node Details:**

- **Code - Remove Think part**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Strips out `<think>...</think>` tags and Markdown JSON fences from the AI output text to produce only the clean LinkedIn URL  
  - *Key Code:*  
  ```js
  let text = $json.text ?? $json.output ?? $json.result ?? "";
  text = String(text)
    .replace(/<think>[\s\S]*?<\/think>/gi, "")
    .replace(/^```(?:json)?\s*|\s*```$/g, "")
    .trim();
  return { json: { cleanText: text } };
  ```  
  - *Connections:* Outputs cleaned text to "HubSpot" update node  
  - *Edge Cases:* Input text missing, unexpected output formats, regex failures  
  - *Version:* v2  

---

#### 2.5 HubSpot Contact Update

**Overview:**  
This block updates the original HubSpot contact record by adding the cleaned LinkedIn URL to a custom contact property.

**Nodes Involved:**  
- HubSpot  
- Sticky Note4 (contextual)  

**Node Details:**

- **HubSpot**  
  - *Type:* HubSpot contact update node  
  - *Role:* Updates the contact’s "linkedinUrl" property using the cleaned LinkedIn URL found by the AI agent  
  - *Parameters:*  
    - Contact identified by email (from "Edit Fields")  
    - linkedinUrl field set to cleaned LinkedIn URL (from "Code - Remove Think part")  
  - *Authentication:* HubSpot Private App Token  
  - *Connections:* Input from "Code - Remove Think part"  
  - *Edge Cases:* Email not found in HubSpot, API write failures, token expiration  
  - *Version:* v2.1  

---

### 3. Summary Table

| Node Name                      | Node Type                         | Functional Role                         | Input Node(s)                   | Output Node(s)                 | Sticky Note                                                                                                                         |
|--------------------------------|----------------------------------|---------------------------------------|---------------------------------|-------------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| HubSpot Trigger                | HubSpot Trigger                  | Fire on contact create/update events | None                            | Edit Fields                   |                                                                                                                                     |
| When clicking ‘Execute workflow’ | Manual Trigger                  | Manual start for batch or test runs   | None                            | Get recently created/updated contacts |                                                                                                                                     |
| Get recently created/updated contacts | HubSpot (getRecentlyCreatedUpdated) | Retrieve recent contacts              | When clicking ‘Execute workflow’ | Edit Fields                   |                                                                                                                                     |
| Edit Fields                   | Set                             | Normalize key contact fields          | HubSpot Trigger, Get recently created/updated contacts | AI Agent                     |                                                                                                                                     |
| Read Google Docs              | Google Docs Tool                | Fetch research instructions document  | None                            | AI Agent (ai_tool)            |                                                                                                                                     |
| SerpAPI                      | Langchain SerpAPI Tool          | Google search for LinkedIn profiles   | None                            | AI Agent (ai_tool)            |                                                                                                                                     |
| AI Agent                     | Langchain Agent                 | Orchestrate AI research and search    | Edit Fields, Read Google Docs, SerpAPI, OpenRouter Chat Model | Code - Remove Think part     | AI Agent performs Google search to find useful information from LinkedIn Profile                                                     |
| OpenRouter Chat Model         | Langchain OpenRouter Chat Model | Language model for AI Agent            | None                            | AI Agent (ai_languageModel)   |                                                                                                                                     |
| Code - Remove Think part      | Code                           | Clean AI output to extract LinkedIn URL | AI Agent                      | HubSpot                      | Think tag portion is common from smaller models and are removed                                                                      |
| HubSpot                      | HubSpot                        | Update contact with LinkedIn URL       | Code - Remove Think part         | None                         | The workflow updates LinkedIn field for the contact                                                                                  |
| Sticky Note                  | Sticky Note                    | Workflow initiation note                | None                            | None                         | ## Innitiate the Workflow                                                                                                            |
| Sticky Note1                 | Sticky Note                    | Data cleanup explanation                | None                            | None                         | ## Data Clean Up                                                                                                                     |
| Sticky Note2                 | Sticky Note                    | AI Agent research explanation           | None                            | None                         | ## AI Agent Led Contact Research<br>AI agent performs Google search to find useful information from LinkedIn Profile                |
| Sticky Note3                 | Sticky Note                    | Data clean up explanation               | None                            | None                         | ## Data Clean up<br>Think tag portion is common from smaller models and are removed                                                 |
| Sticky Note4                 | Sticky Note                    | HubSpot update explanation              | None                            | None                         | ## Update Hubspot Contact<br>The workflow updates LinkedIn field for the contact                                                     |
| Sticky Note5                 | Sticky Note                    | Full workflow explanation and instructions | None                            | None                         | ## HubSpot Contact Refinement<br>🚀 **How it works** ... (see section 5 for full note content)                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the HubSpot Trigger node**  
   - Type: HubSpot Trigger  
   - Credentials: HubSpot Developer OAuth (with subscriptions to contact create/update events)  
   - Purpose: Automatically trigger on contact changes  
   - No input connections  

2. **Create the Manual Trigger node**  
   - Type: Manual Trigger  
   - Purpose: Allow manual execution for testing or batch processing  

3. **Create the Get recently created/updated contacts node**  
   - Type: HubSpot node with operation "getRecentlyCreatedUpdated"  
   - Credentials: HubSpot App Token (Private App)  
   - Connect Manual Trigger output to this node's input  

4. **Create the Edit Fields node (Set node)**  
   - Extract and assign these fields for each contact:  
     - First Name: `{{$json.properties.firstname.value}}`  
     - Last Name: `{{$json.properties.lastname.value}}`  
     - Email: `{{$json['identity-profiles'][0].identities[0].value}}`  
   - Connect outputs of HubSpot Trigger and Get recently created/updated contacts into this node  

5. **Create the Read Google Docs node**  
   - Type: Google Docs Tool (operation: get)  
   - Credentials: Google Service Account (must have Viewer access to the target Google Doc)  
   - Parameter: Paste the URL of the Google Docs research instructions document  
   - No input connections (will connect to AI Agent)  

6. **Create the SerpAPI node**  
   - Type: Langchain SerpAPI Tool  
   - Credentials: SerpAPI API key  
   - No parameters required (default search settings can be configured inside the AI Agent prompt)  

7. **Create the OpenRouter Chat Model node**  
   - Type: Langchain OpenRouter Chat Model  
   - Credentials: OpenRouter API key  

8. **Create the AI Agent node**  
   - Type: Langchain Agent  
   - Parameters:  
     - Prompt: Instruct the AI agent to first read the Google Docs instructions, then perform Google search using contact fields (First Name, Last Name, Email)  
   - Connect:  
     - ai_tool inputs from both "Read Google Docs" and "SerpAPI" nodes  
     - ai_languageModel input from "OpenRouter Chat Model" node  
     - main input from "Edit Fields" node (providing contact data)  

9. **Create the Code node "Remove Think part"**  
   - Type: Code (JavaScript)  
   - Function: Remove `<think>...</think>` tags and ```json fences from AI output text to extract clean LinkedIn URL  
   - Connect main output of AI Agent to this node  

10. **Create the HubSpot update node**  
    - Type: HubSpot (update contact)  
    - Credentials: HubSpot App Token (Private App)  
    - Parameters:  
      - Identify contact by email (`={{ $('Edit Fields').item.json.Email }}`)  
      - Update property "linkedinUrl" with `={{ $json.cleanText }}` (output of Code node)  
    - Connect main output of Code node to this node  

11. **Connect the nodes as per the flow:**  
    - HubSpot Trigger → Edit Fields → AI Agent → Code - Remove Think part → HubSpot update  
    - When clicking ‘Execute workflow’ → Get recently created/updated contacts → Edit Fields → AI Agent ...  
    - Read Google Docs and SerpAPI connect as ai_tool inputs to AI Agent  
    - OpenRouter Chat Model connects as ai_languageModel input to AI Agent  

12. **Create and add Sticky Note nodes** at appropriate places to document workflow sections and explanations as per the original workflow for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Full workflow explanation, setup instructions, and credentials details including HubSpot Private App Token, HubSpot Developer OAuth, Google Service Account, OpenRouter, and SerpAPI. Includes a sample Google Docs URL for research instructions: [https://docs.google.com/document/d/1nn69H3wdwXJolDwu0avlFw7sIwnX_Hpr3bmOD6PgdW4/edit?usp=sharing](https://docs.google.com/document/d/1nn69H3wdwXJolDwu0avlFw7sIwnX_Hpr3bmOD6PgdW4/edit?usp=sharing) | Workflow Sticky Note5 content                                                                                   |
| The Google Docs used by the AI Agent must be shared with the Google Service Account email with at least Viewer permission to allow reading the instructions.                                                                                                                                                                                                                                                                                                                                                                | Google Docs Tool node configuration note                                                                         |
| HubSpot contact property "linkedinUrl" must exist as a Text field in your HubSpot contact schema to store the LinkedIn profile URL found by the AI.                                                                                                                                                                                                                                                                                                                                                                        | HubSpot node update requirements                                                                                 |
| SerpAPI usage should be configured with sensible defaults (e.g., engine=google, hl=en, gl=us, num=5) within the AI prompt or SerpAPI node to control search volume and cost.                                                                                                                                                                                                                                                                                                                                               | AI Agent and SerpAPI node usage                                                                                   |
| The Code node uses regular expressions to remove AI internal `<think>` tags and JSON fences common in smaller language model outputs to yield clean, user-relevant results.                                                                                                                                                                                                                                                                                                                                              | Code - Remove Think part node explanation                                                                         |
| The workflow supports both event-driven (HubSpot Trigger) and batch/manual modes (Manual Trigger + Get recently created contacts) for flexibility in operations.                                                                                                                                                                                                                                                                                                                                                           | Workflow design note                                                                                              |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All processed data is legal and public.