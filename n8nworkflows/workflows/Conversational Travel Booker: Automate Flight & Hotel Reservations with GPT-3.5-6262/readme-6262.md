Conversational Travel Booker: Automate Flight & Hotel Reservations with GPT-3.5

https://n8nworkflows.xyz/workflows/conversational-travel-booker--automate-flight---hotel-reservations-with-gpt-3-5-6262


# Conversational Travel Booker: Automate Flight & Hotel Reservations with GPT-3.5

### 1. Workflow Overview

This workflow, titled **"Conversational Travel Booker: Automate Flight & Hotel Reservations with GPT-3.5"**, is designed to automate travel booking requests via conversational natural language inputs. It accepts HTTP POST requests containing user travel booking intents (flights or hotels), leverages OpenAI GPT-3.5 to parse and understand the requests, determines the booking type (flight or hotel), extracts structured booking parameters, queries external travel APIs or uses mock data for fallback, processes and confirms the booking, then generates and emails a personalized confirmation message. Finally, it responds to the original HTTP request with structured confirmation details.

The workflow is composed of the following logical blocks:

- **1.1 Input Reception:** Receives and accepts booking requests via HTTP webhook POST.
- **1.2 AI Processing:** Uses GPT-3.5 to parse the natural language booking request into structured data.
- **1.3 Booking Type Routing:** Determines if the request is for a flight or hotel booking and routes accordingly.
- **1.4 Data Processing:** Extracts and structures flight or hotel booking parameters from AI output.
- **1.5 External API Search:** Queries flight or hotel search APIs with structured parameters; includes mock data fallback.
- **1.6 Booking Confirmation Processing:** Selects the best option from search results and prepares booking confirmation details.
- **1.7 Confirmation Message Generation:** Generates a personalized confirmation message using GPT-3.5.
- **1.8 Notification & Response:** Sends an email with the confirmation and responds to the initial webhook with booking details.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Accepts incoming HTTP POST booking requests to initiate the workflow.
- **Nodes Involved:** 
  - Webhook Trigger
  - Sticky Note (commentary)

- **Node Details:**

  - **Webhook Trigger**
    - Type: `Webhook`  
    - Role: Entry point for booking requests via HTTP POST on path `/booking-request`  
    - Configuration: HTTP POST method, immediate response handled by response node downstream  
    - Inputs: External HTTP POST request  
    - Outputs: JSON payload forwarded to AI Request Parser  
    - Edge Cases: Invalid HTTP requests, missing/malformed JSON payloads  
    - Sticky Note: "Accepts booking requests via HTTP POST"

---

#### 1.2 AI Processing

- **Overview:** Sends the raw user booking request to OpenAI GPT-3.5 to interpret and parse natural language into a structured form.
- **Nodes Involved:** 
  - AI Request Parser
  - Sticky Note (commentary)

- **Node Details:**

  - **AI Request Parser**
    - Type: `OpenAI`  
    - Role: Uses GPT-3.5-turbo to parse booking request text  
    - Configuration: Model `gpt-3.5-turbo`, temperature 0.3 for determinism, prompt set to "Hello" (placeholder, likely dynamic)  
    - Inputs: JSON from Webhook Trigger  
    - Outputs: AI-generated response containing booking intent and parameters  
    - Credentials: OpenAI API key required  
    - Edge Cases: API quota exceeded, network issues, unexpected AI output format  
    - Sticky Note: "Uses OpenAI to understand natural language booking requests"

---

#### 1.3 Booking Type Routing

- **Overview:** Determines whether the booking request is for a flight or a hotel, routing to respective data extraction nodes.
- **Nodes Involved:** 
  - Booking Type Router
  - Sticky Note (commentary)

- **Node Details:**

  - **Booking Type Router**
    - Type: `Switch`  
    - Role: Examines AI output to route flow to either flight or hotel processing  
    - Configuration: No explicit conditions shown; inferred to check booking type in AI response  
    - Inputs: AI Request Parser output  
    - Outputs: Branch 1 → Flight Data Processor; Branch 2 → Hotel Data Processor  
    - Edge Cases: Unrecognized booking type, missing fields  
    - Sticky Note: "Automatically determines if it's a flight or hotel booking"

