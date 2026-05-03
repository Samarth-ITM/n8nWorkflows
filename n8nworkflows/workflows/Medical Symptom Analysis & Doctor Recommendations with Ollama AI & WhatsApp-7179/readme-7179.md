Medical Symptom Analysis & Doctor Recommendations with Ollama AI & WhatsApp

https://n8nworkflows.xyz/workflows/medical-symptom-analysis---doctor-recommendations-with-ollama-ai---whatsapp-7179


# Medical Symptom Analysis & Doctor Recommendations with Ollama AI & WhatsApp

### 1. Workflow Overview

This workflow, titled **"Medical Symptom Analysis & Doctor Recommendations with Ollama AI & WhatsApp"**, serves as a comprehensive AI-driven medical assistant designed to:

- Receive patient symptom data via a webhook.
- Analyze symptoms using an AI medical model to assess urgency and suggest possible conditions.
- Detect emergencies and provide immediate response instructions.
- Find and recommend relevant doctors based on the analysis and patient location.
- Deliver results and recommendations via WhatsApp and email.
- Respond to the user with a structured success message.

The workflow is logically divided into the following blocks:

**1.1 Input Reception and AI Symptom Analysis**  
Receives patient symptoms and details through a webhook and runs them through an AI medical assistant for analysis.

**1.2 Medical Analysis Parsing and Emergency Check**  
Parses AI output into structured JSON, checks if the case is an emergency, and branches accordingly.

**1.3 Doctor Lookup and Medical Advice Formatting**  
Queries a doctor directory to find specialists matching the AI's recommendations and composes a detailed medical response.

**1.4 Communication Preparation and Delivery**  
Prepares messages for WhatsApp and email, checks contact availability, and sends notifications.

**1.5 Workflow Response and Logging**  
Sends a success response back to the webhook caller confirming delivery.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and AI Symptom Analysis

- **Overview:**  
This block collects patient symptom data via a webhook and sends the data to an AI medical model for symptom analysis.

- **Nodes Involved:**  
  - Symptom Input Webhook  
  - Medical AI Model  
  - AI Symptom Analysis  

- **Node Details:**

  - **Symptom Input Webhook**  
    - *Type:* Webhook  
    - *Role:* Entry point that listens for HTTP POST requests containing patient symptom data in JSON format.  
    - *Configuration:* Path set to a unique webhook ID; accepts POST method; response mode set to wait for response node.  
    - *Input/Output:* Receives JSON with symptoms, age, gender, duration, phone, email, location. Outputs data to AI Symptom Analysis.  
    - *Edge cases:* Missing or malformed input JSON; webhook URL security considerations.

  - **Medical AI Model**  
    - *Type:* Ollama AI language model node.  
    - *Role:* Provides the underlying AI model (llama3.2-16000:latest) used for symptom analysis.  
    - *Configuration:* Uses Ollama API credentials; no extra options.  
    - *Input/Output:* Feeds model into AI Symptom Analysis node as the language model backend.  
    - *Edge cases:* API authentication failures, model unavailability, latency.

  - **AI Symptom Analysis**  
    - *Type:* Langchain Agent node  
    - *Role:* Runs the AI prompt with patient symptoms and details to generate a JSON-formatted medical assessment.  
    - *Configuration:*  
      - Text input uses expression: `{{$json.body.symptoms}}`.  
      - System message instructs AI to analyze symptoms and output a JSON with urgency, conditions, specialist needed, severity, recommended actions, and red flags.  
      - Prompt type: define (fixed system message).  
    - *Input/Output:* Receives webhook JSON and model from Medical AI Model; outputs AI response text.  
    - *Edge cases:* AI output not in expected JSON format, timeout, prompt injection risks.

---

#### 2.2 Medical Analysis Parsing and Emergency Check

- **Overview:**  
Parses the AI response into structured JSON, supplements with patient data, then checks if the urgency level is "emergency" to branch workflow.

- **Nodes Involved:**  
  - Parse Medical Analysis  
  - Check Emergency  

