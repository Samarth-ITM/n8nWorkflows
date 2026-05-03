Dental Clinic Appointment & Admin System with Supabase, Phone Integration & AI

https://n8nworkflows.xyz/workflows/dental-clinic-appointment---admin-system-with-supabase--phone-integration---ai-11463


# Dental Clinic Appointment & Admin System with Supabase, Phone Integration & AI

### 1. Workflow Overview

This workflow automates appointment scheduling and administrative tasks for a dental clinic using Supabase as the database backend, phone integration via webhook, and AI assistance for availability calculations. It supports multiple interaction types triggered by a phone-based assistant, including booking, rescheduling, canceling appointments, checking insurance status, retrieving appointment or doctor lists, and querying doctor availability.

The workflow is structured into these logical blocks:

- **1.1 Input Reception & Routing:** Receives incoming webhook POST requests from the phone assistant, parses the user's intent (`tool_name`), and routes processing accordingly.
- **1.2 Patient & Insurance Lookup:** Queries Supabase to identify patients by phone number or insurance member ID, ensuring subsequent operations have correct user context.
- **1.3 Appointment Booking:** Checks slot availability, verifies or creates patient records, inserts appointment data, and notifies the user of success or conflicts.
- **1.4 Appointment Rescheduling:** Fetches current appointments, validates new requested slots, updates records if available, and sends confirmation or conflict notifications.
- **1.5 Appointment Cancellation:** Deletes the patient’s appointment record and confirms the cancellation.
- **1.6 Insurance Verification:** Looks up insurance records and returns acceptance or rejection messages.
- **1.7 Appointment and Doctor Listing:** Retrieves and summarizes patient appointments or doctor information, formatting results for conversational output.
- **1.8 AI-Driven Availability Search:** Uses a language model agent to analyze existing appointments and generate a list of available 60-minute time slots within working hours, excluding weekends.

Each block finishes by responding to the webhook with structured JSON that the phone assistant uses to communicate to the patient.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Routing

**Overview:**  
This block handles incoming HTTP POST requests from the phone assistant webhook. It extracts the intent (`tool_name`) from the request and routes the flow to the appropriate subsequent logic branch. It also initiates patient or insurance lookups as needed.

**Nodes Involved:**  
- Webhook  
- Switch  
- Find Patient (Booking)  
- Find Patient (Reschedule)  
- Find Patient (Cancel)  
- Find Patient (List Appointments)  
- Find Insurance Record  
- Generate Availability With AI  
- Fetch All Doctors

**Node Details:**

- **Webhook**  
  - Type: Webhook (HTTP POST listener)  
  - Role: Entry point receiving JSON with user request from phone assistant  
  - Config: Path set to a unique ID, POST method, responds asynchronously  
  - Inputs: External HTTP POST  
  - Outputs: Connected to Switch  
  - Failures: Malformed JSON, invalid HTTP method, network errors

- **Switch**  
  - Type: Switch (conditional routing)  
  - Role: Routes flow based on `body.args.tool_name` field (e.g., "set_appointment", "reschedule", "cancel", etc.)  
  - Config: Checks string equality on `tool_name` against known values for routing  
  - Inputs: Webhook output  
  - Outputs: Branches to patient or insurance lookup nodes or AI availability node  
  - Failures: Missing or unknown tool_name leads to no match, flow stops

- **Find Patient (Booking/Reschedule/Cancel/List Appointments)**  
  - Type: Supabase Query  
  - Role: Query `dental_patients_canada` table (or `dental_insurance` for List) filtering by phone number or member ID  
  - Config: Filters on phone_number or member_id extracted from request args  
  - Inputs: From Switch branches  
  - Outputs: Patient or insurance data for next processing nodes  
  - Failures: Query timeout, missing Supabase credentials, empty result sets, case sensitivity issues

- **Find Insurance Record**  
  - Same as above but specifically for insurance lookup  
  - Queries dental_patients_canada by phone number

