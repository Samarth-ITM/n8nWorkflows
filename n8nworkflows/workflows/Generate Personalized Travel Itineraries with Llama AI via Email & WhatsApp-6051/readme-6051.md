Generate Personalized Travel Itineraries with Llama AI via Email & WhatsApp

https://n8nworkflows.xyz/workflows/generate-personalized-travel-itineraries-with-llama-ai-via-email---whatsapp-6051


# Generate Personalized Travel Itineraries with Llama AI via Email & WhatsApp

### 1. Workflow Overview

This workflow automates the generation and delivery of personalized travel itineraries based on user messages received via email or WhatsApp. It leverages an AI language model (Llama) to interpret user requests, create warm, detailed daily travel plans including activities, transport tips, hotel suggestions, and flight info, then sends the itinerary back to the user on the same platform they used (email or WhatsApp).

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Captures user travel requests from two sources — email inbox and WhatsApp messages.
- **1.2 AI Processing:** Sends the user message to a Llama AI agent configured to generate personalized travel itineraries in a friendly style.
- **1.3 Data Preparation:** Extracts and formats key information from the AI response and user metadata (sender email, subject).
- **1.4 Conditional Routing:** Determines whether the response should be sent via email or WhatsApp based on the original input source.
- **1.5 Output Delivery:** Sends the generated itinerary back to the user through the appropriate channel (email or WhatsApp).

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block captures travel itinerary requests from users via two channels: email and WhatsApp. It listens for new incoming emails or WhatsApp messages and triggers the workflow accordingly.

**Nodes Involved:**  
- Get Query from Email  
- Get Query from WhatsApp  

**Node Details:**

- **Get Query from Email**  
  - *Type:* Email Read (IMAP)  
  - *Role:* Polls an email inbox to fetch new emails containing user requests.  
  - *Configuration:* Uses IMAP credentials to connect to the email server; defaults likely to fetching unread emails.  
  - *Expressions/Variables:* Outputs the entire email JSON (including from, subject, body).  
  - *Input/Output:* No input; outputs email data to "Itinerary Creator Agent".  
  - *Failures:* Possible auth errors with IMAP login, network timeouts, empty inbox.  

- **Get Query from WhatsApp**  
  - *Type:* WhatsApp Trigger  
  - *Role:* Listens to incoming WhatsApp messages via webhook.  
  - *Configuration:* Uses WhatsApp API credentials; triggers on new messages.  
  - *Expressions/Variables:* Outputs message JSON including sender and message text.  
  - *Input/Output:* No input; outputs message data to "Itinerary Creator Agent".  
  - *Failures:* Webhook misconfiguration, API auth failures, message format variations.  

---

#### 2.2 AI Processing

**Overview:**  
Receives the user’s message text and invokes the Llama AI model to create a personalized travel itinerary in a friendly, human tone with a strict output format.

**Nodes Involved:**  
- Itinerary Creator Agent  
- Agent  

**Node Details:**

- **Itinerary Creator Agent**  
  - *Type:* LangChain Chain LLM integration  
  - *Role:* Prepares and formats the user’s message as input text for the language model.  
  - *Configuration:* Passes the text extracted from the incoming email or WhatsApp message (`{{$json.textPlain}}`) to the AI prompt.  
  - *Expressions:* Defines a detailed instruction prompt specifying the exact format, tone, and content structure for the itinerary output (greeting, day-wise breakdown, transport, hotel, flight info).  
  - *Input/Output:* Inputs raw user message text; outputs AI-generated itinerary text.  
  - *Failures:* Prompt misinterpretation, API call failures, improper input text format, rate limits.  
  - *Version:* LangChain node v1.6.  

- **Agent**  
  - *Type:* LLM Ollama Node  
  - *Role:* Executes the AI model inference using Ollama’s llama3.2-16000 model.  
  - *Configuration:* Calls the Llama 3.2 model with no additional options; uses configured Ollama API credentials.  
  - *Input/Output:* Receives prompt from "Itinerary Creator Agent", returns generated itinerary text.  
  - *Failures:* API auth errors, model unavailability, network issues.  

---

#### 2.3 Data Preparation

**Overview:**  
Extracts relevant metadata from the original email and formats the AI response text for routing and final delivery.

**Nodes Involved:**  
- Check Proper Data  

**Node Details:**

