Set up cluster centres&thresholds for anomaly detection [2/3 - anomaly]

https://n8nworkflows.xyz/workflows/set-up-cluster-centres-thresholds-for-anomaly-detection--2-3---anomaly--2655


# Set up cluster centres&thresholds for anomaly detection [2/3 - anomaly]

### 1. Workflow Overview

This n8n workflow sets up cluster centers (medoids) and threshold scores for anomaly detection on an agricultural crops image dataset stored in a Qdrant vector database. It is the second pipeline in a three-part system designed for anomaly detection, where:

- The first pipeline uploads and preprocesses the crop images into Qdrant.
- **This second pipeline (current workflow) establishes representative cluster centers and threshold scores using two approaches:**
  - A **Distance Matrix Approach** that uses Qdrant’s distance matrix API and scipy to find medoids by analyzing pairwise distances within clusters.
  - A **Multimodal Embedding Model Approach** that embeds textual descriptions of crops and finds the closest image vector to the description embedding as a cluster representative.
- The third pipeline performs anomaly detection on new images using these prepared medoids and thresholds.

The workflow includes these logical blocks:

- **1.1 Initialization and Variable Setup:** Define global parameters, Qdrant connection details, and medoid selection thresholds.
- **1.2 Crop Cluster Analysis:** Retrieve cluster statistics (counts, unique crop names, largest cluster size).
- **1.3 Distance Matrix Medoid Computation:** For each crop cluster, compute distance matrices, identify medoids using scipy, and mark these points in Qdrant.
- **1.4 Multimodal Embedding Medoid Computation:** Embed hardcoded crop descriptions, find closest image vectors per cluster, and mark them in Qdrant.
- **1.5 Threshold Score Calculation:** For both approaches, find the most dissimilar points to the medoids in each cluster to define threshold scores and store them as metadata in Qdrant.

---

### 2. Block-by-Block Analysis

#### 1.1 Initialization and Variable Setup

**Overview:**  
This block initializes critical workflow parameters such as Qdrant API URLs, collection names, and threshold limits for medoid selection. It acts as the foundation for the entire workflow.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Qdrant cluster variables  
- Medoids Variables  
- Text Medoids Variables  

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Start the workflow manually for testing.  
  - Inputs: None  
  - Outputs: Triggers Qdrant cluster variables node.  
  - Edge Cases: None (manual trigger).

