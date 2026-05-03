Uploading image datasets to Qdrant [1/3 anomaly][1/2 KNN]

https://n8nworkflows.xyz/workflows/uploading-image-datasets-to-qdrant--1-3-anomaly--1-2-knn--2654


# Uploading image datasets to Qdrant [1/3 anomaly][1/2 KNN]

### 1. Workflow Overview

This n8n workflow is designed for batch uploading image datasets to a Qdrant vector database, specifically targeting the agricultural crops dataset. It is the first pipeline in two parallel use cases:

- **Anomaly Detection Pipeline** (1/3 of the full anomaly detection system): Uploads cropped image datasets to Qdrant collections for later clustering and anomaly detection.
- **K-Nearest Neighbors (KNN) Classification Pipeline** (1/2 of the KNN classification system): Uploads land-use image datasets similarly for classification tasks.

**Key logical blocks:**

- **1.1 Initialization and Variable Setup:** Define Qdrant cluster parameters and dataset-specific variables.
- **1.2 Qdrant Collection Management:** Check if the target collection exists, create it if not, and set a payload index on crop names.
- **1.3 Dataset Fetching and Filtering:** Retrieve images from Google Cloud Storage and filter out “tomato” images for anomaly detection testing.
- **1.4 Batch Processing Preparation:** Split images into batches, generate UUIDs for Qdrant points, and transform data into API-compatible formats.
- **1.5 Embedding Generation:** Use Voyage AI API to generate vector embeddings for image batches.
- **1.6 Batch Upload to Qdrant:** Upload embeddings and metadata to Qdrant in batches.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Variable Setup

- **Overview:** This block sets essential variables such as Qdrant Cloud URL, collection name, embedding dimension, and batch size used throughout the workflow.
- **Nodes Involved:**  
  - `Qdrant cluster variables`
  - `Sticky Note1`

- **Node Details:**

  - **Qdrant cluster variables**  
    - *Type:* Set  
    - *Role:* Stores connection parameters and constants: Qdrant Cloud URL, collection name (`agricultural-crops`), embedding dimension (1024), and batch size (4).  
    - *Expressions:* Uses static strings and numbers.  
    - *Connections:* Output to `Check Qdrant Collection Existence` node.  
    - *Edge Cases:* Incorrect or outdated URLs or credentials may cause connection failure.

  - **Sticky Note1**  
    - *Type:* Sticky Note  
    - *Content:* Explains the variables being set.  
    - *No inputs or outputs.*

#### 2.2 Qdrant Collection Management

- **Overview:** Checks if the Qdrant collection exists; creates it if missing; then creates a payload index on `crop_name` to optimize metadata queries.
- **Nodes Involved:**  
  - `Check Qdrant Collection Existence`  
  - `If collection exists`  
  - `Create Qdrant Collection`  
  - `Payload index on crop_name`  
  - `Sticky Note2`, `Sticky Note3`, `Sticky Note4`, `Sticky Note`

- **Node Details:**

  - **Check Qdrant Collection Existence**  
    - *Type:* HTTP Request  
    - *Role:* Sends a GET request to Qdrant to verify collection existence.  
    - *URL:* `${qdrantCloudURL}/collections/${collectionName}/exists`  
    - *Auth:* `qdrantApi` credential used.  
    - *Output:* JSON with boolean `result.exists`.  
    - *Error Cases:* Network issues, invalid credentials, or malformed URLs.

  - **If collection exists**  
    - *Type:* If  
    - *Role:* Routes workflow based on collection existence result (true/false).  
    - *Input:* Output of `Check Qdrant Collection Existence`.  
    - *Output:* True → `Google Cloud Storage`, False → `Create Qdrant Collection`.  
    - *Edge Cases:* Expression failures if input JSON missing `result.exists`.

  - **Create Qdrant Collection**  
    - *Type:* HTTP Request  
    - *Role:* Creates collection with named vector `voyage` of size 1024 and cosine similarity metric.  
    - *Method:* PUT  
    - *Payload:* Defines vector size and similarity metric.  
    - *Auth:* `qdrantApi`  
    - *Output:* Proceeds to `Payload index on crop_name` node.  
    - *Edge Cases:* Error if collection name already exists (should be avoided by prior check).

  - **Payload index on crop_name**  
    - *Type:* HTTP Request  
    - *Role:* Creates a keyword index on the `crop_name` payload field in Qdrant to optimize filtering and aggregation queries.  
    - *Method:* PUT  
    - *Payload:* `{ "field_name": "crop_name", "field_schema": "keyword" }`  
    - *Auth:* `qdrantApi`  
    - *Output:* Proceeds to `Google Cloud Storage`.  
    - *Edge Cases:* API errors if index already exists or network issues.

  - **Sticky Notes**  
    - *Sticky Note2:* Explains collection existence check to avoid duplicate creation errors.  
    - *Sticky Note3:* Explains collection creation including named vectors and cosine similarity.  
    - *Sticky Note4:* Explains purpose of payload index for efficient querying.  
    - *Sticky Note:* Notes on payload index importance for later workflows.

