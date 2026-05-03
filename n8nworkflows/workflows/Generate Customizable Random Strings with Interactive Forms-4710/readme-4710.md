Generate Customizable Random Strings with Interactive Forms

https://n8nworkflows.xyz/workflows/generate-customizable-random-strings-with-interactive-forms-4710


# Generate Customizable Random Strings with Interactive Forms

### 1. Workflow Overview

This workflow, titled **Generate Customizable Random Strings with Interactive Forms**, enables users to generate multiple random alphanumeric strings of a specified length via an interactive web form. The core use case is to provide quick generation of random strings (e.g., passwords or tokens) with customizable length and quantity, and display them in a formatted HTML list.

The workflow is logically organized into these blocks:

- **1.1 Input Reception:** Captures user inputs for string length and number of copies via an interactive form.
- **1.2 Duplication Logic:** Creates multiple copies of the generation request to produce the specified number of random strings.
- **1.3 Random String Generation:** Generates a base64-encoded random string of the requested length for each copy.
- **1.4 Formatting and Aggregation:** Wraps each string in HTML list item tags, concatenates all items, and formats the full output into an HTML page.
- **1.5 Output Delivery:** Returns the formatted HTML as a response to the user via a form response node.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
This block triggers the workflow via a web form where users input the desired length of each random string and how many copies to generate.

- **Nodes Involved:**  
  - `rand_generator_form`

- **Node Details:**  
  - **Name:** rand_generator_form  
  - **Type:** Form Trigger  
  - **Role:** Entry point to capture user input  
  - **Configuration:**  
    - Form titled "rand pass generator" with two required number fields:  
      - `length`: length of each random string (number, required, placeholder 16)  
      - `copies`: number of copies to generate (number, required, placeholder 5)  
    - Button labeled "Generate now"  
    - Response mode set to "lastNode" (response is sent from the last node in the workflow)  
  - **Inputs:** None (trigger node)  
  - **Outputs:** Connects to `duplicates` node  
  - **Edge Cases/Potential Failures:**  
    - User submits invalid or empty input (form validation enforces required fields)  
    - Network issues preventing form triggering  
  - **Version:** 2.2

#### 2.2 Duplication Logic

- **Overview:**  
Duplicates the incoming item to create multiple copies equal to the user-specified `copies` value minus one, preparing for multiple random string generations.

- **Nodes Involved:**  
  - `duplicates`

