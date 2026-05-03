Complete Booking System with Google Calendar, Business Hours & REST API

https://n8nworkflows.xyz/workflows/complete-booking-system-with-google-calendar--business-hours---rest-api-8635


# Complete Booking System with Google Calendar, Business Hours & REST API

### 1. Workflow Overview

This workflow implements a **Complete Booking System** integrating Google Calendar, business hours logic, public holidays, and a REST API interface for frontend integration. It is designed to handle booking requests, validate inputs, check availability considering business hours and public holidays, and create calendar events automatically if slots are available.

The workflow contains two main REST API endpoints:

- **1.1 Make a Booking Endpoint**: Receives booking requests (name, email, phone, date, time), validates inputs and business rules, checks availability, and creates Google Calendar events for confirmed bookings.

- **1.2 Get Booking Slots Endpoint**: Receives a date and returns the availability of timeslots for that day, factoring in public holidays, weekends, business hours, and existing bookings.

Logical blocks within the workflow:

- **Input Reception & Validation**: Receiving webhook requests and validating required fields and formats.

- **Business Hours & Date Validation**: Ensuring requested times are within defined business hours, excluding breaks, weekends, and public holidays.

- **Availability Checking**: Verifying no conflicting events exist in Google Calendar during the requested timeslot.

- **Calendar Event Creation**: Creating a booking event in a dedicated Google Calendar.

- **Response Handling**: Preparing and sending success or error JSON responses back to the API caller.

- **Supporting Logic for Timeslot Availability**: For the timeslot query endpoint, determining which slots are available or booked.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Initial Validation

**Overview:**  
This block receives incoming booking requests via a webhook, extracts and validates required booking details such as name, email, phone, date, and time.

**Nodes Involved:**  
- Booking Webhook  
- Wait  
- Validate Input  
- Validation Check  
- Prepare Validation Error  
- Error Response  

**Node Details:**

- **Booking Webhook**  
  - Type: Webhook  
  - Role: Entry point for booking requests via HTTP POST at `/make-booking`  
  - Config: Allows all origins (`*`), uses response node mode for synchronous response  
  - Input: HTTP POST JSON body  
  - Output: Passes request data to next nodes  
  - Edge Cases: Invalid HTTP method, missing body  

- **Wait**  
  - Type: Wait  
  - Role: Minor delay (2.2 seconds) before validation (can be used for rate limiting or processing stability)  
  - Input: Booking Webhook output  
  - Output: To Validate Input  

- **Validate Input**  
  - Type: Code  
  - Role: Parses input JSON, extracts booking fields, validates required fields and format (email, phone, date, time)  
  - Key Expressions: Regex for email, phone, date (`YYYY-MM-DD`), and time (`HH:MM`) validation  
  - Output: JSON object with `success`, `error`, and parsed `bookingData`  
  - Edge Cases: Missing fields, invalid formats return failure with error messages  

- **Validation Check**  
  - Type: If  
  - Role: Routes flow based on validation result (`success` boolean)  
  - Input: Validate Input output  
  - Output: Passes valid data to Business Hours Check or errors to Prepare Validation Error  

- **Prepare Validation Error**  
  - Type: Code  
  - Role: Prepares structured error response JSON for input validation failures  
  - Input: Validation Check error branch  
  - Output: Error JSON for webhook response  

- **Error Response**  
  - Type: Respond to Webhook  
  - Role: Sends JSON validation error response with HTTP Content-Type `application/json`  
  - Input: Prepare Validation Error output  
  - Output: Ends workflow execution with error response  

---

#### 2.2 Business Hours & Date Validation

**Overview:**  
Validates that the booking time falls within Malaysia business hours (Mon-Fri 9am-9pm excluding lunch and dinner breaks) and prepares timestamps for calendar events.

**Nodes Involved:**  
- Business Hours Check  
- Time Validation Check  
- Prepare Time Error  
- Time Error Response  

**Node Details:**

