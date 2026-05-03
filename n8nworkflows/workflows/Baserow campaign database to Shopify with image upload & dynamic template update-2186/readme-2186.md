Baserow campaign database to Shopify with image upload & dynamic template update

https://n8nworkflows.xyz/workflows/baserow-campaign-database-to-shopify-with-image-upload---dynamic-template-update-2186


# Baserow campaign database to Shopify with image upload & dynamic template update

---
### 1. Workflow Overview

This workflow automates the process of synchronizing marketing campaign data stored in Baserow with a Shopify store. Specifically, it fetches updated campaign images and descriptions from Baserow, uploads images to Shopify’s media library via GraphQL API, dynamically injects campaign data into a Shopify Liquid template file, and finally uploads this updated template to the Shopify theme.

The workflow is designed for marketing teams and e-commerce managers who want to streamline campaign publishing by centralizing asset management in Baserow and automatically reflecting changes on Shopify without manual intervention.

**Logical blocks:**

- **1.1 Input Reception:** Receives campaign update events from Baserow via a webhook.
- **1.2 Data Preparation:** Sets static Shopify configuration parameters and checks if the campaign update meets criteria for processing.
- **1.3 Image Upload:** Uploads the campaign image from Baserow to Shopify’s media library using GraphQL.
- **1.4 Template Update & Upload:** Dynamically generates a Shopify Liquid template snippet with campaign data and uploads it to the Shopify store via REST API.
- **1.5 No-Operation Handling:** Placeholder node for cases when conditions are not met, effectively stopping further processing.

---
### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives POST webhook requests from Baserow whenever rows in the campaign database are updated. It acts as the trigger for the workflow.

- **Nodes Involved:**  
  - Call from Baserow

- **Node Details:**  
  - **Call from Baserow**  
    - Type: Webhook  
    - Role: Entry point; listens for POST requests on a specific webhook path.  
    - Configuration:  
      - HTTP Method: POST  
      - Webhook path: "3041fdd6-4cb5-4286-9034-1337dddc3f45" (unique identifier)  
    - Input: HTTP POST payload from Baserow containing campaign row data, including images and metadata.  
    - Output: JSON data representing the campaign update.  
    - Edge Cases:  
      - Invalid or malformed webhook payload could cause failures downstream.  
      - Network issues or webhook misconfigurations could prevent triggering.  
    - Version: 1.1  

#### 2.2 Data Preparation

- **Overview:**  
  This block sets Shopify-specific configuration parameters and evaluates if the incoming campaign data should be processed further based on update recency, campaign activation status, and presence of images.

- **Nodes Involved:**  
  - Set values here!  
  - Check

- **Node Details:**  
  - **Set values here!**  
    - Type: Set  
    - Role: Defines static configuration values needed for Shopify API calls, such as Shopify subdomain, theme ID, filename, and content template.  
    - Configuration:  
      - Shopify Subdomain (e.g., "n8n-mautic-demo")  
      - Theme ID (numeric string, e.g., "125514514534")  
      - Filename (e.g., "campaign.liquid")  
      - Content (Liquid snippet string with placeholder "IMAGE")  
    - Input: Output from webhook node.  
    - Output: JSON containing the above parameters accessible by later nodes.  
    - Edge Cases:  
      - If values are incorrect or missing, Shopify API calls will fail.  
    - Version: 3.2  

  - **Check**  
    - Type: If  
    - Role: Conditional gate that ensures processing only continues if:  
      1. The "Last modified" timestamp of the campaign data is more recent than the previous version by more than 0.1 minutes (6 seconds).  
      2. The campaign is marked as Active (boolean).  
      3. The campaign has at least one associated image.  
    - Configuration:  
      - Condition 1: DateTime difference calculation using JavaScript expression and Luxon library.  
      - Condition 2: Boolean check on "Active" field.  
      - Condition 3: Checks if the "Campaign Image" array is not empty.  
    - Input: Output from "Set values here!" node.  
    - Output:  
      - True branch: proceeds to upload image.  
      - False branch: goes to no-op node.  
    - Edge Cases:  
      - Date parsing errors if timestamps are malformed.  
      - Missing fields could cause expression evaluation failures.  
    - Version: 2  

#### 2.3 Image Upload

- **Overview:**  
  This block uploads the campaign image from Baserow to the Shopify media library using a GraphQL mutation.

- **Nodes Involved:**  
  - Upload Image

