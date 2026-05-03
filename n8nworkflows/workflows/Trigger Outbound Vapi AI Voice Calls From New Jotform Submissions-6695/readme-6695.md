Trigger Outbound Vapi AI Voice Calls From New Jotform Submissions

https://n8nworkflows.xyz/workflows/trigger-outbound-vapi-ai-voice-calls-from-new-jotform-submissions-6695


# Trigger Outbound Vapi AI Voice Calls From New Jotform Submissions

### 1. Workflow Overview

This workflow automates outbound voice calls through the Vapi AI platform triggered by new submissions on a specific JotForm. Upon receiving a new form submission, it extracts the phone number, formats it internationally, and initiates an AI-powered voice call using Vapi’s API.

The workflow comprises the following logical blocks:

- **1.1 Input Reception:** Listens for new submissions from a configured JotForm.
- **1.2 Data Extraction and Formatting:** Processes the phone number from the submission to generate a correctly formatted international number.
- **1.3 Configuration Setup:** Defines the necessary Vapi credentials and IDs required to make the outbound call.
- **1.4 Outbound Call Trigger:** Sends an HTTP POST request to Vapi’s API to initiate the voice call.
- **1.5 Optional AI Processing (Unused in main flow):** An OpenAI chat node is present but not integrated into the main execution chain, possibly intended for future use or data enrichment.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow upon new submissions in a specified JotForm form, providing the form data as input.

- **Nodes Involved:**  
  - JotForm Trigger

- **Node Details:**  
  - **JotForm Trigger**  
    - Type: Trigger node for JotForm submissions  
    - Configuration: Connected to JotForm form ID `252102909108349`  
    - Credentials: Uses stored JotForm API credentials  
    - Inputs: Webhook trigger activated on new form submission  
    - Outputs: JSON containing all form fields including phone number details  
    - Edge cases:  
      - Possible webhook downtime or lost triggers  
      - Invalid or missing phone number data in submissions  
      - Authentication errors with JotForm API

#### 2.2 Data Extraction and Formatting

- **Overview:**  
  Extracts and constructs an internationally formatted phone number string from the JotForm submission data.

- **Nodes Involved:**  
  - Information Extractor

- **Node Details:**  
  - **Information Extractor**  
    - Type: Langchain information extraction node  
    - Configuration:  
      - Uses a templated text input combining country code, area code, and phone number from the submission JSON.  
      - Defines an output attribute `internationalPhone` which is a required string representing the phone number in international format (e.g., +1234567890).  
    - Inputs: JSON from JotForm Trigger node  
    - Outputs: JSON with `internationalPhone` attribute  
    - Expressions:  
      - `country: {{ $json['Phone Number'].country }}`  
      - `area code: {{ $json['Phone Number'].area }}`  
      - `phone: {{ $json['Phone Number'].phone }}`  
    - Edge cases:  
      - Missing or malformed phone number components  
      - Extraction failures due to unexpected input format  
      - Non-numeric or invalid phone numbers

#### 2.3 Configuration Setup

- **Overview:**  
  Sets static fields containing Vapi API credentials and identifiers required to initiate calls.

- **Nodes Involved:**  
  - Set fields

- **Node Details:**  
  - **Set fields**  
    - Type: Set node to assign static values  
    - Configuration:  
      - `vapiPhoneNumberId`: ID of the phone number used to place calls (to be replaced by user)  
      - `vapiAssistantId`: ID of the Vapi assistant enabled for the call (to be replaced by user)  
      - `vapiApi`: Vapi API key (to be replaced by user)  
    - Inputs: JSON from Information Extractor  
    - Outputs: JSON enriched with Vapi configuration fields  
    - Edge cases:  
      - Failure to replace placeholder IDs with valid values will cause call initiation errors  
      - Credential expiration or invalid API keys

- **Sticky Note associated:**  
  - Instructions to set Vapi phone number ID, assistant ID, and API key from the Vapi account.

#### 2.4 Outbound Call Trigger

- **Overview:**  
  Sends a POST request to Vapi’s call API to initiate an outbound AI voice call using the formatted phone number and configured credentials.

- **Nodes Involved:**  
  - Start outbound Vapi call

- **Node Details:**  
  - **Start outbound Vapi call**  
    - Type: HTTP Request node (POST)  
    - URL: `https://api.vapi.ai/call`  
    - Headers: Authorization Bearer token using `vapiApi`  
    - Body (JSON):  
      ```json
      {
        "assistantId": "{{ $json.vapiAssistantId }}",
        "phoneNumberId": "{{ $json.vapiPhoneNumberId }}",
        "customer": {
          "number": "{{ $('Information Extractor').item.json.output.internationalPhone }}"
        }
      }
      ```  
    - Inputs: JSON from Set fields node  
    - Outputs: API response from Vapi  
    - Edge cases:  
      - API authentication errors (invalid or expired token)  
      - Invalid assistant or phone number IDs  
      - Network timeouts or API rate limits  
      - Malformed or missing phone number field in request body  
    - Version specifics: HTTP Request node version 4.2 used for advanced body and header handling.

#### 2.5 Optional AI Processing (Not in main flow)

- **Overview:**  
  An OpenAI language model node is included but not connected to the main workflow execution path. It likely serves as a placeholder or for future enhancement to process or enrich data.

- **Nodes Involved:**  
  - OpenAI Chat Model