- **Node Details:**

  - **Parse Medical Analysis**  
    - *Type:* Code node  
    - *Role:* Parses AI text output to extract JSON medical assessment; if parsing fails, provides a fallback generic response.  
    - *Configuration:*  
      - JavaScript extracts substring between braces to parse JSON.  
      - Merges original patient data with parsed analysis and timestamp.  
    - *Input/Output:* Input from AI Symptom Analysis; outputs enriched patient data with analysis.  
    - *Edge cases:* AI returns malformed JSON, parsing errors, fallback triggers.

  - **Check Emergency**  
    - *Type:* If node  
    - *Role:* Checks if urgency level equals "emergency" to decide next step.  
    - *Configuration:* String condition comparing `$json.analysis.urgency_level` to "emergency".  
    - *Input/Output:* Input from Parse Medical Analysis.  
    - *Output:*  
      - True branch: goes to Create Emergency Response.  
      - False branch: goes to Find Doctors.  
    - *Edge cases:* Missing urgency_level key or unexpected values.

---

#### 2.3 Doctor Lookup and Medical Advice Formatting

- **Overview:**  
Finds doctors matching the needed specialty in the patient's location and formats a full medical advice message including doctor contacts.

- **Nodes Involved:**  
  - Find Doctors  
  - Format Medical Response  

- **Node Details:**

  - **Find Doctors**  
    - *Type:* HTTP Request node  
    - *Role:* Queries BetterDoctor API for doctors of the specialist type recommended by AI, filtered by location.  
    - *Configuration:*  
      - URL: BetterDoctor API endpoint.  
      - Query parameters: specialty mapped from AI specialist needed, location (default New York, NY), limit 5, user_key to be replaced by actual API key.  
    - *Input/Output:* Input from Check Emergency false branch; outputs JSON list of doctors.  
    - *Edge cases:* API key missing or invalid, API rate limits, no doctors found, network errors.

  - **Format Medical Response**  
    - *Type:* Code node  
    - *Role:* Combines parsed analysis and doctor data to generate a comprehensive message with urgency, possible conditions, recommended specialist, assessment, actions, red flags, and doctor contacts.  
    - *Configuration:*  
      - Formats doctor info into readable list with name, specialty, phone, address.  
      - Adds emojis to urgency levels.  
      - Includes disclaimer emphasizing AI nature and emergency instructions.  
    - *Input/Output:* Inputs from Find Doctors and Parse Medical Analysis; outputs formatted message with patient and doctor info for messaging.  
    - *Edge cases:* Missing or incomplete doctor data, empty doctor list.

---

#### 2.4 Communication Preparation and Delivery

- **Overview:**  
Prepares the message payload for WhatsApp, checks if phone number is available, and sends responses via WhatsApp and/or email.

- **Nodes Involved:**  
  - Check WhatsApp Available  
  - Prepare WhatsApp Message  
  - Send WhatsApp Response  
  - Send Email Response  

- **Node Details:**

  - **Check WhatsApp Available**  
    - *Type:* If node  
    - *Role:* Checks if a phone number is provided to decide if WhatsApp message should be sent.  
    - *Configuration:* String condition: phone_number is not empty.  
    - *Input/Output:* Input from Format Medical Response or Create Emergency Response.  
    - *Output:*  
      - True: goes to Prepare WhatsApp Message.  
      - False: goes to Send Email Response.  
    - *Edge cases:* Phone number format invalid or missing.

  - **Prepare WhatsApp Message**  
    - *Type:* Code node  
    - *Role:* Constructs WhatsApp messaging payload with recipient number and message text.  
    - *Configuration:* Builds JSON object with messaging_product "whatsapp", recipient phone, message type "text", and body containing the formatted medical message.  
    - *Input/Output:* Input from Check WhatsApp Available true branch; outputs WhatsApp API payload.  
    - *Edge cases:* Message size limits, phone number formatting.

  - **Send WhatsApp Response**  
    - *Type:* HTTP Request node  
    - *Role:* Sends the prepared WhatsApp message to Meta's WhatsApp Business API endpoint.  
    - *Configuration:*  
      - URL includes placeholder for phone number ID.  
      - Requires proper authorization headers (not shown in JSON).  
    - *Input/Output:* Input from Prepare WhatsApp Message; outputs API response.  
    - *Edge cases:* Authentication errors, API limits, network issues.

  - **Send Email Response**  
    - *Type:* Email Send node  
    - *Role:* Sends the formatted medical message to patient's email as HTML, transforming line breaks and removing markdown syntax.  
    - *Configuration:*  
      - Subject: "🏥 Your Medical Assessment & Doctor Recommendations".  
      - From: medical-bot@yourdomain.com  
      - SMTP credentials configured.  
    - *Input/Output:* Input from Check WhatsApp Available false branch; outputs email send status.  
    - *Edge cases:* SMTP authentication failure, invalid email address.