- **Node Details:**  
  - **Upload Image**  
    - Type: GraphQL  
    - Role: Executes a mutation `fileCreate` to upload image metadata and source URL to Shopify.  
    - Configuration:  
      - Endpoint URL dynamically constructed from Shopify subdomain.  
      - GraphQL mutation specifies file attributes: alt text (campaign name), contentType as IMAGE, filename from campaign image visible_name, and originalSource URL from Baserow.  
      - Request format: JSON  
      - Authentication: Uses Header Auth credential with "X-Shopify-Access-Token".  
    - Input: JSON data from "Check" node containing campaign image and name.  
    - Output: Shopify response including uploaded file ID.  
    - Edge Cases:  
      - Auth errors if token is invalid or expired.  
      - Network issues or Shopify API rate limiting.  
      - Invalid image URLs or unsupported formats.  
    - Version: 1  

#### 2.4 Template Update & Upload

- **Overview:**  
  After successful image upload, this block dynamically injects the campaign image and content data into a Liquid template and uploads this file as a theme asset to Shopify using REST API.

- **Nodes Involved:**  
  - Save campaign.liquid

- **Node Details:**  
  - **Save campaign.liquid**  
    - Type: HTTP Request  
    - Role: Performs a PUT request to Shopify REST API to upload/update a theme asset file (Liquid snippet).  
    - Configuration:  
      - URL dynamically composed from:  
        - Shopify subdomain  
        - Shopify API version (2024-01)  
        - Theme ID  
        - Asset endpoint `/assets.json`  
      - HTTP method: PUT  
      - Body: JSON containing asset key (filename under "snippets/") and the template content with the placeholder "IMAGE" replaced by the campaign image visible name. Escaping is applied to handle special characters.  
      - Authentication: Generic HTTP Header Auth with header "X-Shopify-Access-Token".  
    - Input: JSON from "Upload Image" node and static values from "Set values here!" node.  
    - Output: Shopify API response confirming asset update.  
    - Edge Cases:  
      - Auth errors if token invalid.  
      - API rate limiting or network errors.  
      - Incorrect theme ID or filename causing upload failure.  
    - Version: 4.1  

#### 2.5 No-Operation Handling

- **Overview:**  
  This node acts as a sink for the workflow execution when the campaign update does not meet processing criteria. It effectively stops the flow without errors.

- **Nodes Involved:**  
  - No Operation, do nothing

- **Node Details:**  
  - **No Operation, do nothing**  
    - Type: NoOp  
    - Role: Terminates workflow branch gracefully, doing no action.  
    - Input: False branch output from "Check" node.  
    - Output: None  
    - Edge Cases: None, safe fallback.  
    - Version: 1  

---
### 3. Summary Table

| Node Name               | Node Type       | Functional Role                                   | Input Node(s)         | Output Node(s)            | Sticky Note                                                                                                                             |
|-------------------------|-----------------|-------------------------------------------------|-----------------------|---------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Call from Baserow       | Webhook         | Receives campaign update events from Baserow    | —                     | Set values here!           |                                                                                                                                         |
| Set values here!         | Set             | Defines Shopify API parameters and template data | Call from Baserow      | Check                     | ## Set values \nPlease edit this node and change the values for your own setup.                                                         |
| Check                   | If              | Conditional check for update recency, active flag, and image presence | Set values here!       | Upload Image, No Operation |                                                                                                                                         |
| Upload Image            | GraphQL         | Uploads campaign image to Shopify media library  | Check (true branch)    | Save campaign.liquid       | ## Shopify API\n\nThis workflow uses GraphQL calls to the Shopify Admin API. See docs:\n[Shopify GraphQL API docs](https://shopify.dev/docs/api/admin-graphql)\nGraphiQL tool: [GraphiQL Admin API](https://shopify.dev/docs/apps/tools/graphiql-admin-api) |
| Save campaign.liquid    | HTTP Request    | Uploads updated Liquid template snippet to Shopify theme | Upload Image           | —                         | ## Shopify \nThe n8n Shopify node cannot upload images or theme assets so we need to make custom calls to the GraphQL and REST Api       |
| No Operation, do nothing | NoOp            | No operation; terminates workflow branch          | Check (false branch)   | —                         |                                                                                                                                         |

---
### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node "Call from Baserow"**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Use a unique identifier (e.g., "3041fdd6-4cb5-4286-9034-1337dddc3f45")  
   - Purpose: To receive campaign update events from Baserow.

2. **Create Set Node "Set values here!"**  
   - Type: Set  
   - Add fields with these names and string values:  
     - Shopify Subdomain: your Shopify shop subdomain (e.g., "yourshopname")  
     - Theme ID: the numeric ID of your Shopify theme (string format)  
     - Filename: "campaign.liquid" (or your desired snippet filename)  
     - Content: Liquid snippet template string, e.g., `<img src="{{ 'IMAGE' | file_img_url: 'grande'}}">`  
   - Connect output of "Call from Baserow" to this node.

