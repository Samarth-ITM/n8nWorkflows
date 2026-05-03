Standardize US Phone Numbers with Multiple Format Options and Validation

https://n8nworkflows.xyz/workflows/standardize-us-phone-numbers-with-multiple-format-options-and-validation-3812


# Standardize US Phone Numbers with Multiple Format Options and Validation

### 1. Workflow Overview

This workflow standardizes US phone numbers by validating and formatting input phone number strings into multiple common US and international formats, including extension handling. It is designed for use cases where consistent and validated US phone number formatting is needed, such as CRM data cleaning, SMS API preparation, or user input validation.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives the phone number input from another workflow trigger.
- **1.2 Formatting Removal:** Strips all non-numeric characters to isolate digits and extensions.
- **1.3 Digit Count Validation:** Checks the length of the numeric phone number to determine validity and processing path.
- **1.4 Country Code Validation:** Verifies or adds the US country code prefix as needed.
- **1.5 Formatting & Output Generation:** Produces multiple formatted versions of the phone number and extracts extension info if present.
- **1.6 Invalid Number Handling:** Clears the phone number field when validation fails.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block triggers the workflow and accepts the input phone number string.

- **Nodes Involved:**  
  - When Executed by Another Workflow

- **Node Details:**  
  - **When Executed by Another Workflow**  
    - *Type:* Execute Workflow Trigger  
    - *Role:* Entry point to accept external input.  
    - *Configuration:* Defines input parameter `"Phone Number"` as type `any`.  
    - *Input/Output:* Outputs JSON with the key `"Phone Number"` containing the input string.  
    - *Edge Cases:* Input might be missing or not a string; no explicit input validation here.  

#### 1.2 Formatting Removal

- **Overview:**  
  Strips all non-numeric characters from the input phone number to isolate digits only.

- **Nodes Involved:**  
  - Strip phone number formatting

- **Node Details:**  
  - **Strip phone number formatting**  
    - *Type:* Set node  
    - *Role:* Cleans input by extracting only digits using regex.  
    - *Configuration:* Uses expression `{{$json["Phone Number"].match(/[0-9]+/gmi).join('')}}` to join all digit groups into one string of digits.  
    - *Input:* From the trigger node.  
    - *Output:* JSON with `"Phone Number"` replaced by digits-only string.  
    - *Edge Cases:* Input without digits causes `.match()` to fail or return null; workflow depends on match success. Needs error handling for null match.  

#### 1.3 Digit Count Validation

- **Overview:**  
  Determines if the numeric string length corresponds to a valid US phone number, too short, or invalid.

- **Nodes Involved:**  
  - Check number of digits in phone number

- **Node Details:**  
  - **Check number of digits in phone number**  
    - *Type:* Switch node  
    - *Role:* Routes flow based on length of digits-only phone number string.  
    - *Configuration:*  
      - Outputs:  
        - `"Full Number"` if length ≥ 11 digits  
        - `"Number"` if length = 10 digits  
        - `"Invalid Number"` if length < 10 digits  
        - `"Not a Number"` if no digit data exists  
      - Uses expression: `$json["Phone Number"].toString().length` for length check.  
    - *Input:* From formatting removal node.  
    - *Output:* Routes to different next steps depending on length.  
    - *Edge Cases:* Assumes length correlates with validity; does not validate digit patterns here.  

#### 1.4 Country Code Validation

- **Overview:**  
  Checks if the first digit of the number is a valid US country code (`1`). If missing, adds it; otherwise, proceeds.

- **Nodes Involved:**  
  - Check if first digit is valid country code  
  - Add valid country code

- **Node Details:**  
  - **Check if first digit is valid country code**  
    - *Type:* If node  
    - *Role:* Tests if first digit equals `1`.  
    - *Configuration:* Expression: `{{$json["Phone Number"].toString().slice(0,1).toNumber()}} === 1`  
    - *Input:* From digit count validation node output `"Full Number"`.  
    - *Output:*  
      - True branch: number has valid country code, continue formatting.  
      - False branch: invalid country code, clear number.  
    - *Edge Cases:* Number not starting with 1 is invalid in this context.  
  - **Add valid country code**  
    - *Type:* Set node  
    - *Role:* Prepends `1` to the phone number when the number has exactly 10 digits (missing country code).  
    - *Configuration:* Sets `"Phone Number"` to string `"1" + original number`.  
    - *Input:* From digit count validation node output `"Number"` (length 10).  
    - *Output:* Passes to formatting node.  

#### 1.5 Formatting & Output Generation

- **Overview:**  
  Formats the validated phone number into multiple standard formats and extracts extension if present.

