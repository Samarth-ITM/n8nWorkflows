Generate Business Leads from Google Maps & Places API to Google Sheets

https://n8nworkflows.xyz/workflows/generate-business-leads-from-google-maps---places-api-to-google-sheets-9857


# Generate Business Leads from Google Maps & Places API to Google Sheets

### 1. Workflow Overview

This workflow automates the generation of business leads from Google Maps using the Google Maps & Places APIs and stores the results in a Google Sheet. It is designed for users who want to find niche-specific businesses by specifying a search term (business type), location, maximum number of results, and a Google Maps API key.

The workflow is logically divided into the following blocks:

- **1.1 User Input Reception:** Collects search parameters through a web form.
- **1.2 Input Preparation:** Standardizes and sets variables from user input.
- **1.3 Location Geocoding:** Converts a location string into geographic coordinates using the Google Geocoding API.
- **1.4 Places Search:** Uses Google Places API to search businesses around the coordinates.
- **1.5 Results Processing:** Splits the list of places and validates each lead.
- **1.6 Lead Formatting:** Formats and enriches each lead’s data.
- **1.7 Lead Storage:** Appends validated leads into a Google Sheet for record keeping.

---

### 2. Block-by-Block Analysis

#### 1.1 User Input Reception

- **Overview:** Captures input from a user via a form to specify search term, location, max results, and API key.
- **Nodes Involved:**  
  - Extraction Configs (Form Trigger)
  - Main Set (Set Node)
- **Node Details:**

  - **Extraction Configs**  
    - Type: Form Trigger  
    - Role: Initiates workflow by receiving user inputs from an HTTP form.  
    - Configuration:  
      - Form fields for: Search Term (business type), Location, Max Results (number), Google Maps API Key.  
      - Webhook ID provided for external form integration.  
    - Inputs: External HTTP request  
    - Outputs: JSON with form data  
    - Edge cases: Missing required fields will prevent trigger; invalid API keys not handled here.

  - **Main Set**  
    - Type: Set  
    - Role: Extracts form data into standardized workflow variables.  
    - Configuration: Assigns the captured form fields into named variables: `location`, `googleApiKey`, `maxResults`, `businessType`.  
    - Inputs: Output from Extraction Configs  
    - Outputs: JSON with clear keys for subsequent nodes  
    - Edge cases: No validation of values; expects valid input.

---

#### 1.2 Location Geocoding

- **Overview:** Converts user-provided location string into latitude and longitude using Google Geocoding API.
- **Nodes Involved:**  
  - Get Location Coordinates (HTTP Request)  
  - Prepare Search Data Set (Set)
- **Node Details:**

  - **Get Location Coordinates**  
    - Type: HTTP Request  
    - Role: Calls Google Geocoding API to get geo-coordinates.  
    - Configuration:  
      - GET request to `https://maps.googleapis.com/maps/api/geocode/json`  
      - Query parameters: `address` from `location` variable, `key` from `googleApiKey`.  
    - Inputs: Output from Main Set  
    - Outputs: JSON containing geocoded results  
    - Edge cases:  
      - Invalid API key or quota exceeded may cause API errors or empty results.  
      - Location string not found returns empty results array.

  - **Prepare Search Data Set**  
    - Type: Set  
    - Role: Extracts latitude and longitude from geocode response and carries forward other required variables.  
    - Configuration:  
      - Extracts `latitude` and `longitude` from first result in geocode response.  
      - Reassigns `businessType`, `googleApiKey`, `maxResults`, and `locationName`.  
    - Inputs: Output from Get Location Coordinates  
    - Outputs: JSON ready for Places API request  
    - Edge cases: Assumes at least one geocode result available; no fallback if empty.

---

#### 1.3 Places Search

- **Overview:** Searches Google Places API for businesses matching the search term around the given coordinates.
- **Nodes Involved:**  
  - Search Google Places (HTTP Request)  
  - Split Results (Split Out)
- **Node Details:**

  - **Search Google Places**  
    - Type: HTTP Request  
    - Role: Executes a POST request to Google Places Text Search API to find businesses.  
    - Configuration:  
      - URL: `https://places.googleapis.com/v1/places:searchText`  
      - Method: POST, JSON body with `textQuery` (business type), `locationBias` (circle with center lat/lng and radius 10,000 meters), `maxResultCount`.  
      - Headers include API key and field mask specifying which fields to return (e.g., name, address, phone, website, rating, types).  
    - Inputs: Output from Prepare Search Data Set  
    - Outputs: JSON with array of `places`  
    - Edge cases:  
      - API limits and quota issues may cause errors.  
      - Empty results if no places match query.

  - **Split Results**  
    - Type: Split Out  
    - Role: Splits each place from the results array into individual workflow items for processing.  
    - Configuration: Splits by the `places` field in the JSON response.  
    - Inputs: Output from Search Google Places  
    - Outputs: Multiple items, each with data of one place  
    - Edge cases: No places found results in no output items.

