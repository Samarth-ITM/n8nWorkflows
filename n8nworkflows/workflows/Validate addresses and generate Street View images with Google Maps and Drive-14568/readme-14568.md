Validate addresses and generate Street View images with Google Maps and Drive

https://n8nworkflows.xyz/workflows/validate-addresses-and-generate-street-view-images-with-google-maps-and-drive-14568


# Validate addresses and generate Street View images with Google Maps and Drive

# 1. Workflow Overview

This workflow collects a postal address from an n8n form, validates it with the Google Geocoding API, requests a Google Street View image for that location, uploads the resulting image to Google Drive, and returns an HTML page that redirects the user to the uploaded file. If the address cannot be resolved or no Street View image is available, it returns an error page instead.

Typical use cases:
- Internal tools for address verification with visual confirmation
- Real-estate or field-operations workflows needing location imagery
- Customer-facing forms that convert addresses into downloadable/viewable Street View assets

## 1.1 Input Reception

The workflow starts from a public form where the user submits an address string.

## 1.2 Address Validation

The submitted address is normalized into request fields and sent to the Google Geocoding API. The response is checked to confirm the address exists and Google returned a valid result.

## 1.3 Location Data Processing

If the address is valid, the workflow extracts the formatted address, latitude, longitude, and place ID, then prepares location data for image retrieval.

## 1.4 Street View Retrieval

The workflow requests a Street View image from Google Maps. The node is configured to receive binary file data rather than JSON.

## 1.5 Image Availability Check and Storage

The binary response is tested. If an image file exists, it is uploaded to a specific Google Drive folder.

## 1.6 User Response Rendering

On success, the workflow generates an HTML page that redirects the user to the uploaded Google Drive file. On failure, it generates an HTML error page with a retry option.

## 1.7 Error Handling

Two failure branches are implemented:
- Address validation failure
- Street View image unavailable

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception

### Overview
This block exposes a form to the user and captures a required address field. It is the workflow entry point and provides the raw user input used throughout the rest of the process.

### Nodes Involved
- On form submission

### Node Details

#### On form submission
- **Type and technical role:** `n8n-nodes-base.formTrigger`  
  Trigger node that starts the workflow when a user submits the hosted n8n form.
- **Configuration choices:**
  - Form title: **Street View Locator**
  - Description: asks the user to enter a physical address to retrieve a street-level or location photo
  - One required form field: **Search Address**
  - Placeholder: `e.g. 1600 Amphitheatre Pkwy, Mountain View, CA`
  - Attribution disabled
- **Key expressions or variables used:**  
  No expressions inside the trigger itself; it produces `Search Address` in the output JSON.
- **Input and output connections:**
  - Input: none, entry point
  - Output: to **Prepare API request data**
- **Version-specific requirements:**  
  Uses `typeVersion: 2.3` of the Form Trigger node.
- **Edge cases or potential failure types:**
  - Form endpoint inaccessible if workflow is inactive in production
  - Required-field validation prevents empty submission
  - Field naming matters downstream; changing `Search Address` requires updating expressions later
- **Sub-workflow reference:**  
  None

---

## Block 2 — Address Validation

### Overview
This block transforms the form submission into API-ready fields, then calls Google Geocoding to determine whether the address exists and can be resolved. It branches immediately based on the validity of the geocoding response.

### Nodes Involved
- Prepare API request data
- Geocode address
- Validate address

### Node Details

#### Prepare API request data
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates a simplified payload containing the user’s address and the Google API key.
- **Configuration choices:**
  - Assigns `address` from the form field
  - Assigns `api_key` as a static string placeholder: `YOU_API_KEY_HERE`
  - `alwaysOutputData` is enabled
- **Key expressions or variables used:**
  - `={{ $json["Search Address"] }}`
- **Input and output connections:**
  - Input: **On form submission**
  - Output: **Geocode address**
- **Version-specific requirements:**  
  Uses Set node `typeVersion: 3.4`.