#### 2.3 Dataset Fetching and Filtering

- **Overview:** Downloads image metadata from Google Cloud Storage bucket filtered by prefix, constructs public URLs for each image, extracts crop names from folder paths, and filters out tomato images.
- **Nodes Involved:**  
  - `Google Cloud Storage`  
  - `Get fields for Qdrant`  
  - `Filtering out tomato to test anomalies`  
  - `Sticky Note5`, `Sticky Note7`, `Sticky Note8`, `Sticky Note9`, `Sticky Note10`, `Sticky Note11`

- **Node Details:**

  - **Google Cloud Storage**  
    - *Type:* Google Cloud Storage  
    - *Role:* Lists all objects in bucket `n8n-qdrant-demo` with prefix `agricultural-crops`.  
    - *Output:* List of all image objects metadata.  
    - *Credentials:* Google Cloud Storage OAuth2.  
    - *Edge Cases:* Permission errors, empty bucket, or incorrect prefix.

  - **Get fields for Qdrant**  
    - *Type:* Set  
    - *Role:* Constructs public image URLs from bucket and object path, extracts crop name from path segments.  
    - *Expressions:*  
      - `publicLink` = `https://storage.googleapis.com/{{bucket}}/{{filename}}`  
      - `cropName` extracted from path components, lowercased.  
    - *Output:* Adds these fields for downstream nodes.  
    - *Edge Cases:* Unexpected path formats may cause wrong crop name extraction.

  - **Filtering out tomato to test anomalies**  
    - *Type:* Filter  
    - *Role:* Filters out images where `cropName` equals `"tomato"`. This avoids uploading tomato images for anomaly detection testing.  
    - *Output:* Passes only non-tomato images.  
    - *Edge Cases:* Case sensitivity if cropName varies in case.

  - **Sticky Notes:**  
    - *Sticky Note5:* Explains public URL construction and crop name extraction.  
    - *Sticky Note7:* Explains filtering out tomatoes for anomaly detection testing.  
    - *Sticky Note8:* General workflow overview and dataset info.  
    - *Sticky Note9:* Explains JSON structure adaptations for Voyage and Qdrant APIs.  
    - *Sticky Note10:* Workflow context for anomaly detection and KNN pipelines.  
    - *Sticky Note11:* Notes on embedding images with Voyage API and importance of `input_type`.

#### 2.4 Batch Processing Preparation

- **Overview:** Splits filtered images into batches of configurable size, generates UUIDs for each image to serve as Qdrant point IDs, and formats data for embedding and uploading.
- **Nodes Involved:**  
  - `Split in batches, generate uuids for Qdrant points`  
  - `Batches in the API's format`  
  - `Sticky Note6`