- **Business Hours Check**  
  - Type: Code  
  - Role: Checks if booking date/time is weekday, during business hours, excluding lunch (12-2pm) and dinner (6-8pm) breaks, all in Malaysia timezone (UTC+8)  
  - Key Logic:  
    - Converts date/time strings to Date objects  
    - Validates day of week and time ranges  
    - Prepares ISO formatted datetime strings for Google Calendar  
  - Output: JSON with validation booleans, error messages, booking data, and formatted times  
  - Edge Cases: Bookings on weekends, outside hours, or during breaks flagged with error messages  

- **Time Validation Check**  
  - Type: If  
  - Role: Routes flow based on business hours validation success  
  - Input: Business Hours Check output  
  - Output: Passes valid time to calendar availability check or errors to Prepare Time Error  

- **Prepare Time Error**  
  - Type: Code  
  - Role: Formats error response JSON for business hours violations, including requested time details and business hours info  
  - Input: Time Validation Check error branch  
  - Output: Detailed error data for webhook response  

- **Time Error Response**  
  - Type: Respond to Webhook  
  - Role: Sends JSON response to client indicating booking outside allowed hours  
  - Input: Prepare Time Error output  
  - Output: Ends workflow with error response  

---

#### 2.3 Availability Checking

**Overview:**  
Checks requested booking slot against two Google Calendars: a public holiday calendar and a main booking calendar to detect conflicts.

**Nodes Involved:**  
- Check Calendar Availability - Main  
- Check Calendar Availability - public holiday  
- Check Availability  
- Availability Check  
- Prepare Availability Error  
- Availability Error Response  

**Node Details:**

- **Check Calendar Availability - Main**  
  - Type: Google Calendar (GetAll)  
  - Role: Retrieves events from main booking calendar between booking start and end times  
  - Config: Uses OAuth2 credentials for Google Calendar, filtered by start/end timestamps  
  - Input: Date/time from Business Hours Check node  
  - Output: List of calendar events  

- **Check Calendar Availability - public holiday**  
  - Type: Google Calendar (GetAll)  
  - Role: Retrieves events from Malaysian public holiday calendar for the booking date  
  - Input: Date/time from Business Hours Check node  
  - Output: Holiday events  

- **Check Availability**  
  - Type: Code  
  - Role: Compares retrieved calendar events for conflicts with requested booking time  
  - Logic: Filters events overlapping the requested time slot; returns availability boolean and conflict details  
  - Input: Events from both calendars, booking data, start/end datetime  
  - Output: JSON indicating availability and conflict messages  

- **Availability Check**  
  - Type: If  
  - Role: Routes flow based on availability boolean  
  - Input: Check Availability output  
  - Output: Proceeds to create event or prepares availability error  

- **Prepare Availability Error**  
  - Type: Code  
  - Role: Formats JSON error response for unavailable booking slots with conflict details and suggestions  
  - Input: Availability Check error branch  
  - Output: Error JSON for webhook response  

- **Availability Error Response**  
  - Type: Respond to Webhook  
  - Role: Sends JSON response indicating slot unavailability  
  - Input: Prepare Availability Error output  
  - Output: Ends workflow execution with error response  

---

#### 2.4 Calendar Event Creation & Success Response

**Overview:**  
Creates a booking event in Google Calendar and prepares a confirmation JSON response to send back to the client.

**Nodes Involved:**  
- Create Calendar Event  
- Prepare Success Response  
- Success Response  

**Node Details:**

- **Create Calendar Event**  
  - Type: Google Calendar (Create)  
  - Role: Creates an event on the main booking calendar with booking details, attendee email, and Google Meet conferencing link  
  - Config: Uses OAuth2 credentials, sets event color, visibility, summary, description, and disables guests inviting others  
  - Input: Booking data and validated datetime from Business Hours Check  
  - Output: Created event data including event ID and link  
  - Edge Cases: API permissions or quota errors, event creation failures  