- **Edge cases or potential failure types:**
  - Placeholder API key must be replaced; otherwise Google API requests will fail
  - If the form field name changes, the expression breaks or becomes empty
  - Hardcoding the API key is functional but not ideal for security
- **Sub-workflow reference:**  
  None

#### Geocode address
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls Google’s Geocoding API with the submitted address.
- **Configuration choices:**
  - Method defaults to GET
  - URL: `https://maps.googleapis.com/maps/api/geocode/json`
  - Query parameters:
    - `address = {{$json.address}}`
    - `key = {{$json.api_key}}`
- **Key expressions or variables used:**
  - `={{ $json.address }}`
  - `={{ $json.api_key }}`
- **Input and output connections:**
  - Input: **Prepare API request data**
  - Output: **Validate address**
- **Version-specific requirements:**  
  Uses HTTP Request node `typeVersion: 4.3`.
- **Edge cases or potential failure types:**
  - Invalid or restricted Google Maps API key
  - Geocoding API not enabled in Google Cloud
  - Quota exceeded or billing disabled
  - Network failures or non-200 responses
  - Ambiguous addresses may still return `OK`, but not necessarily the intended location
- **Sub-workflow reference:**  
  None

#### Validate address
- **Type and technical role:** `n8n-nodes-base.if`  
  Determines whether the geocoding response is usable.
- **Configuration choices:**
  - Logical combinator: AND
  - Conditions:
    1. `results` is not empty
    2. `status` equals `OK`
- **Key expressions or variables used:**
  - `={{ $json.results }}`
  - `={{ $json.status }}`
- **Input and output connections:**
  - Input: **Geocode address**
  - True output: **Extract location data**
  - False output: **Erro address**
- **Version-specific requirements:**  
  Uses If node `typeVersion: 2.2` with conditions version 2.
- **Edge cases or potential failure types:**
  - Some Google responses may include statuses like `ZERO_RESULTS`, `OVER_QUERY_LIMIT`, `REQUEST_DENIED`, or `INVALID_REQUEST`
  - `results` structure assumptions depend on Google API behavior
  - Loose type validation is enabled, which is usually tolerant but can hide data-shape issues
- **Sub-workflow reference:**  
  None

---

## Block 3 — Location Data Processing

### Overview
After a successful geocoding response, this block extracts the most relevant location values and formats them for downstream use. It preserves a normalized address and coordinates from the first geocoding result.

### Nodes Involved
- Extract location data
- Format coordinates

### Node Details

#### Extract location data
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates a compact object from the first geocoding result.
- **Configuration choices:**
  - Produces:
    - `address` from `results[0].formatted_address`
    - `lat` from `results[0].geometry.location.lat`
    - `lng` from `results[0].geometry.location.lng`
    - `place_id` from `results[0].place_id`
  - Configured in raw JSON output mode
- **Key expressions or variables used:**
  - `{{ $json.results[0].formatted_address }}`
  - `{{ $json.results[0].geometry.location.lat }}`
  - `{{ $json.results[0].geometry.location.lng }}`
  - `{{ $json.results[0].place_id }}`
- **Input and output connections:**
  - Input: **Validate address** true branch
  - Output: **Format coordinates**
- **Version-specific requirements:**  
  Set node `typeVersion: 3.4`.
- **Edge cases or potential failure types:**
  - Assumes `results[0]` exists; protected in practice by the previous If node
  - If Google changes the response structure, expressions may fail
  - Multiple valid geocoding results are ignored except the first one
- **Sub-workflow reference:**  
  None

#### Format coordinates
- **Type and technical role:** `n8n-nodes-base.set`  
  Produces a comma-separated `lat,lng` string.
- **Configuration choices:**
  - Raw JSON output
  - Creates `location` as `lat + ',' + lng`
- **Key expressions or variables used:**
  - `{{ $json.lat + ',' + $json.lng }}`
- **Input and output connections:**
  - Input: **Extract location data**
  - Output: **Get street view image**
- **Version-specific requirements:**  
  Set node `typeVersion: 3.4`.
- **Edge cases or potential failure types:**
  - If `lat` or `lng` is missing or non-numeric, the string will be malformed
  - This output is currently not actually used by the next node, which instead references the original address
