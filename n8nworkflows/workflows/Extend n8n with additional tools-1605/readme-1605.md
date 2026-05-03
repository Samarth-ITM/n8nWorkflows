Extend n8n with additional tools

https://n8nworkflows.xyz/workflows/extend-n8n-with-additional-tools-1605


# Extend n8n with additional tools

### 1. Workflow Overview

This workflow implements a Telegram bot that provides weather information for selected European capitals. It fetches real-time weather data from the OpenWeatherMap API, processes it into a CSV file, generates a weather plot image using an external R script with ggplot2, and sends the resulting image back to the Telegram user.

The workflow is logically structured into these blocks:

- **1.1 Telegram Input & Command Switch**: Listens to Telegram messages, detects commands, and routes the flow accordingly.
- **1.2 User Interaction Messages**: Sends greeting, error, and status messages to the user based on command recognition and process outcomes.
- **1.3 Weather Data Preparation**: Defines the list of cities and fetches weather data from OpenWeatherMap API.
- **1.4 Data Conversion and CSV Generation**: Converts API JSON response to a CSV format compatible with the R script.
- **1.5 R Script Execution & Image Generation**: Runs an R script that creates a weather plot image using ggplot2 based on the CSV data.
- **1.6 Sending Image & Error Handling**: Reads the generated image file and sends it via Telegram; handles API and R script errors gracefully.

---

### 2. Block-by-Block Analysis

#### 2.1 Telegram Input & Command Switch

**Overview:**  
Receives Telegram messages and routes workflow execution based on recognized commands (`/start`, `/getweather`) or falls back to error handling for unknown commands.

**Nodes Involved:**  
- Telegram Trigger  
- Switch  
- Merge1  
- Merge  
- Merge2  

**Node Details:**

- **Telegram Trigger**  
  - Type: Telegram Trigger  
  - Role: Entry point; listens for incoming Telegram messages (only "message" updates).  
  - Config: Uses Telegram API credentials for the bot.  
  - Inputs: External Telegram messages.  
  - Outputs: Message JSON containing text and user info.  
  - Edge Cases: Telegram API errors, webhook misconfiguration.

- **Switch**  
  - Type: Switch (string condition)  
  - Role: Checks the message text against `/start` and `/getweather`.  
  - Config: Routes `/start` to output 0, `/getweather` to output 1, else fallback output 3.  
  - Inputs: Message text from Telegram Trigger.  
  - Outputs: Branches to greeting, weather request, or wrong command handling.  
  - Edge Cases: Case sensitivity, unexpected command formats.

- **Merge1, Merge, Merge2**  
  - Type: Merge  
  - Role: Combines separate workflow branches to maintain a single flow.  
  - Modes: `passThrough` for Merge1 and Merge; `wait` for Merge2 (waits for all inputs).  
  - Inputs/Outputs: Connects Switch outputs to subsequent message nodes or data processing nodes.

---

#### 2.2 User Interaction Messages

**Overview:**  
Sends appropriate Telegram messages to the user: greeting on `/start`, waiting status on `/getweather`, error messages for unrecognized commands, API errors, or R script failures.

**Nodes Involved:**  
- msg_greet  
- msg_wrongcommand  
- msg_pleasewait  
- msg_errorAPI  
- msg_errorR  
- msg_getweather  

**Node Details:**

- **msg_greet**  
  - Type: Telegram  
  - Role: Sends a welcome message explaining bot capabilities.  
  - Config: Uses user's first name dynamically; sends on `/start`.  
  - Inputs: From Merge1 (triggered by `/start`).  
  - Outputs: None (endpoint).  
  - Edge Cases: Missing first name field.

- **msg_wrongcommand**  
  - Type: Telegram  
  - Role: Notifies user of unrecognized command, suggests valid command.  
  - Config: Uses user's first name dynamically; triggered on unknown commands.  
  - Inputs: From Merge (fallback of Switch).  
  - Outputs: None.  

- **msg_pleasewait**  
  - Type: Telegram  
  - Role: Informs user that the weather data processing is underway.  
  - Config: Sends during `/getweather` processing.  
  - Inputs: From Switch output 1.  
  - Outputs: Passes control to Merge2 for synchronization.

- **msg_errorAPI**  
  - Type: Telegram  
  - Role: Sends error message if API call fails.  
  - Config: Uses Markdown parse mode; mentions user by first name.  
  - Inputs: From API error detection node.  
  - Outputs: None.