- **Prepare Success Response**  
  - Type: Code  
  - Role: Formats JSON success response including booking details, event ID, event link, and a confirmation message  
  - Input: Create Calendar Event output and original booking data  
  - Output: Success JSON for webhook response  

- **Success Response**  
  - Type: Respond to Webhook  
  - Role: Sends the JSON success confirmation response to the client  
  - Input: Prepare Success Response output  
  - Output: Ends workflow with success response  

---

#### 2.5 Get Booking Slots Endpoint (Fetch Available Time Slots)

**Overview:**  
Handles requests to fetch available booking time slots for a specified date, considering public holidays, weekends, and existing bookings.

**Nodes Involved:**  
- Booking Timeslot webhook  
- Wait1  
- ConfigTimeSlots  
- Validate Date1  
- Check Public Holiday Calendar  
- Check Calendar Availability  
- Process Holiday Check Calendar  
- Holiday Response Calendar  

**Node Details:**

- **Booking Timeslot webhook**  
  - Type: Webhook  
  - Role: Entry point for timeslot availability requests at `/check-booking-date` via POST  
  - Config: Allows all origins, synchronous response mode  

- **Wait1**  
  - Type: Wait  
  - Role: Minor delay (2.2 seconds) for processing stability  

- **ConfigTimeSlots**  
  - Type: Set  
  - Role: Defines a fixed array of configurable timeslots with times, display labels, and availability flags to represent business hours and breaks  
  - Output: Array of timeslot objects  

- **Validate Date1**  
  - Type: Code  
  - Role: Validates the requested date for correct format and checks if it is a weekend  
  - Output: Parsed date details and weekend flag  

- **Check Public Holiday Calendar**  
  - Type: Google Calendar (GetAll)  
  - Role: Retrieves Malaysian public holiday events for the requested date  

- **Check Calendar Availability**  
  - Type: Google Calendar (GetAll)  
  - Role: Retrieves main booking calendar events for the requested date  

- **Process Holiday Check Calendar**  
  - Type: Code  
  - Role: Determines if the date is a public holiday or weekend, prepares messages, and marks timeslots booked if conflicts exist based on calendar events  
  - Output: JSON with success flag, working day flag, holiday/weekend messages, availableSlots array with booking statuses, and conflict counts  

- **Holiday Response Calendar**  
  - Type: Respond to Webhook  
  - Role: Sends JSON response with available booking slots and date information to the client  

---

### 3. Summary Table

