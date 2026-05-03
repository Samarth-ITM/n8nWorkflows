Create an Offline DIGIPIN Microservice API for Precise Location Mapping in India

https://n8nworkflows.xyz/workflows/create-an-offline-digipin-microservice-api-for-precise-location-mapping-in-india-5836


# Create an Offline DIGIPIN Microservice API for Precise Location Mapping in India

---
### 1. Workflow Overview

This workflow provides a standalone microservice API for generating and decoding DIGIPIN codes, India's official geolocation code system. It supports precise location mapping by converting geographic coordinates (latitude and longitude) into a standardized 10-character alphanumeric DIGIPIN and vice versa.

**Target Use Cases:**
- Last-mile delivery geolocation encoding
- Digital address verification systems
- Location-based check-in services
- Simplified location sharing across India

**Logical Blocks:**

- **1.1 Encode Request Handling:** Receives latitude and longitude via webhook, generates a DIGIPIN code.
- **1.2 DIGIPIN Generation Logic:** Core JavaScript code that encodes coordinates into DIGIPIN.
- **1.3 Encode Response Handling:** Returns success or error JSON response for DIGIPIN generation.
- **1.4 Decode Request Handling:** Receives a DIGIPIN via webhook, decodes it back into latitude and longitude.
- **1.5 DIGIPIN Decoding Logic:** Core JavaScript code that decodes a DIGIPIN into coordinates.
- **1.6 Decode Response Handling:** Returns success or error JSON response for DIGIPIN decoding.
- **1.7 Documentation and Usage Notes:** Sticky notes providing usage examples and workflow description.

---

### 2. Block-by-Block Analysis

#### 1.1 Encode Request Handling

- **Overview:**  
  Listens for incoming HTTP GET requests with latitude and longitude query parameters and triggers the DIGIPIN generation process.

- **Nodes Involved:**  
  - Encode_Webhook

- **Node Details:**

  - **Encode_Webhook**  
    - Type: Webhook Trigger  
    - Configured with path `generate-digipin` to receive requests like `/generate-digipin?lat=...&lon=...`.  
    - Responds via downstream nodes (responseMode set to "responseNode").  
    - Input: HTTP GET query parameters `lat` and `lon`.  
    - Output: Passes input data to DIGIPIN generation node.  
    - Failure Modes: Missing or invalid lat/lon parameters may cause errors in downstream processing.

#### 1.2 DIGIPIN Generation Logic

- **Overview:**  
  Encodes the provided latitude and longitude into a 10-character DIGIPIN string using a custom grid-based algorithm.

- **Nodes Involved:**  
  - DIGIPIN_Generation_Code

- **Node Details:**

  - **DIGIPIN_Generation_Code**  
    - Type: Code (JavaScript)  
    - Implements a latitude/longitude bounding box for India and subdivides it into a 4x4 grid recursively to generate each character of the DIGIPIN.  
    - Adds hyphens after the 3rd and 6th characters for readability.  
    - Validates that coordinates fall within India’s geographic bounds; throws error if out of range.  
    - Reads `lat` and `lon` from the webhook query parameters of the first input item.  
    - Outputs JSON with either `digipin` for success or `error` message on failure.  
    - Failure Types: Out-of-range coordinates, missing lat/lon parameters, unexpected exceptions.

#### 1.3 Encode Response Handling

- **Overview:**  
  Routes the result of the DIGIPIN generation to a success or error HTTP JSON response.

- **Nodes Involved:**  
  - Switch - Check for Success  
  - Respond to Webhook - Success  
  - Respond to Webhook - Error

- **Node Details:**

  - **Switch - Check for Success**  
    - Type: Switch  
    - Checks if the output JSON contains a non-empty `digipin` field (success) or an `error` field (failure).  
    - Routes to either Respond to Webhook - Success or Respond to Webhook - Error accordingly.

  - **Respond to Webhook - Success**  
    - Type: Respond to Webhook  
    - Sends HTTP 200 response with JSON body including status "Success", input lat/lon, and generated DIGIPIN.  
    - Uses expressions to insert values from the input JSON.

  - **Respond to Webhook - Error**  
    - Type: Respond to Webhook  
    - Sends HTTP 422 response with JSON body including status "Failed", input lat/lon, and error message.

#### 1.4 Decode Request Handling

- **Overview:**  
  Listens for incoming HTTP GET requests containing a DIGIPIN query parameter to decode.

- **Nodes Involved:**  
  - Decode_Webhook

- **Node Details:**

  - **Decode_Webhook**  
    - Type: Webhook Trigger  
    - Configured with path `decode-digipin` to receive requests like `/decode-digipin?digipin=...`.  
    - Passes the input to the DIGIPIN decoding code node.  
    - Failure Modes: Missing or invalid `digipin` parameter may cause errors in downstream nodes.