- **Generate Availability With AI**  
  - Type: LangChain Agent (AI prompt execution)  
  - Role: Receives doctor ID, start, end dates; uses AI to generate available 60-minute slots excluding weekends and outside working hours  
  - Inputs: From Switch (on get_availability)  
  - Outputs: Availability list to respond node  
  - Failures: AI API errors, prompt malformed, rate limits

- **Fetch All Doctors**  
  - Type: Supabase Query  
  - Role: Retrieves full list of doctors from `dental_doctors_canada` table  
  - Outputs: Aggregated and summarized for response  
  - Failures: Database connectivity, empty doctor table

---

#### 2.2 Patient & Insurance Lookup

**Overview:**  
This block performs database lookups to identify patients or insurance records based on phone number or member ID, providing context for subsequent operations.

**Nodes Involved:**  
- Find Patient (Booking)  
- Find Patient (Reschedule)  
- Find Patient (Cancel)  
- Find Patient (List Appointments)  
- Find Insurance Record

**Node Details:**

- Each node queries `dental_patients_canada` or `dental_insurance` tables with filters on phone_number or member_id from the incoming request.  
- Always outputs data even if empty for downstream conditional checks.  
- Potential failure modes include empty results (patient not found), database errors, or misconfigured filters.

---

#### 2.3 Appointment Booking

**Overview:**  
Checks if the requested time slot is free for the specified doctor, verifies if the patient exists or creates a new patient record, then inserts the appointment record. Sends confirmation or conflict notifications.

**Nodes Involved:**  
- Check Time Availability (If node)  
- Get Existing Appointments (Supabase)  
- Time Slot Already Taken? (If node)  
- Create Appointment (Supabase) [creates patient record]  
- Verify Patient Phone Number (Supabase)  
- Create Appointment Record (Supabase)  
- Notify Appointment Conflict (Respond to Webhook)  
- Notify Appointment Time (Respond to Webhook)

**Node Details:**

- **Check Time Availability**  
  - Checks if patient ID exists after lookup (non-empty)  
  - Forwards flow to get existing appointments or create patient record

- **Get Existing Appointments**  
  - Queries `dental_appointments_canada` for existing appointments on requested date and doctor_id  
  - Returns all matching appointments to check for conflicts

- **Time Slot Already Taken?**  
  - If any existing appointment found, triggers conflict notification  
  - Else proceeds to patient verification and appointment creation

- **Create Appointment**  
  - Inserts patient record if new (name, phone, email, doctor_id)

- **Verify Patient Phone Number**  
  - Re-queries patient to get ID for appointment insertion

- **Create Appointment Record**  
  - Inserts appointment with patient_id, doctor_id, appointment_date, reason/service_type

- **Notify Appointment Conflict**  
  - Responds with JSON message indicating the time slot is already booked

- **Notify Appointment Time**  
  - Responds with JSON confirmation including dynamic variables like date and patient name

- Failure Modes: Slot conflicts, patient record creation failure, DB insert errors, webhook response errors

---

#### 2.4 Appointment Rescheduling

**Overview:**  
Retrieves the patient's current appointment, checks if the doctor is available at the preferred new date, updates the appointment if possible, or notifies about conflicts.

**Nodes Involved:**  
- Fetch Patient's Current Appointment (Supabase)  
- Check Doctor Availability for New Slot (Supabase)  
- Is Reschedule Possible? (If node)  
- Notify: Time Slot Taken (Respond to Webhook)  
- Update Appointment to New Slot (Supabase)  
- Confirm Rescheduled Appointment (Respond to Webhook)

**Node Details:**

- **Fetch Patient's Current Appointment**  
  - Queries `dental_appointments_canada` filtering by patient_id and current appointment date

- **Check Doctor Availability for New Slot**  
  - Queries appointments on preferred_date and doctor_id to detect conflicts

- **Is Reschedule Possible?**  
  - If conflict found, sends notification of taken slot  
  - Else updates appointment date

- **Update Appointment to New Slot**  
  - Updates appointment_date field for matched doctor and old appointment date