---

#### 1.4 Results Processing and Validation

- **Overview:** Checks each split business lead for valid contact information before formatting.
- **Nodes Involved:**  
  - If Lead is Valid (If Condition)  
  - Format Lead Data (Set)
- **Node Details:**

  - **If Lead is Valid**  
    - Type: If  
    - Role: Filters leads to only those with either a phone number or a website.  
    - Configuration:  
      - Condition combinator: OR  
      - Checks if either `nationalPhoneNumber` or `websiteUri` exists and is non-empty.  
    - Inputs: Output from Split Results  
    - Outputs: Leads passing the condition proceed; others are discarded.  
    - Edge cases: Leads lacking both phone and website are dropped.

  - **Format Lead Data**  
    - Type: Set  
    - Role: Normalizes and enriches lead data into a consistent schema for storage.  
    - Configuration:  
      - Assigns standardized fields such as `businessName`, `address`, `phone`, `website`, `googleMapsUrl`, `rating`, `totalReviews`, `businessStatus`, `businessTypes`, `placeId`.  
      - Uses fallback values like `'N/A'` when fields are missing.  
      - Adds metadata like search query, search location, and timestamp (`scrapedAt`).  
    - Inputs: Output from If Lead is Valid  
    - Outputs: JSON with formatted lead data  
    - Edge cases: Handles missing fields gracefully with default values.

---

#### 1.5 Lead Storage

- **Overview:** Appends each validated and formatted lead as a new row in a predefined Google Sheet.
- **Nodes Involved:**  
  - Append row in sheet (Google Sheets)
- **Node Details:**

  - **Append row in sheet**  
    - Type: Google Sheets  
    - Role: Writes lead data to a Google Sheets spreadsheet.  
    - Configuration:  
      - Operation: Append  
      - Document ID and Sheet Name: Fixed to a specific Google Sheet used for leads.  
      - Columns mapped explicitly to lead fields including business name, address, phone, website, ratings, types, status, Google Maps URL, and metadata.  
      - Uses Google Sheets OAuth2 credentials.  
    - Inputs: Output from Format Lead Data  
    - Outputs: None (final node)  
    - Edge cases:  
      - Sheets API quota or permission errors may cause failures.  
      - Mapping expects consistent data schema.

---

### 3. Summary Table

| Node Name             | Node Type          | Functional Role                  | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                         |
|-----------------------|--------------------|--------------------------------|------------------------|--------------------------|---------------------------------------------------------------------------------------------------|
| Extraction Configs     | Form Trigger       | Receives user input via form    | -                      | Main Set                 |                                                                                                   |
| Main Set              | Set                | Standardizes input data         | Extraction Configs      | Get Location Coordinates  |                                                                                                   |
| Get Location Coordinates | HTTP Request      | Converts location to coordinates | Main Set               | Prepare Search Data Set   |                                                                                                   |
| Prepare Search Data Set | Set                | Prepares variables for Places API | Get Location Coordinates | Search Google Places      |                                                                                                   |
| Search Google Places   | HTTP Request       | Searches businesses via Places API | Prepare Search Data Set | Split Results             |                                                                                                   |
| Split Results         | Split Out          | Splits places array into individual leads | Search Google Places   | If Lead is Valid          |                                                                                                   |
| If Lead is Valid      | If                 | Filters leads with valid contacts | Split Results           | Format Lead Data          |                                                                                                   |
| Format Lead Data      | Set                | Formats and enriches lead data  | If Lead is Valid        | Append row in sheet       |                                                                                                   |
| Append row in sheet   | Google Sheets      | Stores lead data in Google Sheet | Format Lead Data        | -                        |                                                                                                   |
| Sticky Note           | Sticky Note        | Documentation note              | -                      | -                        | ## Generate Leads from Google Maps Official way to generate leads using Google Maps, using the Google Maps API Key. Generate Niche-Specific Leads by filling up the business type, location you're targeting, max number of leads, and your google maps API key as the form input. |
| Sticky Note1          | Sticky Note        | Documentation note              | -                      | -                        | ## User Input & Search Fill out the form with your search term (business type), location, and Google Maps API key. The workflow converts your location to coordinates and then finds businesses matching your criteria within the specified area. |
| Sticky Note2          | Sticky Note        | Documentation note              | -                      | -                        | ## Leads Validation & Storage The workflow checks each found business for valid contact info, organizes details, and automatically adds each valid lead to the Google Sheet. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node ("Extraction Configs")**  
   - Set form title: "Generate Leads"  
   - Add form fields (all required except the API key):  
     - "What is the Search Term?" (text)  
     - "What is your Location?" (text)  
     - "Max Results?" (number)  
     - "Google Maps API Key" (text)  
   - Save and note the webhook URL for external triggering.

2. **Add a Set Node ("Main Set")**  
   - Connect from Extraction Configs output.  
   - Assign variables:  
     - `location` = value from "What is your Location?" field  
     - `googleApiKey` = value from "Google Maps API Key" field  
     - `maxResults` = value from "Max Results?" field  
     - `businessType` = value from "What is the Search Term?" field