---

#### 1.4 Data Processing

- **Overview:** Parses and structures detailed booking parameters for flights or hotels from the AI response, using JavaScript code to handle raw text or JSON outputs.
- **Nodes Involved:** 
  - Flight Data Processor
  - Hotel Data Processor
  - Sticky Note (commentary)

- **Node Details:**

  - **Flight Data Processor**
    - Type: `Code` (JavaScript)  
    - Role: Extracts flight booking parameters (origin, destination, departDate, passengers, class) from AI response text or JSON  
    - Configuration: Parses AI message content; fallback regex extraction for keywords; sets defaults for missing data  
    - Inputs: Branch from Booking Type Router  
    - Outputs: Structured `flightSearch` object with search parameters  
    - Edge Cases: Parsing errors, missing or ambiguous dates, passenger count defaults to 1  
    - Sticky Note: "Extracts and structures booking parameters"

  - **Hotel Data Processor**
    - Type: `Code` (JavaScript)  
    - Role: Extracts hotel booking parameters (destination, checkIn, checkOut, guests, rooms) similarly from AI response  
    - Configuration: Same parsing strategy as flight processor; default values for missing data (2 guests, 1 room, dates)  
    - Inputs: Branch from Booking Type Router  
    - Outputs: Structured `hotelSearch` object with search parameters  
    - Edge Cases: Parsing failures, date format inconsistencies, guest number defaults to 2  
    - Sticky Note: "Extracts and structures booking parameters"

---

#### 1.5 External API Search

- **Overview:** Uses the extracted parameters to query external flight or hotel search APIs; falls back to mock data if API responses are missing or invalid.
- **Nodes Involved:** 
  - Flight Search API
  - Hotel Search API
  - Sticky Note (commentary)

- **Node Details:**

  - **Flight Search API**
    - Type: `HTTP Request`  
    - Role: Queries aviationstack API for flights matching origin, destination, limited to 5 results  
    - Configuration: Query parameters: departure IATA, arrival IATA, limit=5  
    - Inputs: Flight Data Processor output  
    - Outputs: Flight search results JSON  
    - Credentials: Predefined aviationStackApi credential  
    - Edge Cases: API downtime, invalid IATA codes, quota limits  
    - Sticky Note: "Searches for flights/hotels (with mock fallbacks)"

  - **Hotel Search API**
    - Type: `HTTP Request`  
    - Role: Queries Booking.com API for hotels with destination ID, check-in/out dates, number of adults and rooms  
    - Configuration: Query parameters mapped from hotelSearch object  
    - Inputs: Hotel Data Processor output  
    - Outputs: Hotel search results JSON  
    - Credentials: Predefined bookingComApi credential  
    - Edge Cases: API errors, invalid destination IDs, date conflicts  
    - Sticky Note: "Searches for flights/hotels (with mock fallbacks)"

---

#### 1.6 Booking Confirmation Processing

- **Overview:** Processes API search results or fallback mock data to select the best flight or hotel option and prepares confirmed booking details.
- **Nodes Involved:** 
  - Flight Booking Processor
  - Hotel Booking Processor
  - Sticky Note (commentary)

- **Node Details:**

  - **Flight Booking Processor**
    - Type: `Code` (JavaScript)  
    - Role: Processes flight search results, or uses mock flights if API fails; selects lowest price flight; prepares confirmation object  
    - Configuration: Sorts flights by price, generates confirmation code `FL{timestamp}`, includes static passenger info (name/email)  
    - Inputs: Flight Search API output  
    - Outputs: Booking confirmation JSON with flight details and total price  
    - Edge Cases: API failure fallback, missing price data, static passenger info (to be replaced in real use)  
    - Sticky Note: "Processes and confirms bookings"

  - **Hotel Booking Processor**
    - Type: `Code` (JavaScript)  
    - Role: Processes hotel search results, or uses mock hotels if API fails; selects highest rating hotel; calculates total price for nights; prepares confirmation  
    - Configuration: Sorts hotels by rating descending, generates confirmation code `HT{timestamp}`, includes static guest info  
    - Inputs: Hotel Search API output  
    - Outputs: Booking confirmation JSON with hotel details, nights, total cost  
    - Edge Cases: API failure fallback, date calculations, static guest info  
    - Sticky Note: "Processes and confirms bookings"

