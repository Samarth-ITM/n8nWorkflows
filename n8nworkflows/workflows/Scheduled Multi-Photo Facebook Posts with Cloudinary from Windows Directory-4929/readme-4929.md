Scheduled Multi-Photo Facebook Posts with Cloudinary from Windows Directory

https://n8nworkflows.xyz/workflows/scheduled-multi-photo-facebook-posts-with-cloudinary-from-windows-directory-4929


# Scheduled Multi-Photo Facebook Posts with Cloudinary from Windows Directory

### 1. Workflow Overview

This n8n workflow automates the scheduled posting of multiple photos to a Facebook Page using images stored in a Windows directory. It leverages Cloudinary as an image hosting intermediary and the Facebook Graph API to upload photos and create multi-photo posts. The process includes reading images and captions from local folders, randomly selecting descriptions and hashtags, uploading images to Cloudinary, posting to Facebook, and finally moving the posted images into dated archive folders.

**Target Use Cases:**  
- Social media managers automating multi-photo Facebook posts from a Windows-based image repository.  
- Scheduled batch uploads with dynamic captions and hashtags loaded from text files.  
- Integration of Cloudinary image hosting for Facebook posts.

**Logical Blocks:**  
- **1.1 Input Reception and File Reading**: Define image folder path and read images, descriptions, and hashtags from local Windows folders.  
- **1.2 Caption and Hashtag Processing**: Extract text content, randomly select lines for captions and hashtags.  
- **1.3 Image Upload and Facebook Photo Upload**: Upload images to Cloudinary, then upload hosted images as unpublished Facebook photos.  
- **1.4 Facebook Multi-Photo Post Creation**: Combine photo IDs and post the multi-photo post with message captions.  
- **1.5 Post-Processing and File Management**: Move posted images to date-stamped archive folders on the local file system.  
- **1.6 Scheduling and Loop Control**: Trigger the workflow every two hours and manage batch processing and waits.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception and File Reading

**Overview:**  
This block sets the working directory path for images and reads image files along with caption and hashtag text files from the Windows directory.

**Nodes Involved:**  
- ImgPath  
- Image  
- Description1  
- Hastag  

**Node Details:**  

- **ImgPath**  
  - Type: Set  
  - Role: Defines the base Windows directory path for images and captions.  
  - Config: Assigns `ImgPath` variable with path `"E:/Autopost-media/PageFB/202506-Images"`.  
  - Outputs to: Image, Description1, Hastag  
  - Edge Cases: Incorrect path or permissions could lead to file read failures.

- **Image**  
  - Type: Read Binary File  
  - Role: Reads all image files (any extension) in `ImgPath`.  
  - Config: Uses a dynamic file selector `={{ $json.ImgPath + '/**.***' }}` to match all files recursively.  
  - Input: From ImgPath  
  - Output: To Merge1  
  - Edge Cases: Empty folder, unsupported file types.

- **Description1**  
  - Type: Read Binary File  
  - Role: Reads description text file at `ImgPath + '/Caption/description.txt'`.  
  - Config: Reads with `.txt` extension, stores content in `description` property.  
  - Input: From ImgPath  
  - Output: To Extract from File  
  - Edge Cases: Missing or empty description file.

- **Hastag**  
  - Type: Read Binary File  
  - Role: Reads hashtag text file at `ImgPath + '/Caption/hashtag.txt'`.  
  - Config: Reads `.txt` with data stored as `hashtag`.  
  - Input: From ImgPath  
  - Output: To Extract from File1  
  - Edge Cases: Missing or empty hashtag file.

---

#### 2.2 Caption and Hashtag Processing

**Overview:**  
Extracts text from binary files and randomly picks one line from description and hashtag files for use in posts.

**Nodes Involved:**  
- Extract from File  
- Code  
- Extract from File1  
- Merge  

**Node Details:**  

- **Extract from File**  
  - Type: Extract From File  
  - Role: Converts binary description file content into text for processing.  
  - Config: Operation `text`, output key `description_random`, input binary `description`.  
  - Input: From Description1  
  - Output: To Code  