- **Nodes Involved:**  
  - Format phone numbers

- **Node Details:**  
  - **Format phone numbers**  
    - *Type:* Set node  
    - *Role:* Creates multiple output fields with different standardized phone number formats.  
    - *Configuration:*  
      - Copies original input string to `"Phone Number (Input)"`.  
      - Uses slices of the numeric phone number string to generate:  
        - `"Phone Number"`: numeric digits (up to 11 digits)  
        - `"=Phone Number (E-164)"`: `+` prefix + full number (e.g., +15551234567)  
        - `"Phone Number (National)"`: `(AAA) BBB-CCCC` format (area code 3 digits, next 3 digits, last 4 digits)  
        - `"Phone Number (Full National)"`: `1 (AAA) BBB-CCCC`  
        - `"Phone Number (International)"`: `00-1-AAA-BBB-CCCC`  
      - Extracts extension digits beyond 11th digit as:  
        - `"Extension"` (number)  
        - `"Extension (String)"` (string, empty if zero)  
      - Expressions use `.slice()` and `.toNumber()` conversions.  
    - *Input:* From either country code valid or added branch.  
    - *Output:* Final formatted phone number data as JSON fields.  
    - *Edge Cases:* Extension parsing assumes digits beyond 11th position; no explicit parsing for extensions with letters or other markers. If extension is not numeric, may fail.  

#### 1.6 Invalid Number Handling

- **Overview:**  
  Clears the phone number field if validation fails or input is invalid.

- **Nodes Involved:**  
  - Clear invalid number

- **Node Details:**  
  - **Clear invalid number**  
    - *Type:* Set node  
    - *Role:* Resets `"Phone Number"` to empty string to indicate invalid input.  
    - *Configuration:* Sets `"Phone Number"` to `""`.  
    - *Input:* From invalid or false branches of previous validation nodes.  
    - *Output:* Passes cleared data to formatting node (which will produce empty or default outputs).  
    - *Edge Cases:* No error thrown but results will be empty phone number formats.  

---

### 3. Summary Table

| Node Name                        | Node Type                    | Functional Role                         | Input Node(s)                     | Output Node(s)                       | Sticky Note                                   |
|---------------------------------|------------------------------|---------------------------------------|----------------------------------|------------------------------------|-----------------------------------------------|
| When Executed by Another Workflow| Execute Workflow Trigger      | Workflow entry and input reception    | None                             | Strip phone number formatting       |                                               |
| Strip phone number formatting    | Set                          | Removes non-digit characters           | When Executed by Another Workflow| Check number of digits in phone number |                                               |
| Check number of digits in phone number | Switch                   | Validates numeric string length        | Strip phone number formatting    | Check if first digit is valid country code, Add valid country code, Clear invalid number |                                               |
| Check if first digit is valid country code | If                     | Validates country code prefix          | Check number of digits in phone number ("Full Number") | Format phone numbers, Clear invalid number |                                               |
| Add valid country code           | Set                          | Adds US country code prefix if missing | Check number of digits in phone number ("Number") | Format phone numbers                 |                                               |
| Format phone numbers             | Set                          | Generates multiple standardized formats and extensions | Check if first digit is valid country code (true), Add valid country code | None                               |                                               |
| Clear invalid number             | Set                          | Clears invalid phone numbers           | Check number of digits in phone number ("Invalid Number","Not a Number"), Check if first digit is valid country code (false) | Format phone numbers                 |                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add **Execute Workflow Trigger** node named `"When Executed by Another Workflow"`.  
   - Configure input parameter:  
     - Name: `"Phone Number"`  
     - Type: `any`  

2. **Create Formatting Removal Node:**  
   - Add **Set** node named `"Strip phone number formatting"`.  
   - Connect input from `"When Executed by Another Workflow"`.  
   - Configure to overwrite `"Phone Number"` with expression:  
     ```javascript
     {{$json["Phone Number"].match(/[0-9]+/gmi).join('')}}
     ```  
   - This extracts all digits from input.

3. **Create Digit Count Validation Node:**  
   - Add **Switch** node named `"Check number of digits in phone number"`.  
   - Connect input from `"Strip phone number formatting"`.  
   - Add rules:  
     - Output `"Full Number"`: when length ≥ 11  
       Expression:  
       ```javascript
       $json["Phone Number"].toString().length >= 11
       ```  
     - Output `"Number"`: when length = 10  
       Expression:  
       ```javascript
       $json["Phone Number"].toString().length === 10
       ```  
     - Output `"Invalid Number"`: when length < 10  
       Expression:  
       ```javascript
       $json["Phone Number"].toString().length < 10
       ```  
     - Output `"Not a Number"`: when `"Phone Number"` field does not exist or empty  
       Expression:  
       ```javascript
       !$json["Phone Number"]
       ```  

