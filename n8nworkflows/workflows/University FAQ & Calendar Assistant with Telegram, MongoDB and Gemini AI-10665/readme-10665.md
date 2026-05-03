University FAQ & Calendar Assistant with Telegram, MongoDB and Gemini AI

https://n8nworkflows.xyz/workflows/university-faq---calendar-assistant-with-telegram--mongodb-and-gemini-ai-10665


# University FAQ & Calendar Assistant with Telegram, MongoDB and Gemini AI

### 1. Workflow Overview

This workflow, titled **"University FAQ & Calendar Assistant with Telegram, MongoDB and Gemini AI"**, is designed as an intelligent Telegram chatbot assistant for ESPOL (Escuela Superior Politécnica del Litoral). Its core purpose is to provide students and users with quick, accurate answers to frequently asked questions (FAQs) related to the university and up-to-date academic calendar information. Additionally, it manages user feedback and sends weekly automated announcements.

The workflow integrates several data sources including MongoDB (for FAQs), Google Sheets (for academic calendar data), and Google Gemini AI (for natural language processing and response generation). It also leverages Telegram’s messaging and callback features for interaction.

**Target Use Cases:**
- Answering student queries about university services, schedules, events, and contacts.
- Providing detailed academic calendar information upon request.
- Collecting and storing user feedback.
- Sending periodic automated personalized announcements about upcoming academic activities.

**Logical Blocks:**

- **1.1 Telegram Input & Command Classification:** Handles all incoming Telegram messages and callback queries, classifies commands, and routes messages accordingly.
- **1.2 FAQ Retrieval & AI Response:** Searches MongoDB for relevant FAQs based on user query, builds prompts, queries Google Gemini AI for answers, and sends responses.
- **1.3 Academic Calendar Query Processing:** Detects calendar-related questions, queries Google Sheets for calendar data, builds prompts, uses Gemini AI for response generation, and sends replies.
- **1.4 User Feedback Handling:** Manages user feedback collection, sends acknowledgement messages, and stores feedback in Google Sheets.
- **1.5 Welcome and Help Messages:** Sends welcome messages and help instructions or suggested questions upon user initiation.
- **1.6 Weekly Announcement System:** Weekly scraping of the academic calendar, processing of data, generation of announcements with AI, and sending messages plus voice notes to users.
- **1.7 Data Scraping and Updating:** Scrapes academic calendar data from the university website, processes and updates Google Sheets with fresh calendar information.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Telegram Input & Command Classification

**Overview:**  
This block listens for any Telegram interaction (messages or callback queries), classifies the input into commands, feedback, or normal queries, and routes the flow accordingly.

**Nodes Involved:**  
- Telegram Trigger - Inicio  
- Switch  
- Comandos1

**Node Details:**

- **Telegram Trigger - Inicio**  
  - **Type:** Telegram Trigger  
  - **Role:** Entry point to capture all user messages and callback queries.  
  - **Configuration:** Listens to message, callback_query, and all update types. Uses ESPOLBot Telegram API credentials.  
  - **Inputs:** Incoming Telegram updates.  
  - **Outputs:** JSON with message and callback data.  
  - **Failures:** Network or webhook errors; Telegram API quota limits.

- **Switch**  
  - **Type:** Switch (conditional routing)  
  - **Role:** Classifies input into callback query, feedback messages, commands (starting with '/'), or fallback message.  
  - **Configuration:** Conditions check whether input has callback_query, whether message is a feedback reply, or if text starts with '/'.  
  - **Inputs:** Output from Telegram Trigger.  
  - **Outputs:** Routes to Respuesta Usuario1, Guardado en cvs Feedback, Comandos1, or Detector_Calendario_Pre.  
  - **Failures:** Expression evaluation failures, unhandled input types.

- **Comandos1**  
  - **Type:** Switch  
  - **Role:** Routes recognized Telegram commands (/help, /faqs, /contact, /events, /feedback, /start) to appropriate response nodes.  
  - **Configuration:** Checks exact equality of incoming message text for each command key.  
  - **Inputs:** Routed from Switch node.  
  - **Outputs:** To nodes like Mensaje Help, FAQs, Respuesta Contact, Respuesta Events, Inicio - Feedback, BIENVENIDA, and OBTENER_ID1.  
  - **Failures:** None specific unless command text missing.

---

#### 1.2 FAQ Retrieval & AI Response

**Overview:**  
For non-calendar queries, this block fetches relevant FAQs from MongoDB, constructs a prompt incorporating FAQ context, sends it to Google Gemini AI for answer generation, cleans the AI response, and sends it back to the user.

**Nodes Involved:**  
- Leer FAQs de MongoDB  
- Buscar_FAQs_Relevantes  
- Construir_Prompt_Gemini1  
- Mensaje de Gemini  
- Limpiar texto Gemini  
- Enviar Respuesta sobre faqs

**Node Details:**