- **Node Details:**  
  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI Chat node  
    - Configuration: Uses GPT-4.1 model, no additional options set  
    - Credentials: OpenAI API key  
    - Inputs/Outputs: Not connected on main path; no live input during execution  
    - Edge cases: N/A as inactive in main execution  
    - Potential use: Data enrichment, validation, or conversational AI integration

---

### 3. Summary Table

| Node Name             | Node Type                      | Functional Role                 | Input Node(s)         | Output Node(s)          | Sticky Note                                                                                                    |
|-----------------------|--------------------------------|--------------------------------|-----------------------|-------------------------|---------------------------------------------------------------------------------------------------------------|
| Sticky Note           | Sticky Note                   | Instruction for Vapi field setup | -                     | -                       | ## Set Vapi fields<br>You must set the following fields that you can obtain inside your Vapi account<br>- phone number id which will make the call<br>- assistant id which will be enabled in the call<br>- your Vapi api key |
| Start outbound Vapi call | HTTP Request                 | Sends POST to Vapi API to start call | Set fields            | -                       |                                                                                                               |
| Set fields            | Set                           | Assign Vapi API credentials and IDs | Information Extractor | Start outbound Vapi call |                                                                                                               |
| Sticky Note1          | Sticky Note                   | Requirements and useful links   | -                     | -                       | ## Requirements<br>### Jotform<br>- [Jotform](https://www.jotform.com/?partner=https://1node.ai) account<br>- Jotform API credentials enabled in n8n<br>- A Jotform form published that includes a phone number field<br><br>### Vapi<br>- A [Vapi](https://vapi.ai/?aff=onenode) account with credit<br>- A connected phone number which will make the calls<br>- An assistant created and ready to make calls<br>- Vapi api key<br><br>### Useful link<br>- [Vapi docs](https://docs.vapi.ai/api-reference/calls/create) |
| JotForm Trigger       | JotForm Trigger               | Trigger on new JotForm submission | -                     | Information Extractor    |                                                                                                               |
| Information Extractor | Langchain Information Extractor | Extracts and formats phone number | JotForm Trigger       | Set fields              |                                                                                                               |
| OpenAI Chat Model     | Langchain OpenAI Chat         | (Unused) Potential AI enrichment | -                     | -                       |                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a JotForm Trigger node:**  
   - Set the node name to `JotForm Trigger`.  
   - Under parameters, specify the JotForm form ID to listen to (e.g., `252102909108349`).  
   - Configure JotForm API credentials (must be previously created in n8n).  
   - Position this node as the workflow entry point.

3. **Add an Information Extractor node:**  
   - Name it `Information Extractor`.  
   - Use Langchain Information Extractor node type.  
   - Configure the `text` parameter with the template:  
     ```
     - country: {{ $json['Phone Number'].country }}
     - area code: {{ $json['Phone Number'].area }}
     - phone: {{ $json['Phone Number'].phone }}
     ```  
   - Define an attribute named `internationalPhone` of type string, marked as required, with the description:  
     "international formatted phone number. Includes the + sign, followed by the country code and phone number, without spaces. Example: +1234567890"  
   - Connect the output of `JotForm Trigger` into this node.

4. **Add a Set node:**  
   - Name it `Set fields`.  
   - Add three string fields with the following keys and placeholder values (to be replaced by actual Vapi data):  
     - `vapiPhoneNumberId`: `"insert-id"`  
     - `vapiAssistantId`: `"insert-id"`  
     - `vapiApi`: `"insert-api"`  
   - Connect the output of `Information Extractor` into this node.

5. **Add an HTTP Request node:**  
   - Name it `Start outbound Vapi call`.  
   - Set method to POST.  
   - Set URL to `https://api.vapi.ai/call`.  
   - Under Headers, add `Authorization` with value: `Bearer {{ $json.vapiApi }}`.  
   - For the request body, select JSON and enter the following template:  
     ```json
     {
       "assistantId": "{{ $json.vapiAssistantId }}",
       "phoneNumberId": "{{ $json.vapiPhoneNumberId }}",
       "customer": {
         "number": "{{ $('Information Extractor').item.json.output.internationalPhone }}"
       }
     }
     ```  
   - Connect the output of `Set fields` into this node.

6. **Optionally, add Sticky Note nodes for documentation:**  
   - Add a sticky note near `Set fields` node with instructions on setting Vapi IDs and API keys.  
   - Add another sticky note near the start of the workflow documenting requirements and links for JotForm and Vapi accounts.

7. **Credentials setup:**  
   - Ensure JotForm API credentials are configured in n8n with access to the target form.  
   - Ensure Vapi API key is active and has permissions for outgoing calls.  
   - No credentials required for the Information Extractor node.  
   - For OpenAI nodes (optional), configure OpenAI API credentials if used.

8. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                    | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| JotForm account required with an active form that includes a phone number field. Ensure JotForm API credentials are properly configured in n8n. | https://www.jotform.com/?partner=https://1node.ai                                                  |
| Vapi account required with connected phone number and assistant, plus an active API key with credit for calls.                                 | https://vapi.ai/?aff=onenode                                                                       |
| Vapi API documentation for creating calls is available here.                                                                                    | https://docs.vapi.ai/api-reference/calls/create                                                    |
| The phone number must be in international format with a leading + sign and no spaces, e.g., +1234567890.                                        | Important for Vapi API call to succeed                                                             |
| Ensure to replace all placeholder IDs and API keys in the Set node before activating the workflow.                                              | Critical to avoid authentication or call initiation failures                                       |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created with n8n, adhering strictly to content policies. It contains no illegal, offensive, or protected elements. All data handled is legal and public.