- **Confirm Rescheduled Appointment**  
  - Sends JSON confirmation with new date in message

- Failures: Patient or appointment not found, DB update failures, concurrency conflicts

---

#### 2.5 Appointment Cancellation

**Overview:**  
Deletes the appointment record for the patient on the specified date and sends confirmation.

**Nodes Involved:**  
- Delete Appointment Record (Supabase)  
- Notify Cancelation Success (Respond to Webhook)

**Node Details:**

- **Delete Appointment Record**  
  - Deletes from `dental_appointments_canada` where patient_id and appointment_date match

- **Notify Cancelation Success**  
  - Responds with confirmation message JSON

- Failures: Missing appointment record, DB delete errors

---

#### 2.6 Insurance Verification

**Overview:**  
Checks if the patient insurance record exists and sends acceptance or rejection message accordingly.

**Nodes Involved:**  
- Check Insurance Status (If node)  
- Notify Insurance Accepted (Respond to Webhook)  
- Notify Insurance Rejected (Respond to Webhook)

**Node Details:**

- **Check Insurance Status**  
  - Checks existence of insurance record by verifying if `id` exists in fetched data

- **Notify Insurance Accepted / Rejected**  
  - Responds to webhook with appropriate message JSON

- Failures: DB query failure, missing insurance record

---

#### 2.7 Appointment and Doctor Listing

**Overview:**  
Fetches all appointments for a patient or all doctors, summarizes and aggregates the data, and formats it for the phone assistant.

**Nodes Involved:**  
- Fetch Patient Appointments (Supabase)  
- Aggregate Patient Appointments  
- Summarize Appointment Data  
- Send Summarized Appointment List (Respond to Webhook)  
- Fetch All Doctors (Supabase)  
- Aggregate Doctors Data  
- Summarize Doctors Data  
- Send Doctors List (Respond to Webhook)

**Node Details:**

- Supabase nodes query tables with filters (patient_id for appointments, no filter for doctors)  
- Aggregate nodes combine all items into unified data arrays  
- Summarize nodes concatenate and clean data fields  
- Respond nodes send structured JSON with dynamic variables to the caller

- Failures: Empty result sets, aggregation errors, malformed responses

---

#### 2.8 AI-Driven Availability Search

**Overview:**  
This block uses an AI model to analyze existing doctor appointments and generate available 60-minute time slots between requested dates, respecting working hours and excluding weekends.

**Nodes Involved:**  
- Fetch Doctor’s Appointments (Supabase)  
- LLM Engine (OpenAI GPT-4o)  
- Generate Availability With AI (LangChain Agent)  
- Send Availability Response (Respond to Webhook)

**Node Details:**

- **Fetch Doctor’s Appointments**  
  - Retrieves appointments for the given doctor_id

- **LLM Engine**  
  - Connects to OpenAI GPT-4o model for text generation

- **Generate Availability With AI**  
  - Uses a prompt defining working hours, weekends, appointment duration, and existing appointments to generate free slots

- **Send Availability Response**  
  - Sends AI-generated availability list as a JSON response

- Failures: AI API errors, prompt misinterpretation, rate limiting, database query issues

---

### 3. Summary Table