- **Leer FAQs de MongoDB**  
  - **Type:** MongoDB node  
  - **Role:** Retrieves the full FAQ collection from “espol_faqs” collection.  
  - **Config:** Uses MongoDB credentials; no query filter (fetches all items).  
  - **Failure:** DB connection errors, empty collection.

- **Buscar_FAQs_Relevantes**  
  - **Type:** Code (JavaScript)  
  - **Role:** Processes user question, cleans and tokenizes it, compares keywords with FAQ questions, scores relevance, and returns top 10 FAQs.  
  - **Key Logic:** Uses string normalization, compares keywords, filters for at least 2 matching words.  
  - **Inputs:** MongoDB FAQs and Telegram Trigger message.  
  - **Outputs:** JSON with user question, relevant FAQ context, count, and chat ID.  
  - **Edge Cases:** Empty user question, no FAQs found, string normalization edge cases.

- **Construir_Prompt_Gemini1**  
  - **Type:** Code (JavaScript)  
  - **Role:** Builds a detailed prompt for Google Gemini AI including user question, relevant FAQ context, contact info, and instructions for response style.  
  - **Inputs:** Output from Buscar_FAQs_Relevantes.  
  - **Outputs:** JSON with prompt and chat ID.  
  - **Failures:** Logic errors in string concatenation.

- **Mensaje de Gemini**  
  - **Type:** Google Gemini AI Node  
  - **Role:** Sends the prompt to Google Gemini (PaLM) model and obtains AI-generated answer.  
  - **Config:** Uses model “gemini-2.5-flash-lite-preview-06-17”, temperature 0.3, max tokens 2000.  
  - **Inputs:** Prompt JSON.  
  - **Failures:** API rate limits, network errors, model timeouts.

- **Limpiar texto Gemini**  
  - **Type:** Code (JavaScript)  
  - **Role:** Extracts and sanitizes AI response text, strips markdown and code blocks, escapes HTML for Telegram.  
  - **Inputs:** Response from Gemini node.  
  - **Outputs:** Cleaned text and chat ID.  
  - **Failures:** Parsing errors if AI response structure changes.

- **Enviar Respuesta sobre faqs**  
  - **Type:** Telegram node  
  - **Role:** Sends cleaned AI response back to the user with inline buttons for feedback (👍 Sí / 👎 No).  
  - **Config:** Uses HTML parse mode, disables attribution, uses chat ID from previous node.  
  - **Failures:** Telegram API errors, invalid chat ID.

---

#### 1.3 Academic Calendar Query Processing

**Overview:**  
This block detects if the user query relates to the academic calendar, fetches calendar data from Google Sheets, analyzes and scores relevant calendar events, builds a prompt for Google Gemini, retrieves and cleans AI response, and sends calendar information back to the user.

**Nodes Involved:**  
- Detector_Calendario_Pre  
- Switch_Que_BD_Leer  
- Calendario Académico  
- Buscar_Eventos_Calendario1  
- Construir_Prompt_Calendario1  
- Message a model1 (Gemini AI)  
- Limpiar_Texto_Gemini1  
- Enviar Respuesta sobre calendario1

**Node Details:**

- **Detector_Calendario_Pre**  
  - **Type:** Code  
  - **Role:** Analyzes user message text for presence of calendar-related keywords to decide if the query is calendar-related.  
  - **Logic:** Scores keywords related to events, times, and terms; decides true if score or keywords found exceeds threshold.  
  - **Failures:** False positives/negatives if keywords ambiguous.

- **Switch_Que_BD_Leer**  
  - **Type:** Switch  
  - **Role:** Routes flow to either “Consultar Calendario” for calendar queries or fallback to FAQ MongoDB reading.  
  - **Failures:** Routing errors if flag missing.

- **Calendario Académico**  
  - **Type:** Google Sheets  
  - **Role:** Reads the “CALENDAR” sheet from the shared Google Sheets document holding academic calendar data.  
  - **Failures:** API errors, sheet access denied.

- **Buscar_Eventos_Calendario1**  
  - **Type:** Code  
  - **Role:** Scores and filters calendar events based on user query tokens, detects intent (e.g., vacations, evaluations), and ranks results by relevance and date. Also builds suggestions to refine query if too many results.  
  - **Failures:** Logic errors in scoring, date parsing issues, empty data.

- **Construir_Prompt_Calendario1**  
  - **Type:** Code  
  - **Role:** Constructs a detailed prompt for Gemini AI incorporating current date, PAO academic periods, top calendar event matches, interpretation rules, and instructions for response formatting.  
  - **Failures:** Logic errors in date formatting or event data extraction.

- **Message a model1**  
  - **Type:** Google Gemini AI Node  
  - **Role:** Sends calendar prompt to Gemini AI for natural language response generation.  
  - **Failures:** API issues, model response timeouts.