- **Qdrant cluster variables**  
  - Type: Set  
  - Role: Stores Qdrant Cloud URL and collection name as string variables.  
  - Configuration:  
    - `qdrantCloudURL`: Qdrant Cloud base API URL (e.g., https://152bc6e2-832a-415c-a1aa-fb529f8baf8d.eu-central-1-0.aws.cloud.qdrant.io)  
    - `collectionName`: The vector collection name, here "agricultural-crops".  
  - Inputs: From manual trigger  
  - Outputs: Triggers Medoids Variables and Text Medoids Variables.  
  - Edge Cases: Incorrect URL or collection name will cause downstream API errors.

- **Medoids Variables**  
  - Type: Set  
  - Role: Defines the threshold parameter `furthestFromCenter` (number of points to consider from center for threshold).  
  - Configuration: `furthestFromCenter` set to 5 (meaning threshold will be based on the 5th furthest point).  
  - Inputs: From Qdrant cluster variables  
  - Outputs: Triggers Total Points in Collection node.  
  - Edge Cases: Improper value may cause incorrect thresholding.

- **Text Medoids Variables**  
  - Type: Set  
  - Role: Similar threshold parameter for text embedding approach, but set to 1 (only the furthest point).  
  - Configuration: `furthestFromCenter` = 1  
  - Inputs: From Qdrant cluster variables  
  - Outputs: Triggers Textual (visual) crop descriptions node.  
  - Edge Cases: Same as above.

---

#### 1.2 Crop Cluster Analysis

**Overview:**  
This block queries Qdrant to obtain counts of data points and unique crop names, and determines cluster statistics like maximum cluster size.

**Nodes Involved:**  
- Total Points in Collection  
- Crop Counts  
- Info About Crop Clusters  
- Split Out  

**Node Details:**

- **Total Points in Collection**  
  - Type: HTTP Request  
  - Role: Posts to Qdrant API to get exact count of points in the collection.  
  - Configuration:  
    - URL: `{QdrantCloudURL}/collections/{collectionName}/points/count`  
    - Method: POST  
    - Body: `{ "exact": true }`  
    - Authentication: Qdrant API credentials  
  - Inputs: From Medoids Variables  
  - Outputs: Triggers Crop Counts  
  - Edge Cases: Auth errors, network failures, incorrect collection name.

- **Crop Counts**  
  - Type: HTTP Request  
  - Role: Retrieves facet counts of unique values for payload key `crop_name` (number of points per crop).  
  - Configuration:  
    - URL: `{QdrantCloudURL}/collections/{collectionName}/facet`  
    - Method: POST  
    - Body: `{ "key": "crop_name", "limit": totalPoints, "exact": true }` (limit set to total points to include all)  
    - Authentication: Qdrant API  
  - Inputs: From Total Points in Collection  
  - Outputs: Triggers Info About Crop Clusters  
  - Edge Cases: Large data may cause slow response or timeouts.

- **Info About Crop Clusters**  
  - Type: Set  
  - Role: Extracts cluster statistics: number of unique crops (`cropsNumber`), max cluster size (`maxClusterSize`), and list of crop names (`cropNames`).  
  - Configuration uses JavaScript expressions to derive these from facet counts.  
  - Inputs: From Crop Counts  
  - Outputs: Triggers Split Out  
  - Edge Cases: Empty or malformed response from Qdrant.

- **Split Out**  
  - Type: Split Out  
  - Role: Splits the list of crop names into individual items to process each cluster separately.  
  - Configuration: Splits on the `cropNames` array field, outputs field `cropName` for each item.  
  - Inputs: From Info About Crop Clusters  
  - Outputs: Triggers Cluster Distance Matrix and Merge nodes.  
  - Edge Cases: Empty clusters handled gracefully.

---

#### 1.3 Distance Matrix Medoid Computation (Upper Branch)

**Overview:**  
For each crop cluster, this block calls Qdrant’s distance matrix API to get pairwise distances within the cluster, identifies the medoid (most representative point) using scipy sparse matrix computations, and updates Qdrant payloads accordingly.

**Nodes Involved:**  
- Cluster Distance Matrix  
- Scipy Sparse Matrix  
- Set medoid id  
- Get Medoid Vector  
- Prepare for Searching Threshold  
- Searching Score  
- Threshold Score  
- Set medoid threshold score  

**Node Details:**

- **Cluster Distance Matrix**  
  - Type: HTTP Request  
  - Role: Calls Qdrant API `/points/search/matrix/offsets` to get the distance matrix of points within the cluster filtered by `crop_name`.  
  - Configuration:  
    - URL: `{QdrantCloudURL}/collections/{collectionName}/points/search/matrix/offsets`  
    - Method: POST  
    - Body:  
      ```
      {
        "sample": maxClusterSize,
        "limit": maxClusterSize,
        "using": "voyage",
        "filter": {
          "must": {
            "key": "crop_name",
            "match": { "value": cropName }
          }
        }
      }
      ```  
    - Authentication: Qdrant API  
  - Inputs: From Split Out (each cropName)  
  - Outputs: Triggers Scipy Sparse Matrix  
  - Edge Cases: Large clusters may cause timeouts or memory issues.

- **Scipy Sparse Matrix**  
  - Type: Code (Python)  
  - Role: Converts sparse distance matrix data into a `coo_array`, sums similarity scores per point, and identifies the medoid as the point with highest aggregate similarity.  
  - Key code points:  
    - Uses `scipy.sparse.coo_array` to reconstruct matrix.  
    - Calculates sum of each row, finds argmax (medoid index).  
    - Returns medoid point ID.  
  - Inputs: From Cluster Distance Matrix  
  - Outputs: Triggers Set medoid id and Get Medoid Vector  
  - Edge Cases: Python environment required; code errors if matrix data malformed.

- **Set medoid id**  
  - Type: HTTP Request  
  - Role: Marks the identified medoid point in Qdrant by setting payload `is_medoid` to true.  
  - Configuration:  
    - URL: `{QdrantCloudURL}/collections/{collectionName}/points/payload`  
    - Method: POST  
    - Body:  
      ```
      {
        "payload": { "is_medoid": true },
        "points": [medoid_id]
      }
      ```  
    - Authentication: Qdrant API  
  - Inputs: From Scipy Sparse Matrix  
  - Outputs: None (end of this branch)  
  - Edge Cases: API errors if point ID invalid.

- **Get Medoid Vector**  
  - Type: HTTP Request  
  - Role: Fetches the vector and payload of the medoid point identified.  
  - Configuration:  
    - URL: `{QdrantCloudURL}/collections/{collectionName}/points`  
    - Method: POST  
    - Body:  
      ```
      {
        "ids": [medoid_id],
        "with_vector": true,
        "with_payload": true
      }
      ```  
    - Authentication: Qdrant API  
  - Inputs: From Scipy Sparse Matrix  
  - Outputs: Triggers Prepare for Searching Threshold  
  - Edge Cases: API errors if point ID invalid.

- **Prepare for Searching Threshold**  
  - Type: Set  
  - Role: Prepares variables to find the threshold score for the medoid cluster by:  
    - Creating `oppositeOfCenterVector` (medoid vector multiplied by -1 element-wise)  
    - Passing `cropName` and medoid point ID as `centerId`  
  - Inputs: From Get Medoid Vector  
  - Outputs: Triggers Searching Score  
  - Edge Cases: Vector missing or malformed.

- **Searching Score**  
  - Type: HTTP Request  
  - Role: Queries Qdrant to find points most similar to the negated medoid vector (`oppositeOfCenterVector`) within the crop cluster, limited by `furthestFromCenter` parameter. This identifies the most dissimilar points to medoid.  
  - Configuration:  
    - URL: `{QdrantCloudURL}/collections/{collectionName}/points/query`  
    - Method: POST  
    - Body:  
      ```
      {
        "query": oppositeOfCenterVector,
        "using": "voyage",
        "exact": true,
        "filter": {
          "must": [
            { "key": "crop_name", "match": { "value": cropName } }
          ]
        },
        "limit": furthestFromCenter,
        "with_payload": true
      }
      ```  
    - Authentication: Qdrant API  
  - Inputs: From Prepare for Searching Threshold  
  - Outputs: Triggers Threshold Score  
  - Edge Cases: API errors or no points found.

- **Threshold Score**  
  - Type: Set  
  - Role: Extracts the similarity score of the last (furthest) point returned by Searching Score node, negates it, and stores as `thresholdScore`. Also passes `centerId`.  
  - Inputs: From Searching Score  
  - Outputs: Triggers Set medoid threshold score  
  - Edge Cases: Empty result sets.

- **Set medoid threshold score**  
  - Type: HTTP Request  
  - Role: Sets the threshold score as a payload `is_medoid_cluster_threshold` on the medoid point in Qdrant.  
  - Configuration:  
    - URL: `{QdrantCloudURL}/collections/{collectionName}/points/payload`  
    - Method: POST  
    - Body:  
      ```
      {
        "payload": { "is_medoid_cluster_threshold": thresholdScore },
        "points": [centerId]
      }
      ```  
    - Authentication: Qdrant API  
  - Inputs: From Threshold Score  
  - Outputs: None  
  - Edge Cases: API failures.

---

#### 1.4 Multimodal Embedding Medoid Computation (Lower Branch)

**Overview:**  
This branch uses predefined textual descriptions for each crop, embeds them using the Voyage AI multimodal model, searches for the closest image vector in Qdrant to these embeddings, and marks these as medoids.

**Nodes Involved:**  
- Textual (visual) crop descriptions  
- Split Out1  
- Embed text  
- Get Medoid by Text  
- Set text medoid id  
- Prepare for Searching Threshold1  
- Searching Text Medoid Score  
- Threshold Score1  
- Set text medoid threshold score  

**Node Details:**

- **Textual (visual) crop descriptions**  
  - Type: Set  
  - Role: Contains hardcoded JSON array with crop names and textual descriptions of their typical visual appearance.  
  - Configuration: Raw JSON assigned to field `text anchors`.  
  - Inputs: From Text Medoids Variables  
  - Outputs: Triggers Split Out1  
  - Edge Cases: Static data; can be updated or generated dynamically.

- **Split Out1**  
  - Type: Split Out  
  - Role: Splits the array of crop descriptions into individual items to process each crop separately.  
  - Configuration: Splits on `['text anchors']` field.  
  - Inputs: From Textual (visual) crop descriptions  
  - Outputs: Triggers Merge node (merging with crop names from distance matrix branch)  
  - Edge Cases: Empty data handled.

- **Merge**  
  - Type: Merge  
  - Role: Combines crop description items with crop names from the Distance Matrix branch, matching on `cropName`.  
  - Inputs: From Split Out1 (left input) and Split Out (right input)  
  - Outputs: Triggers Embed text  
  - Edge Cases: Mismatched crop names may cause missing data.

- **Embed text**  
  - Type: HTTP Request  
  - Role: Calls Voyage AI multimodal embedding API to embed crop descriptions as vectors.  
  - Configuration:  
    - URL: `https://api.voyageai.com/v1/multimodalembeddings`  
    - Method: POST  
    - Body:  
      ```
      {
        "inputs": [
          {
            "content": [
              { "type": "text", "text": cropDescription }
            ]
          }
        ],
        "model": "voyage-multimodal-3",
        "input_type": "query"
      }
      ```  
    - Authentication: HTTP Header Auth with Voyage API key  
  - Inputs: From Merge  
  - Outputs: Triggers Get Medoid by Text  
  - Edge Cases: API rate limits, invalid auth.

- **Get Medoid by Text**  
  - Type: HTTP Request  
  - Role: Queries Qdrant to find the closest image point vector to the description embedding within the crop cluster.  
  - Configuration:  
    - URL: `{QdrantCloudURL}/collections/{collectionName}/points/query`  
    - Method: POST  
    - Body:  
      ```
      {
        "query": embedding,
        "using": "voyage",
        "exact": true,
        "filter": {
          "must": [
            { "key": "crop_name", "match": { "value": cropName } }
          ]
        },
        "limit": 1,
        "with_payload": true,
        "with_vector": true
      }
      ```  
    - Authentication: Qdrant API  
  - Inputs: From Embed text  
  - Outputs: Triggers Set text medoid id and Prepare for Searching Threshold1  
  - Edge Cases: No matching points found.

- **Set text medoid id**  
  - Type: HTTP Request  
  - Role: Marks the found medoid point with payload `is_text_anchor_medoid` set to true in Qdrant.  
  - Configuration:  
    - URL: `{QdrantCloudURL}/collections/{collectionName}/points/payload`  
    - Method: POST  
    - Body:  
      ```
      {
        "payload": { "is_text_anchor_medoid": true },
        "points": [medoid_id]
      }
      ```  
    - Authentication: Qdrant API  
  - Inputs: From Get Medoid by Text  
  - Outputs: None  
  - Edge Cases: API errors.

- **Prepare for Searching Threshold1**  
  - Type: Set  
  - Role: Similar to Prepare for Searching Threshold but for text medoid; calculates negated vector and stores cropName and centerId.  
  - Inputs: From Get Medoid by Text  
  - Outputs: Triggers Searching Text Medoid Score  
  - Edge Cases: Vector missing.

- **Searching Text Medoid Score**  
  - Type: HTTP Request  
  - Role: Finds most dissimilar points to text medoid center vector by querying Qdrant with negated vector.  
  - Configuration: Uses similar JSON structure as Searching Score but limited by `Text Medoids Variables.furthestFromCenter` (usually 1).  
  - Inputs: From Prepare for Searching Threshold1  
  - Outputs: Triggers Threshold Score1  
  - Edge Cases: No points found.

- **Threshold Score1**  
  - Type: Set  
  - Role: Extracts threshold score from last point’s similarity score and tags with centerId.  
  - Inputs: From Searching Text Medoid Score  
  - Outputs: Triggers Set text medoid threshold score  
  - Edge Cases: Empty results.

- **Set text medoid threshold score**  
  - Type: HTTP Request  
  - Role: Updates Qdrant payload of text medoid point with threshold score `is_text_anchor_medoid_cluster_threshold`.  
  - Inputs: From Threshold Score1  
  - Outputs: None  
  - Edge Cases: API errors.

---

### 3. Summary Table

| Node Name                      | Node Type               | Functional Role                                     | Input Node(s)                 | Output Node(s)                                  | Sticky Note                                                                                                                      |
|-------------------------------|-------------------------|---------------------------------------------------|------------------------------|------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’  | Manual Trigger          | Manual start of workflow                           | None                         | Qdrant cluster variables                        |                                                                                                                                 |
| Qdrant cluster variables       | Set                     | Store Qdrant API URL and collection name          | When clicking ‘Test workflow’ | Medoids Variables, Text Medoids Variables       | Once again, variables for Qdrant: cluster URL and a collection we're working with                                                |
| Medoids Variables              | Set                     | Set threshold parameter for medoid distance matrix| Qdrant cluster variables      | Total Points in Collection                      | Which point in the cluster we're using to draw threshold on: the furthest one from center, or the 2nd, ... Xth furthest one;    |
| Text Medoids Variables         | Set                     | Set threshold parameter for text medoid approach  | Qdrant cluster variables      | Textual (visual) crop descriptions              | Which point in the cluster we're using to draw threshold on: the furthest one from center, or the 2nd, ... Xth furthest one; (2nd approach) |
| Total Points in Collection     | HTTP Request            | Get total count of points in Qdrant collection    | Medoids Variables             | Crop Counts                                     | We need to get the total amount of points in Qdrant collection to use it as a limit in the "Crop Counts" node, so we won't lose info |
| Crop Counts                   | HTTP Request            | Get facet counts of unique crop names              | Total Points in Collection    | Info About Crop Clusters                        | Here we are getting facet counts: unique values behind "crop_name" payload and counts                                           |
| Info About Crop Clusters       | Set                     | Extract number of clusters, max size, crop names  | Crop Counts                   | Split Out                                       | Extract all info needed to call Qdrant distance matrix API per cluster                                                          |
| Split Out                     | Split Out               | Split crop names to process each cluster individually | Info About Crop Clusters      | Cluster Distance Matrix, Merge                   | Splitting out into each unique crop cluster                                                                                    |
| Cluster Distance Matrix        | HTTP Request            | Get pairwise distance matrix for cluster points   | Split Out                    | Scipy Sparse Matrix                             | Calling distance matrix API once per cluster                                                                                   |
| Scipy Sparse Matrix           | Code (Python)           | Compute medoid by summing similarity scores       | Cluster Distance Matrix       | Set medoid id, Get Medoid Vector                | Using scipy coo_array, find representative point (medoid) based on cosine similarity                                           |
| Set medoid id                 | HTTP Request            | Mark medoid point in Qdrant payload                | Scipy Sparse Matrix           | None                                           | Set "is_medoid" payload to true for medoid points                                                                               |
| Get Medoid Vector             | HTTP Request            | Fetch medoid vector and payload                    | Scipy Sparse Matrix           | Prepare for Searching Threshold                 | Fetching vectors of centres by their IDs                                                                                       |
| Prepare for Searching Threshold | Set                   | Prepare negated vector and metadata for threshold search | Get Medoid Vector             | Searching Score                                 | Starting from here, nodes find class threshold scores for anomaly detection                                                    |
| Searching Score              | HTTP Request             | Find most dissimilar points using negated vector  | Prepare for Searching Threshold | Threshold Score                                 | Finding most dissimilar points by querying opposite vector                                                                     |
| Threshold Score              | Set                      | Extract threshold score and centerId               | Searching Score               | Set medoid threshold score                      | Threshold score is similarity score between medoid and furthest point                                                          |
| Set medoid threshold score   | HTTP Request             | Save threshold score to medoid point payload       | Threshold Score               | None                                           | Set "is_medoid_cluster_threshold" payload                                                                                      |
| Textual (visual) crop descriptions | Set               | Hardcoded crop descriptions for embedding          | Text Medoids Variables        | Split Out1                                     | Hardcoded descriptions generated with chatGPT                                                                                 |
| Split Out1                   | Split Out                | Split descriptions into individual crops            | Textual (visual) crop descriptions | Merge                                          |                                                                                                                               |
| Merge                        | Merge                    | Combine crop descriptions with crop names           | Split Out1, Split Out         | Embed text                                     |                                                                                                                               |
| Embed text                   | HTTP Request             | Embed crop descriptions using Voyage multimodal model | Merge                       | Get Medoid by Text                             | Embedding descriptions with Voyage model, input_type = "query"                                                                |
| Get Medoid by Text           | HTTP Request             | Find closest image vector to description embedding  | Embed text                   | Set text medoid id, Prepare for Searching Threshold1 | Find closest image to description embedding                                                                                 |
| Set text medoid id           | HTTP Request             | Mark text medoid in Qdrant payload                   | Get Medoid by Text           | None                                           | Set "is_text_anchor_medoid" payload                                                                                            |
| Prepare for Searching Threshold1 | Set                  | Prepare negated vector and metadata for threshold search (text medoid) | Get Medoid by Text           | Searching Text Medoid Score                    |                                                                                                                               |
| Searching Text Medoid Score  | HTTP Request             | Find most dissimilar points for text medoid          | Prepare for Searching Threshold1 | Threshold Score1                              |                                                                                                                               |
| Threshold Score1             | Set                      | Extract text medoid threshold score                  | Searching Text Medoid Score  | Set text medoid threshold score                |                                                                                                                               |
| Set text medoid threshold score | HTTP Request          | Save threshold score to text medoid point payload    | Threshold Score1             | None                                           | Set "is_text_anchor_medoid_cluster_threshold" payload                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start the workflow manually.

2. **Create Set Node: Qdrant cluster variables**  
   - Fields:  
     - `qdrantCloudURL` (string): Your Qdrant Cloud API URL (e.g., `https://your-cluster-url`)  
     - `collectionName` (string): Your Qdrant collection name, e.g., `agricultural-crops`

3. **Create Set Node: Medoids Variables**  
   - Field:  
     - `furthestFromCenter` (number): 5 (number of furthest points to consider for threshold in distance matrix approach)

4. **Create Set Node: Text Medoids Variables**  
   - Field:  
     - `furthestFromCenter` (number): 1 (for text medoid threshold)

5. **Connect Qdrant cluster variables to Medoids Variables and Text Medoids Variables**

6. **Create HTTP Request Node: Total Points in Collection**  
   - URL: `={{ $json.qdrantCloudURL }}/collections/{{$json.collectionName}}/points/count`  
   - Method: POST  
   - Body: `{ "exact": true }` (JSON)  
   - Auth: Use Qdrant API credentials  
   - Connect Medoids Variables → Total Points in Collection

7. **Create HTTP Request Node: Crop Counts**  
   - URL: `={{ $json.qdrantCloudURL }}/collections/{{$json.collectionName}}/facet`  
   - Method: POST  
   - Body:  
     ```json
     {
       "key": "crop_name",
       "limit": {{$json.result.count}},
       "exact": true
     }
     ```  
   - Auth: Qdrant API  
   - Connect Total Points in Collection → Crop Counts

8. **Create Set Node: Info About Crop Clusters**  
   - Assign fields using expressions to extract:  
     - `cropsNumber`: length of facet hits  
     - `maxClusterSize`: max count among hits  
     - `cropNames`: array of crop names  
   - Connect Crop Counts → Info About Crop Clusters

9. **Create Split Out Node: Split Out**  
   - Field to split: `cropNames`  
   - Destination field: `cropName`  
   - Connect Info About Crop Clusters → Split Out

10. **Create HTTP Request Node: Cluster Distance Matrix**  
    - URL: `={{ $json.qdrantCloudURL }}/collections/{{$json.collectionName}}/points/search/matrix/offsets`  
    - Method: POST  
    - Body:  
      ```json
      {
        "sample": {{$json.maxClusterSize}},
        "limit": {{$json.maxClusterSize}},
        "using": "voyage",
        "filter": {
          "must": {
            "key": "crop_name",
            "match": { "value": {{$json.cropName}} }
          }
        }
      }
      ```  
    - Auth: Qdrant API  
    - Connect Split Out → Cluster Distance Matrix

11. **Create Code Node (Python): Scipy Sparse Matrix**  
    - Language: Python  
    - Code: Use scipy.sparse.coo_array to build matrix from scores and offsets, sum rows, find argmax for medoid, output medoid id.  
    - Connect Cluster Distance Matrix → Scipy Sparse Matrix

12. **Create HTTP Request Node: Set medoid id**  
    - URL: `={{ $json.qdrantCloudURL }}/collections/{{$json.collectionName}}/points/payload`  
    - Method: POST  
    - Body:  
      ```json
      {
        "payload": {"is_medoid": true},
        "points": [{{$json.medoid_id}}]
      }
      ```  
    - Auth: Qdrant API  
    - Connect Scipy Sparse Matrix → Set medoid id

13. **Create HTTP Request Node: Get Medoid Vector**  
    - URL: `={{ $json.qdrantCloudURL }}/collections/{{$json.collectionName}}/points`  
    - Method: POST  
    - Body:  
      ```json
      {
        "ids": [{{$json.medoid_id}}],
        "with_vector": true,
        "with_payload": true
      }
      ```  
    - Auth: Qdrant API  
    - Connect Scipy Sparse Matrix → Get Medoid Vector

14. **Create Set Node: Prepare for Searching Threshold**  
    - Assign:  
      - `oppositeOfCenterVector`: negated medoid vector (map each element * -1)  
      - `cropName`: medoid’s `crop_name` payload  
      - `centerId`: medoid point id  
    - Connect Get Medoid Vector → Prepare for Searching Threshold

15. **Create HTTP Request Node: Searching Score**  
    - URL: `={{ $json.qdrantCloudURL }}/collections/{{$json.collectionName}}/points/query`  
    - Method: POST  
    - Body:  
      ```json
      {
        "query": {{$json.oppositeOfCenterVector}},
        "using": "voyage",
        "exact": true,
        "filter": {"must": [{"key": "crop_name", "match": {"value": {{$json.cropName}}}}]},
        "limit": {{$('Medoids Variables').first().json.furthestFromCenter}},
        "with_payload": true
      }
      ```  
    - Auth: Qdrant API  
    - Connect Prepare for Searching Threshold → Searching Score

16. **Create Set Node: Threshold Score**  
    - Assign:  
      - `thresholdScore`: negative of last point’s score from Searching Score  
      - `centerId`: from Prepare for Searching Threshold  
    - Connect Searching Score → Threshold Score

17. **Create HTTP Request Node: Set medoid threshold score**  
    - URL: `={{ $json.qdrantCloudURL }}/collections/{{$json.collectionName}}/points/payload`  
    - Method: POST  
    - Body:  
      ```json
      {
        "payload": {"is_medoid_cluster_threshold": {{$json.thresholdScore}}},
        "points": [{{$json.centerId}}]
      }
      ```  
    - Auth: Qdrant API  
    - Connect Threshold Score → Set medoid threshold score

18. **Create Set Node: Textual (visual) crop descriptions**  
    - Assign raw JSON array of crop names and descriptions (hardcoded or generated) to `text anchors`.  
    - Connect Text Medoids Variables → Textual (visual) crop descriptions

19. **Create Split Out Node: Split Out1**  
    - Split on `['text anchors']`  
    - Connect Textual (visual) crop descriptions → Split Out1

20. **Create Merge Node**  
    - Mode: Combine  
    - Fields to match: `cropName`  
    - Connect Split Out1 → Merge (left)  
    - Connect Split Out → Merge (right)

21. **Create HTTP Request Node: Embed text**  
    - URL: `https://api.voyageai.com/v1/multimodalembeddings`  
    - Method: POST  
    - Body:  
      ```json
      {
        "inputs": [{"content": [{"type": "text", "text": {{$json.cropDescription}}}]}],
        "model": "voyage-multimodal-3",
        "input_type": "query"
      }
      ```  
    - Auth: HTTP Header Auth with Voyage AI API key  
    - Connect Merge → Embed text

22. **Create HTTP Request Node: Get Medoid by Text**  
    - URL: `={{ $json.qdrantCloudURL }}/collections/{{$json.collectionName}}/points/query`  
    - Method: POST  
    - Body:  
      ```json
      {
        "query": {{$json.data[0].embedding}},
        "using": "voyage",
        "exact": true,
        "filter": {"must": [{"key": "crop_name", "match": {"value": {{$json.cropName}}}}]},
        "limit": 1,
        "with_payload": true,
        "with_vector": true
      }
      ```  
    - Auth: Qdrant API  
    - Connect Embed text → Get Medoid by Text

23. **Create HTTP Request Node: Set text medoid id**  
    - URL: `={{ $json.qdrantCloudURL }}/collections/{{$json.collectionName}}/points/payload`  
    - Method: POST  
    - Body:  
      ```json
      {
        "payload": {"is_text_anchor_medoid": true},
        "points": [{{$json.result.points[0].id}}]
      }
      ```  
    - Auth: Qdrant API  
    - Connect Get Medoid by Text → Set text medoid id

24. **Create Set Node: Prepare for Searching Threshold1**  
    - Assign:  
      - `oppositeOfCenterVector`: negated medoid vector (each coordinate * -1)  
      - `cropName`: medoid’s crop_name  
      - `centerId`: medoid point ID  
    - Connect Get Medoid by Text → Prepare for Searching Threshold1

25. **Create HTTP Request Node: Searching Text Medoid Score**  
    - URL: `={{ $json.qdrantCloudURL }}/collections/{{$json.collectionName}}/points/query`  
    - Method: POST  
    - Body:  
      ```json
      {
        "query": {{$json.oppositeOfCenterVector}},
        "using": "voyage",
        "exact": true,
        "filter": {"must": [{"key": "crop_name", "match": {"value": {{$json.cropName}}}}]},
        "limit": {{$('Text Medoids Variables').first().json.furthestFromCenter}},
        "with_payload": true
      }
      ```  
    - Auth: Qdrant API  
    - Connect Prepare for Searching Threshold1 → Searching Text Medoid Score

26. **Create Set Node: Threshold Score1**  
    - Assign:  
      - `thresholdScore`: negative of last point’s score from Searching Text Medoid Score  
      - `centerId`: from Prepare for Searching Threshold1  
    - Connect Searching Text Medoid Score → Threshold Score1

27. **Create HTTP Request Node: Set text medoid threshold score**  
    - URL: `={{ $json.qdrantCloudURL }}/collections/{{$json.collectionName}}/points/payload`  
    - Method: POST  
    - Body:  
      ```json
      {
        "payload": {"is_text_anchor_medoid_cluster_threshold": {{$json.thresholdScore}}},
        "points": [{{$json.centerId}}]
      }
      ```  
    - Auth: Qdrant API  
    - Connect Threshold Score1 → Set text medoid threshold score

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow is part two of a three-pipeline system for anomaly detection using Qdrant and Voyage AI.                 | Workflow description                                                                                 |
| Data source: Agricultural crops image classification dataset from Kaggle. Upload to Google Storage bucket first.       | Dataset link: https://www.kaggle.com/datasets/mdwaquarazam/agricultural-crops-image-classification   |
| Qdrant Cloud free tier cluster can be used; requires setting up Qdrant API credentials in n8n.                         | Qdrant Cloud quickstart: https://qdrant.tech/documentation/quickstart-cloud/                         |
| Voyage AI API key needed for embedding crop description texts.                                                        | Voyage AI: https://www.voyageai.com/                                                                |
| Distance matrix approach scales poorly with very large clusters; recommended for labeled data with clusters of hundreds.| Qdrant distance matrix API docs: https://qdrant.tech/documentation/concepts/explore/#distance-matrix |
| Payload keys used in Qdrant: `is_medoid`, `is_medoid_cluster_threshold`, `is_text_anchor_medoid`, `is_text_anchor_medoid_cluster_threshold` | Payload documentation: https://qdrant.tech/documentation/concepts/payload/                          |
| The cosine similarity metric is used for vector comparisons; negating vectors is a valid approach to find dissimilar points.| Math insight: https://mathinsight.org/image/vector_opposite                                         |
| Crop visual descriptions were generated with ChatGPT but can be dynamically generated in n8n for other datasets.      |                                                                                                    |

---

This documentation enables full understanding, modification, and recreation of the workflow for setting up medoids and thresholds for anomaly detection in Qdrant. It covers both algorithmic logic and integration details with external APIs.