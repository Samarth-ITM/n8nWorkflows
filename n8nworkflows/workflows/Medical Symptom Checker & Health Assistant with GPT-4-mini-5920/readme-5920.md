Medical Symptom Checker & Health Assistant with GPT-4-mini

https://n8nworkflows.xyz/workflows/medical-symptom-checker---health-assistant-with-gpt-4-mini-5920


# Medical Symptom Checker & Health Assistant with GPT-4-mini

---

### 1. Workflow Overview

This workflow, titled **"AI Medical Symptom Checker & Health Assistant with GPT-4-mini"**, serves as an automated health information assistant designed to provide general guidance based on user-submitted health queries. It targets users seeking symptom analysis, medication information, appointment advice, wellness tips, and emergency detection through conversational input. The system is privacy-conscious, does not store user data by default, and emphasizes that it is not a substitute for professional medical advice.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Accepts user queries via a secure webhook with privacy headers.
- **1.2 Safety and Categorization:** Analyzes input for emergencies and categorizes the query type.
- **1.3 Emergency Handling:** Routes emergency cases to an immediate automated response.
- **1.4 Health Information Processing:** For non-emergencies, uses GPT-4-mini AI to provide general health info.
- **1.5 Response Formatting:** Adds disclaimers, health resources, and organizes response metadata.
- **1.6 Response Compilation & Delivery:** Combines all outputs into a structured JSON and returns it to the user.
- **1.7 (Optional) Audit Logging:** Optional logging of queries for compliance and analysis.
- **1.8 Documentation & Notes:** Multiple sticky notes provide disclaimers, configuration requirements, feature roadmap, and integration options.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception

- **Overview:**  
  Receives POST requests containing health-related queries, ensuring privacy headers are set for CORS and content security.

- **Nodes Involved:**  
  - Health Query Webhook

- **Node Details:**

  - **Health Query Webhook**  
    - Type: Webhook node  
    - Configuration:  
      - Method: POST  
      - Path: `/health-assistant`  
      - Response Mode: Responds via a response node downstream  
      - Response Headers:  
        - Access-Control-Allow-Origin: * (enables cross-origin requests)  
        - X-Content-Type-Options: nosniff (prevents MIME-type sniffing)  
    - Inputs: External HTTP requests  
    - Outputs: JSON containing the raw user query and metadata  
    - Edge Cases:  
      - Invalid HTTP methods will not be processed  
      - Missing or malformed JSON body could cause downstream errors  
    - Version: 1.1

---

#### Block 1.2: Safety and Categorization

- **Overview:**  
  Parses and analyzes the incoming query text to detect emergency keywords, categorize the query type (symptom, medication, appointment, wellness, or general), and extracts metadata such as language, age, and session ID.

- **Nodes Involved:**  
  - Safety Check & Categorization

- **Node Details:**

  - **Safety Check & Categorization**  
    - Type: Code node (JavaScript)  
    - Function:  
      - Extracts user input from request body or direct JSON  
      - Detects if input contains emergency keywords (e.g., chest pain, stroke)  
      - Categorizes query based on keywords into types: symptom, medication, appointment, wellness, or general  
      - Generates a session ID if not provided  
      - Adds privacy notice and disclaimers in output  
    - Inputs: Output from Webhook node  
    - Outputs: JSON with userInput, language, age, sessionId, isEmergency (boolean), queryType, privacyNotice, timestamp, disclaimer  
    - Edge Cases:  
      - Empty or missing input results in default categorization 'general' and no emergency  
      - Case-insensitive keyword detection ensures robustness  
      - Could miss emergencies if phrasing is unusual or keywords absent  
    - Version: 2

---

#### Block 1.3: Emergency Handling

- **Overview:**  
  Routes queries flagged as emergencies to an immediate response with emergency contact information and safety instructions.

- **Nodes Involved:**  
  - Emergency Router  
  - Emergency Response

- **Node Details:**

  - **Emergency Router**  
    - Type: If node  
    - Function: Checks if `isEmergency` is true  
    - Inputs: Output from Safety Check & Categorization  
    - Outputs:  
      - True branch: Emergency Response  
      - False branch: Health Information AI  
    - Version: 2

  - **Emergency Response**  
    - Type: Set node  
    - Function:  
      - Sets a predefined emergency message with emergency phone numbers (911, 999, 112, 000)  
      - Provides safety tips while waiting for help  
      - Sets responseType = "emergency" and severity = "critical"  
    - Inputs: True branch from Emergency Router  
    - Outputs: JSON with emergency response content and metadata  
    - Edge Cases: None expected, message is static and deterministic  
    - Version: 3.4