- **Node Details:**

  - **Split in batches, generate uuids for Qdrant points**  
    - *Type:* Code (Python)  
    - *Role:*  
      - Receives all filtered image JSON objects.  
      - Splits them into batches of size defined by `batchSize` (4).  
      - Generates UUIDs for each image in every batch.  
    - *Code Summary:*  
      Uses `uuid.uuid4()` to generate a unique string ID per image.  
    - *Output:* Array of objects each with `batch` (images) and `uuids` (corresponding IDs).  
    - *Edge Cases:* Very small datasets (less than batch size), empty inputs.

  - **Batches in the API's format**  
    - *Type:* Set  
    - *Role:* Transforms batch images into JSON structures required by:  
      - Voyage AI embeddings API (field `batchVoyage`) - array of objects with image URLs under `"content"` key.  
      - Qdrant payload (field `batchPayloadQdrant`) - array of metadata objects with `crop_name` and `image_path`.  
      - Passes UUIDs for batch IDs.  
    - *Expressions:* Uses map-functions on batch and uuids.  
    - *Edge Cases:* Mismatched array lengths between UUIDs and batch images.

  - **Sticky Note6:** Explains batch splitting and UUID generation to satisfy Qdrant's requirement for explicit point IDs.

#### 2.5 Embedding Generation

- **Overview:** Sends each batch of images to Voyage AI multimodal embeddings API to get 1024-dimensional vector embeddings.
- **Nodes Involved:**  
  - `Embed crop image`  
  - `Sticky Note11`

- **Node Details:**

  - **Embed crop image**  
    - *Type:* HTTP Request  
    - *Role:* Calls Voyage AI API endpoint `/v1/multimodalembeddings` with POST method to generate embeddings.  
    - *Request Body:*  
      `{ "inputs": <batchVoyage>, "model": "voyage-multimodal-3", "input_type": "document" }`  
    - *Authentication:* HTTP Header with API key in `httpHeaderAuth` credential.  
    - *Output:* The response contains embeddings per image.  
    - *Edge Cases:* API rate limiting, invalid API key, malformed requests, empty batches.

  - **Sticky Note11:** Notes importance of correct `input_type` for embedding request.

#### 2.6 Batch Upload to Qdrant

- **Overview:** Uploads the generated embeddings along with metadata and UUIDs to Qdrant collection in batch mode.
- **Nodes Involved:**  
  - `Batch Upload to Qdrant`

- **Node Details:**

  - **Batch Upload to Qdrant**  
    - *Type:* HTTP Request  
    - *Role:* Sends PUT request to Qdrant API endpoint `/collections/{collectionName}/points` to upsert batch points.  
    - *Request Body:*  
      ```json
      {
        "batch": {
          "ids": <uuids>,
          "vectors": { "voyage": <array of embeddings> },
          "payloads": <array of metadata objects>
        }
      }
      ```  
    - *Auth:* `qdrantApi` credential.  
    - *Edge Cases:* API failure, network issues, mismatched array lengths, quota limits.

---

### 3. Summary Table

| Node Name                          | Node Type              | Functional Role                                       | Input Node(s)                        | Output Node(s)                         | Sticky Note                                                                                              |
|-----------------------------------|------------------------|------------------------------------------------------|------------------------------------|--------------------------------------|--------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’      | Manual Trigger         | Starts the workflow manually                          | None                               | Qdrant cluster variables             |                                                                                                        |
| Qdrant cluster variables           | Set                    | Defines Qdrant URL, collection name, embedding dim, batch size | When clicking ‘Test workflow’       | Check Qdrant Collection Existence    | Sticky Note1: Setting up variables (Cloud URL, collection name, etc.)                                   |
| Check Qdrant Collection Existence  | HTTP Request           | Checks if Qdrant collection exists                    | Qdrant cluster variables           | If collection exists                 | Sticky Note2: Explains collection existence check                                                      |
| If collection exists               | If                     | Routes workflow based on collection existence         | Check Qdrant Collection Existence  | Google Cloud Storage / Create Qdrant Collection |                                                                                                        |
| Create Qdrant Collection           | HTTP Request           | Creates Qdrant collection if missing                   | If collection exists (false path)  | Payload index on crop_name           | Sticky Note3: Explains collection creation details                                                     |
| Payload index on crop_name         | HTTP Request           | Creates index on `crop_name` payload field             | Create Qdrant Collection           | Google Cloud Storage                 | Sticky Note4: Explains importance of creating payload index                                           |
| Google Cloud Storage               | Google Cloud Storage   | Lists images from GCS bucket filtered by prefix        | If collection exists (true path), Payload index on crop_name | Get fields for Qdrant               | Sticky Note8: Overview of dataset fetch and processing                                                 |
| Get fields for Qdrant              | Set                    | Constructs public URLs and extracts crop names         | Google Cloud Storage               | Filtering out tomato to test anomalies | Sticky Note5: Explains public link construction and crop name extraction                              |
| Filtering out tomato to test anomalies | Filter                 | Filters out images labeled as "tomato"                  | Get fields for Qdrant              | Split in batches, generate uuids for Qdrant points | Sticky Note7: Explains tomato filtering                                                              |
| Split in batches, generate uuids for Qdrant points | Code (Python)           | Splits images into batches and generates UUIDs         | Filtering out tomato to test anomalies | Batches in the API's format           | Sticky Note6: Explains batch splitting and UUID generation                                            |
| Batches in the API's format        | Set                    | Formats batch data for Voyage API and Qdrant upload     | Split in batches, generate uuids for Qdrant points | Embed crop image                   | Sticky Note9: Explains JSON adaptation for APIs                                                      |
| Embed crop image                  | HTTP Request           | Calls Voyage AI API to generate embeddings              | Batches in the API's format        | Batch Upload to Qdrant              | Sticky Note11: Notes on embedding images and input_type                                              |
| Batch Upload to Qdrant             | HTTP Request           | Uploads embeddings and metadata points to Qdrant        | Embed crop image                   | None                               |                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named `When clicking ‘Test workflow’` to start the workflow manually.