---

#### 2.5 Workflow Response and Logging

- **Overview:**  
Sends a success JSON response to the original webhook caller confirming the medical assessment was sent successfully.

- **Nodes Involved:**  
  - Success Response  

- **Node Details:**

  - **Success Response**  
    - *Type:* Respond to Webhook node  
    - *Role:* Returns a JSON success object including status, message, urgency level, and timestamp to the HTTP client that triggered the webhook.  
    - *Configuration:* Responds with JSON body containing status "success" and echoes urgency level from patient data.  
    - *Input/Output:* Input from Send Email Response or Send WhatsApp Response; outputs HTTP response.  
    - *Edge cases:* Timing out if previous nodes delayed, missing data fields.

---

### 3. Summary Table

| Node Name              | Node Type                       | Functional Role                                  | Input Node(s)                  | Output Node(s)                    | Sticky Note                                                                                                           |
|------------------------|--------------------------------|-------------------------------------------------|-------------------------------|----------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Symptom Input Webhook   | Webhook                        | Receives symptom input JSON via HTTP POST       | -                             | AI Symptom Analysis              |                                                                                                                       |
| Medical AI Model        | Ollama AI Language Model       | Provides AI model for symptom analysis           | -                             | AI Symptom Analysis (LM)         |                                                                                                                       |
| AI Symptom Analysis     | Langchain Agent                | Analyzes symptoms and outputs medical JSON       | Symptom Input Webhook, Medical AI Model | Parse Medical Analysis           |                                                                                                                       |
| Parse Medical Analysis  | Code                          | Parses AI output into structured medical data    | AI Symptom Analysis            | Check Emergency                  |                                                                                                                       |
| Check Emergency         | If                            | Checks if urgency level is "emergency"           | Parse Medical Analysis         | Create Emergency Response, Find Doctors |                                                                                                                       |
| Create Emergency Response | Code                        | Generates urgent medical alert message            | Check Emergency (true)         | Check WhatsApp Available         |                                                                                                                       |
| Find Doctors            | HTTP Request                  | Queries BetterDoctor API for specialists          | Check Emergency (false)        | Format Medical Response          |                                                                                                                       |
| Format Medical Response | Code                          | Formats medical advice and doctor contact info    | Find Doctors                  | Check WhatsApp Available         |                                                                                                                       |
| Check WhatsApp Available | If                            | Checks for presence of phone number               | Format Medical Response, Create Emergency Response | Prepare WhatsApp Message, Send Email Response |                                                                                                                       |
| Prepare WhatsApp Message | Code                          | Builds WhatsApp message payload                    | Check WhatsApp Available (true) | Send WhatsApp Response           |                                                                                                                       |
| Send WhatsApp Response  | HTTP Request                  | Sends message via WhatsApp Business API           | Prepare WhatsApp Message       | Success Response                 |                                                                                                                       |
| Send Email Response     | Email Send                    | Sends medical advice via email                     | Check WhatsApp Available (false) | Success Response                 |                                                                                                                       |
| Success Response        | Respond to Webhook            | Sends confirmation JSON response to webhook caller | Send WhatsApp Response, Send Email Response | -                              |                                                                                                                       |
| Medical Bot Overview    | Sticky Note                   | Describes workflow purpose and features           | -                             | -                              | ## 🏥 AI Symptom Checker & Doctor Suggestion\n\nComplete Medical Assistant Workflow:\n\n🔍 Symptom Analysis...\n✅ HIPAA-compliant data handling\n✅ Professional medical disclaimers\n✅ Emergency response protocols |
| Setup Guide            | Sticky Note                   | Lists setup requirements and input JSON example   | -                             | -                              | ## 🛠️ Setup Requirements:\n\nAPIs & Services:\n1. BetterDoctor API\n2. WhatsApp Business API\n3. SMTP Email\n4. Ollama\n\nInput Format example and test URL |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "Symptom Input Webhook"**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Unique identifier (e.g., "symptom-checker-webhook")  
   - Response Mode: "Response Node"  
   - Purpose: Accept JSON with fields: symptoms, age, gender, duration, phone, email, location.