---

#### Block 1.4: Health Information Processing

- **Overview:**  
  For non-emergency queries, the workflow uses OpenAI's GPT-4-mini model to generate general health information, strictly constrained to avoid diagnosis or prescription.

- **Nodes Involved:**  
  - Health Information AI  
  - Format Health Response

- **Node Details:**

  - **Health Information AI**  
    - Type: OpenAI node (via Langchain)  
    - Model: GPT-4-mini  
    - Configuration:  
      - Max tokens: 1000  
      - Temperature: 0.3 (low randomness, focused on factual responses)  
      - System prompt: Detailed instructions to avoid diagnosis, prescriptions, and to recommend professional consultation  
      - User prompt: Includes user input text, query type, language, and age group  
    - Inputs: False branch from Emergency Router  
    - Outputs: AI-generated message content in JSON  
    - Credentials: Requires valid OpenAI API key  
    - Edge Cases:  
      - API rate limits or authentication errors may disrupt response  
      - Model hallucination potential minimized by prompt design but still possible  
      - Language understanding depends on input and model capabilities  
    - Version: 1

  - **Format Health Response**  
    - Type: Set node  
    - Function:  
      - Extracts AI message content  
      - Adds disclaimers emphasizing educational purpose only  
      - Adds curated health resource links (doctor finder, symptom checker, mental health, poison control)  
      - Copies query type from Safety Check & Categorization node for metadata  
    - Inputs: Output from Health Information AI  
    - Outputs: JSON with formatted response, disclaimer, and resources  
    - Edge Cases: None critical; static disclaimers and links  
    - Version: 3.4

---

#### Block 1.5: Response Compilation & Delivery

- **Overview:**  
  Merges emergency and AI responses, compiles a structured JSON response including session info, disclaimers, suggested actions, and next steps, then sends it back to the requester.

- **Nodes Involved:**  
  - Merge  
  - Compile Final Response  
  - Send Response

- **Node Details:**

  - **Merge**  
    - Type: Merge node  
    - Mode: Combine multiplex (merges responses from emergency and AI branches)  
    - Inputs:  
      - From Emergency Response (emergency branch)  
      - From Format Health Response (non-emergency branch)  
    - Outputs: Combined response for final processing  
    - Version: 3

  - **Compile Final Response**  
    - Type: Code node  
    - Function:  
      - Extracts input metadata and merged response content  
      - Organizes output JSON with:  
        - success flag  
        - sessionId, timestamp  
        - original query and metadata  
        - response content, type, severity  
        - medical and privacy disclaimers  
        - health resources  
        - suggested actions and next steps based on queryType and emergency status  
      - Helper functions define suggested actions and next steps for each query category  
    - Inputs: Output from Merge node  
    - Outputs: Final structured JSON response ready for delivery  
    - Edge Cases: Handles missing severity by defaulting to "normal"  
    - Version: 2

  - **Send Response**  
    - Type: Respond to Webhook node  
    - Function: Sends HTTP 200 with JSON response body  
    - Headers include:  
      - Content-Type: application/json  
      - X-Health-Disclaimer: "This is not medical advice"  
    - Inputs: Output from Compile Final Response  
    - Outputs: HTTP response to original requester  
    - Version: 1.1

---

#### Block 1.6: Optional Audit Logging

- **Overview:**  
  Optionally logs query metadata into an Airtable base for compliance, audit, or analytics.

- **Nodes Involved:**  
  - Audit Log (Optional)

- **Node Details:**

  - **Audit Log (Optional)**  
    - Type: Airtable node  
    - Function: Creates new record with sessionId, queryType, isEmergency, timestamp  
    - Disabled by default (can be enabled if audit logging is required)  
    - Requires Airtable credentials and configuration of the health-assistant-logs base  
    - Edge Cases: Network or credential failures; should not block main flow if disabled  
    - Version: 2

---

#### Block 1.7: Documentation & Notes

- **Overview:**  
  Multiple Sticky Note nodes document key aspects of the workflow: disclaimers, safety protocols, symptom categories, example interactions, configuration requirements, feature roadmap, and integration options.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2  
  - Sticky Note3  
  - Sticky Note4  
  - Sticky Note5  
  - Sticky Note6