- **Check Proper Data**  
  - *Type:* Set node  
  - *Role:* Assigns and prepares variables such as sender email ("from"), email subject (prefixed with "Re:"), and the generated itinerary text for downstream routing.  
  - *Configuration:*  
    - `from` is set from the "Get Query from Email" node output (`$('Get Query from Email').first().json.from`).  
    - `subject` prepends "Re:" to the original email subject.  
    - `text` holds the AI-generated itinerary text.  
  - *Input/Output:* Inputs AI response and original email data; outputs structured data for routing.  
  - *Failures:* Missing email data if input was from WhatsApp, potential undefined expressions.  
  - *Version:* v3.4.  

---

#### 2.4 Conditional Routing

**Overview:**  
Determines the platform to which the itinerary should be sent, based on the presence of an email sender address. Routes either to email sending or WhatsApp sending nodes.

**Nodes Involved:**  
- Check where to send Answer  

**Node Details:**

- **Check where to send Answer**  
  - *Type:* If node  
  - *Role:* Checks if the incoming request came from an email by verifying if the sender email field is non-empty.  
  - *Configuration:* Condition: `$('Get Query from Email').first().json.from` is non-empty.  
  - *Input/Output:*  
    - True branch: routes to "Sending Itinery from Email".  
    - False branch: routes to "Send Itinery from message" (WhatsApp).  
  - *Failures:* Misrouting if data inconsistent; expression failures if "Get Query from Email" node output missing.  

---

#### 2.5 Output Delivery

**Overview:**  
Sends the generated itinerary back to the requesting user on the correct platform.

**Nodes Involved:**  
- Sending Itinery from Email  
- Send Itinery from message  

**Node Details:**

- **Sending Itinery from Email**  
  - *Type:* Email Send (SMTP)  
  - *Role:* Sends the generated itinerary as a plain text email reply to the user’s email address.  
  - *Configuration:*  
    - `toEmail` set to user email (`{{$json.from}}`).  
    - `fromEmail` hardcoded as "XYZ@gmail.com".  
    - Subject prepended with "Re: " plus original subject.  
    - Email format plain text.  
    - Uses SMTP credentials.  
  - *Input/Output:* Inputs structured data with text, subject, recipient; outputs success/failure status.  
  - *Failures:* SMTP auth errors, invalid recipient email, network issues.  

- **Send Itinery from message**  
  - *Type:* WhatsApp Send  
  - *Role:* Sends the itinerary text as a WhatsApp message back to the user’s phone number.  
  - *Configuration:*  
    - `textBody` is the AI itinerary text.  
    - `operation` set to "send".  
    - `phoneNumberId` and `recipientPhoneNumber` hardcoded placeholders (e.g., "+918888888888", "+919999999999").  
    - Uses WhatsApp API credentials.  
  - *Input/Output:* Inputs text message; outputs send confirmation.  
  - *Failures:* Incorrect phone numbers, API auth errors, message formatting issues.  

---

### 3. Summary Table

| Node Name                 | Node Type                      | Functional Role                 | Input Node(s)           | Output Node(s)                | Sticky Note                                                                                                                                                                  |
|---------------------------|--------------------------------|--------------------------------|-------------------------|------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Get Query from Email       | Email Read (IMAP)              | Receive user request via email | None                    | Itinerary Creator Agent       |                                                                                                                                                                              |
| Get Query from WhatsApp    | WhatsApp Trigger               | Receive user request via WhatsApp | None                  | Itinerary Creator Agent       |                                                                                                                                                                              |
| Itinerary Creator Agent    | LangChain Chain LLM            | Prepare prompt for AI itinerary | Get Query from Email, Get Query from WhatsApp | Agent                        |                                                                                                                                                                              |
| Agent                     | LLM Ollama                    | Generate personalized itinerary | Itinerary Creator Agent | Check Proper Data             |                                                                                                                                                                              |
| Check Proper Data          | Set                           | Format and assign output data   | Agent                   | Check where to send Answer    |                                                                                                                                                                              |
| Check where to send Answer | If                            | Route response to email or WhatsApp | Check Proper Data       | Sending Itinery from Email, Send Itinery from message |                                                                                                                                                                              |
| Sending Itinery from Email | Email Send (SMTP)             | Send itinerary via email        | Check where to send Answer (true) | None                         |                                                                                                                                                                              |
| Send Itinery from message  | WhatsApp Send                 | Send itinerary via WhatsApp     | Check where to send Answer (false) | None                         |                                                                                                                                                                              |
| Sticky Note               | Sticky Note                   | Documentation note              | None                    | None                         | This workflow automatically creates friendly, personalized travel itineraries based on messages received via email or WhatsApp. Whether a user says "I want to go to Dubai with friends for 5 days" or similar... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Email Input Node**  
   - Add *Email Read (IMAP)* node named `Get Query from Email`.  
   - Configure with IMAP credentials to monitor the inbox for new emails.  
   - No input connections needed.