- **Sub-workflow reference:**  
  None

**Important implementation note:**  
Although this node prepares a coordinate string, the Street View request uses the original address from **Prepare API request data** rather than `{{$json.location}}`. So this block is informative/preparatory, but not operationally necessary in the current version.

---

## Block 4 — Street View Retrieval

### Overview
This block requests a Street View image from Google Maps and expects a binary file response. It is the core media-fetch step of the workflow.

### Nodes Involved
- Get street view image
- Check street view availability

### Node Details

#### Get street view image
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the Google Street View API and stores the result as binary data.
- **Configuration choices:**
  - URL: `https://maps.googleapis.com/maps/api/streetview`
  - Query parameters:
    - `size = 600x400`
    - `location = {{$('Prepare API request data').item.json.address}}`
    - `key = {{$('Prepare API request data').item.json.api_key}}`
  - Response format set to **file**
- **Key expressions or variables used:**
  - `={{ $('Prepare API request data').item.json.address }}`
  - `={{ $('Prepare API request data').item.json.api_key }}`
- **Input and output connections:**
  - Input: **Format coordinates**
  - Output: **Check street view availability**
- **Version-specific requirements:**  
  HTTP Request node `typeVersion: 4.3`.
- **Edge cases or potential failure types:**
  - Street View API not enabled
  - Invalid or restricted API key
  - Quota/billing issues
  - A file may still be returned even when Street View imagery is unavailable, depending on API behavior and parameters
  - Because it uses the original address instead of coordinates, ambiguous addresses may fetch a different result than the geocoded coordinates imply
- **Sub-workflow reference:**  
  None

#### Check street view availability
- **Type and technical role:** `n8n-nodes-base.if`  
  Checks whether binary data exists on the response.
- **Configuration choices:**
  - Condition tests whether `$binary.data !== undefined`
- **Key expressions or variables used:**
  - `=={{ $binary.data !== undefined }}`
- **Input and output connections:**
  - Input: **Get street view image**
  - True output: **Upload image**
  - False output: **Erro image**
- **Version-specific requirements:**  
  If node `typeVersion: 2.2`.
- **Edge cases or potential failure types:**
  - This check only confirms binary presence, not image validity
  - Google Street View can return a generic placeholder image as a valid binary file
  - The expression begins with `=={{ ... }}`; if copied manually, ensure it is accepted exactly as configured in n8n
- **Sub-workflow reference:**  
  None

**Important implementation note:**  
A more robust Street View availability check would call Street View metadata first, or inspect HTTP status / image content. In the current workflow, any binary file is treated as success.

---

## Block 5 — Image Storage

### Overview
If a binary image is present, this block uploads it to a specific Google Drive folder. The output from Google Drive includes a link later used for redirection.

### Nodes Involved
- Upload image

### Node Details

#### Upload image
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Uploads the binary file stored under `data` to Google Drive.
- **Configuration choices:**
  - Binary property name: `data`
  - Drive: `My Drive`
  - Folder: `StreetViewImages`
  - Uses OAuth2 Google Drive credentials
- **Key expressions or variables used:**  
  No custom expressions shown in the visible configuration.
- **Input and output connections:**
  - Input: **Check street view availability** true branch
  - Output: **Display page**
- **Version-specific requirements:**  
  Google Drive node `typeVersion: 3`.
- **Edge cases or potential failure types:**
  - Missing or expired Google Drive OAuth2 credentials
  - Folder deleted or access revoked
  - Binary property name mismatch if the prior node output changes
  - Drive permission constraints may affect generated share links
  - If the uploaded file is private, `webViewLink` may not work for external users
- **Sub-workflow reference:**  
  None

---

## Block 6 — Success Response Rendering

### Overview
This block returns a small HTML page to the user after a successful upload. It immediately redirects the browser to the Google Drive `webViewLink`.

### Nodes Involved
- Display page

### Node Details

#### Display page
- **Type and technical role:** `n8n-nodes-base.html`  
  Sends an HTML response back to the browser.