- **Limpiar_Texto_Gemini1**  
  - **Type:** Code  
  - **Role:** Extracts and cleans AI response text, strips markdown and code blocks, escapes for Telegram HTML, trims to safe length.  
  - **Failures:** Parsing errors if AI changes response format.

- **Enviar Respuesta sobre calendario1**  
  - **Type:** Telegram node  
  - **Role:** Sends cleaned calendar response to user with inline feedback buttons.  
  - **Failures:** Telegram API errors, invalid chat ID.

---

#### 1.4 User Feedback Handling

**Overview:**  
Manages user feedback after they rate responses. If negative feedback is given, asks for textual comments, stores feedback in Google Sheets, and sends acknowledgments.

**Nodes Involved:**  
- Respuesta Usuario1 (Switch)  
- Get a chat  
- Sí - Agradecimiento1  
- No - Mensaje Feedback1  
- Guardado en cvs Feedback

**Node Details:**

- **Respuesta Usuario1**  
  - **Type:** Switch  
  - **Role:** Detects whether user clicked “feedback_yes” or “feedback_no” inline buttons.  
  - **Outputs:** “Sí - Agradecimiento1” node or “No - Mensaje Feedback1” node.  
  - **Failures:** Missing callback data.

- **Get a chat**  
  - **Type:** Telegram  
  - **Role:** Retrieves chat info to maintain active conversation.  
  - **Failures:** Telegram API errors.

- **Sí - Agradecimiento1**  
  - **Type:** Telegram  
  - **Role:** Sends thank-you message for positive feedback.  
  - **Failures:** Telegram API errors.

- **No - Mensaje Feedback1**  
  - **Type:** Telegram  
  - **Role:** Prompts user to describe what they needed to improve assistance, enabling a reply message.  
  - **Failures:** Telegram API errors.

- **Guardado en cvs Feedback**  
  - **Type:** Google Sheets  
  - **Role:** Stores user feedback text along with user ID and name in a Google Sheet for analysis.  
  - **Failures:** Google Sheets API errors.

---

#### 1.5 Welcome and Help Messages

**Overview:**  
Sends welcome messages, help instructions, and suggestions for user queries when users start interaction or request help.

**Nodes Involved:**  
- BIENVENIDA  
- PREGUNTAS AL AZAR DE GUIA  
- Wait  
- GUIA DE COMO PREGUNTAR  
- Mensaje Help

**Node Details:**

- **BIENVENIDA**  
  - **Type:** Telegram  
  - **Role:** Sends an introductory welcome message describing the bot’s capabilities and encouraging engagement.  
  - **Failures:** Telegram API errors.

- **PREGUNTAS AL AZAR DE GUIA**  
  - **Type:** Langchain Agent (Google Gemini)  
  - **Role:** Generates 5 random example questions users can ask, based on stored FAQ and calendar knowledge.  
  - **Failures:** AI API errors.

- **Wait**  
  - **Type:** Wait  
  - **Role:** Pauses briefly before sending the guide message to ensure user receives messages sequentially.  
  - **Failures:** None.

- **GUIA DE COMO PREGUNTAR**  
  - **Type:** Telegram  
  - **Role:** Sends the generated example questions and usage tips.  
  - **Failures:** Telegram API errors.

- **Mensaje Help**  
  - **Type:** Telegram  
  - **Role:** Responds to /help command listing available commands and usage explanations.  
  - **Failures:** Telegram API errors.

---

#### 1.6 Weekly Announcement System

**Overview:**  
Each Sunday evening, this system scrapes the academic calendar, compiles weekly activities, uses AI to generate a personalized announcement message, converts it to audio, and sends both text and audio announcements to subscribed users on Telegram.

**Nodes Involved:**  
- ANUNCIO SEMANAL (Schedule Trigger)  
- BD_CALENDARIO1 (Google Sheets read)  
- ACTIVIDADES DE LA SEMANA1 (Code)  
- Split Out2  
- Code in JavaScript1  
- GENERADOR DE ANUNCIOS1 (Langchain Agent)  
- OPTIMIZADOR (Code)  
- ENVIO DE MENSAJE DE TEXTO1 (Telegram)  
- GENERADOR DE MENSAJE DE VOZ (HTTP Request to TTS)  
- OPTIMIZADOR1 (Code)  
- ENVIO DE MENSAJE DE VOZ1 (Telegram)

**Node Details:**

- **ANUNCIO SEMANAL**  
  - **Type:** Schedule Trigger  
  - **Role:** Triggers workflow every week on Sunday at 7 PM.  
  - **Failures:** Scheduling misconfiguration.

- **BD_CALENDARIO1**  
  - **Type:** Google Sheets  
  - **Role:** Reads calendar data for events.  
  - **Failures:** API access errors.

- **ACTIVIDADES DE LA SEMANA1**  
  - **Type:** Code  
  - **Role:** Filters and organizes calendar events into three key weeks: previous, current, and next week. Collects unique chat IDs for recipients.  
  - **Failures:** Date parsing errors, empty data.