| Node Name                      | Node Type                         | Functional Role                               | Input Node(s)                     | Output Node(s)                       | Sticky Note                                                                                                          |
|--------------------------------|----------------------------------|-----------------------------------------------|----------------------------------|------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Webhook                        | Webhook                         | Entry point receiving webhook POST             | External HTTP POST               | Switch                             |                                                                                                                      |
| Switch                        | Switch                         | Routes flow based on `tool_name`               | Webhook                        | Find Patient (Booking), Find Patient (Reschedule), Find Patient (Cancel), Find Patient (List Appointments), Find Insurance Record, Generate Availability With AI, Fetch All Doctors | Sticky Note1: Request Routing & Patient Lookup                                                                        |
| Find Patient (Booking)         | Supabase                       | Lookup patient by phone for booking             | Switch                        | Check Time Availability            | Sticky Note1                                                                                                         |
| Find Patient (Reschedule)      | Supabase                       | Lookup patient by phone for rescheduling         | Switch                        | Fetch Patient's Current Appointment | Sticky Note1                                                                                                         |
| Find Patient (Cancel)           | Supabase                       | Lookup patient by phone for cancellation         | Switch                        | Delete Appointment Record          | Sticky Note1                                                                                                         |
| Find Patient (List Appointments) | Supabase                     | Lookup patient insurance by member ID            | Switch                        | Check Insurance Status             | Sticky Note1                                                                                                         |
| Find Insurance Record           | Supabase                       | Lookup insurance record by phone number          | Switch                        | Fetch Patient Appointments         | Sticky Note1                                                                                                         |
| Generate Availability With AI  | LangChain Agent                | AI generates available time slots                | Switch, LLM Engine, Fetch Doctor’s Appointments | Send Availability Response         | Sticky Note7                                                                                                         |
| Fetch All Doctors              | Supabase                       | Retrieve all doctors                             | Switch                        | Aggregate Doctors Data             | Sticky Note8                                                                                                         |
| Check Time Availability        | If                            | Checks if patient exists before booking           | Find Patient (Booking)          | Get Existing Appointments, Create Appointment | Sticky Note2                                                                                                         |
| Get Existing Appointments      | Supabase                       | Retrieve appointments for doctor and date         | Check Time Availability        | Time Slot Already Taken?           | Sticky Note2                                                                                                         |
| Time Slot Already Taken?       | If                            | Checks if time slot is already booked              | Get Existing Appointments       | Notify Appointment Conflict, Verify Patient Phone Number | Sticky Note3                                                                                                         |
| Create Appointment             | Supabase                       | Create patient record                             | Check Time Availability        | Get Existing Appointments          | Sticky Note2                                                                                                         |
| Verify Patient Phone Number    | Supabase                       | Verify patient exists for appointment creation    | Time Slot Already Taken?       | Create Appointment Record          | Sticky Note3                                                                                                         |
| Create Appointment Record      | Supabase                       | Insert new appointment record                      | Verify Patient Phone Number    | Notify Appointment Time            | Sticky Note3                                                                                                         |
| Notify Appointment Conflict    | Respond to Webhook             | Notify user that time slot is already booked       | Time Slot Already Taken?       | -                                | Sticky Note3                                                                                                         |
| Notify Appointment Time        | Respond to Webhook             | Confirm appointment booking                         | Create Appointment Record      | -                                | Sticky Note3                                                                                                         |
| Fetch Patient's Current Appointment | Supabase                | Retrieve current appointment for rescheduling      | Find Patient (Reschedule)      | Check Doctor Availability for New Slot | Sticky Note4                                                                                                         |
| Check Doctor Availability for New Slot | Supabase              | Check if doctor is free at new preferred date      | Fetch Patient's Current Appointment | Is Reschedule Possible?           | Sticky Note4                                                                                                         |
| Is Reschedule Possible?        | If                            | Conditional on slot availability for reschedule    | Check Doctor Availability for New Slot | Notify: Time Slot Taken, Update Appointment to New Slot | Sticky Note5                                                                                                         |
| Notify: Time Slot Taken        | Respond to Webhook             | Notify user that preferred reschedule date is booked | Is Reschedule Possible?        | -                                | Sticky Note5                                                                                                         |
| Update Appointment to New Slot | Supabase                       | Update appointment date in DB                       | Is Reschedule Possible?        | Confirm Rescheduled Appointment    | Sticky Note5                                                                                                         |
| Confirm Rescheduled Appointment | Respond to Webhook            | Send confirmation of reschedule                     | Update Appointment to New Slot | -                                | Sticky Note5                                                                                                         |
| Delete Appointment Record      | Supabase                       | Delete appointment record for cancellation          | Find Patient (Cancel)          | Notify Cancelation Success         | Sticky Note9                                                                                                         |
| Notify Cancelation Success     | Respond to Webhook             | Confirm cancellation                                | Delete Appointment Record      | -                                | Sticky Note9                                                                                                         |
| Check Insurance Status         | If                            | Validate insurance record existence                  | Find Patient (List Appointments) | Notify Insurance Accepted, Notify Insurance Rejected | Sticky Note10                                                                                                        |
| Notify Insurance Accepted      | Respond to Webhook             | Notify user insurance is accepted                     | Check Insurance Status         | -                                | Sticky Note10                                                                                                        |
| Notify Insurance Rejected      | Respond to Webhook             | Notify user insurance is not accepted                 | Check Insurance Status         | -                                | Sticky Note10                                                                                                        |
| Fetch Patient Appointments     | Supabase                       | Retrieve all appointments for patient                  | Find Insurance Record          | Aggregate Patient Appointments     | Sticky Note6                                                                                                         |
| Aggregate Patient Appointments | Aggregate                     | Aggregate all appointment data                         | Fetch Patient Appointments     | Summarize Appointment Data         | Sticky Note6                                                                                                         |
| Summarize Appointment Data    | Summarize                     | Summarize appointment data for response                | Aggregate Patient Appointments | Send Summarized Appointment List   | Sticky Note6                                                                                                         |
| Send Summarized Appointment List | Respond to Webhook          | Send summarized list of patient appointments          | Summarize Appointment Data    | -                                | Sticky Note6                                                                                                         |
| Aggregate Doctors Data        | Aggregate                     | Aggregate all doctors data                              | Fetch All Doctors              | Summarize Doctors Data             | Sticky Note8                                                                                                         |
| Summarize Doctors Data        | Summarize                     | Summarize doctors data for response                     | Aggregate Doctors Data         | Send Doctors List                 | Sticky Note8                                                                                                         |
| Send Doctors List             | Respond to Webhook             | Send list of doctors                                   | Summarize Doctors Data         | -                                | Sticky Note8                                                                                                         |
| Fetch Doctor’s Appointments   | Supabase Tool                 | Retrieve appointments for a specific doctor            | -                             | LLM Engine, Generate Availability With AI | Sticky Note7                                                                                                         |
| LLM Engine                   | LangChain LM Chat OpenAI       | Provides GPT-4o model for AI prompt execution          | Fetch Doctor’s Appointments    | Generate Availability With AI     | Sticky Note7                                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Use unique path (e.g., "3beec299-79db-453e-8986-b1b98e82209d")  
   - Response Mode: Response Node  
   - Purpose: Accept incoming requests from phone system