#### 1.5 DIGIPIN Decoding Logic

- **Overview:**  
  Decodes a valid DIGIPIN string back into latitude and longitude coordinates by reversing the grid subdivision process.

- **Nodes Involved:**  
  - DIGIPIN_Decode_Code

- **Node Details:**

  - **DIGIPIN_Decode_Code**  
    - Type: Code (JavaScript)  
    - Removes hyphens from DIGIPIN and validates length and characters against the DIGIPIN grid.  
    - Iteratively narrows latitude and longitude bounds for each character to find target coordinate center.  
    - Outputs JSON with `decodedLatitude` and `decodedLongitude` formatted to 6 decimal places.  
    - Throws error if DIGIPIN is invalid or empty.  
    - Uses the `digipin` query parameter from the webhook input JSON.  
    - Failure Types: Invalid length, invalid characters, missing DIGIPIN parameter.

#### 1.6 Decode Response Handling

- **Overview:**  
  Routes decoded coordinates or error messages to appropriate HTTP JSON responses.

- **Nodes Involved:**  
  - Switch 2 - Check for Success1  
  - Respond to Webhook - Success 2  
  - Respond to Webhook - Error 2

- **Node Details:**

  - **Switch 2 - Check for Success1**  
    - Type: Switch  
    - Checks for presence of `decodedLatitude` (success) or `error` field (failure) in the JSON.  
    - Routes to success or error response nodes accordingly.

  - **Respond to Webhook - Success 2**  
    - Type: Respond to Webhook  
    - Returns HTTP 200 with success JSON including original DIGIPIN and decoded coordinates.

  - **Respond to Webhook - Error 2**  
    - Type: Respond to Webhook  
    - Returns HTTP 422 with failure JSON including DIGIPIN and error message.

#### 1.7 Documentation and Usage Notes

- **Overview:**  
  Provides user guidance and example API calls via sticky notes on the workflow canvas for easier understanding and testing.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1

- **Node Details:**

  - **Sticky Note1** (Positioned near Encode block)  
    - Contains detailed explanation of the workflow purpose, DIGIPIN system, usage, and no external API or credential requirements.

  - **Sticky Note** (Positioned near Decode block)  
    - Provides example `curl` commands for generating and decoding DIGIPINs via the webhook endpoints.

---

### 3. Summary Table

| Node Name                    | Node Type              | Functional Role                      | Input Node(s)                   | Output Node(s)                         | Sticky Note                                                                                     |
|------------------------------|------------------------|------------------------------------|--------------------------------|---------------------------------------|------------------------------------------------------------------------------------------------|
| Encode_Webhook                | Webhook                | Entry point for DIGIPIN generation | None                           | DIGIPIN_Generation_Code                |                                                                                                |
| DIGIPIN_Generation_Code       | Code                   | Encodes lat/lon into DIGIPIN       | Encode_Webhook                 | Switch - Check for Success             |                                                                                                |
| Switch - Check for Success    | Switch                 | Routes success or error response   | DIGIPIN_Generation_Code        | Respond to Webhook - Success, Respond to Webhook - Error |                                                                                                |
| Respond to Webhook - Success  | Respond to Webhook     | Sends success HTTP response (generate) | Switch - Check for Success     | None                                  |                                                                                                |
| Respond to Webhook - Error    | Respond to Webhook     | Sends error HTTP response (generate) | Switch - Check for Success     | None                                  |                                                                                                |
| Decode_Webhook                | Webhook                | Entry point for DIGIPIN decoding   | None                           | DIGIPIN_Decode_Code                    |                                                                                                |
| DIGIPIN_Decode_Code           | Code                   | Decodes DIGIPIN into lat/lon       | Decode_Webhook                | Switch 2 - Check for Success1          |                                                                                                |
| Switch 2 - Check for Success1 | Switch                 | Routes success or error response   | DIGIPIN_Decode_Code           | Respond to Webhook - Success 2, Respond to Webhook - Error 2 |                                                                                                |
| Respond to Webhook - Success 2| Respond to Webhook     | Sends success HTTP response (decode) | Switch 2 - Check for Success1 | None                                  |                                                                                                |
| Respond to Webhook - Error 2  | Respond to Webhook     | Sends error HTTP response (decode) | Switch 2 - Check for Success1 | None                                  |                                                                                                |
| Sticky Note1                 | Sticky Note            | Workflow description and usage     | None                           | None                                  | DIGIPIN Generator and Decoder explanation block with usage instructions and no API needed      |
| Sticky Note                  | Sticky Note            | Example API usage commands          | None                           | None                                  | Example curl commands for generate and decode endpoints                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node "Encode_Webhook":**  
   - Type: Webhook  
   - Path: `generate-digipin`  
   - HTTP Method: GET (default)  
   - Response Mode: `responseNode` (to defer response to downstream nodes)  