---

#### 1.7 Confirmation Message Generation

- **Overview:** Generates a personalized confirmation message text using GPT-3.5 based on the confirmed booking details.
- **Nodes Involved:** 
  - Confirmation Message Generator
  - Sticky Note (commentary)

- **Node Details:**

  - **Confirmation Message Generator**
    - Type: `OpenAI`  
    - Role: Uses GPT-3.5-turbo to generate a friendly confirmation message using booking data  
    - Configuration: Prompt set dynamically to `{{json.data}}` (booking details), temperature 0.7 for creativity  
    - Inputs: Booking Processor outputs (flight or hotel)  
    - Outputs: Generated confirmation message text  
    - Credentials: OpenAI API key required  
    - Edge Cases: API errors, unexpected output format  
    - Sticky Note: "Creates personalized confirmation messages"

---

#### 1.8 Notification & Response

- **Overview:** Sends the confirmation email to the user and responds to the original HTTP webhook request with structured booking confirmation data.
- **Nodes Involved:** 
  - Send Confirmation Email
  - Send Response
  - Sticky Notes (commentary)

- **Node Details:**

  - **Send Confirmation Email**
    - Type: `Email Send`  
    - Role: Sends the generated confirmation message via email to the user  
    - Configuration: From and To emails are statically set (`abc@gmail.com` to `xyz@gmail.com`), subject line includes emoji, email body uses confirmation message content  
    - Inputs: Confirmation Message Generator output  
    - Outputs: Passes data to Send Response node  
    - Credentials: SMTP credentials configured  
    - Edge Cases: Email sending failures, invalid email addresses  
    - Sticky Note: "Sends booking confirmations via email"

  - **Send Response**
    - Type: `Respond to Webhook`  
    - Role: Returns JSON response to the original HTTP request including booking status, booking details, confirmation message, and confirmation code  
    - Configuration: JSON response with keys: status, booking, message, confirmation_code; dynamically references flight or hotel booking nodes for data  
    - Inputs: Send Confirmation Email output  
    - Outputs: Final response to webhook client  
    - Edge Cases: Missing booking data, expression evaluation failures  
    - Sticky Note: "Returns structured booking data"

---

### 3. Summary Table