- **Split Out2**  
  - **Type:** Split Out  
  - **Role:** Splits event data arrays into separate fields for message generation.  
  - **Failures:** Field mismatch.

- **Code in JavaScript1**  
  - **Type:** Code  
  - **Role:** Combines weekly event data into combined strings for input into AI.  
  - **Failures:** Data aggregation errors.

- **GENERADOR DE ANUNCIOS1**  
  - **Type:** Langchain Agent (Google Gemini)  
  - **Role:** Generates a casual, personalized weekly announcement message using AI, following Ecuadorian colloquial style and university context.  
  - **Failures:** AI API errors.

- **OPTIMIZADOR**  
  - **Type:** Code  
  - **Role:** Creates individual message items for each chat ID to send personalized messages.  
  - **Failures:** Incorrect chat ID handling.

- **ENVIO DE MENSAJE DE TEXTO1**  
  - **Type:** Telegram  
  - **Role:** Sends the AI-generated text announcement to each user.  
  - **Failures:** Telegram API errors.

- **GENERADOR DE MENSAJE DE VOZ**  
  - **Type:** HTTP Request  
  - **Role:** Converts the text announcement to audio using TTS service (ttsmp3.com).  
  - **Failures:** External service downtime or request failures.

- **OPTIMIZADOR1**  
  - **Type:** Code  
  - **Role:** Prepares audio message sending per chat ID.  
  - **Failures:** Same as OPTIMIZADOR.

- **ENVIO DE MENSAJE DE VOZ1**  
  - **Type:** Telegram  
  - **Role:** Sends the generated audio announcement as voice message to users with caption including the date range of the announcement week.  
  - **Failures:** Telegram API errors, file upload limits.

---

#### 1.7 Data Scraping and Updating

**Overview:**  
Periodically scrapes the official ESPOL academic calendar web page, extracts date and event information, corrects date formats and years, and updates the Google Sheets database with the latest calendar data.

**Nodes Involved:**  
- ACTUALIZAR CALENDARIO (Schedule Trigger)  
- SCRRAPPING (HTTP Request)  
- HTML  
- Split Out  
- CORRECTOR DE FECHAS (Code)  
- BD-CALENDARIO (Google Sheets append or update)

**Node Details:**

- **ACTUALIZAR CALENDARIO**  
  - **Type:** Schedule Trigger  
  - **Role:** Triggers calendar data update every 7 days.  
  - **Failures:** Scheduling issues.

- **SCRRAPPING**  
  - **Type:** HTTP Request  
  - **Role:** Downloads the academic calendar page HTML content with custom User-Agent header.  
  - **Failures:** Network errors, page structure changes.

- **HTML**  
  - **Type:** HTML Extraction  
  - **Role:** Parses HTML content, extracts table data for period dates, activities, processes, and technical formation activities.  
  - **Failures:** Changed HTML structure or selectors.

- **Split Out**  
  - **Type:** Split Out  
  - **Role:** Splits concatenated fields to separate items for processing.  
  - **Failures:** Incorrect field splitting.

- **CORRECTOR DE FECHAS**  
  - **Type:** Code  
  - **Role:** Normalizes date strings, infers missing years, adjusts for year transitions, and formats period strings correctly.  
  - **Failures:** Date parsing errors, unexpected date formats.

- **BD-CALENDARIO**  
  - **Type:** Google Sheets  
  - **Role:** Appends or updates the calendar data sheet with the processed and corrected calendar events.  
  - **Failures:** API permission or quota issues.

---

### 3. Summary Table