| Node Name                         | Node Type               | Functional Role                                    | Input Node(s)              | Output Node(s)                      | Sticky Note                                                                                                                               |
|----------------------------------|-------------------------|--------------------------------------------------|----------------------------|-----------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| Booking Webhook                  | Webhook                 | Entry point for booking requests                  |                            | Wait                              | ## Endpoint for making a booking<br>Receives the input name,email,phone and datetime for booking                                            |
| Wait                            | Wait                    | Delay before validation                            | Booking Webhook            | Validate Input                    |                                                                                                                                          |
| Validate Input                  | Code                    | Validate booking input fields and formats         | Wait                      | Validation Check                  | ## Handle validation<br>1. name + email validation<br>2. phone validation<br>3. date time validation                                         |
| Validation Check               | If                      | Routes based on input validation                   | Validate Input             | Business Hours Check, Prepare Validation Error |                                                                                                                                          |
| Prepare Validation Error       | Code                    | Format validation error response                   | Validation Check (Error)   | Error Response                   |                                                                                                                                          |
| Error Response                | Respond to Webhook       | Respond with validation error JSON                 | Prepare Validation Error    |                                   |                                                                                                                                          |
| Business Hours Check          | Code                    | Validate booking time within business hours        | Validation Check (Success) | Time Validation Check            | ## Handle validation<br>1. datetime is weekend working day?<br>2. is a non-business hours?<br>3. is a lunch or dinner hours?               |
| Time Validation Check         | If                      | Routes on business hours validation                 | Business Hours Check       | Check Calendar Availability - Main, Prepare Time Error |                                                                                                                                          |
| Prepare Time Error            | Code                    | Format error response for business hours violation | Time Validation Check (Error) | Time Error Response              |                                                                                                                                          |
| Time Error Response           | Respond to Webhook       | Respond with business hours error JSON             | Prepare Time Error         |                                   |                                                                                                                                          |
| Check Calendar Availability - Main | Google Calendar (GetAll) | Fetch events from main booking calendar            | Time Validation Check      | Check Calendar Availability - public holiday |                                                                                                                                          |
| Check Calendar Availability - public holiday | Google Calendar (GetAll) | Fetch public holiday events                         | Check Calendar Availability - Main | Check Availability              |                                                                                                                                          |
| Check Availability            | Code                    | Check for conflicting events in calendars          | Check Calendar Availability - public holiday | Availability Check              | ## Checking datetime<br>1. Malaysian public holiday<br>2. Event booking conflict                                                           |
| Availability Check            | If                      | Routes on availability result                       | Check Availability         | Create Calendar Event, Prepare Availability Error | ## Handle validation<br>1. datetime is conflicting with another event<br>2. datetime is a malaysian public holiday                         |
| Prepare Availability Error    | Code                    | Format error response for slot unavailability      | Availability Check (Error) | Availability Error Response      |                                                                                                                                          |
| Availability Error Response   | Respond to Webhook       | Respond with slot unavailability error JSON        | Prepare Availability Error |                                   |                                                                                                                                          |
| Create Calendar Event         | Google Calendar (Create) | Create booking event in Google Calendar            | Availability Check (Success) | Prepare Success Response         | ## Creating the calendar entry                                                                                                             |
| Prepare Success Response      | Code                    | Format success response JSON with booking details  | Create Calendar Event      | Success Response                 |                                                                                                                                          |
| Success Response              | Respond to Webhook       | Respond with booking confirmation JSON             | Prepare Success Response   |                                   |                                                                                                                                          |
| Booking Timeslot webhook      | Webhook                 | Entry point for fetching booking timeslots         |                            | Wait1                           | ## Endpoint for getting booking slots by specific date<br>Receives the timeslots for booking                                               |
| Wait1                        | Wait                    | Delay before processing timeslot request            | Booking Timeslot webhook   | ConfigTimeSlots                  |                                                                                                                                          |
| ConfigTimeSlots              | Set                     | Defines configurable booking timeslots              | Wait1                     | Validate Date1                  |                                                                                                                                          |
| Validate Date1              | Code                    | Validate date format and weekend status             | ConfigTimeSlots            | Check Public Holiday Calendar    |                                                                                                                                          |
| Check Public Holiday Calendar | Google Calendar (GetAll) | Fetch public holiday events for requested date      | Validate Date1             | Check Calendar Availability       |                                                                                                                                          |
| Check Calendar Availability  | Google Calendar (GetAll) | Fetch calendar events for requested date            | Check Public Holiday Calendar | Process Holiday Check Calendar  |                                                                                                                                          |
| Process Holiday Check Calendar | Code                    | Determine holiday/weekend status and time slot availability | Check Calendar Availability | Holiday Response Calendar       |                                                                                                                                          |
| Holiday Response Calendar    | Respond to Webhook       | Respond with available timeslots and date info      | Process Holiday Check Calendar |                                 |                                                                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "Booking Webhook"**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `make-booking`  
   - Allowed Origins: `*`  
   - Response Mode: `responseNode`  

2. **Add Wait Node: "Wait"**  
   - Delay: 2.2 seconds  
   - Connect output of "Booking Webhook" to this node  

3. **Add Code Node: "Validate Input"**  
   - Paste JavaScript code to parse JSON body and validate fields: name, email, phone, date, time  
   - Use regex for email, phone, date (`YYYY-MM-DD`), time (`HH:MM`)  
   - Output JSON with `success`, `error`, and `bookingData` fields  
   - Connect "Wait" output to this node  