| Node Name                    | Node Type          | Functional Role                            | Input Node(s)             | Output Node(s)                | Sticky Note                                         |
|------------------------------|--------------------|--------------------------------------------|---------------------------|------------------------------|----------------------------------------------------|
| Webhook Trigger              | Webhook            | Accepts booking requests via HTTP POST     | External HTTP request      | AI Request Parser            | Accepts booking requests via HTTP POST             |
| AI Request Parser            | OpenAI             | Parses natural language booking request    | Webhook Trigger           | Booking Type Router          | Uses OpenAI to understand natural language booking requests |
| Booking Type Router          | Switch             | Routes based on booking type (flight/hotel)| AI Request Parser         | Flight Data Processor, Hotel Data Processor | Automatically determines if it's a flight or hotel booking |
| Flight Data Processor        | Code               | Extracts & structures flight booking data  | Booking Type Router (flight branch) | Flight Search API            | Extracts and structures booking parameters          |
| Hotel Data Processor         | Code               | Extracts & structures hotel booking data   | Booking Type Router (hotel branch) | Hotel Search API             | Extracts and structures booking parameters          |
| Flight Search API            | HTTP Request       | Searches flights via aviationstack API     | Flight Data Processor     | Flight Booking Processor     | Searches for flights/hotels (with mock fallbacks)   |
| Hotel Search API             | HTTP Request       | Searches hotels via Booking.com API         | Hotel Data Processor      | Hotel Booking Processor      | Searches for flights/hotels (with mock fallbacks)   |
| Flight Booking Processor     | Code               | Processes flight results & confirms booking| Flight Search API         | Confirmation Message Generator| Processes and confirms bookings                      |
| Hotel Booking Processor      | Code               | Processes hotel results & confirms booking | Hotel Search API          | Confirmation Message Generator| Processes and confirms bookings                      |
| Confirmation Message Generator| OpenAI             | Creates personalized confirmation message  | Flight/Hotel Booking Processor | Send Confirmation Email      | Creates personalized confirmation messages           |
| Send Confirmation Email      | Email Send         | Sends booking confirmation to user email   | Confirmation Message Generator | Send Response               | Sends booking confirmations via email                |
| Send Response               | Respond to Webhook  | Returns booking confirmation via webhook   | Send Confirmation Email   | External HTTP response       | Returns structured booking data                       |
| Sticky Note                 | Sticky Note        | Comment: Accepts booking requests           | -                         | -                            | Accepts booking requests via HTTP POST               |
| Sticky Note1                | Sticky Note        | Comment: Searches for flights/hotels        | -                         | -                            | Searches for flights/hotels (with mock fallbacks)    |
| Sticky Note2                | Sticky Note        | Comment: Extracts and structures parameters | -                         | -                            | Extracts and structures booking parameters           |
| Sticky Note3                | Sticky Note        | Comment: Automatically determines booking type | -                         | -                            | Automatically determines if it's a flight or hotel booking |
| Sticky Note4                | Sticky Note        | Comment: Uses OpenAI to parse requests       | -                         | -                            | Uses OpenAI to understand natural language booking requests |
| Sticky Note5                | Sticky Note        | Comment: Sends booking confirmations         | -                         | -                            | Sends booking confirmations via email                |
| Sticky Note6                | Sticky Note        | Comment: Creates personalized messages       | -                         | -                            | Creates personalized confirmation messages           |
| Sticky Note10               | Sticky Note        | Comment: Processes and confirms bookings     | -                         | -                            | Processes and confirms bookings                       |
| Sticky Note11               | Sticky Note        | Comment: Returns structured booking data     | -                         | -                            | Returns structured booking data                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger node**
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `booking-request`  
   - Response Mode: responseNode (to allow later nodes to respond)  
   - No credentials required

2. **Create OpenAI node (AI Request Parser)**
   - Type: OpenAI  
   - Model: `gpt-3.5-turbo`  
   - Temperature: 0.3  
   - Prompt: Initially "Hello" (should be dynamically replaced with webhook input text)  
   - Connect input from Webhook Trigger  
   - Set OpenAI API credentials accordingly

3. **Create Switch node (Booking Type Router)**
   - Type: Switch  
   - Condition: Evaluate AI response to determine booking type ("flight" or "hotel")  
   - Input from AI Request Parser  
   - Branch 1: Flight Data Processor  
   - Branch 2: Hotel Data Processor

4. **Create Code node (Flight Data Processor)**
   - Type: Code (JavaScript)  
   - Script: Parses AI output for flight parameters (origin, destination, departure date, passengers, class)  
   - Uses regex fallback if JSON parsing fails  
   - Defaults: origin='NYC', destination='LAX', date=today, passengers=1, class='economy'  
   - Input: Switch node flight branch

5. **Create Code node (Hotel Data Processor)**
   - Type: Code (JavaScript)  
   - Script: Parses AI output for hotel parameters (destination, check-in/out, guests, rooms)  
   - Uses regex fallback if JSON parsing fails  
   - Defaults: destination='New York', checkIn=today, checkOut=tomorrow, guests=2, rooms=1  
   - Input: Switch node hotel branch