- **Code**  
  - Type: Code  
  - Role: Parses the description text lines and randomly selects one non-empty line.  
  - Config: JavaScript code splits by newline and filters empty lines, returns one random line as `description`.  
  - Input: From Extract from File  
  - Output: To Merge  

- **Extract from File1**  
  - Type: Extract From File  
  - Role: Extracts text from binary hashtag file.  
  - Config: Similar to description extraction, outputs `hashtag_random`.  
  - Input: From Hastag  
  - Output: To Merge  

- **Merge**  
  - Type: Merge  
  - Role: Combines the single random description and hashtag into one data stream.  
  - Config: Mode `combine` by position (column-wise merge).  
  - Input: From Code and Extract from File1  
  - Output: To Merge1  

---

#### 2.3 Image Upload and Facebook Photo Upload

**Overview:**  
Uploads image files to Cloudinary, then posts them as unpublished photos on Facebook, preparing for multi-photo post creation.

**Nodes Involved:**  
- Merge1  
- Facebook Graph API - Photo  
- Wait1  

**Node Details:**  

- **Merge1**  
  - Type: Merge  
  - Role: Combines image binary data and caption/hashtag text data.  
  - Config: Mode `combine` by position.  
  - Input: From Image and Merge  
  - Output: To Facebook Graph API - Photo  

- **Facebook Graph API - Photo**  
  - Type: Facebook Graph API  
  - Role: Uploads a photo to Facebook Page as published photo with message.  
  - Config: Edge `photos`, node `me`, HTTP POST, binary data sent in property `data`. Message combines description and hashtag with newline separation.  
  - Credentials: Facebook Graph API PageFB credential.  
  - Input: From Merge1  
  - Output: To Wait1  
  - Edge Cases: Authentication failure, API rate limits, binary data issues.

- **Wait1**  
  - Type: Wait  
  - Role: Pauses workflow for 10 seconds after photo upload, likely to avoid throttling.  
  - Config: Fixed 10 seconds.  
  - Input: From Facebook Graph API - Photo  
  - Output: To Move image to Posted date folder  

---

#### 2.4 Facebook Multi-Photo Post Creation

**Overview:**  
Uploads images to Cloudinary, creates unpublished Facebook photos, aggregates photo IDs, and publishes a multi-photo post with the combined message.

**Nodes Involved:**  
- ImgPath2  
- Image3  
- Description2  
- Hastag2  
- Extract from File4  
- Code2  
- Extract from File5  
- Caption  
- upload to file server1  
- HTTP Request  
- Aggregate1  
- PreparePost  
- Multi-Photo1  
- Wait2  
- GET  
- If  
- Loop Over Items  
- Move image to Posted date folder2  
- MoveFile  
- ImagePath  

**Node Details:**  

- **ImgPath2**  
  - Type: Set  
  - Role: Sets image path (`E:\\Autopost-media\\PageFB\\202506-Images\\`) and Facebook Page ID (`694076880451435`).  
  - Output: To Image3, Description2, Hastag2  

- **Image3**  
  - Type: Read Binary File  
  - Role: Reads all image files from ImgPath2 directory.  
  - Output: To Limit1  

- **Limit1**  
  - Type: Limit  
  - Role: Restricts to maximum 3 images per batch (Cloudinary upload limit).  
  - Output: To upload to file server1 and MoveFile  

- **upload to file server1**  
  - Type: HTTP Request  
  - Role: Uploads images to Cloudinary using multipart/form-data with preset `srk_uploads`.  
  - Config: POST to Cloudinary API URL, binary field `data`.  
  - Output: To HTTP Request (Facebook photo upload)  

- **HTTP Request**  
  - Type: HTTP Request  
  - Role: Posts the image URL from Cloudinary to Facebook photos API as unpublished photo (`published=false`).  
  - Output: To Aggregate1  