- **Configuration choices:**
  - Displays a processing message
  - JavaScript redirect to `{{$json.webViewLink}}`
  - Fallback clickable link
- **Key expressions or variables used:**
  - `{{ $json.webViewLink }}`
- **Input and output connections:**
  - Input: **Upload image**
  - Output: none
- **Version-specific requirements:**  
  HTML node `typeVersion: 1.2`.
- **Edge cases or potential failure types:**
  - `webViewLink` must exist in the Google Drive node output
  - Redirect may fail if browser blocks script execution
  - If Drive file sharing is restricted, the user may land on an access-denied page
- **Sub-workflow reference:**  
  None

---

## Block 7 — Error Handling

### Overview
This block builds user-friendly error payloads and renders them as HTML pages. It handles both invalid-address and unavailable-image scenarios.

### Nodes Involved
- Erro address
- Erro image
- Display error page

### Node Details

#### Erro address
- **Type and technical role:** `n8n-nodes-base.set`  
  Builds an error object for geocoding failures.
- **Configuration choices:**
  - Raw JSON output:
    - `error = "Address not found"`
    - `input = {{ $json.address }}`
- **Key expressions or variables used:**
  - `{{ $json.address }}`
- **Input and output connections:**
  - Input: **Validate address** false branch
  - Output: **Display error page**
- **Version-specific requirements:**  
  Set node `typeVersion: 3.4`.
- **Edge cases or potential failure types:**
  - If `address` is absent, the input field on the error page becomes blank
  - Spelling of the node name is nonstandard but harmless
- **Sub-workflow reference:**  
  None

#### Erro image
- **Type and technical role:** `n8n-nodes-base.set`  
  Builds an error object for Street View retrieval failures.
- **Configuration choices:**
  - Raw JSON output:
    - `error = "Street View not available for this location"`
- **Key expressions or variables used:**  
  None
- **Input and output connections:**
  - Input: **Check street view availability** false branch
  - Output: **Display error page**
- **Version-specific requirements:**  
  Set node `typeVersion: 3.4`.
- **Edge cases or potential failure types:**
  - Because binary existence is the only check upstream, this branch may rarely trigger even when imagery is effectively unusable
- **Sub-workflow reference:**  
  None

#### Display error page
- **Type and technical role:** `n8n-nodes-base.html`  
  Renders an HTML error page with a retry button.
- **Configuration choices:**
  - Styled error page
  - Displays `{{$json.error}}`
  - Optionally displays the input value if present
  - Retry button uses `javascript:history.back()`
- **Key expressions or variables used:**
  - `{{ $json.error }}`
  - `{{ $json.input ? "Input: " + $json.input : "" }}`
- **Input and output connections:**
  - Input: **Erro address**, **Erro image**
  - Output: none
- **Version-specific requirements:**  
  HTML node `typeVersion: 1.2`.
- **Edge cases or potential failure types:**
  - `history.back()` may not behave as expected if the page was opened directly
  - If `error` is missing, the page still renders but message content is poor
- **Sub-workflow reference:**  
  None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On form submission | Form Trigger | Receives the user-submitted address from the hosted form |  | Prepare API request data | ## User input |
| Prepare API request data | Set | Maps form field to `address` and injects Google API key | On form submission | Geocode address | ## Address validation |
| Geocode address | HTTP Request | Calls Google Geocoding API to validate and resolve the address | Prepare API request data | Validate address | ## Address validation |
| Validate address | If | Branches based on geocoding success (`results` not empty and `status = OK`) | Geocode address | Extract location data, Erro address | ## Address validation |
| Extract location data | Set | Extracts normalized address, latitude, longitude, and place ID from geocoding result | Validate address | Format coordinates | ## Process location data |
| Format coordinates | Set | Builds a `lat,lng` string for location formatting | Extract location data | Get street view image | ## Process location data |
| Get street view image | HTTP Request | Requests Street View image as binary file from Google Maps | Format coordinates | Check street view availability | ## Retrieve street view |
| Check street view availability | If | Confirms whether binary image data exists | Get street view image | Upload image, Erro image | ## Retrieve street view |
| Upload image | Google Drive | Uploads the Street View image to Google Drive folder `StreetViewImages` | Check street view availability | Display page | ## Store image |
| Display page | HTML | Returns HTML that redirects user to the Google Drive file link | Upload image |  | ## Generate response page |
| Erro address | Set | Creates a structured error payload for invalid/unresolved addresses | Validate address | Display error page | ## Error handling |
| Erro image | Set | Creates a structured error payload when Street View is unavailable | Check street view availability | Display error page | ## Error handling |
| Display error page | HTML | Renders an HTML error response with retry button | Erro address, Erro image |  | ## Error handling |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like **Find address photo**.