- **Node Details:**  
  - **Name:** duplicates  
  - **Type:** Set (with duplication feature)  
  - **Role:** Duplicate input item to generate multiple random string requests  
  - **Configuration:**  
    - No new fields assigned  
    - Duplicate count set dynamically: `={{ $json.copies -1 }}` (duplicates original once plus these copies → total copies)  
    - Duplicate item enabled  
  - **Input:** From `rand_generator_form`  
  - **Output:** To `Generate a random string`  
  - **Edge Cases/Potential Failures:**  
    - If `copies` = 0 or negative, duplication count may be invalid or zero (workflow behavior depends on n8n's handling)  
  - **Version:** 3.4

#### 2.3 Random String Generation

- **Overview:**  
Generates a random string encoded in base64 of the length specified by the user for each duplicated item.

- **Nodes Involved:**  
  - `Generate a random string`

- **Node Details:**  
  - **Name:** Generate a random string  
  - **Type:** Crypto  
  - **Role:** Produce base64-encoded random string of specified length  
  - **Configuration:**  
    - Action: Generate  
    - Encoding: base64  
    - String length: dynamic expression `={{ $('rand_generator_form').item.json.length }}` (fetch length input from form trigger)  
  - **Input:** From `duplicates`  
  - **Output:** To `format an item`  
  - **Edge Cases/Potential Failures:**  
    - If length is zero or invalid, generation might fail or produce empty output  
    - Crypto node errors, such as internal crypto library issues  
  - **Version:** 1

#### 2.4 Formatting and Aggregation

- **Overview:**  
Formats each random string into an HTML list item, then concatenates all generated strings into a single string, and finally embeds them into a complete HTML page.

- **Nodes Involved:**  
  - `format an item`  
  - `concatenate items`  
  - `format into html`

- **Node Details:**  

  - **format an item**  
    - Type: Set  
    - Role: Wrap each generated string into an `<li>` HTML element  
    - Configuration: Assigns field `data` to `<li>{{ $json.data }}</li>`  
    - Input: From `Generate a random string`  
    - Output: To `concatenate items`  
    - Edge cases: Expression failures if `data` field missing  

  - **concatenate items**  
    - Type: Summarize  
    - Role: Concatenate all `data` fields into a single string separated by newlines  
    - Configuration: Field to summarize: `data`, aggregation: concatenate, separator: newline  
    - Input: From `format an item`  
    - Output: To `format into html`  
    - Edge cases: Empty input array yields empty concatenation  

  - **format into html**  
    - Type: HTML  
    - Role: Embed concatenated items into a full HTML template  
    - Configuration:  
      - Static HTML template with title "random strings" and container div  
      - Includes dynamic header showing number of copies and string length from form trigger  
      - Inserts concatenated list items into `<ul>` element via `{{ $json.concatenated_data }}`  
      - Inline CSS styles and a trivial JavaScript console log  
    - Input: From `concatenate items`  
    - Output: To `Display results`  
    - Edge cases: If concatenated data empty, list will be empty  

#### 2.5 Output Delivery

- **Overview:**  
Sends the final formatted HTML response back to the user as a form completion response.

- **Nodes Involved:**  
  - `Display results`

- **Node Details:**  
  - **Name:** Display results  
  - **Type:** Form  
  - **Role:** Return the HTML output as the final response to the form trigger  
  - **Configuration:**  
    - Operation: completion  
    - Respond with: showText  
    - Response text set dynamically to `={{ $json.html }}` (HTML content from previous node)  
  - **Input:** From `format into html`  
  - **Output:** None (terminal node)  
  - **Edge cases:**  
    - n8n form node may not render complex HTML perfectly (noted in sticky note)  
  - **Version:** 1

---

### 3. Summary Table

| Node Name              | Node Type            | Functional Role                      | Input Node(s)         | Output Node(s)          | Sticky Note                                                                                 |
|------------------------|----------------------|------------------------------------|-----------------------|-------------------------|---------------------------------------------------------------------------------------------|
| rand_generator_form     | Form Trigger         | Capture user input for length and copies | None                  | duplicates              |                                                                                             |
| duplicates             | Set                  | Duplicate item to generate multiple copies | rand_generator_form   | Generate a random string |                                                                                             |
| Generate a random string| Crypto               | Generate random base64 string of given length | duplicates            | format an item           |                                                                                             |
| format an item          | Set                  | Format each string as HTML list item | Generate a random string | concatenate items       |                                                                                             |
| concatenate items       | Summarize            | Concatenate all list items into one string | format an item         | format into html         |                                                                                             |
| format into html        | HTML                 | Embed concatenated strings in full HTML page | concatenate items      | Display results          |                                                                                             |
| Display results         | Form                 | Return HTML response to the user   | format into html       | None                    | n8n Form apparently can't display HTML properly yet.                                        |
| Sticky Note            | Sticky Note          | Workflow summary and notes         | None                  | None                    | ## random-string generator<br><br>### Summary<br>This n8n workflow generates random strings containing only alphanumeric characters.<br>You can specify the length of the string and how many copies you like to generate.<br><br>### Nodes used<br>* formTrigger<br>* set<br>* crypto<br>* summarize (concatenate)<br>* HTML<br>* form (form ending)<br><br>### Notes<br>* Dups operation is used to generate multipe copies as specified.<br>* n8n Form apparently can't display HTML properly yet. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node named `rand_generator_form`:**  
   - Set form title: "rand pass generator"  
   - Add two required number fields:  
     - "length" (placeholder: 16)  
     - "copies" (placeholder: 5)  
   - Set button label: "Generate now"  
   - Response mode: "lastNode"  
   - Save and note the webhook URL generated.

2. **Add a Set node named `duplicates`:**  
   - Enable "Duplicate Item" option  
   - Set "Duplicate Count" to expression: `={{ $json.copies - 1 }}`  
   - Do not assign new fields  
   - Connect `rand_generator_form` output to `duplicates` input.

3. **Add a Crypto node named `Generate a random string`:**  
   - Action: Generate  
   - Encoding Type: Base64  
   - String Length: expression `={{ $('rand_generator_form').item.json.length }}`  
   - Connect `duplicates` output to this node.

4. **Add a Set node named `format an item`:**  
   - Add a field assignment:  
     - Field name: `data`  
     - Type: String  
     - Value: `<li>{{ $json.data }}</li>` (use expression to wrap generated string)  
   - Connect `Generate a random string` output to this node.

5. **Add a Summarize node named `concatenate items`:**  
   - Set summarization field: `data`  
   - Aggregation: Concatenate  
   - Separator: Newline (`\n`)  
   - Connect `format an item` output to this node.

6. **Add an HTML node named `format into html`:**  
   - Paste the following HTML template as static content, replacing interpolations with expressions:  

   ```html
   <!DOCTYPE html>
   <html>
   <head>
     <meta charset="UTF-8" />
     <title>random strings</title>
   </head>
   <body>
     <div class="container">
       <h2>{{ $('rand_generator_form').item.json.copies }}X {{ $('rand_generator_form').item.json.length }}-char random strings</h2>
       <ul>{{ $json.concatenated_data }}</ul>
     </div>
   </body>
   </html>

   <style>
   container {
     background-color: #ffffff;
     text-align: center;
     padding: 16px;
     border-radius: 8px;
   }

   h1 {
     color: #ff6d5a;
     font-size: 24px;
     font-weight: bold;
     padding: 8px;
   }

   h2 {
     color: #909399;
     font-size: 18px;
     font-weight: bold;
     padding: 8px;
   }
   </style>

   <script>
   console.log("Hello World!");
   </script>
   ```

   - Connect `concatenate items` output to this node.

7. **Add a Form node named `Display results`:**  
   - Operation: Completion  
   - Respond with: Show Text  
   - Response Text: expression `={{ $json.html }}` (HTML from previous node)  
   - Connect `format into html` output to this node.

8. **Activate the workflow:**  
   - Test by submitting the form via the provided webhook URL with values for length and copies.  
   - The workflow will generate the requested number of random base64 strings and return them in an HTML list.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                  | Context or Link                        |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------|
| This workflow generates random strings containing only alphanumeric characters. Users specify the string length and number of copies to generate. The duplication node is used to replicate the generation requests.                         | Workflow description                  |
| n8n Form node currently has limitations rendering complex HTML properly, so the output may display as plain text in some environments.                                                                                                       | Sticky note content                   |
| Nodes involved include formTrigger, set, crypto, summarize, HTML, and form to create an end-to-end interactive string generator.                                                                                                            | Sticky note content                   |

---

**Disclaimer:** The text provided stems exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.