KNN (images) classifier tool [2/2 KNN]

https://n8nworkflows.xyz/workflows/knn--images--classifier-tool--2-2-knn--2657


# KNN (images) classifier tool [2/2 KNN]

### 1. Workflow Overview

This workflow implements a K-Nearest Neighbors (KNN) classifier tool designed to classify images based on their similarity to a pre-labeled dataset stored in a Qdrant vector database. The workflow specifically targets satellite imagery of land use types, taking as input an image URL and outputting the predicted land class.

The workflow's main logical blocks are:

- **1.1 Input Reception and Preparation:** Receives the image URL as input and prepares it for embedding.
- **1.2 Image Embedding:** Sends the image URL to the Voyage AI Multimodal Embeddings API to obtain a vector representation of the image.
- **1.3 Query Qdrant Vector Database:** Uses the embedding to query Qdrant for the nearest neighbors in the dataset, retrieving their classes.
- **1.4 Majority Voting & Tie Resolution Loop:** Performs majority voting on the retrieved neighbors' classes. If a tie occurs, it increases the number of neighbors queried and repeats until the tie is resolved or a maximum limit is reached.
- **1.5 Return Classification Result:** Outputs the final predicted class to the calling workflow.

This workflow is the second part of a two-part system where the first workflow uploads and indexes the dataset into Qdrant.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Preparation

**Overview:**  
This block receives an image URL from an external trigger and sets it into the workflow's internal data structure for subsequent embedding.

**Nodes Involved:**  
- Execute Workflow Trigger  
- Image Test URL

**Node Details:**

- **Execute Workflow Trigger**  
  - Type: Trigger node  
  - Role: Entry point for the workflow; receives input JSON containing `imageURL`  
  - Config: No special parameters; expects input with `query.imageURL`  
  - Input: External trigger invocation  
  - Output: Passes input JSON downstream  
  - Edge Cases: Missing or malformed `imageURL` could cause failure downstream.

- **Image Test URL**  
  - Type: Set node  
  - Role: Extracts and assigns the `imageURL` from the input query to a top-level JSON property `imageURL`  
  - Config: Assigns `imageURL` = `{{$json.query.imageURL}}`  
  - Input: From Execute Workflow Trigger  
  - Output: JSON with `imageURL` property for the embedding node  
  - Edge Cases: If input lacks `query.imageURL`, the value will be undefined, causing issues in the embedding request.

---

#### 2.2 Image Embedding

**Overview:**  
This block sends the input image URL to the Voyage AI Multimodal Embedding API to convert the image into a vector embedding suitable for similarity search.

**Nodes Involved:**  
- Embed image  
- Qdrant variables + embedding + KNN neighbours

**Node Details:**

- **Embed image**  
  - Type: HTTP Request node  
  - Role: Calls Voyage AI API for image embedding  
  - Config:  
    - Method: POST  
    - URL: `https://api.voyageai.com/v1/multimodalembeddings`  
    - JSON Body includes the image URL extracted from `$json.imageURL` and specifies model `"voyage-multimodal-3"`  
    - Authentication: HTTP Header with Voyage API key credential  
  - Input: Image URL JSON from previous node  
  - Output: Embedding vector in `data[0].embedding`  
  - Edge Cases: API key invalid, request timeout, or malformed image URL may cause failure or empty embeddings.

- **Qdrant variables + embedding + KNN neighbours**  
  - Type: Set node  
  - Role: Defines static variables for Qdrant connection and packaging the embedding and KNN parameters for querying  
  - Config:  
    - Sets `ImageEmbedding` from API response embedding vector  
    - Sets `qdrantCloudURL` to the Qdrant Cloud endpoint URL (hardcoded)  
    - Sets `collectionName` to `"land-use"` (Qdrant collection name)  
    - Sets initial `limitKNN` to 10 (number of neighbors to query)  
  - Input: Embedding output from previous node  
  - Output: JSON with all params for Qdrant query  
  - Edge Cases: Hardcoded Qdrant URL and collection imply need for user customization; mismatches cause query failures.

---

#### 2.3 Query Qdrant Vector Database

**Overview:**  
Using the embedding and parameters, this block queries Qdrant for the nearest neighbors, retrieving their class payloads.

**Nodes Involved:**  
- Query Qdrant  
- Propagate loop variables