3. **Create If Node "Check"**  
   - Type: If  
   - Add three conditions combined with AND:  
     1. Numeric condition:  
        - Left expression:  
          ```js
          DateTime.fromISO($json["body"]["items"][0]["Last modified"])
            .diff(DateTime.fromISO($json["body"]["old_items"][0]["Last modified"]), 'minutes')
            .toObject()["minutes"]
          ```  
        - Operator: greater than  
        - Right value: 0.1  
     2. Boolean condition:  
        - Left expression: `$json.body.items[0].Active`  
        - Operator: is true  
     3. Array not empty condition:  
        - Left expression: `$json["body"]["items"][0]["Campaign Image"]`  
        - Operator: not empty array  
   - Connect output of "Set values here!" to "Check".

4. **Create No Operation Node "No Operation, do nothing"**  
   - Type: NoOp  
   - Connect false output of "Check" to this node.

5. **Create GraphQL Node "Upload Image"**  
   - Type: GraphQL  
   - Endpoint URL: `https://{{ $('Set values here!').params["fields"]["values"][0]["stringValue"] }}.myshopify.com/admin/api/2024-01/graphql.json`  
   - Query:  
     ```
     mutation fileCreate($files: [FileCreateInput!]!) {
       fileCreate(files: $files) {
         files {
           id
         }
       }
     }
     ```  
   - Variables (JSON format):  
     ```json
     {
       "files": {
         "alt": "{{ $json.body.items[0].Name }}",
         "contentType": "IMAGE",
         "filename": "{{ $json.body.items[0]['Campaign Image'][0].visible_name }}",
         "originalSource": "{{ $json.body.items[0]['Campaign Image'][0].url }}"
       }
     }
     ```  
   - Authentication: Use HTTP Header Auth credentials with header "X-Shopify-Access-Token" set to your Shopify access token.  
   - Connect true output of "Check" to this node.

6. **Create HTTP Request Node "Save campaign.liquid"**  
   - Type: HTTP Request  
   - Method: PUT  
   - URL:  
     ```
     https://{{ $('Set values here!').params["fields"]["values"][0]["stringValue"] }}.myshopify.com/admin/api/2024-01/themes/{{ $('Set values here!').params["fields"]["values"][1]["stringValue"] }}/assets.json
     ```  
   - Body Content-Type: JSON  
   - Body:  
     ```json
     {
       "asset": {
         "key": "snippets/{{ $('Set values here!').params["fields"]["values"][2]["stringValue"] }}",
         "value": "{{ $('Set values here!').params["fields"]["values"][3]["stringValue"].replace('IMAGE', $('Check').item.json.body.items[0]['Campaign Image'][0].visible_name).replace(/\\/g, '\\\\').replace(/\"/g, '\\\"').replace(/\n/g, '\\n') }}"
       }
     }
     ```  
   - Authentication: Use Generic HTTP Header Auth with header "X-Shopify-Access-Token" and your access token value.  
   - Connect output of "Upload Image" node to this node.

7. **Verify Credentials:**  
   - Create Shopify Access Token API credentials in n8n with your Shopify app access token.  
   - Create HTTP Header Auth credentials named "Header Auth Shopify" with header name "X-Shopify-Access-Token" and token value.  
   - Assign these credentials appropriately to GraphQL and HTTP Request nodes.

8. **Test Workflow:**  
   - Send a test POST request to the webhook URL with a sample Baserow campaign update payload.  
   - Confirm that the image uploads to Shopify and the template snippet is updated in the theme assets.

---
### 5. General Notes & Resources

| Note Content                                                                                                                                     | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow uses GraphQL calls to the Shopify Admin API. For details on the API, see the official docs: [Shopify GraphQL API docs](https://shopify.dev/docs/api/admin-graphql) | Sticky Note on GraphQL usage                                                                        |
| To build and test GraphQL queries easily, use the Shopify GraphiQL app: [GraphiQL Admin API](https://shopify.dev/docs/apps/tools/graphiql-admin-api) | Sticky Note on GraphQL tooling                                                                      |
| The n8n Shopify node does not support uploading images or theme assets. Hence, this workflow uses custom HTTP and GraphQL calls to Shopify APIs. | Sticky Note on Shopify API limitations                                                             |
| Baserow is used here as a centralized campaign asset manager with webhook support to trigger workflow on data updates.                          | Main integration context                                                                            |
| For a detailed video walkthrough of this workflow setup and operation, see: [https://youtu.be/Ky-dYlljGiY](https://youtu.be/Ky-dYlljGiY)           | Provided by the original workflow author                                                           |

---

This completes the comprehensive reference documentation of the "Baserow campaign database to Shopify with image upload & dynamic template update" n8n workflow.