- **Aggregate1**  
  - Type: Aggregate  
  - Role: Aggregates uploaded photo Facebook IDs into an array (field `id`).  
  - Output: To PreparePost  

- **PreparePost**  
  - Type: Merge  
  - Role: Combines aggregated photo IDs and caption/hashtag content.  
  - Output: To Multi-Photo1  

- **Multi-Photo1**  
  - Type: Facebook Graph API  
  - Role: Creates a Facebook multi-photo post on Page feed with message and attached media from aggregated photo IDs.  
  - Config: Edge `feed`, node Page ID, message from description + hashtags, attached_media is JSON array of media_fbid objects.  
  - Output: To Wait2  

- **Wait2**  
  - Type: Wait  
  - Role: Waits 10 seconds before querying Facebook feed for confirmation.  
  - Output: To GET  

- **GET**  
  - Type: Facebook Graph API  
  - Role: Retrieves latest feed posts from Facebook Page (Page ID from ImgPath2).  
  - Output: To If  

- **If**  
  - Type: If  
  - Role: Checks if the last feed item has a valid `id` to confirm successful post.  
  - Config: Checks if `id` is not empty.  
  - True Output: To MoveFile (for post-processing)  
  - False Output: (No connection, stops workflow)  

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes each image post sequentially to move files after posting.  
  - Output: True branch to Move image to Posted date folder2, False branch empty  

- **Move image to Posted date folder2**  
  - Type: Execute Command  
  - Role: Moves posted image file(s) to a dated subfolder in the source directory using PowerShell script.  
  - Script dynamically creates dated folder and moves files.  
  - Input: From Loop Over Items  
  - Output: To Loop Over Items (allows batch processing loop)  
  - Edge Cases: File not found, permission errors.

- **MoveFile**  
  - Type: Merge  
  - Role: Combines data to set full file path of media for moving.  
  - Output: To ImagePath  

- **ImagePath**  
  - Type: Set  
  - Role: Constructs full image file path string from base directory and filename.  
  - Output: To Loop Over Items  

- **Description2, Hastag2, Extract from File4, Extract from File5, Code2, Caption**  
  - Parallel to block 2.2, these nodes extract and randomize captions and hashtags for the multi-photo post.  

---

#### 2.5 Post-Processing and File Management

**Overview:**  
After successful posting, moves posted images to date-stamped folders to archive them and avoid reposting.

**Nodes Involved:**  
- Move image to Posted date folder  
- Move image to Posted date folder2  

**Node Details:**  

- **Move image to Posted date folder**  
  - Type: Execute Command  
  - Role: Moves a single image to a dated folder using PowerShell.  
  - Input: From Wait1  
  - Command creates folder for current date and moves the posted image.  
  - Edge Cases: Folder creation fails, move fails if file locked.

- **Move image to Posted date folder2**  
  - As described in block 2.4, moves multiple images in batch mode.

---

#### 2.6 Scheduling and Loop Control

**Overview:**  
Schedules workflow trigger and manages batch processing and looping.

**Nodes Involved:**  
- Schedule Trigger  
- Wait1  
- Wait2  
- Loop Over Items  

**Node Details:**  

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Starts the workflow every 2 hours at minute 10.  
  - Config: Interval hours = 2, trigger at minute 10.  
  - Output: Starts ImgPath2 block (multi-photo post flow).  

- **Wait1, Wait2**  
  - Used to insert deliberate delays to avoid API throttling or timing issues.

- **Loop Over Items**  
  - Controls batch processing of multiple images sequentially with the ability to move files after posting.

---

### 3. Summary Table