**Node Details:**

- **Query Qdrant**  
  - Type: HTTP Request node  
  - Role: Queries Qdrant collection with embedding vector for nearest neighbors  
  - Config:  
    - POST to `${qdrantCloudURL}/collections/${collectionName}/points/query`  
    - JSON Body includes:  
      - `query`: embedding vector  
      - `using`: `"voyage"` (distance metric or search method)  
      - `limit`: number of neighbors (`limitKNN`)  
      - `with_payload`: true (to get labels along with points)  
    - Authentication: Qdrant API credentials  
  - Input: JSON with embedding and parameters from previous node  
  - Output: Query result including nearest points and their payloads  
  - Edge Cases: Authentication errors, network issues, invalid embedding vector or collection name.

- **Propagate loop variables**  
  - Type: Set node  
  - Role: Updates internal workflow variables after each query iteration for controlling the loop  
  - Config:  
    - Sets `limitKNN` to the number of points returned (`$json.result.points.length`)  
    - Propagates full `result` object for downstream nodes  
  - Input: Qdrant query results  
  - Output: JSON including current `limitKNN` and query result for majority voting  
  - Edge Cases: Empty results or unexpected response shape.

---

#### 2.4 Majority Voting & Tie Resolution Loop

**Overview:**  
This block analyzes the classes of the nearest neighbors, performs majority voting, and handles tie cases by increasing the number of neighbors queried iteratively.

**Nodes Involved:**  
- Majority Vote  
- Check tie  
- Increase limitKNN

**Node Details:**

- **Majority Vote**  
  - Type: Code node (Python)  
  - Role: Counts occurrences of classes among neighbors and returns the two most common with counts  
  - Config: Uses `collections.Counter` to tally `landscape_name` from payloads in points  
  - Input: JSON with Qdrant query result points  
  - Output: JSON list of tuples `[('class_name', count), ...]` containing two most common classes  
  - Edge Cases: No points returned, missing payload keys, code exceptions.

- **Check tie**  
  - Type: If node  
  - Role: Determines if there is a tie between the top two classes and whether to continue looping  
  - Config:  
    - Conditions:  
      - Result length > 1  
      - Top two classes have equal count  
      - `limitKNN` ≤ 100 (max neighbor count)  
    - If true: loop continues (Increase limitKNN)  
    - If false: loop ends (Return class)  
  - Input: Majority Vote output and loop variable state  
  - Output: Two branches — loop or exit  
  - Edge Cases: Edge tie conditions, max limit reached.

- **Increase limitKNN**  
  - Type: Set node  
  - Role: Increases `limitKNN` by 5 for the next query iteration and propagates variables unchanged  
  - Config:  
    - `limitKNN` = previous `limitKNN` + 5  
    - Carries forward `ImageEmbedding`, `qdrantCloudURL`, and `collectionName` unchanged  
  - Input: From Check tie (true branch)  
  - Output: Updated parameters for next Qdrant query  
  - Edge Cases: Exceeding max limit handled by Check tie node.

---

#### 2.5 Return Classification Result

**Overview:**  
This block extracts the final predicted class from the majority vote result and outputs it as the workflow result.

**Nodes Involved:**  
- Return class

**Node Details:**

- **Return class**  
  - Type: Set node  
  - Role: Extracts the winning class name from the majority vote result and sets it as output property `class`  
  - Config:  
    - `class` = first element's class name: `{{$json.result[0][0]}}`  
  - Input: From Check tie (false branch)  
  - Output: Final classification JSON with property `class` for calling workflows  
  - Edge Cases: If result is empty or malformed, output may be invalid.

---

### 3. Summary Table

