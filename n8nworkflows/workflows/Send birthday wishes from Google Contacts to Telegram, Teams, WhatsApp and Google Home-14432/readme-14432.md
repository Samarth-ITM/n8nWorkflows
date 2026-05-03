Send birthday wishes from Google Contacts to Telegram, Teams, WhatsApp and Google Home

https://n8nworkflows.xyz/workflows/send-birthday-wishes-from-google-contacts-to-telegram--teams--whatsapp-and-google-home-14432


# Send birthday wishes from Google Contacts to Telegram, Teams, WhatsApp and Google Home

# 1. Workflow Overview

This workflow automates daily birthday detection from Google Contacts and sends birthday wishes across several channels: Telegram, Microsoft Teams, Google Home via Home Assistant, and WhatsApp via Rapiwa. It runs every day at 10:00 AM, inspects all Google Contacts, determines whether each contact has a birthday matching the current date, and then either sends birthday messages or skips to the next contact. It also includes a “no birthday” Telegram notification path, although its current placement causes it to trigger per non-matching contact rather than once globally.

## 1.1 Scheduling and Initialization
The workflow starts on a daily schedule and creates a date value used in later checks.

## 1.2 Contact Retrieval and Iteration
It loads all Google Contacts with birthday and name data, then processes them one by one using a batch loop.

## 1.3 Birthday Presence and Date Matching
For each contact, it first checks whether a birthday field exists, then runs custom JavaScript to compare the contact’s birthday month/day against today’s date.

## 1.4 Birthday Notification Delivery
If the current contact’s birthday is today, the workflow sends greetings through Telegram, Teams, Google Home, and then fetches the contact again to obtain phone numbers for WhatsApp delivery.

## 1.5 WhatsApp Verification and Sending
The contact’s phone numbers are cleaned, verified through Rapiwa, and then used to send a WhatsApp birthday message if the number is considered valid by the verification branch.

## 1.6 Non-Birthday Path
If a contact has no birthday, or the birthday is not today, the workflow follows fallback branches. One branch does nothing and continues looping; another sends a Telegram message stating that today is not anyone’s birthday, but this is implemented at item level rather than as a single summary notification.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduling and Initialization

**Overview:**  
This block triggers the workflow every day at 10:00 AM and prepares a date field. That date field is referenced in a later existence check, although the logic does not actually require adding one day for the birthday comparison code.

**Nodes Involved:**  
- Start Everyday at 10AM
- Add 1 Day With Currant Date