4. **Add If Node: "Validation Check"**  
   - Condition: `$json.success === true`  
   - True branch: proceed to business hours check  
   - False branch: proceed to validation error preparation  
   - Connect "Validate Input" output to this node  

5. **Add Code Node: "Prepare Validation Error"**  
   - Prepare JSON response with `success: false`, error message, and details  
   - Connect "Validation Check" false branch here  

6. **Add Respond to Webhook Node: "Error Response"**  
   - Respond with JSON from "Prepare Validation Error"  
   - Content-Type header: `application/json`  
   - Connect "Prepare Validation Error" output here  

7. **Add Code Node: "Business Hours Check"**  
   - JavaScript to check if booking is on Mon-Fri, 9am-9pm Malaysia time, excluding 12-2pm and 6-8pm breaks  
   - Prepare ISO datetime strings for calendar event start and end  
   - Output includes `isValidTime` boolean and `errorMessage` string  
   - Connect "Validation Check" true branch here  

8. **Add If Node: "Time Validation Check"**  
   - Condition: `$json.isValidTime === true`  
   - True branch: proceed to check calendar availability  
   - False branch: proceed to prepare time error  
   - Connect "Business Hours Check" output here  

9. **Add Code Node: "Prepare Time Error"**  
   - Format error JSON with business hours violation details and requested time info  
   - Connect "Time Validation Check" false branch here  

10. **Add Respond to Webhook Node: "Time Error Response"**  
    - Respond with JSON from "Prepare Time Error"  
    - Content-Type header: `application/json`  
    - Connect "Prepare Time Error" output here  

11. **Add Google Calendar Node: "Check Calendar Availability - Main"**  
    - Operation: GetAll events  
    - Calendar ID: Your booking calendar ID  
    - TimeMin: `={{ $json.startDateTime }}`  
    - TimeMax: `={{ $json.endDateTime }}`  
    - Use Google Calendar OAuth2 credentials  
    - Connect "Time Validation Check" true branch here  

12. **Add Google Calendar Node: "Check Calendar Availability - public holiday"**  
    - Operation: GetAll events  
    - Calendar ID: public holiday calendar (e.g., `en.malaysia#holiday@group.v.calendar.google.com`)  
    - TimeMin and TimeMax same as above  
    - Use Google Calendar OAuth2 credentials  
    - Connect "Check Calendar Availability - Main" output here  

13. **Add Code Node: "Check Availability"**  
    - JavaScript to check for event conflicts in both calendars  
    - Returns `isAvailable` boolean, conflict messages, and booking data  
    - Connect "Check Calendar Availability - public holiday" output here  

14. **Add If Node: "Availability Check"**  
    - Condition: `$json.isAvailable === true`  
    - True branch: proceed to create calendar event  
    - False branch: prepare availability error  
    - Connect "Check Availability" output here  

15. **Add Code Node: "Prepare Availability Error"**  
    - Format JSON error for conflicting booking time slot  
    - Connect "Availability Check" false branch here  

16. **Add Respond to Webhook Node: "Availability Error Response"**  
    - Respond with JSON from "Prepare Availability Error"  
    - Content-Type header: `application/json`  
    - Connect "Prepare Availability Error" output here  

17. **Add Google Calendar Node: "Create Calendar Event"**  
    - Operation: Create event  
    - Calendar ID: Your booking calendar ID  
    - Start: `={{ $json.startDateTimeCal }}`  
    - End: `={{ $json.endDateTimeCal }}`  
    - Additional fields: summary, attendees (email), description, disable guests inviting others, add Google Meet conferencing  
    - Use Google Calendar OAuth2 credentials  
    - Connect "Availability Check" true branch here  

18. **Add Code Node: "Prepare Success Response"**  
    - Format success JSON with booking details and calendar event info  
    - Connect "Create Calendar Event" output here  