2. **Add Switch Node**  
   - Connect Webhook output to Switch input  
   - Configure rules on `{{$json.body.args.tool_name}}` with cases:  
     - "set_appointment" → Find Patient (Booking)  
     - "reschedule" → Find Patient (Reschedule)  
     - "cancel" → Find Patient (Cancel)  
     - "insurance_check" → Find Insurance Record  
     - "get_appointmnets_list" → Find Patient (List Appointments)  
     - "get_availability" → Generate Availability With AI  
     - "get_doctors_list" → Fetch All Doctors

3. **Set Up Supabase Credentials**  
   - Add Supabase API credential with your project details

4. **Create Patient Lookup Nodes**  
   - For Booking, Reschedule, Cancel, List Appointments, Insurance:  
     - Supabase node querying `dental_patients_canada` or `dental_insurance`  
     - Filters on phone_number or member_id as per node  
     - Operation: Get or GetAll  
     - Always output data

5. **Build Booking Branch:**  
   - After Find Patient (Booking), add If Node "Check Time Availability" to test if patient ID exists  
   - Connect to Supabase node "Get Existing Appointments" filtering by appointment_date and doctor_id  
   - Add If Node "Time Slot Already Taken?" checking if any existing appointment exists  
   - If taken, connect to Respond Node "Notify Appointment Conflict" with JSON message  
   - If free, connect to Supabase node "Verify Patient Phone Number" to confirm patient data  
   - Then Supabase node "Create Appointment Record" inserting appointment with patient_id, doctor_id, appointment_date, reason/service_type  
   - Finally, Respond Node "Notify Appointment Time" sending JSON confirmation with dynamic variables (date, name)