| Node Name                  | Node Type                       | Functional Role                                                | Input Node(s)                         | Output Node(s)                        | Sticky Note                                                                                                   |
|----------------------------|--------------------------------|----------------------------------------------------------------|-------------------------------------|-------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Telegram Trigger - Inicio   | Telegram Trigger               | Entry point for Telegram updates (messages and callbacks)      | -                                   | Switch                              | TRIGGER TELEGRAM: SE ACTIVA EN CUALQUIER INTERACCIÓN DEL USUARIO CON EL BOT                                  |
| Switch                     | Switch                        | Classifies input into callback, feedback, commands, or fallback| Telegram Trigger - Inicio            | Respuesta Usuario1, Guardado en cvs Feedback, Comandos1, Detector_Calendario_Pre | CLASIFICADOR DE MENSAJES: 0=FEEDBACK, 1=COMENTARIO, 2=TAGS, Fallback=MENSAJE                                 |
| Comandos1                  | Switch                        | Routes recognized commands to appropriate response nodes       | Switch                              | Mensaje Help, FAQs, Respuesta Contact, Respuesta Events, Inicio - Feedback, BIENVENIDA, OBTENER_ID1 | Comandos explained in sticky notes 3-7                                                                        |
| BIENVENIDA                 | Telegram                      | Sends welcome message to new user                              | Comandos1                          | PREGUNTAS AL AZAR DE GUIA            | BIENVENIDA AL USUARIO                                                                                         |
| PREGUNTAS AL AZAR DE GUIA  | Langchain Agent (Google Gemini) | Generates example questions for new users                      | BIENVENIDA                        | Wait                                | BIENVENIDA AL USUARIO                                                                                         |
| Wait                       | Wait                          | Small delay for message ordering                               | PREGUNTAS AL AZAR DE GUIA          | GUIA DE COMO PREGUNTAR               | BIENVENIDA AL USUARIO                                                                                         |
| GUIA DE COMO PREGUNTAR     | Telegram                      | Sends generated example questions to user                     | Wait                              | -                                   | BIENVENIDA AL USUARIO                                                                                         |
| Mensaje Help               | Telegram                      | Sends help command info                                        | Comandos1 (/help)                  | -                                   | AYUDA (/HELP)                                                                                                |
| FAQs                       | Telegram                      | Sends prompt to start FAQ query section                       | Comandos1 (/faqs)                  | -                                   | CONSULTAS (/FAQS)                                                                                            |
| Respuesta Contact          | Telegram                      | Sends contact information                                    | Comandos1 (/contact)               | -                                   | CONTACTOS (/CONTACT)                                                                                          |
| Respuesta Events           | Telegram                      | Sends upcoming academic events info                         | Comandos1 (/events)                | -                                   | EVENTOS (/EVENTS)                                                                                            |
| Inicio - Feedback          | Telegram                      | Sends feedback input prompt with inline buttons             | Comandos1 (/feedback)              | -                                   | FEEDBACK (/FEEDBACK)                                                                                          |
| OBTENER_ID1                | Set                           | Extracts chat ID from messages                               | Comandos1 (/start)                 | AGREGA ID_UNICAS1                   | BIENVENIDA AL USUARIO                                                                                         |
| AGREGA ID_UNICAS1          | Google Sheets                 | Stores unique chat IDs                                      | OBTENER_ID1                      | -                                   | BIENVENIDA AL USUARIO                                                                                         |
| Respuesta Usuario1         | Switch                        | Detects feedback button clicks ("yes" or "no")              | Switch (callback_query)            | Get a chat, No - Mensaje Feedback1 | FLUJO: RESPUESTAS DEL USUARIO & FEEDBACK                                                                     |
| Get a chat                 | Telegram                      | Retrieves chat info for active conversation                 | Respuesta Usuario1                | Sí - Agradecimiento1                | FLUJO: RESPUESTAS DEL USUARIO & FEEDBACK                                                                     |
| Sí - Agradecimiento1       | Telegram                      | Sends thank you for positive feedback message               | Get a chat                      | -                                   | FLUJO: RESPUESTAS DEL USUARIO & FEEDBACK                                                                     |
| No - Mensaje Feedback1     | Telegram                      | Requests detailed feedback from user after negative rating | Respuesta Usuario1                | Guardado en cvs Feedback            | FLUJO: RESPUESTAS DEL USUARIO & FEEDBACK                                                                     |
| Guardado en cvs Feedback   | Google Sheets                 | Saves user feedback comments                                | No - Mensaje Feedback1            | -                                   | FEEDBACK (/FEEDBACK)                                                                                          |
| Leer FAQs de MongoDB       | MongoDB                      | Reads all FAQ entries from database                          | Switch_Que_BD_Leer (fallback)      | Buscar_FAQs_Relevantes             | CONSULTAS (/FAQS)                                                                                            |
| Buscar_FAQs_Relevantes     | Code                         | Filters top 10 relevant FAQs based on user question         | Leer FAQs de MongoDB, Telegram Trigger - Inicio | Construir_Prompt_Gemini1          | CONSULTAS (/FAQS)                                                                                            |
| Construir_Prompt_Gemini1   | Code                         | Builds AI prompt with FAQ context and instructions          | Buscar_FAQs_Relevantes            | Mensaje de Gemini                  | CONSULTAS (/FAQS)                                                                                            |
| Mensaje de Gemini          | Google Gemini AI             | Sends prompt to AI and receives response                    | Construir_Prompt_Gemini1          | Limpiar texto Gemini              | CONSULTAS (/FAQS)                                                                                            |
| Limpiar texto Gemini       | Code                         | Cleans AI response text for Telegram                        | Mensaje de Gemini                | Enviar Respuesta sobre faqs       | CONSULTAS (/FAQS)                                                                                            |
| Enviar Respuesta sobre faqs| Telegram                      | Sends AI-generated FAQ answer with feedback buttons        | Limpiar texto Gemini             | -                                   | CONSULTAS (/FAQS)                                                                                            |
| Detector_Calendario_Pre    | Code                         | Detects if user query relates to academic calendar          | Switch                          | Switch_Que_BD_Leer               | SISTEMA GENERADOR DE MENSAJES AUTOMÁTICOS Y PERSONALIZADOS                                                  |
| Switch_Que_BD_Leer         | Switch                       | Routes to calendar or FAQ data source                        | Detector_Calendario_Pre          | Calendario Académico, Leer FAQs de MongoDB | SISTEMA GENERADOR DE MENSAJES AUTOMÁTICOS Y PERSONALIZADOS                                                  |
| Calendario Académico       | Google Sheets                | Reads academic calendar data from Google Sheets             | Switch_Que_BD_Leer (calendar)    | Buscar_Eventos_Calendario1       | SISTEMA GENERADOR DE MENSAJES AUTOMÁTICOS Y PERSONALIZADOS                                                  |
| Buscar_Eventos_Calendario1 | Code                         | Scores and filters calendar events by relevance             | Calendario Académico             | Construir_Prompt_Calendario1     | SISTEMA GENERADOR DE MENSAJES AUTOMÁTICOS Y PERSONALIZADOS                                                  |
| Construir_Prompt_Calendario1| Code                        | Builds AI prompt for calendar query                          | Buscar_Eventos_Calendario1       | Message a model1                 | SISTEMA GENERADOR DE MENSAJES AUTOMÁTICOS Y PERSONALIZADOS                                                  |
| Message a model1           | Google Gemini AI             | Sends calendar prompt to AI and receives response           | Construir_Prompt_Calendario1     | Limpiar_Texto_Gemini1           | SISTEMA GENERADOR DE MENSAJES AUTOMÁTICOS Y PERSONALIZADOS                                                  |
| Limpiar_Texto_Gemini1      | Code                         | Cleans AI calendar response for Telegram                    | Message a model1                | Enviar Respuesta sobre calendario1 | SISTEMA GENERADOR DE MENSAJES AUTOMÁTICOS Y PERSONALIZADOS                                                  |
| Enviar Respuesta sobre calendario1| Telegram               | Sends AI-generated calendar answer with feedback buttons    | Limpiar_Texto_Gemini1           | -                                   | SISTEMA GENERADOR DE MENSAJES AUTOMÁTICOS Y PERSONALIZADOS                                                  |
| ACTUALIZAR CALENDARIO      | Schedule Trigger             | Triggers weekly scraping and update of calendar data        | -                               | SCRRAPPING                      | SCRAPING WEB Y ACTUALIZACIÓN DE BASE DE DATOS                                                               |
| SCRRAPPING                 | HTTP Request                | Downloads the ESPOL academic calendar webpage               | ACTUALIZAR CALENDARIO            | HTML                           | SCRAPING WEB Y ACTUALIZACIÓN DE BASE DE DATOS                                                               |
| HTML                       | HTML Extraction             | Extracts calendar table data from HTML                       | SCRRAPPING                     | Split Out                      | SCRAPING WEB Y ACTUALIZACIÓN DE BASE DE DATOS                                                               |
| Split Out                  | Split Out                   | Splits extracted table columns into items                   | HTML                           | CORRECTOR DE FECHAS             | SCRAPING WEB Y ACTUALIZACIÓN DE BASE DE DATOS                                                               |
| CORRECTOR DE FECHAS        | Code                         | Normalizes date periods, infers missing years               | Split Out                     | BD-CALENDARIO                  | SCRAPING WEB Y ACTUALIZACIÓN DE BASE DE DATOS                                                               |
| BD-CALENDARIO              | Google Sheets               | Updates Google Sheets calendar data                          | CORRECTOR DE FECHAS             | BD_CALENDARIO1                 | SCRAPING WEB Y ACTUALIZACIÓN DE BASE DE DATOS                                                               |
| BD_CALENDARIO1             | Google Sheets               | Reads updated calendar data for weekly announcement         | ANUNCIO SEMANAL                | ACTIVIDADES DE LA SEMANA1       | SISTEMA DE MENSAJES AUTOMÁTICOS SEMANALES                                                                   |
| ACTIVIDADES DE LA SEMANA1  | Code                         | Filters and organizes calendar events per week, collects chat IDs| BD_CALENDARIO1             | Split Out2                    | SISTEMA DE MENSAJES AUTOMÁTICOS SEMANALES                                                                   |
| Split Out2                 | Split Out                   | Splits weekly event data into separate fields               | ACTIVIDADES DE LA SEMANA1       | Code in JavaScript1            | SISTEMA DE MENSAJES AUTOMÁTICOS SEMANALES                                                                   |
| Code in JavaScript1        | Code                         | Combines weekly event data for AI message generation        | Split Out2                    | GENERADOR DE ANUNCIOS1         | SISTEMA DE MENSAJES AUTOMÁTICOS SEMANALES                                                                   |
| GENERADOR DE ANUNCIOS1     | Langchain Agent (Google Gemini) | Generates personalized weekly announcement message          | Code in JavaScript1           | OPTIMIZADOR                   | SISTEMA DE MENSAJES AUTOMÁTICOS SEMANALES                                                                   |
| OPTIMIZADOR                | Code                         | Prepares individual text messages per chat ID               | GENERADOR DE ANUNCIOS1        | ENVIO DE MENSAJE DE TEXTO1    | SISTEMA DE MENSAJES AUTOMÁTICOS SEMANALES                                                                   |
| ENVIO DE MENSAJE DE TEXTO1 | Telegram                    | Sends weekly text announcement to users                     | OPTIMIZADOR                   | -                             | SISTEMA DE MENSAJES AUTOMÁTICOS SEMANALES                                                                   |
| GENERADOR DE MENSAJE DE VOZ| HTTP Request                | Converts text announcement to audio using TTS service       | GENERADOR DE ANUNCIOS1        | OPTIMIZADOR1                  | SISTEMA DE MENSAJES AUTOMÁTICOS SEMANALES                                                                   |
| OPTIMIZADOR1               | Code                         | Prepares individual audio messages per chat ID              | GENERADOR DE MENSAJE DE VOZ   | ENVIO DE MENSAJE DE VOZ1      | SISTEMA DE MENSAJES AUTOMÁTICOS SEMANALES                                                                   |
| ENVIO DE MENSAJE DE VOZ1   | Telegram                    | Sends weekly audio announcements to users                   | OPTIMIZADOR1                  | -                             | SISTEMA DE MENSAJES AUTOMÁTICOS SEMANALES                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node:**  
   - Type: Telegram Trigger  
   - Credentials: ESPOLBot Telegram API  
   - Listen for updates: message, callback_query, * (all)  