3. **Add HTTP Request Node ("Get Location Coordinates")**  
   - Connect from Main Set.  
   - Method: GET  
   - URL: `https://maps.googleapis.com/maps/api/geocode/json`  
   - Add query parameters:  
     - `address` = `={{ $json.location }}`  
     - `key` = `={{ $json.googleApiKey }}`

4. **Add a Set Node ("Prepare Search Data Set")**  
   - Connect from Get Location Coordinates.  
   - Assign variables:  
     - `latitude` = `={{ $json.results[0].geometry.location.lat }}`  
     - `longitude` = `={{ $json.results[0].geometry.location.lng }}`  
     - Pass through `businessType`, `googleApiKey`, `maxResults`, and assign `locationName` = input `location`.

5. **Add HTTP Request Node ("Search Google Places")**  
   - Connect from Prepare Search Data Set.  
   - Method: POST  
   - URL: `https://places.googleapis.com/v1/places:searchText`  
   - Headers:  
     - `X-Goog-Api-Key` = `={{ $json.googleApiKey }}`  
     - `X-Goog-FieldMask` = `places.displayName,places.formattedAddress,places.nationalPhoneNumber,places.internationalPhoneNumber,places.websiteUri,places.googleMapsUri,places.rating,places.userRatingCount,places.businessStatus,places.types,places.id`  
     - `Content-Type` = `application/json`  
   - Body (JSON):  
   ```
   {
     "textQuery": "{{ $json.businessType }}",
     "locationBias": {
       "circle": {
         "center": {
           "latitude": {{ $json.latitude }},
           "longitude": {{ $json.longitude }}
         },
         "radius": 10000
       }
     },
     "maxResultCount": {{ $json.maxResults }}
   }
   ```
   - Send body as JSON.

6. **Add Split Out Node ("Split Results")**  
   - Connect from Search Google Places.  
   - Configure to split by the `places` field.

7. **Add If Node ("If Lead is Valid")**  
   - Connect from Split Results.  
   - Condition: OR  
     - Check if `nationalPhoneNumber` exists and is non-empty  
     - OR `websiteUri` exists and is non-empty

8. **Add Set Node ("Format Lead Data")**  
   - Connect from If Lead is Valid (true output).  
   - Assign the following fields with fallback defaults where applicable:  
     - `businessName` = `={{ $json.displayName?.text || 'N/A' }}`  
     - `address` = `={{ $json.formattedAddress || 'N/A' }}`  
     - `phone` = `={{ $json.nationalPhoneNumber || $json.internationalPhoneNumber || 'N/A' }}`  
     - `website` = `={{ $json.websiteUri || 'N/A' }}`  
     - `googleMapsUrl` = `={{ $json.googleMapsUri || 'N/A' }}`  
     - `rating` = `={{ $json.rating || 'N/A' }}`  
     - `totalReviews` = `={{ $json.userRatingCount || 0 }}` (number)  
     - `businessStatus` = `={{ $json.businessStatus || 'UNKNOWN' }}`  
     - `businessTypes` = `={{ $json.types?.join(', ') || 'N/A' }}`  
     - `placeId` = `={{ $json.id || 'N/A' }}`  
     - `searchQuery` = `={{ $('Prepare Search Data Set').item.json.businessType }}`  
     - `searchLocation` = `={{ $('Prepare Search Data Set').item.json.locationName }}`  
     - `scrapedAt` = `={{ $now.toISO() }}` (current ISO timestamp)

9. **Add Google Sheets Node ("Append row in sheet")**  
   - Connect from Format Lead Data.  
   - Operation: Append  
   - Configure with Google Sheets OAuth2 credentials.  
   - Spreadsheet: Use the desired Google Sheet document ID and sheet name.  
   - Map columns explicitly to the formatted fields including:  
     Business Name, Address, Phone, Website, Google Maps URL, Rating, Total Reviews, Business Types, Business Status, Place ID, Search Query, Search location, Scraped At.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                              |
|-------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow uses the official Google Maps & Places APIs, requiring a valid API key with Places and Geocoding enabled. | https://developers.google.com/maps/documentation/places/web-service/overview                                   |
| The Google Sheets node requires OAuth2 credentials with edit access to the target spreadsheet for appending rows. | n8n credentials configuration for Google Sheets OAuth2                                                       |
| The workflow assumes valid user input and does not include error handling for API quota limits or invalid locations. | Consider adding error handling or alerts for production use.                                                |
| The API field mask in the Places search is configured to retrieve rich business info including phone numbers and website URLs. | https://developers.google.com/maps/documentation/places/web-service/search                                   |
| The workflow includes sticky notes describing the main functional blocks, useful for documentation and quick understanding. | Internal documentation nodes within the workflow                                                            |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected content. All data handled is legal and public.