4. **Create Country Code Validation Node:**  
   - Add **If** node named `"Check if first digit is valid country code"`.  
   - Connect input from `"Check number of digits in phone number"` output `"Full Number"`.  
   - Condition:  
     - Check if first digit equals `1`:  
       Expression:  
       ```javascript
       $json["Phone Number"].toString().slice(0,1).toNumber() === 1
       ```  

5. **Create Add Country Code Node:**  
   - Add **Set** node named `"Add valid country code"`.  
   - Connect input from `"Check number of digits in phone number"` output `"Number"`.  
   - Assign `"Phone Number"` field to expression:  
     ```javascript
     '1' + $json["Phone Number"]
     ```  

6. **Create Format Output Node:**  
   - Add **Set** node named `"Format phone numbers"`.  
   - Connect input from:  
     - `"Check if first digit is valid country code"` true output branch  
     - `"Add valid country code"` output  
     - `"Clear invalid number"` output (see next step)  
   - Assign fields with these expressions:  
     - `"Phone Number (Input)"`: `$('When Executed by Another Workflow').item.json['Phone Number']`  
     - `"Phone Number"`: first 11 digits as number:  
       ```javascript
       $json['Phone Number'].toString().slice(0,11).toNumber()
       ```  
     - `"=Phone Number (E-164)"`: `+` + full number string:  
       ```javascript
       $json['Phone Number'] ? '+' + $json['Phone Number'] : ''
       ```  
     - `"Phone Number (National)"`: `(AAA) BBB-CCCC` format:  
       ```javascript
       $json['Phone Number'] ? '(' + $json['Phone Number'].toString().slice(1,4) + ') ' + $json['Phone Number'].toString().slice(4,7) + '-' + $json['Phone Number'].toString().slice(7,11) : ''
       ```  
     - `"Phone Number (Full National)"`: `1 (AAA) BBB-CCCC`:  
       ```javascript
       $json['Phone Number'] ? '1 (' + $json['Phone Number'].toString().slice(1,4) + ') ' + $json['Phone Number'].toString().slice(4,7) + '-' + $json['Phone Number'].toString().slice(7,11) : ''
       ```  
     - `"Phone Number (International)"`: `00-1-AAA-BBB-CCCC`:  
       ```javascript
       $json['Phone Number'] ? '00-1-' + $json['Phone Number'].toString().slice(1,4) + '-' + $json['Phone Number'].toString().slice(4,7) + '-' + $json['Phone Number'].toString().slice(7,11) : ''
       ```  
     - `"Extension"`: digits after 11th position as number:  
       ```javascript
       $json['Phone Number'].toString().slice(11).toNumber()
       ```  
     - `"Extension (String)"`: string version or empty if zero:  
       ```javascript
       $json['Phone Number'].toString().slice(11).toNumber() > 0 ? $json['Phone Number'].toString().slice(11).toNumber() : ''
       ```  

7. **Create Clear Invalid Number Node:**  
   - Add **Set** node named `"Clear invalid number"`.  
   - Connect inputs from:  
     - `"Check number of digits in phone number"` outputs `"Invalid Number"` and `"Not a Number"`  
     - `"Check if first digit is valid country code"` false output branch  
   - Set `"Phone Number"` to empty string `""`.  
   - Connect output to `"Format phone numbers"` node to produce empty standardized output.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                   | Context or Link                          |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------|
| This workflow handles US phone numbers only and expects input strings that may contain various formatting characters or extensions.                                           | General Workflow Scope                  |
| Extensions are assumed to be numeric digits following the 11th digit in the cleaned number; non-numeric or letter-based extensions are not explicitly parsed.                   | Extension Handling                      |
| The workflow assumes US country code `1` is either present or added; other country codes are not supported and cause invalidation.                                            | Country Code Validation                 |
| Regex used for digit extraction: `/[0-9]+/gmi` extracts continuous digit groups and joins them without separators.                                                             | Formatting Removal                      |
| Validation is based on length checks and first digit checks; no advanced phone number pattern validation (e.g., NANP area codes) is performed.                                 | Validation Logic                        |
| The workflow can be integrated as a sub-workflow triggered by other workflows to standardize phone numbers on demand.                                                         | Integration Note                       |

---

This documentation fully describes the structure, logic, node configuration, and edge considerations of the "Standardize US Phone Numbers with Multiple Format Options and Validation" workflow, enabling accurate reproduction and modification.