### Node: Start Everyday at 10AM
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger` — entry-point trigger node.
- **Configuration choices:** Configured to trigger daily at hour 10.
- **Key expressions or variables used:** None.
- **Input and output connections:** Entry node → outputs to **Add 1 Day With Currant Date**.
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**  
  - Time zone behavior depends on the n8n instance timezone.  
  - If the server timezone differs from the expected user timezone, birthdays may be checked at the wrong local time.
- **Sub-workflow reference:** None.

### Node: Add 1 Day With Currant Date
- **Type and technical role:** `n8n-nodes-base.set` — assigns a computed date field.
- **Configuration choices:** Creates a string field named `date` with value `{{$today.plus({days:1})}}`.
- **Key expressions or variables used:**  
  - `$today.plus({days:1})`
- **Input and output connections:** Input from **Start Everyday at 10AM** → output to **Google Contacts (Get All Contact)**.
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**  
  - The node name suggests “current” date but actually adds one day.  
  - This date is only used in a later birthday existence check, not in the actual birthday match code.  
  - Can introduce confusion for maintainers because the real birthday comparison uses JavaScript `new Date()`, not this field.
- **Sub-workflow reference:** None.

---

## 2.2 Contact Retrieval and Iteration

**Overview:**  
This block retrieves all contacts from Google Contacts and processes them one by one. It is the main item-processing backbone of the workflow.

**Nodes Involved:**  
- Google Contacts (Get All Contact)
- Loop Over Items

### Node: Google Contacts (Get All Contact)
- **Type and technical role:** `n8n-nodes-base.googleContacts` — fetches contacts from Google Contacts.
- **Configuration choices:**  
  - Operation: `getAll`  
  - Return all: enabled  
  - Requested fields: `birthdays`, `names`
- **Key expressions or variables used:** None in parameters.
- **Input and output connections:** Input from **Add 1 Day With Currant Date** → output to **Loop Over Items**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**  
  - OAuth credential expiration or missing consent scopes.  
  - Some contacts may not have `birthdays` or `names`.  
  - Large contact lists may increase execution time.
- **Sub-workflow reference:** None.

### Node: Loop Over Items
- **Type and technical role:** `n8n-nodes-base.splitInBatches` — loops through items sequentially.
- **Configuration choices:** Uses default options; acts as an iterator.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input from **Google Contacts (Get All Contact)**  
  - Receives loop-back connections from **No Birthday**, **Sent Telegram Message**, **Rapiwa (Send Wish Message On WhatsApp**, **Sent Telegram Message (not anyone's birthday today)**  
  - Secondary output goes to **If (Check Is there a birthday?)**
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**  
  - If miswired, loops can terminate early or create unintended repeated notifications.  
  - Since multiple branches loop back into it, debugging item state can be harder.
- **Sub-workflow reference:** None.

---

## 2.3 Birthday Presence and Date Matching

**Overview:**  
This block checks whether the current contact has a birthday field and, if so, compares the contact birthday’s day and month against today. It uses a custom JavaScript node to normalize the date format.

**Nodes Involved:**  
- If (Check Is there a birthday?)
- Code (Match Birthday 'Date and Month')
- If Check (Today Birthday?)
- No Birthday

### Node: If (Check Is there a birthday?)
- **Type and technical role:** `n8n-nodes-base.if` — checks whether a birthday value exists.
- **Configuration choices:**  
  - Condition type: `dateTime`
  - Operation: `exists`
  - Left value: `={{ $('Google Contacts (Get All Contact)').item.json.birthdays }}`
  - Right value: references month from **Add 1 Day With Currant Date**, but for an `exists` operator this right side is not meaningfully needed.
- **Key expressions or variables used:**  
  - `$('Google Contacts (Get All Contact)').item.json.birthdays`
  - `$('Add 1 Day With Currant Date').item.json.date.toDateTime().month`
- **Input and output connections:**  
  - Input from **Loop Over Items**  
  - True → **Code (Match Birthday 'Date and Month')**  
  - False → **No Birthday**
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**  
  - If `birthdays` is not in a standard date-like format, the dateTime existence validation may behave unexpectedly.  
  - The condition is more complex than necessary; a simple existence/string check would be clearer.
- **Sub-workflow reference:** None.

### Node: No Birthday
- **Type and technical role:** `n8n-nodes-base.noOp` — does nothing and passes control back to the loop.
- **Configuration choices:** No parameters.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from **If (Check Is there a birthday?)** false branch → output back to **Loop Over Items**.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**  
  - No technical failure expected.  
  - Functionally, it silently skips contacts without birthdays.
- **Sub-workflow reference:** None.

### Node: Code (Match Birthday 'Date and Month')
- **Type and technical role:** `n8n-nodes-base.code` — custom JavaScript date comparison.
- **Configuration choices:**  
  - Uses JavaScript `new Date()` for the current day.  
  - Converts today into `dd/MM`.  
  - Reads each contact birthday as string in presumed `mm/dd/yyyy` format.  
  - Converts birthday to `dd/MM`.  
  - Outputs `birthdayToday: "YES"` or `"NO"`.
- **Key expressions or variables used:**  
  - `$input.all()`
  - `item.json.birthdays`
- **Input and output connections:** Input from **If (Check Is there a birthday?)** true branch → output to **If Check (Today Birthday?)**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Assumes birthday format is exactly `mm/dd/yyyy`. If Google Contacts returns another structure or locale-dependent format, parsing will fail or miscompare.  
  - Missing or malformed birthday strings produce fallback `{ birthdayToday: "No" }`, which differs in case from `"NO"`.  
  - It discards original contact fields except the new flag, which forces later nodes to rely on `$('...').item` cross-node references.
  - Uses server current date, which may differ from expected timezone.
- **Sub-workflow reference:** None.

### Node: If Check (Today Birthday?)
- **Type and technical role:** `n8n-nodes-base.if` — checks whether the code node marked the birthday as today.
- **Configuration choices:**  
  - String equals condition  
  - Left value: `={{ $json.birthdayToday }}`
  - Right value: `YES`
- **Key expressions or variables used:**  
  - `$json.birthdayToday`
- **Input and output connections:**  
  - Input from **Code (Match Birthday 'Date and Month')**  
  - True → **Sent Telegram Message**, **Send to Google Home Speaker**, **Create chat message**, **Get a contact ID to contact Number**  
  - False → **Sent Telegram Message (not anyone's birthday today)**
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**  
  - Any variation in output casing (`No`, `NO`, `YES`) affects branching.  
  - The false branch triggers per contact, not as a single daily summary.
- **Sub-workflow reference:** None.

---

## 2.4 Birthday Notification Delivery

**Overview:**  
When a birthday match is found, this block sends personalized birthday messages to Telegram, Teams, and Google Home. It also retrieves the full contact record to obtain phone numbers for WhatsApp processing.

**Nodes Involved:**  
- Sent Telegram Message
- Send to Google Home Speaker
- Create chat message
- Get a contact ID to contact Number

### Node: Sent Telegram Message
- **Type and technical role:** `n8n-nodes-base.telegram` — sends a Telegram message.
- **Configuration choices:**  
  - Sends formatted text to a specific chat ID.  
  - Attribution disabled.
- **Key expressions or variables used:**  
  - `$('If Check (Today Birthday?)').item.json.formattedDate.toDateTime().setLocale('bd').format('dd MMMM')`
  - `$('Get a contact ID to contact Number').item.json.names.displayName`
- **Input and output connections:**  
  - Input from **If Check (Today Birthday?)** true branch  
  - Output back to **Loop Over Items**
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**  
  - The expression references `formattedDate`, but no node creates this field. This can fail at runtime.  
  - It also references **Get a contact ID to contact Number**, which executes in parallel from the same IF node. Depending on item linking, this may be unreliable.  
  - Invalid Telegram credentials or chat ID will cause API errors.
- **Sub-workflow reference:** None.

### Node: Send to Google Home Speaker
- **Type and technical role:** `n8n-nodes-base.homeAssistant` — calls Home Assistant TTS service.
- **Configuration choices:**  
  - Domain: `tts`  
  - Service: `speak`  
  - Service attributes include:
    - `entity_id = tts.google_translate_bn_bn`
    - `message = dynamic birthday text`
    - `media_player_entity_id = media_player.google_cuisine`
- **Key expressions or variables used:**  
  - `$('Get a contact ID to contact Number').item.json.names.displayName`
  - `$('If Check (Today Birthday?)').item.json.formattedDate.toDateTime().setLocale('bd').format('dd MMMM')`
- **Input and output connections:** Input from **If Check (Today Birthday?)** true branch.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**  
  - Requires a working Home Assistant credential and accessible TTS/media player entities.  
  - Same `formattedDate` issue as above.  
  - `bd` locale may not be available depending on environment/date library support.
- **Sub-workflow reference:** None.

### Node: Create chat message
- **Type and technical role:** `n8n-nodes-base.microsoftTeams` — posts a message to a Teams chat.
- **Configuration choices:**  
  - Resource: `chatMessage`  
  - Target chat ID set manually  
  - Dynamic message text
- **Key expressions or variables used:**  
  - `$('Get a contact ID to contact Number').item.json.names.displayName`
  - `$('If Check (Today Birthday?)').item.json.formattedDate.toDateTime().setLocale('bd').format('dd MMMM')`
- **Input and output connections:** Input from **If Check (Today Birthday?)** true branch.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - OAuth token expiry or insufficient Teams permissions.  
  - Invalid chat ID.  
  - Same missing `formattedDate` field issue.
- **Sub-workflow reference:** None.

### Node: Get a contact ID to contact Number
- **Type and technical role:** `n8n-nodes-base.googleContacts` — fetches one contact by ID with phone numbers.
- **Configuration choices:**  
  - Operation: `get`
  - Contact ID: `={{ $('Loop Over Items').item.json.contactId }}`
  - Requested fields: `phoneNumbers`, `names`
- **Key expressions or variables used:**  
  - `$('Loop Over Items').item.json.contactId`
- **Input and output connections:**  
  - Input from **If Check (Today Birthday?)** true branch  
  - Output to **Code (Get & Clean Phone Number)**
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**  
  - Assumes `contactId` is available on loop items from the first Google Contacts node.  
  - If contact ID is absent or permissions are insufficient, the lookup fails.  
  - Contacts without phone numbers will affect downstream code.
- **Sub-workflow reference:** None.

---

## 2.5 WhatsApp Verification and Sending

**Overview:**  
This block transforms contact phone data, verifies WhatsApp availability through Rapiwa, and sends a WhatsApp birthday message. Its IF logic appears inverted or overly permissive because it checks whether `exists` itself exists, not whether it is true.

**Nodes Involved:**  
- Code (Get & Clean Phone Number)
- Rapiwa (Verify WhatsApp Exists or Not)
- If (Check WhatsApp Number "exists or not")
- Rapiwa (Send Wish Message On WhatsApp

### Node: Code (Get & Clean Phone Number)
- **Type and technical role:** `n8n-nodes-base.code` — extracts and normalizes phone numbers.
- **Configuration choices:**  
  - Reads the first incoming item.  
  - Safely accesses `phoneNumbers`.  
  - Converts values to strings and removes non-digit characters.  
  - Returns `phoneNumbers: cleanedNumbers`.
- **Key expressions or variables used:**  
  - `$input.first().json`
  - `data?.phoneNumbers`
- **Input and output connections:** Input from **Get a contact ID to contact Number** → output to **Rapiwa (Verify WhatsApp Exists or Not)**.
- **Version-specific requirements:** Type version `2`.
- **Edge cases or potential failure types:**  
  - Returns an array under `phoneNumbers`, but downstream nodes may expect a single string number.  
  - If multiple phone numbers exist, behavior depends on how Rapiwa handles arrays.  
  - If no phone numbers exist, returns an empty array.
- **Sub-workflow reference:** None.

### Node: Rapiwa (Verify WhatsApp Exists or Not)
- **Type and technical role:** `n8n-nodes-rapiwa.rapiwa` — custom/community node to verify WhatsApp registration.
- **Configuration choices:**  
  - Operation: `verifyWhatsAppNumber`
  - Number: `={{ $json.phoneNumbers }}`
- **Key expressions or variables used:**  
  - `$json.phoneNumbers`
- **Input and output connections:** Input from **Code (Get & Clean Phone Number)** → output to **If (Check WhatsApp Number "exists or not")**.
- **Version-specific requirements:** Type version `1`; requires the Rapiwa community node installed.
- **Edge cases or potential failure types:**  
  - External API auth issues.  
  - Array input may not match API expectations.  
  - Verification API limits, network failures, or unsupported number formats.
- **Sub-workflow reference:** None.

### Node: If (Check WhatsApp Number "exists or not")
- **Type and technical role:** `n8n-nodes-base.if` — branches based on Rapiwa verification output.
- **Configuration choices:**  
  - Boolean `exists` operator on left value `={{ $json.exists }}`
  - Right value `false`
- **Key expressions or variables used:**  
  - `$json.exists`
- **Input and output connections:**  
  - Input from **Rapiwa (Verify WhatsApp Exists or Not)**  
  - True → **Rapiwa (Send Wish Message On WhatsApp**  
  - False → **Loop Over Items**
- **Version-specific requirements:** Type version `2.3`.
- **Edge cases or potential failure types:**  
  - The logic is suspicious: “boolean exists” checks whether the field exists, not whether it equals true. This may route almost all valid responses into the true branch.  
  - If the API returns a different schema, the branch may misfire.
- **Sub-workflow reference:** None.

### Node: Rapiwa (Send Wish Message On WhatsApp
- **Type and technical role:** `n8n-nodes-rapiwa.rapiwa` — sends WhatsApp message via Rapiwa.
- **Configuration choices:**  
  - Number: `={{ $json.number }}`
  - Message: dynamic birthday message
- **Key expressions or variables used:**  
  - `$json.number`
  - `$('Get a contact ID to contact Number').item.json.names.displayName`
  - `$('If (Check Is there a birthday?)').item.json.birthdays.toDateTime().setLocale('bd').format('dd MMMM')`
- **Input and output connections:**  
  - Input from **If (Check WhatsApp Number "exists or not")** true branch  
  - Output back to **Loop Over Items**
- **Version-specific requirements:** Type version `1`; requires installed Rapiwa node.
- **Edge cases or potential failure types:**  
  - The message references `$('If (Check Is there a birthday?)').item.json.birthdays`, but that IF node does not output a `birthdays` field directly in its own JSON. This expression may fail.  
  - The node uses `$json.number`, but upstream verification used `$json.phoneNumbers`; unless Rapiwa returns `number`, this may break.  
  - WhatsApp sending depends on external provider reliability and number validity.
- **Sub-workflow reference:** None.

---

## 2.6 Non-Birthday Notification Path

**Overview:**  
This block sends a Telegram message when the birthday match check is false. However, because it is connected directly from the per-contact birthday IF node, it will execute once for every contact whose birthday is not today, not once per day.

**Nodes Involved:**  
- Sent Telegram Message (not anyone's birthday today)

### Node: Sent Telegram Message (not anyone's birthday today)
- **Type and technical role:** `n8n-nodes-base.telegram` — sends Telegram notification.
- **Configuration choices:**  
  - Sends a message to a configured chat ID.  
  - Attribution disabled.
- **Key expressions or variables used:**  
  - `$('If Check (Today Birthday?)').item.json.formattedDate.toDateTime().setLocale('bd').format('dd MMMM')`
- **Input and output connections:**  
  - Input from **If Check (Today Birthday?)** false branch  
  - Output back to **Loop Over Items**
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**  
  - References missing `formattedDate` field.  
  - Functionally noisy: sends “Not anyone's birthday today” repeatedly for all non-matching contacts.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Start Everyday at 10AM | Schedule Trigger | Daily workflow entry point |  | Add 1 Day With Currant Date | ## Scheduling and Initialization Group |
| Add 1 Day With Currant Date | Set | Creates date field for downstream checks | Start Everyday at 10AM | Google Contacts (Get All Contact) | ## Scheduling and Initialization Group |
| Google Contacts (Get All Contact) | Google Contacts | Fetches all contacts with birthdays and names | Add 1 Day With Currant Date | Loop Over Items | ## Contact Retrieval Group |
| Loop Over Items | Split In Batches | Iterates through contacts one by one | Google Contacts (Get All Contact); No Birthday; Sent Telegram Message; Rapiwa (Send Wish Message On WhatsApp; Sent Telegram Message (not anyone's birthday today) | If (Check Is there a birthday?) | # Birthday Checking Group |
| If (Check Is there a birthday?) | If | Checks whether a birthday field exists | Loop Over Items | Code (Match Birthday 'Date and Month'); No Birthday | # Birthday Checking Group |
| No Birthday | No Operation | Skips contacts without birthdays and continues loop | If (Check Is there a birthday?) | Loop Over Items | # No Birthday Group |
| Code (Match Birthday 'Date and Month') | Code | Compares contact birthday day/month against today | If (Check Is there a birthday?) | If Check (Today Birthday?) | # Birthday Checking Group |
| If Check (Today Birthday?) | If | Branches on whether today matches the birthday | Code (Match Birthday 'Date and Month') | Sent Telegram Message; Send to Google Home Speaker; Create chat message; Get a contact ID to contact Number; Sent Telegram Message (not anyone's birthday today) | # Birthday Checking Group |
| Sent Telegram Message | Telegram | Sends birthday greeting to Telegram | If Check (Today Birthday?) | Loop Over Items | # Birthday Wishing Group |
| Send to Google Home Speaker | Home Assistant | Plays birthday greeting over Google Home via TTS | If Check (Today Birthday?) |  | # Birthday Wishing Group |
| Create chat message | Microsoft Teams | Sends birthday greeting into Teams chat | If Check (Today Birthday?) |  | # Birthday Wishing Group |
| Get a contact ID to contact Number | Google Contacts | Retrieves specific contact phone numbers and names | If Check (Today Birthday?) | Code (Get & Clean Phone Number) | # Birthday Wishing Group |
| Code (Get & Clean Phone Number) | Code | Extracts and sanitizes phone numbers | Get a contact ID to contact Number | Rapiwa (Verify WhatsApp Exists or Not) | # Birthday Wishing Group |
| Rapiwa (Verify WhatsApp Exists or Not) | Rapiwa | Verifies whether the phone number is on WhatsApp | Code (Get & Clean Phone Number) | If (Check WhatsApp Number "exists or not") | # Birthday Wishing Group |
| If (Check WhatsApp Number "exists or not") | If | Decides whether to send WhatsApp message | Rapiwa (Verify WhatsApp Exists or Not) | Rapiwa (Send Wish Message On WhatsApp; Loop Over Items | # Birthday Wishing Group |
| Rapiwa (Send Wish Message On WhatsApp | Rapiwa | Sends WhatsApp birthday message | If (Check WhatsApp Number "exists or not") | Loop Over Items | # Birthday Wishing Group |
| Sent Telegram Message (not anyone's birthday today) | Telegram | Sends negative daily status message | If Check (Today Birthday?) | Loop Over Items | # No Birthday Group |
| Sticky Note1 | Sticky Note | Documentation note |  |  |  |
| Sticky Note | Sticky Note | Documentation note |  |  |  |
| Sticky Note2 | Sticky Note | Documentation note |  |  |  |
| Sticky Note3 | Sticky Note | Documentation note |  |  |  |
| Sticky Note4 | Sticky Note | Documentation note |  |  |  |
| Sticky Note5 | Sticky Note | Documentation note |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add a Schedule Trigger node** named **Start Everyday at 10AM**.
   - Type: `Schedule Trigger`
   - Configure it to run daily at hour `10`.
   - Confirm the n8n instance timezone matches your intended local timezone.

3. **Add a Set node** named **Add 1 Day With Currant Date**.
   - Connect it after the trigger.
   - Add one field:
     - Name: `date`
     - Type: `String`
     - Value: `{{$today.plus({days:1})}}`

4. **Add a Google Contacts node** named **Google Contacts (Get All Contact)**.
   - Connect it after the Set node.
   - Operation: `Get All`
   - Return All: enabled
   - Fields: `birthdays`, `names`
   - Configure **Google Contacts OAuth2 credentials** with permission to read contacts.

5. **Add a Split In Batches node** named **Loop Over Items**.
   - Connect it after the Google Contacts node.
   - Keep default settings unless you want a specific batch size.

6. **Add an IF node** named **If (Check Is there a birthday?)**.
   - Connect the second/main iteration output of **Loop Over Items** to this node.
   - Configure a condition that checks whether the current contact has a birthday.
   - To match the supplied workflow:
     - Condition type: `DateTime`
     - Operation: `Exists`
     - Left value: `{{ $('Google Contacts (Get All Contact)').item.json.birthdays }}`
   - Recommended improvement if rebuilding cleanly: use a simpler existence check directly on the current item.

7. **Add a No Operation node** named **No Birthday**.
   - Connect the false branch of **If (Check Is there a birthday?)** to it.
   - Connect **No Birthday** back to **Loop Over Items** to continue iteration.

8. **Add a Code node** named **Code (Match Birthday 'Date and Month')**.
   - Connect the true branch of **If (Check Is there a birthday?)** to it.
   - Paste the birthday comparison JavaScript logic:
     - Build today in `dd/MM`
     - Parse the birthday string in `mm/dd/yyyy`
     - Compare day/month only
     - Output `birthdayToday` as `YES` or `NO`
   - If rebuilding robustly, preserve the original contact data in output as well.

9. **Add another IF node** named **If Check (Today Birthday?)**.
   - Connect it after the code node.
   - Configure:
     - Left value: `{{ $json.birthdayToday }}`
     - Operator: `equals`
     - Right value: `YES`

10. **Add a Telegram node** named **Sent Telegram Message**.
    - Connect it from the true branch of **If Check (Today Birthday?)**.
    - Operation: send message.
    - Configure Telegram credentials.
    - Set your chat ID manually.
    - Message content can follow the workflow’s intent:
      - Mention the contact name
      - Mention today’s date formatted in Bengali or your preferred locale
    - Note: the supplied workflow references `formattedDate`, which does not exist. You should replace it with a valid date expression such as `$now`.

11. **Connect Sent Telegram Message back to Loop Over Items**.
    - This allows the workflow to continue processing the remaining contacts.

12. **Add a Microsoft Teams node** named **Create chat message**.
    - Connect it from the true branch of **If Check (Today Birthday?)**.
    - Resource: `chatMessage`
    - Set the chat ID manually.
    - Configure **Microsoft Teams OAuth2 credentials**.
    - Compose the birthday greeting message.
    - Ensure the app/credential has rights to post into the target chat.

13. **Add a Home Assistant node** named **Send to Google Home Speaker**.
    - Connect it from the true branch of **If Check (Today Birthday?)**.
    - Configure:
      - Resource: service call
      - Domain: `tts`
      - Service: `speak`
    - Add attributes:
      - `entity_id = tts.google_translate_bn_bn`
      - `message = your dynamic birthday message`
      - `media_player_entity_id = media_player.google_cuisine`
    - Configure Home Assistant credentials and verify the TTS and media player entities exist.

14. **Add a Google Contacts node** named **Get a contact ID to contact Number**.
    - Connect it from the true branch of **If Check (Today Birthday?)**.
    - Operation: `Get`
    - Contact ID: `{{ $('Loop Over Items').item.json.contactId }}`
    - Fields: `phoneNumbers`, `names`

15. **Add a Code node** named **Code (Get & Clean Phone Number)**.
    - Connect it after the contact-by-ID node.
    - Implement logic to:
      - Read `phoneNumbers`
      - Strip non-digit characters
      - Return cleaned values
    - If rebuilding more robustly, return one item per phone number instead of a single array field.

16. **Add a Rapiwa node** named **Rapiwa (Verify WhatsApp Exists or Not)**.
    - Connect it after the phone cleaning code.
    - Operation: `verifyWhatsAppNumber`
    - Number: map the cleaned phone number field
    - Configure **Rapiwa API credentials**
    - Ensure the community node `n8n-nodes-rapiwa` is installed in your n8n environment.

17. **Add an IF node** named **If (Check WhatsApp Number "exists or not")**.
    - Connect it after the WhatsApp verification node.
    - The original workflow checks whether `$json.exists` exists, which is weak logic.
    - Recommended rebuild:
      - Left value: `{{ $json.exists }}`
      - Operator: `is true`
    - If you want an exact replica, use the same existence-based boolean check.

18. **Add a Rapiwa node** named **Rapiwa (Send Wish Message On WhatsApp**.
    - Connect it from the true branch of the WhatsApp IF node.
    - Configure send operation according to the Rapiwa node capabilities.
    - Number: map to the verified number field returned by Rapiwa, or your cleaned phone number if supported.
    - Message: compose the birthday text with contact name and date.
    - Connect its output back to **Loop Over Items**.

19. **Connect the false branch of the WhatsApp IF node back to Loop Over Items**.
    - This skips WhatsApp sending when the number is not valid.

20. **Add a Telegram node** named **Sent Telegram Message (not anyone's birthday today)**.
    - Connect it from the false branch of **If Check (Today Birthday?)**.
    - Configure Telegram credentials and chat ID.
    - Message example: “Today is X. Not anyone’s birthday today.”
    - Connect its output back to **Loop Over Items**.

21. **Be aware of the current logic issue**:
    - This “not anyone’s birthday today” message is sent once per non-matching contact.
    - If you want the intended behavior, redesign this path so it sends only once after all contacts are checked and no matches were found.

22. **Optional: Add sticky notes** to match the original layout:
    - Scheduling and Initialization Group
    - Contact Retrieval Group
    - Birthday Checking Group
    - No Birthday Group
    - Birthday Wishing Group
    - Plus the large overview note if you want in-canvas documentation.

23. **Credential setup checklist**
    - **Google Contacts OAuth2**: read access to contacts
    - **Telegram API**: bot token and correct chat ID
    - **Microsoft Teams OAuth2**: permission to post messages
    - **Home Assistant**: authenticated instance with valid TTS and media player entities
    - **Rapiwa API**: account/API key and installed custom node

24. **Test with sample contacts**
    - One with no birthday
    - One with birthday not today
    - One with birthday today
    - One with malformed birthday
    - One with no phone number
    - One with multiple phone numbers

25. **Recommended corrections before production**
    - Replace all references to nonexistent `formattedDate`
    - Preserve original contact data after the date-matching code node
    - Fix the WhatsApp verification IF to check actual boolean value
    - Decide whether WhatsApp should receive one phone number or many
    - Redesign the “no birthday today” notification to run once per workflow, not once per contact

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow is an automated birthday reminder and greeting system that checks for birthdays from Google Contacts daily at 10 AM and sends personalized birthday wishes through multiple channels. It integrates with Google Contacts to retrieve contact information, checks for birthdays occurring on the current day, and sends customized birthday messages via Telegram, Microsoft Teams, Google Home Speaker, and WhatsApp. The workflow uses Rapiwa for WhatsApp messaging and includes date formatting in Bengali locale for personalized messages. | Canvas overview note |
| Google Contacts API is used for retrieving contact information including birthdays, names, and phone numbers. | Data source note |
| Birthday detection compares contact birthdays stored as `mm/dd/yyyy` against the current date by extracting day and month. | Logic note |
| Telegram, Microsoft Teams, Google Home Speaker, and WhatsApp are all used as delivery channels. | Messaging note |
| Customization options mentioned in the workflow notes include schedule time, message templates, date format, notification channels, language settings, and contact fields. | Customization note |

## Additional implementation observations
- Several expressions reference fields that do not exist in the actual data flow, especially `formattedDate`.
- The workflow depends heavily on cross-node item references instead of keeping the full contact payload through the main branch.
- The WhatsApp verification branch should be reviewed carefully before use.
- The “not anyone’s birthday today” notification does not currently represent a true workflow-level summary.