| Node Name                           | Node Type              | Functional Role                                  | Input Node(s)                         | Output Node(s)                    | Sticky Note                                                                                                          |
|-----------------------------------|------------------------|-------------------------------------------------|-------------------------------------|----------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Execute Workflow Trigger           | Execute Workflow Trigger | Receives external trigger with image URL        | None                                | Image Test URL                   | See Sticky Note2: Overview of KNN classification workflow and process details                                        |
| Image Test URL                    | Set                    | Extracts and sets image URL for embedding        | Execute Workflow Trigger             | Embed image                     | See Sticky Note4: Embedding input image with Voyage model                                                            |
| Embed image                      | HTTP Request           | Calls Voyage AI API to get image embedding        | Image Test URL                      | Qdrant variables + embedding + KNN neighbours | See Sticky Note4                                                                                                    |
| Qdrant variables + embedding + KNN neighbours | Set                    | Sets embedding and Qdrant query parameters       | Embed image                        | Query Qdrant                   | See Sticky Note3: Variables defining Qdrant collection and KNN limit                                               |
| Query Qdrant                    | HTTP Request           | Queries Qdrant for nearest neighbors              | Qdrant variables + embedding + KNN neighbours | Propagate loop variables        | See Sticky Note5: Tie loop explanation and Qdrant query details                                                     |
| Propagate loop variables         | Set                    | Updates loop variables including limitKNN         | Query Qdrant                       | Majority Vote                  | See Sticky Note5                                                                                                     |
| Majority Vote                   | Code (Python)          | Performs majority voting on retrieved neighbors' classes | Propagate loop variables           | Check tie                     | See Sticky Note5                                                                                                     |
| Check tie                       | If                     | Checks if tie exists and controls loop continuation | Majority Vote                     | Increase limitKNN, Return class | See Sticky Note5                                                                                                     |
| Increase limitKNN               | Set                    | Increases neighbor limit by 5 for next query       | Check tie (true branch)             | Query Qdrant                   | See Sticky Note5                                                                                                     |
| Return class                   | Set                    | Extracts and outputs final classification result   | Check tie (false branch)            | None                         | See Sticky Note6: Extracting decided class name                                                                     |
| Sticky Note                    | Sticky Note            | Lists land use classes being classified             | None                              | None                          | See Sticky Note content                                                                                                |
| Sticky Note1                   | Sticky Note            | Reports tested accuracy on validation dataset       | None                              | None                          |                                                                                                                      |
| Sticky Note2                   | Sticky Note            | Overview of KNN classification workflow             | None                              | None                          | See sticky note content                                                                                              |
| Sticky Note3                   | Sticky Note            | Explains variables for Qdrant collection and KNN limit | None                              | None                          | See sticky note content                                                                                              |
| Sticky Note4                   | Sticky Note            | Notes embedding input image with Voyage model       | None                              | None                          | See sticky note content                                                                                              |
| Sticky Note5                   | Sticky Note            | Explains tie loop mechanism and Qdrant querying     | None                              | None                          | See sticky note content                                                                                              |
| Sticky Note6                   | Sticky Note            | Notes final class extraction                          | None                              | None                          | See sticky note content                                                                                              |
| Sticky Note10                  | Sticky Note            | Summarizes the entire KNN classification pipeline   | None                              | None                          | See sticky note content                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**  
   - Add **Execute Workflow Trigger** node as the workflow's entry point. No special configuration needed.

2. **Create Input Preparation Node**  
   - Add a **Set** node named `Image Test URL`.  
   - Assign a new variable `imageURL` with the expression `{{$json.query.imageURL}}`.  
   - Connect the trigger node's output to this node.

3. **Create Image Embedding Node**  
   - Add an **HTTP Request** node named `Embed image`.  
   - Set method to `POST` and URL to `https://api.voyageai.com/v1/multimodalembeddings`.  
   - Set body content type to JSON with the following body:  
     ```json
     {
       "inputs": [
         {
           "content": [
             {
               "type": "image_url",
               "image_url": "={{$json.imageURL}}"
             }
           ]
         }
       ],
       "model": "voyage-multimodal-3",
       "input_type": "document"
     }
     ```  
   - Enable sending the body and specify authentication as HTTP Header Auth with your **Voyage API** credentials.  
   - Connect `Image Test URL` to this node.

4. **Create Parameters Setup Node**  
   - Add a **Set** node named `Qdrant variables + embedding + KNN neighbours`.  
   - Set the following variables:  
     - `ImageEmbedding`: `={{ $json.data[0].embedding }}`  
     - `qdrantCloudURL`: your Qdrant Cloud URL, e.g., `"https://<your-cluster>.eu-central-1-0.aws.cloud.qdrant.io"`  
     - `collectionName`: `"land-use"` (or your collection name)  
     - `limitKNN`: `10` (initial neighbors to query)  
   - Connect `Embed image` to this node.