6. **Create HTTP Request node (Flight Search API)**
   - Type: HTTP Request  
   - URL: `https://api.aviationstack.com/v1/flights`  
   - Method: GET  
   - Query parameters: `dep_iata`, `arr_iata`, `limit=5` extracted from flightSearch object  
   - Authentication: Use aviationStackApi credential  
   - Input: Flight Data Processor output

7. **Create HTTP Request node (Hotel Search API)**
   - Type: HTTP Request  
   - URL: `https://api.booking.com/v1/hotels/search`  
   - Method: GET  
   - Query parameters: `dest_id`, `checkin_date`, `checkout_date`, `adults_number`, `room_number` extracted from hotelSearch object  
   - Authentication: Use bookingComApi credential  
   - Input: Hotel Data Processor output

8. **Create Code node (Flight Booking Processor)**
   - Type: Code (JavaScript)  
   - Script: Processes flight search results or falls back to mock flights if API fails  
   - Selects lowest price flight  
   - Generates confirmation code `FL{timestamp}`  
   - Uses static passenger info (replace in actual deployment)  
   - Input: Flight Search API output

9. **Create Code node (Hotel Booking Processor)**
   - Type: Code (JavaScript)  
   - Script: Processes hotel search results or falls back to mock hotels if API fails  
   - Selects highest rating hotel  
   - Calculates nights from check-in/out dates  
   - Generates confirmation code `HT{timestamp}`  
   - Uses static guest info (replace in actual deployment)  
   - Input: Hotel Search API output

10. **Create OpenAI node (Confirmation Message Generator)**
    - Type: OpenAI  
    - Model: `gpt-3.5-turbo`  
    - Temperature: 0.7 (more creative)  
    - Prompt: Use booking confirmation JSON (`{{json.data}}`)  
    - Credentials: OpenAI API key  
    - Input: Flight or Hotel Booking Processor output

11. **Create Email Send node (Send Confirmation Email)**
    - Type: Email Send  
    - From Email: `abc@gmail.com` (static, replace with real sender)  
    - To Email: `xyz@gmail.com` (static, replace with user email)  
    - Subject: `🎉 Your Travel Booking is Confirmed!`  
    - Text: Use generated confirmation message (`{{json.data}}`)  
    - Credentials: SMTP configured  
    - Input: Confirmation Message Generator output

12. **Create Respond to Webhook node (Send Response)**
    - Type: Respond to Webhook  
    - Respond with JSON  
    - Response Body:  
      ```json
      {
        "status": "success",
        "booking": "{{ $('Flight Booking Processor').item.json.booking || $('Hotel Booking Processor').item.json.booking }}",
        "message": "{{ $('Confirmation Message Generator').item.json.choices[0].message.content }}",
        "confirmation_code": "{{ $('Flight Booking Processor').item.json.booking?.confirmation || $('Hotel Booking Processor').item.json.booking?.confirmation }}"
      }
      ```  
    - Input: Send Confirmation Email output

13. **Connect all nodes accordingly** in the sequence described above.

14. **Add Sticky Notes** to document each part of the workflow as per the original comments for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                  |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------|
| The workflow uses mock fallback data for flights and hotels when external APIs fail or return empty results. | Ensures robustness against API downtime         |
| Passenger and guest info (name/email) are statically set to "John Doe" / "john@example.com". Replace with dynamic user data in production. | User personalization enhancement                 |
| Confirmation emails are sent to a static address (`xyz@gmail.com`). Update to use dynamic user emails from input or user profiles. | Email notification customization                  |
| OpenAI GPT-3.5 API usage is central for natural language understanding and message generation; ensure API keys have sufficient quota. | OpenAI API best practices                         |
| Aviationstack and Booking.com API credentials must be preconfigured in n8n with proper scopes and permissions. | External API integration setup                    |

---

This documentation fully captures the workflow logic, node configurations, data flow, and potential failure modes to enable understanding, reproduction, and maintenance for advanced users and automation agents.