- **Node Details:**  
  - These nodes do not affect workflow logic but provide critical contextual information and instructions for operators and integrators.  
  - They cover:  
    - AI Health Assistant disclaimer and features  
    - Emergency detection and privacy protocols  
    - Supported symptom categories  
    - Setup and compliance checklist  
    - Example user-bot dialogues  
    - Planned and future feature roadmap  
    - Integration channels and external systems  

---

### 3. Summary Table

| Node Name                   | Node Type                     | Functional Role                          | Input Node(s)               | Output Node(s)                   | Sticky Note                                                                                                             |
|-----------------------------|-------------------------------|----------------------------------------|-----------------------------|---------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| Sticky Note                 | Sticky Note                   | Documentation: AI Health Assistant info | -                           | -                               | ## 🏥 AI HEALTH ASSISTANT - Disclaimer, features, and use cases                                                         |
| Sticky Note1                | Sticky Note                   | Documentation: Safety protocols         | -                           | -                               | ## ⚠️ SAFETY PROTOCOLS - Emergency detection, privacy, auto-escalation                                                   |
| Sticky Note2                | Sticky Note                   | Documentation: Symptom categories       | -                           | -                               | ## 📋 SYMPTOM CATEGORIES - Supported symptom areas                                                                        |
| Sticky Note3                | Sticky Note                   | Documentation: Configuration checklist  | -                           | -                               | ## 🔧 CONFIGURATION - Required and optional setup, compliance notes                                                      |
| Sticky Note4                | Sticky Note                   | Documentation: Example interactions     | -                           | -                               | ## 💬 EXAMPLE INTERACTIONS - Sample user queries and bot responses                                                       |
| Sticky Note5                | Sticky Note                   | Documentation: Features roadmap         | -                           | -                               | ## 📊 FEATURES ROADMAP - Current, planned, and future features                                                          |
| Sticky Note6                | Sticky Note                   | Documentation: Integration options      | -                           | -                               | ## 📱 INTEGRATION OPTIONS - Supported channels, healthcare systems, data sources                                         |
| Health Query Webhook        | Webhook                      | Input reception of health queries       | -                           | Safety Check & Categorization    | Receives health-related queries with strict privacy                                                                      |
| Safety Check & Categorization | Code                        | Emergency detection & query categorization | Health Query Webhook         | Emergency Router                 | Checks for emergencies and categorizes queries                                                                           |
| Emergency Router            | If                           | Routes to emergency or AI processing    | Safety Check & Categorization | Emergency Response / Health Information AI |                                                                                                                         |
| Emergency Response          | Set                          | Immediate emergency response             | Emergency Router (true)       | Merge                           | Immediate emergency response with contact numbers                                                                        |
| Health Information AI       | OpenAI                       | Generates general health information    | Emergency Router (false)      | Format Health Response           | Provides general health information with strict guidelines                                                              |
| Format Health Response      | Set                          | Adds disclaimers and health resources   | Health Information AI         | Merge                           | Adds disclaimers and resources to response                                                                               |
| Merge                      | Merge                        | Combines emergency and AI responses     | Emergency Response, Format Health Response | Compile Final Response          |                                                                                                                         |
| Compile Final Response      | Code                         | Structures final response JSON           | Merge                        | Send Response                   | Creates structured response with all necessary information                                                              |
| Send Response              | Respond to Webhook           | Sends HTTP response to requester         | Compile Final Response         | -                               | Returns health information with appropriate headers                                                                     |
| Audit Log (Optional)        | Airtable                     | Optional logging for compliance          | Format Health Response         | -                               | Optional: Log queries for compliance (configure data retention)                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Name: Health Query Webhook  
   - HTTP Method: POST  
   - Path: `health-assistant`  
   - Response Mode: Response Node  
   - Add Response Headers:  
     - `Access-Control-Allow-Origin: *`  
     - `X-Content-Type-Options: nosniff`

2. **Add Code Node for Safety & Categorization**  
   - Name: Safety Check & Categorization  
   - Paste the JavaScript code to:  
     - Extract user input, language, age, session ID  
     - Detect emergencies via keywords  
     - Categorize query type (symptom, medication, appointment, wellness, general)  
     - Output all metadata plus privacy notice and disclaimers

3. **Add If Node for Emergency Routing**  
   - Name: Emergency Router  
   - Condition: `{{$json.isEmergency}} == true`  
   - True Output: Emergency Response node  
   - False Output: Health Information AI node

4. **Add Set Node for Emergency Response**  
   - Name: Emergency Response  
   - Assign these values:  
     - response: Predefined emergency message with emergency numbers and safety tips  
     - responseType: "emergency"  
     - severity: "critical"