- **msg_errorR**  
  - Type: Telegram  
  - Role: Sends error message if R script fails to produce image.  
  - Config: Similar to msg_errorAPI but for R processing errors.  
  - Inputs: From R script exit code check (failure branch).  
  - Outputs: None.

- **msg_getweather**  
  - Type: Telegram (sendPhoto)  
  - Role: Sends the generated weather image to user with caption.  
  - Config: Sends the PNG image read from disk; caption includes user first name.  
  - Inputs: From Read Binary File node.  
  - Outputs: None.

---

#### 2.3 Weather Data Preparation

**Overview:**  
Defines a fixed list of European cities with IDs for OpenWeatherMap, then initiates API requests for weather data for each city.

**Nodes Involved:**  
- Filename  
- City List  
- Get weather data  
- Any errors API?  

**Node Details:**

- **Filename**  
  - Type: Set  
  - Role: Defines filename and folder path for temporary CSV and image files.  
  - Config: Filename includes user ID and timestamp to ensure unique naming; folder path fixed to `/home/node/.n8n/weather-bot/`.  
  - Inputs: From Merge2 (start of weather data block).  
  - Outputs: To City List.

- **City List**  
  - Type: Function  
  - Role: Returns an array of city objects (ID, name, country) for 10 European capitals.  
  - Config: Hardcoded city IDs from OpenWeatherMap.  
  - Inputs: From Filename.  
  - Outputs: To Get weather data node (one call per city).  
  - Edge Cases: City IDs must be valid; missing IDs cause API errors.

- **Get weather data**  
  - Type: HTTP Request  
  - Role: Fetches current weather data for each city ID from OpenWeatherMap API.  
  - Config: URL template uses city ID, metric units, and a fixed API key (`6d3fff582a101700576faf74734f9535`).  
  - Inputs: From City List (iterates over cities).  
  - Outputs: To Any errors API?.  
  - Edge Cases: API key invalid or rate limits; possible network timeouts; continueOnFail allows partial failure.

- **Any errors API?**  
  - Type: If  
  - Role: Checks if API response contains an error by inspecting for an `error.name` field equal to `"Error"`.  
  - Config: Routes error responses to msg_errorAPI, successful responses to Convert API response.  
  - Inputs: From Get weather data.  
  - Outputs: Error or success branch.

---

#### 2.4 Data Conversion and CSV Generation

**Overview:**  
Transforms the JSON weather data into a simplified object suitable for CSV export, then writes the CSV file for R script consumption.

**Nodes Involved:**  
- Convert API response  
- Spreadsheet File  
- Write csv  

**Node Details:**

- **Convert API response**  
  - Type: Function  
  - Role: Extracts city name, country, current temp, min temp, and max temp from API JSON.  
  - Config: Constructs an array of objects with keys: CityName, TempCur, TempMin, TempMax.  
  - Inputs: Success branch from Any errors API?.  
  - Outputs: To Spreadsheet File.  
  - Edge Cases: Missing JSON fields or unexpected response structure.

- **Spreadsheet File**  
  - Type: Spreadsheet File  
  - Role: Converts the JSON data array into a CSV file binary.  
  - Config: Filename set dynamically based on Filename node; output file format is CSV.  
  - Inputs: From Convert API response.  
  - Outputs: To Write csv.  
  - Edge Cases: File write permission issues.

- **Write csv**  
  - Type: Write Binary File  
  - Role: Saves the CSV binary data to disk using the folder and filename from Filename node.  
  - Config: File path concatenates folder path and filename.  
  - Inputs: From Spreadsheet File.  
  - Outputs: To Run R script.

---

#### 2.5 R Script Execution & Image Generation

**Overview:**  
Executes an external R script to generate a weather plot image (PNG) based on the CSV data, capturing logs and handling success/failure.

**Nodes Involved:**  
- Run R script  
- R successful?  

**Node Details:**

- **Run R script**  
  - Type: Execute Command  
  - Role: Runs the R script `dumbbell_plot.R` with input CSV and output PNG file paths as arguments.  
  - Config: Command constructed dynamically using Filename folder and file names; standard output redirected to a log file.  
  - Inputs: From Write csv.  
  - Outputs: To R successful?.  
  - Edge Cases: Rscript not installed, script errors, file permission issues, continueOnFail enabled to allow error branch.

- **R successful?**  
  - Type: If  
  - Role: Checks the R script exit code equals 0 (success).  
  - Inputs: From Run R script.  
  - Outputs: Success branch to Read Binary File; failure branch to msg_errorR.