| Node Name                      | Node Type               | Functional Role                              | Input Node(s)                          | Output Node(s)                            | Sticky Note                                                  |
|--------------------------------|-------------------------|----------------------------------------------|--------------------------------------|-------------------------------------------|--------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger        | Triggers workflow every 2 hours              |                                      | ImgPath2                                  |                                                              |
| ImgPath2                      | Set                     | Defines base image path and Facebook Page ID | Schedule Trigger                    | Image3, Description2, Hastag2              |                                                              |
| Image3                        | Read Binary File        | Reads image files from ImgPath2               | ImgPath2                            | Limit1                                    |                                                              |
| Limit1                        | Limit                   | Limits number of images to 3                   | Image3                             | upload to file server1, MoveFile           |                                                              |
| upload to file server1         | HTTP Request            | Uploads images to Cloudinary                   | Limit1                             | HTTP Request                              |                                                              |
| HTTP Request                  | HTTP Request            | Uploads image URL as unpublished Facebook photo | upload to file server1           | Aggregate1                                |                                                              |
| Aggregate1                   | Aggregate               | Aggregates photo IDs from Facebook            | HTTP Request                      | PreparePost                               |                                                              |
| PreparePost                  | Merge                   | Combines photo IDs and captions                | Aggregate1, Caption               | Multi-Photo1                              |                                                              |
| Multi-Photo1                 | Facebook Graph API      | Posts multi-photo Facebook post                 | PreparePost                      | Wait2                                     |                                                              |
| Wait2                        | Wait                    | Waits 10 seconds after posting                  | Multi-Photo1                     | GET                                       |                                                              |
| GET                          | Facebook Graph API      | Retrieves Facebook feed                          | Wait2                            | If                                        |                                                              |
| If                           | If                      | Checks if Facebook post succeeded                | GET                             | MoveFile                                   |                                                              |
| MoveFile                     | Merge                   | Prepares file path for moving post images       | If                              | ImagePath                                  |                                                              |
| ImagePath                    | Set                     | Sets full path of image file                      | MoveFile                        | Loop Over Items                            |                                                              |
| Loop Over Items              | SplitInBatches          | Processes images sequentially                    | ImagePath                       | Move image to Posted date folder2          |                                                              |
| Move image to Posted date folder2 | Execute Command     | Moves posted images to date folder               | Loop Over Items                 | Loop Over Items                            |                                                              |
| ImgPath                      | Set                     | Defines base image path                           |                                  | Image, Description1, Hastag                |                                                              |
| Image                       | Read Binary File        | Reads image files from ImgPath                    | ImgPath                         | Merge1                                     |                                                              |
| Description1                | Read Binary File        | Reads description text file                        | ImgPath                         | Extract from File                          |                                                              |
| Hastag                      | Read Binary File        | Reads hashtag text file                            | ImgPath                         | Extract from File1                         |                                                              |
| Extract from File           | Extract From File       | Extracts text from description file               | Description1                   | Code                                       |                                                              |
| Code                       | Code                    | Selects random description line                    | Extract from File              | Merge                                      |                                                              |
| Extract from File1          | Extract From File       | Extracts text from hashtag file                     | Hastag                         | Merge                                      |                                                              |
| Merge                      | Merge                   | Combines description and hashtag                   | Code, Extract from File1       | Merge1                                     |                                                              |
| Merge1                     | Merge                   | Combines image and caption data                     | Image, Merge                  | Facebook Graph API - Photo                  |                                                              |
| Facebook Graph API - Photo  | Facebook Graph API      | Uploads photo to Facebook with caption             | Merge1                        | Wait1                                      |                                                              |
| Wait1                      | Wait                    | Waits 10 seconds after photo upload                 | Facebook Graph API - Photo    | Move image to Posted date folder            |                                                              |
| Move image to Posted date folder | Execute Command     | Moves single posted image to dated folder           | Wait1                        |                                           |                                                              |
| Description2                | Read Binary File        | Reads description text file for multi-photo posts  | ImgPath2                      | Extract from File4                          |                                                              |
| Hastag2                    | Read Binary File        | Reads hashtag text file for multi-photo posts        | ImgPath2                      | Extract from File5                          |                                                              |
| Extract from File4          | Extract From File       | Extracts text from description file                   | Description2                  | Code2                                      |                                                              |
| Code2                      | Code                    | Selects random description line                        | Extract from File4            | Caption                                    |                                                              |
| Extract from File5          | Extract From File       | Extracts text from hashtag file                         | Hastag2                      | Caption                                    |                                                              |
| Caption                    | Merge                   | Combines description and hashtag for multi-photo post | Code2, Extract from File5     | PreparePost                                |                                                              |
| Sticky Note                 | Sticky Note             | Label: "# A Image Facebook post"                       |                               |                                            | Covers initial image Facebook post nodes                     |
| Sticky Note1                | Sticky Note             | Label: "# Multi-Photo FB Post\n- Cloudunary\n- Limit 3 Images/post" |                               |                                            | Covers multi-photo post logic and Cloudinary integration      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set interval to every 2 hours, trigger at minute 10.