2. **Add Ollama AI Model Node: "Medical AI Model"**  
   - Type: Ollama LM Chat node (langchain)  
   - Model: "llama3.2-16000:latest"  
   - Credentials: Configure Ollama API credentials  
   - No additional options needed.

3. **Add Langchain Agent Node: "AI Symptom Analysis"**  
   - Type: Langchain Agent  
   - Text parameter: Expression `{{$json.body.symptoms}}`  
   - System Message:  
     ```
     You are a medical AI assistant that analyzes symptoms and provides preliminary assessments. IMPORTANT: Always emphasize that this is not a substitute for professional medical advice.

     Analyze the symptoms and provide a JSON response with:
     {
       "urgency_level": "low/medium/high/emergency",
       "possible_conditions": ["condition1", "condition2", "condition3"],
       "specialist_needed": "General Practitioner/Cardiologist/Dermatologist/Neurologist/Orthopedist/Gastroenterologist/Pulmonologist/Emergency",
       "severity_assessment": "brief description",
       "recommended_actions": ["action1", "action2"],
       "red_flags": ["warning sign if any"]
     }

     Symptoms to analyze: {{ $json.body.symptoms }}

     Patient details:
     - Age: {{ $json.body.age || 'Not provided' }}
     - Gender: {{ $json.body.gender || 'Not provided' }}
     - Duration: {{ $json.body.duration || 'Not provided' }}
     - Additional info: {{ $json.body.additional_info || 'None' }}
     ```
   - Prompt type: Define  
   - Connect "Symptom Input Webhook" and "Medical AI Model" as inputs.

4. **Add Code Node: "Parse Medical Analysis"**  
   - Type: Code  
   - JavaScript:  
     - Extract JSON substring from AI response text.  
     - Parse JSON or fallback to generic analysis.  
     - Append original webhook data and timestamp.  
   - Input: "AI Symptom Analysis"

5. **Add If Node: "Check Emergency"**  
   - Condition: `$json.analysis.urgency_level` equals "emergency" (string comparison)  
   - Input: "Parse Medical Analysis"

6. **Add Code Node: "Create Emergency Response"**  
   - Type: Code  
   - JavaScript:  
     - Compose urgent medical message with instructions to call emergency services.  
     - Include symptoms and disclaimers.  
   - Input: "Check Emergency" true branch

7. **Add HTTP Request Node: "Find Doctors"**  
   - Type: HTTP Request  
   - URL: `https://api.betterdoctor.com/2016-03-01/doctors`  
   - Query parameters:  
     - specialty_uid: map from `$json.analysis.specialist_needed` to BetterDoctor specialty string with fallback to "family-practitioner"  
     - location: `$json.location` or default "New York, NY"  
     - limit: 5  
     - user_key: your BetterDoctor API key  
   - Input: "Check Emergency" false branch

8. **Add Code Node: "Format Medical Response"**  
   - Type: Code  
   - JavaScript:  
     - Format patient symptoms, urgency (with emoji), conditions, specialist, assessment, recommended actions, red flags.  
     - Format doctor list with name, specialty, phone, address for up to 3 doctors.  
     - Append disclaimers and emergency instructions.  
   - Inputs: "Find Doctors" and "Parse Medical Analysis"