2. **Add a Set node** named `Qdrant cluster variables`:  
   - Define variables:  
     - `qdrantCloudURL`: Your Qdrant Cloud cluster URL (string).  
     - `collectionName`: `"agricultural-crops"` (string).  
     - `VoyageEmbeddingsDim`: `1024` (number).  
     - `batchSize`: `4` (number).  
   - Connect `When clicking ‘Test workflow’` output to this node.

3. **Add HTTP Request node** `Check Qdrant Collection Existence`:  
   - Method: GET  
   - URL: `{{$json.qdrantCloudURL}}/collections/{{$json.collectionName}}/exists`  
   - Authentication: Use `qdrantApi` credentials configured with your Qdrant Cloud API key.  
   - Connect from `Qdrant cluster variables`.

4. **Add an If node** `If collection exists`:  
   - Condition: `$json.result.exists` equals `true` (boolean).  
   - Connect input from `Check Qdrant Collection Existence`.

5. **Add HTTP Request node** `Create Qdrant Collection`:  
   - Method: PUT  
   - URL: `{{$json.qdrantCloudURL}}/collections/{{$json.collectionName}}`  
   - Body (JSON):  
     ```json
     {
       "vectors": {
         "voyage": {
           "size": {{$json.VoyageEmbeddingsDim}},
           "distance": "Cosine"
         }
       }
     }
     ```  
   - Authentication: `qdrantApi` credentials.  
   - Connect `If collection exists` (False output) to this node.

6. **Add HTTP Request node** `Payload index on crop_name`:  
   - Method: PUT  
   - URL: `{{$json.qdrantCloudURL}}/collections/{{$json.collectionName}}/index`  
   - Body:  
     ```json
     {
       "field_name": "crop_name",
       "field_schema": "keyword"
     }
     ```  
   - Authentication: `qdrantApi`.  
   - Connect from `Create Qdrant Collection`.

7. **Connect the True output of `If collection exists` and the output of `Payload index on crop_name` to a Google Cloud Storage node** named `Google Cloud Storage`:  
   - Operation: List objects (`resource` = object).  
   - Bucket name: Your bucket (e.g., `"n8n-qdrant-demo"`).  
   - Prefix filter: `"agricultural-crops"`.  
   - Credentials: Google Cloud Storage OAuth2 credentials configured.  

8. **Add a Set node** `Get fields for Qdrant`:  
   - Assignments:  
     - `publicLink` = `https://storage.googleapis.com/{{$json.bucket}}/{{$json.selfLink.split('/').splice(-1)}}`  
     - `cropName` = `{{$json.id.split('/').slice(-3, -2)[0].toLowerCase()}}`  
   - Connect from `Google Cloud Storage`.