2. **Add Switch Node for Input Classification:**  
   - Connect from Telegram Trigger  
   - Conditions:  
     - If callback_query.data equals `feedback_yes` → output 1  
     - If callback_query.data equals `feedback_no` → output 2  
     - If message.text starts with `/` → output 3  
     - Else → output 4 (fallback)  

3. **Add Comandos1 Switch Node for Command Routing:**  
   - Connect output 3 of previous switch  
   - Conditions routing by exact message text: `/help`, `/faqs`, `/contact`, `/events`, `/feedback`, `/start`  
   - Connect outputs to respective nodes (see below)  

4. **Build Welcome Message Node (Telegram):**  
   - Text: Welcome message describing bot capabilities (Spanish)  
   - Triggered by `/start` command from Comandos1  

5. **Add Langchain Agent Node for Example Questions:**  
   - Model: Google Gemini  
   - Prompt: Generate 5 random example questions for new users  
   - Connect after Welcome message node  

6. **Add Wait Node:**  
   - Small delay before sending guide message  

7. **Add Telegram Node to Send Guide Message:**  
   - Text: Output from Langchain agent  
   - Connect after Wait  

8. **Add Telegram Nodes for Help, FAQs, Contact, Events, Feedback:**  
   - Each sends respective static or dynamic messages  
   - Connected from Comandos1 outputs  