2. **Create Set node "ImgPath2"**  
   - Assign two variables:  
     - `ImgPath = "E:\\Autopost-media\\PageFB\\202506-Images\\"` (note double backslashes)  
     - `PageFBFB = "694076880451435"` (Facebook Page ID)  
   - Connect Schedule Trigger → ImgPath2.

3. **Create Read Binary File node "Image3"**  
   - File Selector: `={{ $json.ImgPath + '**.***' }}` to read all images recursively.  
   - Connect ImgPath2 → Image3.

4. **Create Read Binary File nodes "Description2" and "Hastag2"**  
   - Description2 file selector: `={{ $json.ImgPath + 'Caption/description.txt' }}`  
   - Hastag2 file selector: `={{ $json.ImgPath + 'Caption\\hashtag.txt' }}` (escaped backslashes)  
   - Connect ImgPath2 → Description2 and Hastag2.

5. **Create Extract From File nodes "Extract from File4" and "Extract from File5"**  
   - Extract description text from Description2 (operation: text, output key: `description_random`, binary field `description`).  
   - Extract hashtag text from Hastag2 similarly (output key: `hashtag_random`, binary field `hashtag`).  
   - Connect Description2 → Extract from File4; Hastag2 → Extract from File5.

6. **Create Code node "Code2"**  
   - JavaScript code to split description lines and pick a random non-empty line:  
     ```js
     const lines = $input.first().json.description_random.split(/\r?\n/).filter(Boolean);
     const random = lines[Math.floor(Math.random() * lines.length)];
     return [{ json: { description: random } }];
     ```  
   - Connect Extract from File4 → Code2.

7. **Create Merge node "Caption"**  
   - Mode: Combine by position  
   - Connect Code2 and Extract from File5 → Caption.

8. **Create Limit node "Limit1"**  
   - Max Items: 3 (for max 3 images per batch).

9. **Connect Image3 → Limit1**

10. **Create HTTP Request node "upload to file server1"**  
    - URL: `https://api.cloudinary.com/v1_1/dxepor8fh/image/upload`  
    - Method: POST  
    - Content-Type: multipart-form-data  
    - Body parameters:  
      - `upload_preset` = "srk_uploads"  
      - `file` = binary data field `data`  
    - Connect Limit1 → upload to file server1.

11. **Create HTTP Request node "HTTP Request"**  
    - URL: `https://graph.facebook.com/v23.0/me/photos`  
    - Method: POST  
    - Content-Type: multipart-form-data  
    - Body parameters:  
      - `url` = `={{ $json.url }}` (Cloudinary URL)  
      - `published` = "false" (upload unpublished photos)  
    - Authentication: Facebook Graph API with PageFB credential  
    - Connect upload to file server1 → HTTP Request.

12. **Create Aggregate node "Aggregate1"**  
    - Aggregate field: `id` (collects photo IDs)  
    - Connect HTTP Request → Aggregate1.

13. **Create Merge node "PreparePost"**  
    - Mode: Combine by position  
    - Connect Aggregate1 and Caption → PreparePost.