---

#### 2.6 Sending Image & Error Handling

**Overview:**  
Reads the generated PNG image from disk and sends it to the user via Telegram; also handles and notifies about API or R script errors.

**Nodes Involved:**  
- Read Binary File  
- msg_getweather  

**Node Details:**

- **Read Binary File**  
  - Type: Read Binary File  
  - Role: Loads the generated PNG image from the specified file path.  
  - Config: File path built from folder and image name from Filename node.  
  - Inputs: Success branch from R successful?.  
  - Outputs: To msg_getweather.  
  - Edge Cases: Missing or unreadable image file.

- **msg_getweather**  
  - See section 2.2.

---

### 3. Summary Table

| Node Name         | Node Type             | Functional Role                         | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                          |
|-------------------|-----------------------|---------------------------------------|------------------------|-------------------------|----------------------------------------------------------------------------------------------------|
| Telegram Trigger   | telegramTrigger       | Entry: receives Telegram messages     | -                      | Switch                  |                                                                                                    |
| Switch            | switch                | Routes commands (/start, /getweather) | Telegram Trigger       | Merge1, msg_pleasewait, Merge | check bot commands                                                                                  |
| Merge1            | merge                 | Merges `/start` branch                 | Switch                  | msg_greet               |                                                                                                    |
| msg_greet         | telegram              | Sends greeting message                 | Merge1                  | -                       |                                                                                                    |
| msg_pleasewait    | telegram              | Sends processing status                | Switch                  | Merge2                  |                                                                                                    |
| Merge2            | merge                 | Waits for inputs before proceeding    | msg_pleasewait, Switch  | Filename                |                                                                                                    |
| Filename          | set                   | Sets dynamic filenames & folder path  | Merge2                  | City List               |                                                                                                    |
| City List         | function              | Provides city IDs and names            | Filename                 | Get weather data         |                                                                                                    |
| Get weather data  | httpRequest           | Fetches weather data from OpenWeatherMap API | City List           | Any errors API?          |                                                                                                    |
| Any errors API?   | if                    | Checks for API errors                  | Get weather data         | msg_errorAPI, Convert API response |                                                                                                    |
| msg_errorAPI      | telegram              | Sends API error message                | Any errors API?          | -                       |                                                                                                    |
| Convert API response | function            | Converts JSON to simplified CSV data  | Any errors API?          | Spreadsheet File         | this data is stored as a CSV file and then processed in the R script. Please check the R code here: https://gist.github.com/ed-parsadanyan/0561cd12d545e642fcef17dcb0872b00 |
| Spreadsheet File  | spreadsheetFile       | Converts JSON to CSV binary            | Convert API response     | Write csv                |                                                                                                    |
| Write csv         | writeBinaryFile       | Writes CSV file to disk                | Spreadsheet File        | Run R script             |                                                                                                    |
| Run R script      | executeCommand        | Runs R script for image generation    | Write csv                | R successful?            |                                                                                                    |
| R successful?     | if                    | Checks R script success                | Run R script             | Read Binary File, msg_errorR |                                                                                                    |
| Read Binary File  | readBinaryFile        | Reads generated PNG file               | R successful?            | msg_getweather           |                                                                                                    |
| msg_getweather    | telegram              | Sends weather image to user            | Read Binary File         | -                       |                                                                                                    |
| msg_errorR        | telegram              | Sends R script error message           | R successful?            | -                       |                                                                                                    |
| msg_wrongcommand  | telegram              | Sends unknown command message          | Merge                    | -                       |                                                                                                    |
| Merge             | merge                 | Merges unknown command branch          | Switch                   | msg_wrongcommand         |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node:**  
   - Type: Telegram Trigger  
   - Parameters: Listen to "message" updates only  
   - Credentials: Select your Telegram Bot API credentials  
   - Position: Start of workflow  

2. **Add Switch node:**  
   - Type: Switch (String)  
   - Parameter: Evaluate `{{$json["message"]["text"]}}`  
   - Rules:  
     - If equals `/start`, output 0  
     - If equals `/getweather`, output 1  
     - Fallback output 3  
   - Connect input from Telegram Trigger’s main output.  

3. **Add Merge nodes:**  
   - Merge1: PassThrough mode, connect Switch output 0 (start command) to Merge1.  
   - Merge2: Wait mode, connect Switch output 1 (getweather command) and msg_pleasewait node to Merge2.  
   - Merge: PassThrough mode, connect Switch fallback output 3 (unknown commands) to Merge.