9. **Create MongoDB Node to Read FAQs:**  
   - Connect from Switch fallback output (non-command, non-feedback, non-callback)  
   - Collection: espol_faqs  
   - Fetch all documents  

10. **Add Code Node to Find Relevant FAQs:**  
    - Input: MongoDB FAQs and Telegram Trigger message  
    - Logic: Clean text, tokenize, find FAQs with >=2 matching keywords  
    - Output: Top 10 FAQ items and user question  

11. **Add Code Node to Build Gemini Prompt for FAQs:**  
    - Incorporate user question, matching FAQs, contact info, instructions  

12. **Add Google Gemini AI Node:**  
    - Model: “gemini-2.5-flash-lite-preview-06-17”  
    - Input: Prompt from previous node  

13. **Add Code Node to Clean Gemini Response:**  
    - Remove markdown, code blocks, escape HTML for Telegram  

14. **Add Telegram Node to Send FAQ Response:**  
    - Text: Cleaned response  
    - Reply markup: Inline buttons for feedback (👍 Sí / 👎 No)  

15. **Add Switch Node for Feedback Handling:**  
    - Detect callback_data feedback_yes or feedback_no  
    - On yes: Send thank-you Telegram message  
    - On no: Send Telegram message requesting detailed feedback (force reply)  

16. **Add Google Sheets Node to Save Feedback:**  
    - Append user feedback message to Google Sheet with user ID and name  

17. **Create Code Node for Calendar Query Detection:**  
    - Analyze incoming messages for calendar-related keywords  
    - Output flag `es_consulta_calendario` true/false  

18. **Add Switch Node to Route Calendar or FAQs:**  
    - If calendar query, route to Google Sheets calendar read  
    - Else route to MongoDB FAQs read  

19. **Add Google Sheets Node to Read Calendar Data:**  
    - Document and sheet containing academic calendar  