5. **Add OpenAI Node**  
   - Name: Health Information AI  
   - Credentials: Setup OpenAI API key  
   - Model: GPT-4-mini  
   - Temperature: 0.3, Max Tokens: 1000  
   - System Prompt: Instructions to avoid diagnosis/prescription, emphasize consulting professionals  
   - User Prompt: User input and metadata injected via expressions

6. **Add Set Node for Formatting Response**  
   - Name: Format Health Response  
   - Extract AI message content to `response`  
   - Copy query type from Safety Check & Categorization  
   - Add medical disclaimer text and health resource links as static strings

7. **Add Merge Node**  
   - Name: Merge  
   - Mode: Combine multiplex  
   - Inputs:  
     - Emergency Response output  
     - Format Health Response output

8. **Add Code Node to Compile Final Response**  
   - Name: Compile Final Response  
   - Implement JS code to:  
     - Structure final JSON with session, timestamps, query metadata, response content, disclaimers, resources  
     - Include helper functions for suggested actions and next steps based on query type and emergency status

9. **Add Respond to Webhook Node**  
   - Name: Send Response  
   - Response Code: 200  
   - Response Headers:  
     - Content-Type: application/json  
     - X-Health-Disclaimer: "This is not medical advice"  
   - Response Body: JSON stringified output from Compile Final Response

10. **(Optional) Add Airtable Node for Audit Log**  
    - Name: Audit Log (Optional)  
    - Configure Airtable credentials and database "health-assistant-logs"  
    - Map fields: sessionId, queryType, isEmergency, timestamp  
    - Connect output of Format Health Response node  
    - Disable by default; enable if logging is needed

11. **Add Sticky Note Nodes for Documentation**  
    - Create notes with content for:  
      - AI Health Assistant overview and disclaimer  
      - Safety protocols and emergency detection  
      - Symptom categories supported  
      - Configuration requirements and compliance  
      - Example user interactions  
      - Features roadmap  
      - Integration options  

12. **Connect Nodes in Sequence:**  
    Health Query Webhook → Safety Check & Categorization → Emergency Router →  
    → Emergency Response (true branch) → Merge → Compile Final Response → Send Response  
    → Health Information AI (false branch) → Format Health Response → Merge

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                    | Context or Link                                                                                                                       |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| **IMPORTANT DISCLAIMER:** This tool provides general health information only and is NOT a substitute for professional medical advice, diagnosis, or treatment. Always consult qualified healthcare providers for medical concerns.                 | Sticky Note: AI Health Assistant Disclaimer                                                                                          |
| **Emergency Detection Protocols:** Includes immediate call instructions for chest pain, stroke, breathing difficulty, severe bleeding, unconsciousness. Automatically escalates detected emergencies. Privacy-first design included.              | Sticky Note1: Safety protocols                                                                                                       |
| **Supported Symptom Categories:** General, Respiratory, Digestive, Mental Health, Skin-related symptoms.                                                                                                                                       | Sticky Note2: Symptom categories                                                                                                     |
| **Configuration Requirements:** OpenAI API key, Emergency contacts database, Disclaimer acceptance, Language settings. Optional: Medical database API, Translation, SMS notifications, Appointment systems. Compliance reminders included.         | Sticky Note3: Configuration checklist                                                                                                |
| **Example Interactions:** Sample user queries and expected bot responses for symptoms, emergencies, medication reminders, rashes, mental health concerns.                                                                                      | Sticky Note4: Example interactions                                                                                                   |
| **Feature Roadmap:** Current features include symptom info, emergency detection, general guidance, multi-language support. Planned: medicine interactions, appointment booking, health tracking, telemedicine prep, insurance info, etc.         | Sticky Note5: Features roadmap                                                                                                       |
| **Integration Options:** Channels such as web chat, WhatsApp, Telegram, SMS, voice assistants; Healthcare systems integration like EHR, pharmacies, telemedicine; Data sources including medical databases and provider directories.              | Sticky Note6: Integration options                                                                                                   |
| **Health Resources Links:**  
  - Find a doctor: https://doctor.webmd.com/  
  - Symptom checker: https://www.mayoclinic.org/symptom-checker/  
  - Mental health helpline: https://www.samhsa.gov/find-help/national-helpline  
  - Poison control US: 1-800-222-1222                                                                                                                                | Included in Format Health Response node                                                                                             |

---

**Disclaimer:** The provided workflow is designed solely for informational and educational purposes. It does not replace professional medical advice or emergency services. Users should always seek licensed healthcare providers for diagnosis and treatment.