2. **Create WhatsApp Input Node**  
   - Add *WhatsApp Trigger* node named `Get Query from WhatsApp`.  
   - Configure with WhatsApp API credentials and webhook enabled to listen for new incoming messages.

3. **Add LangChain Chain LLM Node**  
   - Add node `Itinerary Creator Agent` of type *LangChain Chain LLM*.  
   - Set parameter `text` to `={{ $json.textPlain }}` to pass user message text.  
   - Define messages with a detailed instruction prompt:  
     - Instruct to create a personalized travel itinerary with warm greeting, day-wise breakdown, hotel, transport, and flight info.  
     - Specify output format strictly as per example.  
   - Connect outputs of both email and WhatsApp input nodes to this node.

4. **Add LLM Ollama Node**  
   - Add node `Agent` of type *LLM Ollama*.  
   - Select model `llama3.2-16000:latest`.  
   - Attach configured Ollama API credentials.  
   - Connect `Itinerary Creator Agent` output to this node.

5. **Add Data Preparation Node**  
   - Add *Set* node named `Check Proper Data`.  
   - Assign the following fields:  
     - `from` = `={{ $('Get Query from Email').first().json.from }}`  
     - `subject` = `=Re: {{ $('Get Query from Email').first().json.subject }}`  
     - `text` = AI-generated itinerary from previous node (`$json.text`).  
   - Connect `Agent` output to this node.

6. **Add Conditional Routing Node**  
   - Add *If* node named `Check where to send Answer`.  
   - Condition: Check if `$('Get Query from Email').first().json.from` is not empty.  
   - Connect `Check Proper Data` output to this node.

7. **Create Email Sending Node**  
   - Add *Email Send (SMTP)* node named `Sending Itinery from Email`.  
   - Set `toEmail` to `={{ $json.from }}`.  
   - Set `subject` to `={{ $json.subject }}`.  
   - Set `text` to `={{ $json.text }}`.  
   - Use SMTP credentials for sending.  
   - Connect the *true* branch of the If node to this node.

8. **Create WhatsApp Sending Node**  
   - Add *WhatsApp Send* node named `Send Itinery from message`.  
   - Set `textBody` to `={{ $json.text }}`.  
   - Configure `phoneNumberId` and `recipientPhoneNumber` appropriately (dynamic mapping to sender number recommended).  
   - Use WhatsApp API credentials.  
   - Connect the *false* branch of the If node to this node.

9. **Add Sticky Note**  
   - Add a *Sticky Note* node with the content describing the workflow purpose and capabilities.

10. **Check Connections and Activate Workflow**  
    - Verify all connections follow the diagram: Email & WhatsApp inputs → AI prompt → AI model → Data preparation → Conditional routing → Email or WhatsApp sending.  
    - Activate workflow for real-time operation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                               | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow automatically creates friendly, personalized travel itineraries based on messages received via email or WhatsApp. It saves time and delivers ready-to-send itineraries without manual effort.                                  | Sticky note in workflow overview section                                                         |
| Ollama LLM model "llama3.2-16000:latest" is used for AI generation. Ensure correct API credentials and model availability.                                                                                                                | Agent node configuration                                                                           |
| Email SMTP and IMAP credentials must be properly configured with correct host, port, username, and password for seamless operation.                                                                                                        | Email nodes configuration                                                                          |
| WhatsApp trigger and send nodes require WhatsApp Business API credentials and correctly configured webhooks with valid phone numbers.                                                                                                     | WhatsApp node configuration                                                                        |
| The AI prompt instructs strict output formatting; errors in prompt or user input may cause unexpected results. Validate input text length and format for reliable responses.                                                              | Itinerary Creator Agent node prompt detail                                                        |

---

*Disclaimer:* This document is generated exclusively from an n8n workflow automation and complies with content policies. All processed data is legal and publicly acceptable.