2. **Add a Form Trigger node**.
   - Node type: **Form Trigger**
   - Title: `Street View Locator`
   - Description: `Enter a physical address to retrieve its corresponding street view or location photo.`
   - Add one form field:
     - Label: `Search Address`
     - Placeholder: `e.g. 1600 Amphitheatre Pkwy, Mountain View, CA`
     - Required: enabled
   - Disable attribution if desired.

3. **Add a Set node** after the form.
   - Name: `Prepare API request data`
   - Create fields:
     - `address` as string with expression: `{{$json["Search Address"]}}`
     - `api_key` as string with your Google Maps API key
   - Replace the placeholder with a real key.
   - Prefer using an environment variable or credential strategy if available in your deployment.

4. **Connect `On form submission` → `Prepare API request data`.**

5. **Add an HTTP Request node** named `Geocode address`.
   - Method: GET
   - URL: `https://maps.googleapis.com/maps/api/geocode/json`
   - Enable query parameters:
     - `address` = `{{$json.address}}`
     - `key` = `{{$json.api_key}}`

6. **Connect `Prepare API request data` → `Geocode address`.**

7. **Add an If node** named `Validate address`.
   - Set condition combinator to **AND**
   - Condition 1:
     - Left value: `{{$json.results}}`
     - Operator: **is not empty**
   - Condition 2:
     - Left value: `{{$json.status}}`
     - Operator: **equals**
     - Right value: `OK`

8. **Connect `Geocode address` → `Validate address`.**

9. **Add a Set node** on the true branch named `Extract location data`.
   - Use raw JSON output mode.
   - Output fields:
     - `address` = `{{$json.results[0].formatted_address}}`
     - `lat` = `{{$json.results[0].geometry.location.lat}}`
     - `lng` = `{{$json.results[0].geometry.location.lng}}`
     - `place_id` = `{{$json.results[0].place_id}}`

10. **Connect the true output of `Validate address` → `Extract location data`.**

11. **Add another Set node** named `Format coordinates`.
   - Use raw JSON output mode.
   - Output:
     - `location` = `{{$json.lat + ',' + $json.lng}}`

12. **Connect `Extract location data` → `Format coordinates`.**

13. **Add an HTTP Request node** named `Get street view image`.
   - Method: GET
   - URL: `https://maps.googleapis.com/maps/api/streetview`
   - Query parameters:
     - `size` = `600x400`
     - `location` = `{{$('Prepare API request data').item.json.address}}`
     - `key` = `{{$('Prepare API request data').item.json.api_key}}`
   - In response settings, choose **File** as the response format.

14. **Connect `Format coordinates` → `Get street view image`.**

15. **Add an If node** named `Check street view availability`.
   - Condition:
     - Left value expression checking binary presence, equivalent to: `$binary.data !== undefined`
   - In n8n UI, configure it so the branch is true when the binary property `data` exists.

16. **Connect `Get street view image` → `Check street view availability`.**

17. **Add a Google Drive node** named `Upload image`.
   - Operation: upload file
   - Binary property name: `data`
   - Drive: `My Drive`
   - Folder: choose or create a folder such as `StreetViewImages`
   - Authenticate using **Google Drive OAuth2** credentials

18. **Connect the true branch of `Check street view availability` → `Upload image`.**