2. **Create Code Node "DIGIPIN_Generation_Code":**  
   - Type: Code (JavaScript)  
   - Paste the provided DIGIPIN generation JavaScript code that:  
     - Validates latitude/longitude within India bounds  
     - Converts coordinates into a 10-character DIGIPIN using a 4x4 grid subdivision algorithm  
     - Sets `digipin` or `error` fields in JSON  
   - Configure to process input items, reading `lat` and `lon` from `$input.first().json.query.lat` and `.lon`.

3. **Connect "Encode_Webhook" output to "DIGIPIN_Generation_Code" input.**

4. **Create Switch Node "Switch - Check for Success":**  
   - Type: Switch  
   - Add two rules:  
     - Success: Check if `digipin` exists and is non-empty in JSON (`string exists`)  
     - Error: Check if `error` exists and is non-empty in JSON  

5. **Connect "DIGIPIN_Generation_Code" output to "Switch - Check for Success" input.**

6. **Create Respond to Webhook Node "Respond to Webhook - Success":**  
   - Response Code: 200  
   - Respond With: JSON  
   - Response Body: JSON string with keys:  
     - `status`: "Success"  
     - `lat`: from `$json.query.lat`  
     - `lon`: from `$json.query.lon`  
     - `digipin`: from `$json.digipin`

7. **Create Respond to Webhook Node "Respond to Webhook - Error":**  
   - Response Code: 422  
   - Respond With: JSON  
   - Response Body: JSON string with keys:  
     - `status`: "Failed"  
     - `lat`: from `$json.query.lat`  
     - `lon`: from `$json.query.lon`  
     - `error`: from `$json.error`

8. **Connect "Switch - Check for Success" outputs:**  
   - Success to "Respond to Webhook - Success"  
   - Error to "Respond to Webhook - Error"

9. **Create Webhook Node "Decode_Webhook":**  
   - Type: Webhook  
   - Path: `decode-digipin`  
   - HTTP Method: GET (default)  
   - Response Mode: `responseNode`

10. **Create Code Node "DIGIPIN_Decode_Code":**  
    - Type: Code (JavaScript)  
    - Paste the provided DIGIPIN decoding JavaScript code that:  
      - Validates the DIGIPIN string format and characters  
      - Converts DIGIPIN back to latitude and longitude, formatted to 6 decimals  
      - Sets `decodedLatitude`, `decodedLongitude`, or `error` in JSON  
    - Reads DIGIPIN from `$input.first().json.query.digipin`.

11. **Connect "Decode_Webhook" output to "DIGIPIN_Decode_Code" input.**

12. **Create Switch Node "Switch 2 - Check for Success1":**  
    - Type: Switch  
    - Add two rules:  
      - Success: Check if `decodedLatitude` exists and is non-empty   
      - Error: Check if `error` exists and is non-empty

13. **Connect "DIGIPIN_Decode_Code" output to "Switch 2 - Check for Success1" input.**

14. **Create Respond to Webhook Node "Respond to Webhook - Success 2":**  
    - Response Code: 200  
    - Respond With: JSON  
    - Response Body: JSON string with keys:  
      - `status`: "Success"  
      - `digipin`: from `$json.query.digipin`  
      - `lat`: from `$json.decodedLatitude`  
      - `lon`: from `$json.decodedLongitude`

15. **Create Respond to Webhook Node "Respond to Webhook - Error 2":**  
    - Response Code: 422  
    - Respond With: JSON  
    - Response Body: JSON string with keys:  
      - `status`: "Failed"  
      - `digipin`: from `$json.query.digipin`  
      - `error`: from `$json.error`

16. **Connect "Switch 2 - Check for Success1" outputs:**  
    - Success to "Respond to Webhook - Success 2"  
    - Error to "Respond to Webhook - Error 2"

17. **Optional: Add Sticky Note1 near Encode block:**  
    - Content describing workflow purpose, no external API or credentials needed, usage instructions.

18. **Optional: Add Sticky Note near Decode block:**  
    - Content showing example curl commands for generate and decode endpoints.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                  | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| DIGIPIN is India Post’s official geolocation code system encoding lat/lon into a 10-character alphanumeric string with hyphens, simplifying precise location sharing in India. No external APIs or credentials are required.                      | Workflow description and design rationale                                                         |
| Example curl usage for generation: `curl --request GET 'https://n8n.example.in/webhook/generate-digipin?lat=27.175063&lon=78.042169'` and decoding: `curl --request GET 'https://n8n.example.in/webhook/decode-digipin?digipin=32C-849-5CJ6'` | Usage examples sticky note                                                                          |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.