19. **Add Respond to Webhook Node: "Success Response"**  
    - Respond with JSON from "Prepare Success Response"  
    - Content-Type header: `application/json`  
    - Connect "Prepare Success Response" output here  

---

**Setup for Get Booking Slots Endpoint:**

20. **Create Webhook Node: "Booking Timeslot webhook"**  
    - HTTP Method: POST  
    - Path: `check-booking-date`  
    - Allowed Origins: `*`  

21. **Add Wait Node: "Wait1"**  
    - Delay: 2.2 seconds  
    - Connect "Booking Timeslot webhook" output here  

22. **Add Set Node: "ConfigTimeSlots"**  
    - Define array of timeslots with time, display label, and availability flags representing business hours and breaks  

23. **Add Code Node: "Validate Date1"**  
    - Validate the date field in request body  
    - Check if date is weekend  
    - Output parsed date info and success status  

24. **Add Google Calendar Node: "Check Public Holiday Calendar"**  
    - Get all events from public holiday calendar for the requested date  

25. **Add Google Calendar Node: "Check Calendar Availability"**  
    - Get all events from main booking calendar for the requested date  

26. **Add Code Node: "Process Holiday Check Calendar"**  
    - Determine if date is holiday or weekend  
    - Mark timeslots as booked if conflicts exist  
    - Output available slots and date info  

27. **Add Respond to Webhook Node: "Holiday Response Calendar"**  
    - Respond with available timeslots and date information JSON  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                                                                                                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| ## Create a Booking : MAKE A BOOKING ENDPOINT - This workflow provides a webhook API endpoint for frontend integration to submit bookings. It validates inputs, checks business hours and conflicts, and creates Google Calendar events. Example curl is included. ![Demo](https://raw.githubusercontent.com/dragonjump/suarify-booking/59a519ce4ebbaf58e4173c0914dce3ec617c44f8/demo-makebooking.gif)                                                                                                                                               | See Sticky Note12 for detailed instructions and example request/response                                                                                                                                                                            |
| ## Get Booking Slots : FETCH BOOKING SLOTS ENDPOINT - Provides available time slots for a selected date, marking booked slots, and indicating non-working days like holidays or weekends. Example curl and response included. ![Demo](https://raw.githubusercontent.com/dragonjump/suarify-booking/59a519ce4ebbaf58e4173c0914dce3ec617c44f8/demo-timeslot.gif)                                                                                                                                                                  | See Sticky Note13 for detailed instructions and example request/response                                                                                                                                                                            |
| Google Calendar Setup: Add your public holiday calendar (e.g., Malaysia holidays) and create a dedicated booking calendar. Assign Google OAuth2 credentials with Calendar access to the Google Calendar nodes.                                                                                                                                                                                                                                                                                                                                                         | Links: [n8n Google Calendar Setup](https://docs.n8n.io/integrations/builtin/credentials/google/), [Google Calendar Node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlecalendar/)                                            |
| Business Hours can be adjusted by modifying the `ConfigTimeSlots` Set node or editing the JavaScript in the Business Hours Check node to change allowed times or breaks.                                                                                                                                                                                                                                                                                                                                                                                           |                                                                                                                                                                                                                                                     |
| Demo and frontend example code are available at https://github.com/dragonjump/suarify-booking and live demo at https://dragonjump.github.io/suarify-booking/                                                                                                                                                                                                                                                                                                                                                                                                         | See Sticky Note14 for links and setup instructions                                                                                                                                                                                                  |
| Timezone note: The workflow uses Malaysia timezone (Asia/Kuala_Lumpur, UTC+8) for all date/time computations and Google Calendar event times. JavaScript Date objects are carefully adjusted to ensure proper timezone offsets in ISO strings.                                                                                                                                                                                                                                                                                                                      |                                                                                                                                                                                                                                                     |

---

**Disclaimer:**  
This documentation is generated from an automated n8n workflow export. It complies with all content policies, contains no illegal or protected content, and processes only legal and public data.