19. **Configure Google Drive credentials**.
   - In n8n, create Google Drive OAuth2 credentials
   - Authorize access to the target Google account
   - Ensure the selected folder exists and is writable
   - If you want end users to open the file, verify your Drive sharing settings permit access

20. **Add an HTML node** named `Display page`.
   - Configure it to output an HTML page containing:
     - A “processing” message
     - JavaScript redirect to `{{$json.webViewLink}}`
     - A fallback anchor link to `{{$json.webViewLink}}`

21. **Connect `Upload image` → `Display page`.**

22. **Add a Set node** named `Erro address`.
   - Use raw JSON output mode
   - Output:
     - `error` = `Address not found`
     - `input` = `{{$json.address}}`

23. **Connect the false branch of `Validate address` → `Erro address`.**

24. **Add a Set node** named `Erro image`.
   - Use raw JSON output mode
   - Output:
     - `error` = `Street View not available for this location`

25. **Connect the false branch of `Check street view availability` → `Erro image`.**

26. **Add an HTML node** named `Display error page`.
   - Configure a styled HTML page that:
     - Displays `{{$json.error}}`
     - Optionally displays `{{$json.input ? "Input: " + $json.input : ""}}`
     - Includes a button or link using `javascript:history.back()` for retry

27. **Connect both error nodes to the error page**:
   - `Erro address` → `Display error page`
   - `Erro image` → `Display error page`

28. **Enable required Google APIs** in Google Cloud for the API key used in the Set node:
   - **Geocoding API**
   - **Street View Static API**
   - Ensure billing is enabled if required by your Google project

29. **Restrict and secure the API key**.
   - Recommended:
     - Restrict by API
     - Restrict by IP or referrer where appropriate
   - Avoid leaving the hardcoded placeholder in production

30. **Test the workflow** using a known valid address.
   - Expected success path:
     - Form Trigger
     - Prepare API request data
     - Geocode address
     - Validate address = true
     - Extract location data
     - Format coordinates
     - Get street view image
     - Check street view availability = true
     - Upload image
     - Display page

31. **Test invalid input** such as a fake or malformed address.
   - Expected path:
     - Validate address = false
     - Erro address
     - Display error page

32. **Consider strengthening the Street View validation** if reproducing for production.
   - Better approach:
     - Call Street View metadata first
     - Or use the geocoded coordinates in the `location` parameter
     - Or inspect whether the returned image is a fallback “no imagery” placeholder

33. **Optional improvement when rebuilding**:
   - In `Get street view image`, use `{{$json.location}}` from `Format coordinates` instead of reusing the original address
   - This makes the Street View request consistent with the validated geocoding result

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Validate Address + Generate Street View: This workflow allows users to generate a Street View image from any address using Google Maps APIs. A form collects the address input, which is then validated using the Geocoding API. If valid, the workflow extracts coordinates, retrieves the Street View image, stores it in Google Drive, and returns a visual response page. Error handling is included to manage invalid addresses or unavailable Street View images, ensuring a reliable and user-friendly experience. | Workflow description note |
| How it works: 1. User submits an address through the form  2. The address is validated using Google Geocoding API  3. Latitude and longitude are extracted  4. The Street View image is requested using coordinates  5. The image is validated and uploaded to Google Drive  6. A response page is generated (success or error) | Workflow description note |
| Setup steps: 1. Create a Google Maps API key (Geocoding + Street View enabled)  2. Add your API key in the workflow (or use environment variables)  3. Connect your Google Drive credentials  4. Adjust image size if needed (default: 600x400)  5. Activate the workflow and test using the form | Workflow description note |

## Additional implementation observations
- The workflow contains a minor inconsistency: it computes `lat,lng` in `Format coordinates`, but `Get street view image` actually uses the original address instead.
- The image availability check only verifies whether binary data exists; it does not confirm that Google returned a meaningful Street View image.
- The node names `Erro address` and `Erro image` are misspelled, but they are valid and functional as-is.
- There are no sub-workflows or secondary entry points in this workflow.
- The workflow is marked **active**, and execution order is set to **v1**.