9. **Add If Node: "Check WhatsApp Available"**  
   - Condition: `phone_number` is not empty  
   - Input: from "Create Emergency Response" and "Format Medical Response"

10. **Add Code Node: "Prepare WhatsApp Message"**  
    - Type: Code  
    - JavaScript:  
      - Build WhatsApp API payload with messaging product, recipient phone, message text.  
    - Input: "Check WhatsApp Available" true branch

11. **Add HTTP Request Node: "Send WhatsApp Response"**  
    - Type: HTTP Request  
    - URL: `https://graph.facebook.com/v17.0/YOUR_PHONE_NUMBER_ID/messages` (replace YOUR_PHONE_NUMBER_ID)  
    - Configure authorization headers for WhatsApp Business API  
    - Input: "Prepare WhatsApp Message"

12. **Add Email Send Node: "Send Email Response"**  
    - Type: Email Send  
    - Subject: "🏥 Your Medical Assessment & Doctor Recommendations"  
    - From Email: medical-bot@yourdomain.com  
    - To Email: `$json.email`  
    - HTML Body: Convert message text line breaks to `<br>`, remove markdown asterisks  
    - Credentials: SMTP setup with valid credentials  
    - Input: "Check WhatsApp Available" false branch

13. **Add Respond to Webhook Node: "Success Response"**  
    - Type: Respond to Webhook  
    - Respond with JSON:  
      ```
      {
        "status": "success",
        "message": "Medical assessment sent successfully",
        "urgency_level": $json.patient_data ? $json.patient_data.analysis.urgency_level : "unknown",
        "timestamp": new Date().toISOString()
      }
      ```
    - Inputs: From both "Send WhatsApp Response" and "Send Email Response"

14. **Connect Nodes According to Logical Flow:**  
    - Symptom Input Webhook -> AI Symptom Analysis  
    - Medical AI Model -> AI Symptom Analysis (model input)  
    - AI Symptom Analysis -> Parse Medical Analysis  
    - Parse Medical Analysis -> Check Emergency  
    - Check Emergency true -> Create Emergency Response -> Check WhatsApp Available  
    - Check Emergency false -> Find Doctors -> Format Medical Response -> Check WhatsApp Available  
    - Check WhatsApp Available true -> Prepare WhatsApp Message -> Send WhatsApp Response -> Success Response  
    - Check WhatsApp Available false -> Send Email Response -> Success Response

15. **Additional Setup:**  
    - Configure credentials for Ollama API, BetterDoctor API key, WhatsApp Business API (Meta Developer), and SMTP email.  
    - Ensure webhook URL is exposed publicly and secured as per needs.  
    - Replace placeholders with real API keys and credentials.  
    - Test input JSON format per Setup Guide below.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| **AI Symptom Checker & Doctor Suggestion**: Complete Medical Assistant Workflow includes symptom analysis, doctor matching, multi-channel delivery, emergency detection, and data logging. HIPAA-compliant data handling and professional disclaimers included. | See "Medical Bot Overview" sticky note in workflow |
| **Setup Requirements**: Requires BetterDoctor API (free tier), WhatsApp Business API (Meta Developer account), SMTP email service, Ollama local AI model. Input JSON format example and test webhook URL provided. | See "Setup Guide" sticky note in workflow |
| BetterDoctor API Documentation: https://developer.betterdoctor.com/ | For doctor lookup API usage |
| WhatsApp Business API Documentation: https://developers.facebook.com/docs/whatsapp/ | For WhatsApp messaging integration |
| Ollama AI Documentation: https://ollama.com/docs | For AI model setup and usage |
| HIPAA Compliance Reminder: Ensure all patient data transmission and storage comply with healthcare data protection standards. | General best practice |

---

This detailed analysis and reconstruction guide provides a clear, stepwise roadmap for understanding, reproducing, and maintaining the "Medical Symptom Analysis & Doctor Recommendations with Ollama AI & WhatsApp" n8n workflow.