6. **Build Reschedule Branch:**  
   - After Find Patient (Reschedule), add Supabase node "Fetch Patient's Current Appointment" filtering by patient_id and current appointment_date  
   - Then Supabase node "Check Doctor Availability for New Slot" filtering by preferred_date and doctor_id  
   - Add If Node "Is Reschedule Possible?" checking if new slot is free  
   - If slot taken, Respond Node "Notify: Time Slot Taken" with JSON message  
   - Else Supabase node "Update Appointment to New Slot" updating appointment_date  
   - Then Respond Node "Confirm Rescheduled Appointment" confirming new date

7. **Build Cancellation Branch:**  
   - After Find Patient (Cancel), Supabase node "Delete Appointment Record" filtering by patient_id and appointment_date  
   - Then Respond Node "Notify Cancelation Success" with confirmation message

8. **Build Insurance Check Branch:**  
   - After Find Patient (List Appointments), add If Node "Check Insurance Status" testing if insurance record exists (id present)  
   - If yes, Respond Node "Notify Insurance Accepted"  
   - Else Respond Node "Notify Insurance Rejected"

9. **Build Appointment List Branch:**  
   - After Find Insurance Record, Supabase node "Fetch Patient Appointments" filtering by patient_id  
   - Aggregate node to aggregate all appointment items  
   - Summarize node to concatenate appointment data fields  
   - Respond Node "Send Summarized Appointment List" sending appointment data JSON

10. **Build Doctor List Branch:**  
    - Supabase node "Fetch All Doctors" retrieving all from `dental_doctors_canada`  
    - Aggregate node to collect all doctor items  
    - Summarize node to concatenate doctor data fields  
    - Respond Node "Send Doctors List" sending doctor data JSON

11. **Build Availability Search Branch:**  
    - Supabase node "Fetch Doctor’s Appointments" filtering by doctor_id  
    - LangChain LLM Engine node configured with OpenAI GPT-4o credentials  
    - LangChain Agent node "Generate Availability With AI" with prompt specifying working hours, weekends, duration, and existing appointments  
    - Respond Node "Send Availability Response" sending AI-generated time slots JSON

12. **Connect all nodes as per workflow connections:**  
    - Ensure each branch flows from Switch to respective lookup, checks, updates, and response nodes

13. **Test all branches with sample inputs:**  
    - Validate booking, rescheduling, canceling, insurance check, listing appointments/doctors, and availability functions

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                   | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow automates dental appointment management via phone assistant including booking, rescheduling, canceling, insurance checking, and availability queries.                                                                                  | Sticky Note (positioned near Webhook and Switch nodes)                                             |
| Setup requires adding Supabase credentials, connecting phone system to the webhook URL, matching table/column names in Supabase, and thorough testing before production deployment.                                                             | Sticky Note (positioned near Webhook and Switch nodes)                                             |
| The AI availability uses working hours 8:00 AM to 5:00 PM, excludes Saturdays and Sundays, and finds free 60-minute slots.                                                                                                                    | Sticky Note7                                                                                        |
| The appointment conflict and reschedule logic ensure no double-booking occurs, with clear user notifications.                                                                                                                                  | Sticky Note3, Sticky Note5                                                                          |
| Use valid OpenAI credentials for LangChain nodes to enable AI-driven availability calculations.                                                                                                                                                 | LLM Engine node credential configuration                                                          |
| Supabase tables used: dental_patients_canada, dental_appointments_canada, dental_doctors_canada, dental_insurance. Ensure schema matches expected fields (patient_id, phone_number, doctor_id, appointment_date, member_id, etc.)                   | Across all Supabase nodes                                                                           |

---

This completes the comprehensive, structured reference document for the provided n8n workflow, enabling understanding, reproduction, and modification by advanced users and AI agents.