14. **Create Facebook Graph API node "Multi-Photo1"**  
    - Edge: `feed`  
    - Node: `={{ $('ImgPath2').item.json.PageFBFB }}`  
    - HTTP Method: POST  
    - Parameters:  
      - `message` = `={{ $json.description + "\n\n" + $json.hashtag_random }}`  
      - `attached_media` = `={{ JSON.stringify($json.id.map(media_id => ({ media_fbid: media_id }))) }}`  
    - Credential: Facebook Graph API PageFB  
    - Connect PreparePost → Multi-Photo1.

15. **Create Wait node "Wait2"**  
    - Wait 10 seconds  
    - Connect Multi-Photo1 → Wait2.

16. **Create Facebook Graph API node "GET"**  
    - Edge: `feed`  
    - Node: `={{ $('ImgPath2').item.json.PageFBFB }}`  
    - Method: GET  
    - Connect Wait2 → GET.

17. **Create If node "If"**  
    - Condition: Check if `id` from GET response is not empty  
    - Connect GET → If.

18. **Create Merge node "MoveFile"**  
    - Connect If (true) → MoveFile.

19. **Create Set node "ImagePath"**  
    - Assign `mediafile` = `={{ $('ImgPath2').item.json.ImgPath +  $json.fileName}}`  
    - Connect MoveFile → ImagePath.

20. **Create SplitInBatches node "Loop Over Items"**  
    - Connect ImagePath → Loop Over Items.

21. **Create Execute Command node "Move image to Posted date folder2"**  
    - Powershell command:  
      ```powershell
      $d = Get-Date -Format 'yyyyMMdd'; 
      $p = '{{ $('ImgPath2').item.json.ImgPath }}' + $d; 
      New-Item -ItemType Directory -Path $p -Force; 
      $files = @('{{ $json.mediafile }}'); 
      foreach ($file in $files) { 
        if (Test-Path $file) { 
          Move-Item -Path $file -Destination $p -Force; Write-Host "Moved: $file" 
        } else { 
          Write-Host "File not found: $file" 
        } 
      }
      ```  
    - Connect Loop Over Items (true) → Move image to Posted date folder2 → Loop Over Items (loop).

22. **Build the initial single photo posting branch similarly:**  
    - Create Set node "ImgPath" with path `"E:/Autopost-media/PageFB/202506-Images"`  
    - Read image files, description, and hashtags as in steps 3-7 but for single files.  
    - Combine and upload photos to Facebook as published photos.  
    - Wait, then move posted images similarly.

23. **Add waits (Wait1, Wait2) to mitigate API throttling.**

24. **Add sticky notes to label major workflow sections for clarity.**

25. **Configure Facebook Graph API credentials with proper Page tokens and OAuth2.**

26. **Test workflow thoroughly with sample images and captions to handle edge cases like missing files or API errors.**

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow designed for automated multi-photo Facebook posts with Cloudinary as image host.                        | Workflow Description                                                                                |
| To avoid hitting API rate limits, waits of 10 seconds are added after photo uploads and posts.                   | Wait1 and Wait2 nodes                                                                               |
| Cloudinary upload preset used: `srk_uploads`. Ensure this preset exists in your Cloudinary account.             | Cloudinary API upload configuration                                                                |
| Facebook Graph API version used: v22.0 and v23.0 for photo upload and feed posting.                              | Facebook Graph API nodes                                                                            |
| PowerShell scripts used for moving files require Windows environment with appropriate permissions.               | Execute Command nodes                                                                               |
| Facebook Page ID `694076880451435` is hardcoded; replace with your own Page ID when replicating the workflow.   | ImgPath2 Set node                                                                                   |
| For multi-photo posts, max 3 images are uploaded per post due to Facebook/Cloudinary limits.                     | Limit1 node                                                                                        |
| Documentation and node details useful for debugging auth errors, file access errors, and API limits.             | General troubleshooting                                                                              |

---

This document provides a comprehensive insight into the Scheduled Multi-Photo Facebook Posts workflow with Cloudinary integration, enabling reproduction, modification, and troubleshooting by both technical users and automation agents.