5. **Create Qdrant Query Node**  
   - Add an **HTTP Request** node named `Query Qdrant`.  
   - Set method to `POST` and URL to:  
     `={{ $json.qdrantCloudURL + '/collections/' + $json.collectionName + '/points/query' }}`  
   - Set JSON body as:  
     ```json
     {
       "query": "={{ $json.ImageEmbedding }}",
       "using": "voyage",
       "limit": "={{ $json.limitKNN }}",
       "with_payload": true
     }
     ```  
   - Use your **Qdrant API** credentials for authentication.  
   - Connect `Qdrant variables + embedding + KNN neighbours` to this node.

6. **Create Loop Variables Propagation Node**  
   - Add a **Set** node named `Propagate loop variables`.  
   - Set:  
     - `limitKNN`: `={{ $json.result.points.length }}`  
     - `result`: `={{ $json.result }}`  
   - Connect `Query Qdrant` to this node.

7. **Create Majority Vote Node**  
   - Add a **Code** node named `Majority Vote`.  
   - Select language Python.  
   - Use the following code:
     ```python
     from collections import Counter

     input_json = _input.all()[0]
     points = input_json['json']['result']['points']
     majority_vote_two_most_common = Counter([point["payload"]["landscape_name"] for point in points]).most_common(2)

     return [{
         "json": {
             "result": majority_vote_two_most_common    
         }
     }]
     ```  
   - Connect `Propagate loop variables` to this node.

8. **Create Tie Check Node**  
   - Add an **If** node named `Check tie`.  
   - Configure with the following conditions (all must be true):  
     1. Number of classes in result > 1: `{{$json.result.length}} > 1`  
     2. Counts of top two classes equal: `{{$json.result[0][1]}} == {{$json.result[1][1]}}`  
     3. `limitKNN` ≤ 100: `{{$json.limitKNN}} <= 100`  
   - Connect `Majority Vote` to this node.

9. **Create Increase limit Node**  
   - Add a **Set** node named `Increase limitKNN`.  
   - Set:  
     - `limitKNN`: `={{ $('Propagate loop variables').item.json.limitKNN + 5 }}`  
     - `ImageEmbedding`: `={{ $('Qdrant variables + embedding + KNN neighbours').first().json.ImageEmbedding }}`  
     - `qdrantCloudURL`: `={{ $('Qdrant variables + embedding + KNN neighbours').first().json.qdrantCloudURL }}`  
     - `collectionName`: `={{ $('Qdrant variables + embedding + KNN neighbours').first().json.collectionName }}`  
   - Connect `Check tie` true branch to this node.

10. **Loop Back to Qdrant Query**  
    - Connect `Increase limitKNN` output to `Query Qdrant` input to repeat the loop.

11. **Create Return Class Node**  
    - Add a **Set** node named `Return class`.  
    - Set variable `class` to `{{$json.result[0][0]}}` to extract the winning class.  
    - Connect `Check tie` false branch to this node.

12. **Final Output**  
    - The output of `Return class` node is the workflow result, containing the predicted land use class.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                                                      |
|---------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| This workflow classifies satellite imagery into 21 land use classes such as 'forest', 'beach', 'airport', etc.       | Sticky Note content listing classes                                                                                               |
| Tested classification accuracy on the test dataset is 93.24% without fine-tuning or metric learning.                | Sticky Note1                                                                                                                       |
| The KNN classifier is designed as a companion to a dataset upload pipeline; both need to be configured with matching data. | Workflow description & Sticky Note10                                                                                              |
| To recreate, upload the [lands dataset](https://www.kaggle.com/datasets/apollo2506/landuse-scene-classification) to Google Storage and set up Qdrant Cloud and Voyage AI API credentials. | Workflow description                                                                                                              |
| Qdrant Cloud Quickstart: https://qdrant.tech/documentation/quickstart-cloud/                                         | Official Qdrant documentation                                                                                                     |
| Voyage AI API documentation: https://www.voyageai.com/                                                              | Official Voyage AI documentation                                                                                                  |
| The tie resolution loop increases the number of neighbors queried by increments of 5 up to a maximum of 100 neighbors. | Sticky Note5                                                                                                                      |

---

This documentation provides a comprehensive understanding of the KNN classifier workflow for satellite imagery classification, allowing for reproduction, modification, and troubleshooting of the pipeline.