4. **Create User Messaging nodes:**  
   - msg_greet (Telegram): Sends greeting message with dynamic `{{$node["Telegram Trigger"].json["message"]["from"]["first_name"]}}`  
     - Connect from Merge1.  
   - msg_wrongcommand (Telegram): Sends unknown command message similarly personalized.  
     - Connect from Merge.  
   - msg_pleasewait (Telegram): Sends "Please wait..." message during processing.  
     - Connect from Switch output 1 and to Merge2.

5. **Create Filename node (Set):**  
   - Set variables:  
     - `filename`: `"request_from" + userId + timestamp`  
     - `foldername`: `"/home/node/.n8n/weather-bot/"`  
     - `imgname`: `"request_from" + userId`  
   - Use expressions with user ID from Telegram Trigger and current timestamp.  
   - Input from Merge2.

6. **Create City List node (Function):**  
   - Return hardcoded array of city objects with IDs and names for 10 European capitals.  
   - Input from Filename.

7. **Create Get weather data node (HTTP Request):**  
   - Method: GET  
   - URL: `https://api.openweathermap.org/data/2.5/weather?id={{$json["Cityid"]}}&units=metric&appid=YOUR_API_KEY` (replace with your API key)  
   - Input from City List node.

8. **Add Any errors API? node (If):**  
   - Condition: Check if `{{$json["error"]["name"]}}` equals `"Error"`  
   - True branch: Connect to msg_errorAPI node.  
   - False branch: Connect to Convert API response.

9. **Create msg_errorAPI node (Telegram):**  
   - Sends API error notification with user first name.  
   - Input from Any errors API? true branch.

10. **Create Convert API response node (Function):**  
    - Extract city name, country, current/min/max temperatures from API JSON.  
    - Return simplified array for CSV conversion.  
    - Input from Any errors API? false branch.

11. **Add Spreadsheet File node:**  
    - Operation: toFile  
    - File format: CSV  
    - Filename: Use expression from Filename node for naming.  
    - Input from Convert API response.

12. **Add Write csv node (Write Binary File):**  
    - File path: Combine folder path and filename from Filename node.  
    - Input from Spreadsheet File node.

13. **Add Run R script node (Execute Command):**  
    - Command:  
      ```bash
      Rscript --vanilla '{{folder}}{{filename}}.csv' '{{folder}}{{imgname}}.png' >& '{{folder}}{{filename}}.log'
      ```  
      Using folder, filename, and imgname from Filename node.  
    - continueOnFail: true  
    - Input from Write csv node.

14. **Add R successful? node (If):**  
    - Condition: Check if exitCode equals 0  
    - True branch: Connect to Read Binary File node.  
    - False branch: Connect to msg_errorR node.

15. **Add Read Binary File node:**  
    - File path: `'{{folder}}{{imgname}}.png'` from Filename node.  
    - Input from R successful? true branch.

16. **Add msg_getweather node (Telegram):**  
    - Operation: sendPhoto  
    - Photo: Binary data from Read Binary File node  
    - Caption: Personalized to user first name  
    - Input from Read Binary File node.

17. **Add msg_errorR node (Telegram):**  
    - Sends R script failure message with user first name.  
    - Input from R successful? false branch.

18. **Wire all nodes as per the logical flow described, ensuring the correct connections for branching and merging.**

19. **Configure Telegram API credentials:**  
    - Create or link bot credentials in n8n with Bot API Token.  

20. **Ensure R environment:**  
    - On the n8n host machine, install R, ggplot2 package, and place the `dumbbell_plot.R` script in the specified folder.  
    - Confirm `Rscript` command is available in system path.

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                                                                                              |
|--------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| The R script used for plotting is available here: https://gist.github.com/ed-parsadanyan/0561cd12d545e642fcef17dcb0872b00 | Reference for CSV structure and plotting logic used by the Execute Command node.                                              |
| To generate weather plots, the R environment with ggplot2 package must be installed on the n8n host machine. | Ensure R and required packages are installed and accessible to n8n's execution environment.                                   |
| The Telegram Bot API credentials must be properly configured in n8n with OAuth2 or bot token for the bot.     | Without valid credentials, the Telegram Trigger and Telegram nodes will fail.                                                |
| OpenWeatherMap API key is hardcoded in the HTTP Request node URL; replace with a valid API key to avoid errors. | Consider externalizing API keys for security in production workflows.                                                        |