20. **Add Code Node to Score and Filter Calendar Events:**  
    - Tokenize user question and calendar fields  
    - Score matches and detect intent (vacations, exams, start/end)  
    - Sort results by score and date  

21. **Add Code Node to Build Gemini Prompt for Calendar:**  
    - Include current date, academic period (PAO), top event matches, instructions  

22. **Add Google Gemini AI Node for Calendar Response:**  
    - Model: “gemini-2.5-flash-preview-05-20” or newer  

23. **Add Code Node to Clean Calendar AI Response:**  
    - Similar sanitization as FAQ response  

24. **Add Telegram Node to Send Calendar Response:**  
    - Include inline feedback buttons  

25. **Create Schedule Trigger Node for Weekly Calendar Update:**  
    - Runs every 7 days to update calendar data  

26. **Add HTTP Request Node to Scrape ESPOL Calendar Web Page:**  
    - Use User-Agent header to mimic browser  

27. **Add HTML Extraction Node:**  
    - Extract calendar table columns (date periods, activities, processes)  

28. **Add Split Out Node:**  
    - Split extracted arrays for processing  

29. **Add Code Node to Correct Dates:**  
    - Normalize date text, add missing years, manage year transitions  

30. **Add Google Sheets Node to Append or Update Calendar Data:**  
    - Write cleaned calendar data to Sheets  

31. **Create Schedule Trigger Node for Weekly Announcement:**  
    - Runs every Sunday at 7 PM  

32. **Add Google Sheets Node to Read Current Calendar Data:**  
    - Read updated calendar events  

33. **Add Code Node to Filter Events by Week:**  
    - Identify previous, current, next week events  
    - Collect unique chat IDs for announcement recipients  

34. **Add Split Out Node:**  
    - Separate event data fields for AI input  

35. **Add Code Node to Combine Weekly Data:**  
    - Merge events into strings for AI prompt  

36. **Add Langchain Agent Node to Generate Announcement Message:**  
    - Generate friendly, casual weekly announcement in Ecuadorian Spanish  

37. **Add Code Node to Create Per-User Messages:**  
    - Prepare individual text messages for each user  

38. **Add Telegram Node to Send Text Announcement:**  
    - Send to each chat ID  

39. **Add HTTP Request Node to Convert Text to Audio:**  
    - Use TTS service (ttsmp3.com) with Spanish voice “Mia”  

40. **Add Code Node to Prepare Audio Messages:**  
    - Similar to text message optimizer  

41. **Add Telegram Node to Send Audio Announcement:**  
    - Send voice message with caption including week dates  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                  | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Workflow designed for ESPOL university to assist students via Telegram with FAQs and academic calendar queries, integrating MongoDB, Google Sheets, and Google Gemini AI.                                                                       | Workflow general description                                                                            |
| Google Gemini AI credentials and models used: "gemini-2.5-flash-lite-preview-06-17" and "gemini-2.5-flash-preview-05-20" for different tasks.                                                                                                | AI Integration                                                                                         |
| Telegram bot is named ESPOLBot and uses OAuth2 credentials for Google Sheets and MongoDB account for data access.                                                                                                                           | Credential setup                                                                                        |
| Weekly announcements include text and voice messages generated every Sunday at 7 PM automatically.                                                                                                                                           | Weekly announcement system                                                                              |
| Scraping academic calendar from https://www.espol.edu.ec/es/vida-politecnica/calendario-grado with custom User-Agent header.                                                                                                                | Scraping details                                                                                       |
| Contact information for ESPOL included in FAQ responses and contact commands for user convenience.                                                                                                                                           | Contact info in prompt: admision@espol.edu.ec, phone (04) 2269-269, web https://www.espol.edu.ec         |
| Feedback system stores user comments in a Google Sheet for later analysis to improve bot functionality.                                                                                                                                      | Feedback management                                                                                     |
| Inline keyboard buttons used extensively for feedback collection and interaction flow management in Telegram.                                                                                                                               | Telegram UI/UX features                                                                                  |
| Message sanitization is critical to avoid Telegram parse errors, especially escaping HTML and removing markdown/code blocks from AI responses.                                                                                              | Telegram message formatting                                                                             |
| Several sticky notes in workflow provide contextual explanations, e.g., about database tags, user welcome, feedback flow, calendar scraping, and announcement generation — useful for maintainers and collaborators.                          | Workflow documentation embedded in sticky notes                                                       |
| Project credits mention the team (Nicole Guevara, Doménica Amores, Adrián Villamar) and mentor Jaren, emphasizing the academic and collaborative nature of this automation.                                                                   | Project credits                                                                                        |
| Workflow respects legal and content policies, handling only public and legal data.                                                                                                                                                            | Disclaimer                                                                                             |

---

**Disclaimer:** The provided content is derived exclusively from an automated n8n workflow and complies fully with content policies. It contains no illegal or protected elements. All data handled is legal and publicly available.