9. **Add a Filter node** `Filtering out tomato to test anomalies`:  
   - Condition: `cropName` not equals `"tomato"`.  
   - Connect from `Get fields for Qdrant`.

10. **Add a Code node** `Split in batches, generate uuids for Qdrant points`:  
    - Language: Python  
    - Code to:  
      - Collect all incoming JSON items.  
      - Split into batches of size `batchSize` from the variable node.  
      - Generate UUID strings for each image.  
    - Connect from `Filtering out tomato to test anomalies`.

11. **Add a Set node** `Batches in the API's format`:  
    - Assignments:  
      - `batchVoyage`: Map each item in batch to `{ content: [{ type: "image_url", image_url: <publicLink> }] }`  
      - `batchPayloadQdrant`: Map each item to `{ crop_name: <cropName>, image_path: <publicLink> }`  
      - `uuids`: UUIDs array from code node.  
    - Connect from `Split in batches, generate uuids for Qdrant points`.

12. **Add HTTP Request node** `Embed crop image`:  
    - Method: POST  
    - URL: `https://api.voyageai.com/v1/multimodalembeddings`  
    - Body (JSON):  
      ```json
      {
        "inputs": {{$json.batchVoyage}},
        "model": "voyage-multimodal-3",
        "input_type": "document"
      }
      ```  
    - Authentication: HTTP Header Auth with your Voyage AI API key.  
    - Connect from `Batches in the API's format`.

13. **Add HTTP Request node** `Batch Upload to Qdrant`:  
    - Method: PUT  
    - URL: `{{$node["Qdrant cluster variables"].json.qdrantCloudURL}}/collections/{{$node["Qdrant cluster variables"].json.collectionName}}/points`  
    - Body (JSON):  
      ```json
      {
        "batch": {
          "ids": {{$node["Batches in the API's format"].json.uuids}},
          "vectors": {
            "voyage": {{$json.data.map(item => item.embedding)}}
          },
          "payloads": {{$node["Batches in the API's format"].json.batchPayloadQdrant}}
        }
      }
      ```  
    - Authentication: `qdrantApi` credentials.  
    - Connect from `Embed crop image`.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                        | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The workflow uses [Voyage AI multimodal embeddings API](https://docs.voyageai.com/reference/multimodal-embeddings-api) for vectorization.         | Voyage AI API documentation                                                                        |
| Qdrant collection is created with named vectors "voyage" and cosine similarity metric to support vector search.                                   | Qdrant concepts: https://qdrant.tech/documentation/concepts/vectors/                               |
| Payload index on `crop_name` is created to enable efficient filtering and aggregation on this metadata field.                                      | Qdrant indexing docs: https://qdrant.tech/documentation/concepts/indexing/#payload-index           |
| UUIDs are generated per image point to uniquely identify Qdrant points; Qdrant requires user-assigned IDs.                                         | Qdrant points: https://qdrant.tech/documentation/concepts/points/#point-ids                        |
| Dataset images are stored in Google Cloud Storage; public URLs are constructed for use in embedding and upload.                                    | Google Cloud Storage bucket and object layout                                                     |
| Tomato images are deliberately excluded to enable anomaly detection testing with crops dataset.                                                     | Filtering logic in `Filtering out tomato to test anomalies` node                                  |
| The workflow can be adapted to other image datasets (e.g., land-use scenes) by modifying bucket names, prefixes, and collection names accordingly. | See [lands dataset](https://www.kaggle.com/datasets/apollo2506/landuse-scene-classification)      |
| Qdrant free-tier cloud cluster is sufficient for testing this pipeline.                                                                               | Qdrant Cloud quickstart: https://qdrant.tech/documentation/quickstart-cloud/                       |
| Batch size for embeddings and uploads is set to 4 but can be adjusted based on API limits and performance considerations.                          | Variable `batchSize` in `Qdrant cluster variables` node                                           |
| Detailed workflow context and explanations are available in sticky notes embedded within the workflow for easier understanding and maintenance.    | n8n sticky notes within the workflow                                                              |

---

This document provides a detailed, node-level understanding of the batch uploading workflow for image datasets to Qdrant, enabling developers and AI agents to comprehend, reproduce, and modify the process effectively.