Create and manage ERC-20 tokens with a Telegram bot and 1Shot API wallets

https://n8nworkflows.xyz/workflows/create-and-manage-erc-20-tokens-with-a-telegram-bot-and-1shot-api-wallets-11264


# Create and manage ERC-20 tokens with a Telegram bot and 1Shot API wallets

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Create and manage ERC-20 tokens with a Telegram bot and 1Shot API wallets  
**Workflow name (n8n):** Telegram Crypto Bot Template

**Purpose:**  
This workflow powers a Telegram bot that:
- Creates a dedicated EVM wallet for each user via **1Shot API**
- Displays wallet address and balance on `/start`
- Provides an **inline keyboard** to:
  1) Deploy a standard ERC-20 token (via 1Shot Token Factory method)  
  2) Mint/send tokens (via 1Shot contract method)  
  3) List the user’s deployed token addresses (from an n8n Data Table)

**Primary use cases:**
- Lightweight Telegram “wallet + token launcher” experience for testnets/EVM networks
- Custodial/managed transaction submission through 1Shot API wallets (walletId-based)
- Token deployment and minting using predefined 1Shot contract methods

### Logical Blocks
1.1 **Telegram Entry & Event Routing** (message vs callback query)  
1.2 **Main `/start` Path: Wallet Provisioning + Balance + Keyboard**  
1.3 **Callback Path: Wallet Lookup + Action Switch**  
1.4 **Deploy ERC‑20 Token (Token Factory)**  
1.5 **Send/Mint Tokens (contract method call)**  
1.6 **List User Tokens (Data Table → formatted Telegram message)**

---

## 2. Block-by-Block Analysis

### 2.1 Telegram Entry & Event Routing
**Overview:** Receives Telegram updates and routes execution depending on whether the update is a user message (e.g., `/start`) or an inline keyboard callback query (e.g., `create-token`).  
**Nodes involved:** `Telegram Trigger`, `Callback or Message`

#### Node: Telegram Trigger
- **Type / role:** `telegramTrigger` — entrypoint webhook for Telegram updates.
- **Configuration (interpreted):**
  - Listens to: `message`, `callback_query`
  - Uses Telegram bot credentials `@1shotdemobot`
- **Key data produced:**
  - For messages: `item.json.message.*` (e.g., `.text`, `.chat.id`, `.from`)
  - For callbacks: `item.json.callback_query.*` (e.g., `.data`, `.message.chat.id`)
- **Connections:**
  - Output → `Callback or Message`
- **Potential failures / edge cases:**
  - Telegram credential invalid/revoked
  - Bot webhook misconfigured
  - Users interacting in group chats (explicitly warned against in sticky note; chat.id would be shared by group)

#### Node: Callback or Message
- **Type / role:** `switch` — branches based on presence of `callback_query` vs `message`.
- **Configuration:**
  - Rule 1: if `$('Telegram Trigger').item.json.callback_query` **exists** → callback branch
  - Rule 2: if `$('Telegram Trigger').item.json.message` **exists** → message branch
- **Connections:**
  - Callback branch → `Lookup User Wallet`
  - Message branch → `Only respond to /start Messages`
- **Potential failures / edge cases:**
  - Unexpected update types (neither message nor callback_query) will not match any rule (no downstream)
  - Expressions assume `Telegram Trigger` item structure

---

### 2.2 Main `/start` Path: Wallet Provisioning + Balance + Keyboard
**Overview:** Handles the user’s `/start` command. If the user already has a wallet (stored in Data Table), it fetches wallet balance and shows an action keyboard. If not, it creates a new wallet in 1Shot API, stores it, and sends the address.  
**Nodes involved:** `Only respond to /start Messages`, `Check for Existing User`, `Does the User Have A Wallet`, `Create a User Wallet`, `Record User Wallet`, `Send User Wallet Address`, `Get User Wallet Balance`, `Send Action Keyboard Options`

#### Node: Only respond to /start Messages
- **Type / role:** `if` — gatekeeper to only respond to `/start`.
- **Configuration:**
  - Condition: `$('Telegram Trigger').item.json.message.text == "/start"`
- **Connections:**
  - True → `Check for Existing User`
  - False → (no output connected; bot ignores other messages)
- **Edge cases:**
  - If message text contains `/start` with parameters (e.g., `/start foo`) it will not match
  - If user sends non-text message, `.text` may be missing

#### Node: Check for Existing User
- **Type / role:** `dataTable` (operation: get) — query “User Table” by `chatid`.
- **Configuration:**
  - Filter: `chatid = {{$json.message.chat.id}}` (from incoming message payload)
  - `alwaysOutputData: true` (so downstream executes even if no row found)
- **Connections:**
  - Output → `Does the User Have A Wallet`
- **Failure modes:**
  - Data Table not created / wrong ID
  - Schema mismatch (missing `chatid`, `wallet`, `walletid`)
- **Edge cases:**
  - Multiple matching rows could return multiple items; downstream IF checks existence per item

#### Node: Does the User Have A Wallet
- **Type / role:** `if` — decides whether to create a wallet.
- **Configuration:**
  - Condition: field `$json.wallet` **exists**
- **Connections:**
  - True → `Get User Wallet Balance`
  - False → `Create a User Wallet`
- **Edge cases:**
  - If Data Table returns an empty item but with `alwaysOutputData`, `$json.wallet` may be undefined (expected)
  - If the table returns multiple items, you may unexpectedly fork multiple times

#### Node: Create a User Wallet
- **Type / role:** `1shot.oneShot` — creates a managed wallet in 1Shot.
- **Configuration:**
  - Resource: `wallets`, Operation: `create`
  - Chain ID: `97` (BSC testnet)
  - Wallet name: `chatid-{{ Telegram chat id }}`
  - Description: `"first_name, last_name, chat.id"`
- **Connections:**
  - Output → `Record User Wallet`
- **Failure modes:**
  - 1Shot OAuth2 credential invalid
  - ChainId not supported by your 1Shot account/config
  - Rate limits / API downtime

#### Node: Record User Wallet
- **Type / role:** `dataTable` (operation: insert/create row) — stores wallet mapping.
- **Configuration:**
  - Target: “User Table”
  - Columns written:
    - `chatid` = Telegram `message.chat.id`
    - `wallet` = 1Shot `accountAddress`
    - `walletid` = 1Shot `id`
- **Connections:**
  - Output → `Send User Wallet Address`
- **Failure modes / edge cases:**
  - Table missing or columns differ
  - Duplicate entries for same chatid (no dedupe logic shown)

#### Node: Send User Wallet Address
- **Type / role:** `telegram` — informs user of new wallet address.
- **Configuration:**
  - Chat ID: `Telegram Trigger → message.chat.id`
  - Text: `"You're wallet address is: {{ Create a User Wallet.accountAddress }}"`
  - Attribution disabled
- **Connections:** none further (end of branch)
- **Edge cases:**
  - Grammar: “You’re” vs “Your”
  - If wallet creation returned unexpected payload, expression may fail

#### Node: Get User Wallet Balance
- **Type / role:** `1shot.oneShot` — retrieves wallet info/balance.
- **Configuration:**
  - Resource: `wallets` (read)
  - ChainId: `97`
  - Wallet “name”: `{{$json.chatid}}` (this is unusual: the input item here is from `Check for Existing User`, so `$json.chatid` is the stored chat id column; it assumes 1Shot lookup by name is supported/configured this way)
- **Connections:**
  - Output → `Send Action Keyboard Options`
- **Potential issues:**
  - If 1Shot expects wallet ID or specific identifier, using chatid as `name` may not resolve
  - If the “get wallet” operation returns a different shape, downstream message expects `accountBalanceDetails.balance`

#### Node: Send Action Keyboard Options
- **Type / role:** `telegram` — shows wallet address + BNB balance and inline keyboard.
- **Configuration:**
  - Chat ID: `Telegram Trigger → message.chat.id`
  - Text (Markdown-like formatting with backticks):
    - Wallet: ``{{ $json.accountAddress }}``
    - Balance: `{{ $json.accountBalanceDetails.balance }} BNB`
  - Inline keyboard buttons:
    - “Create a token” → callback_data `create-token`
    - “Send Some Tokens” → `send-tokens`
    - “See My Tokens” → `list-tokens`
- **Connections:** none further (user continues via callbacks)
- **Edge cases:**
  - Balance field missing → expression renders empty or fails depending on n8n settings
  - Telegram Markdown parsing: backticks require correct parse mode (node doesn’t explicitly set parse mode)

---

### 2.3 Callback Path: Wallet Lookup + Action Switch
**Overview:** For any inline keyboard action, the workflow looks up the user’s wallet from the “User Table” using callback query chat id, then routes to the chosen feature branch.  
**Nodes involved:** `Lookup User Wallet`, `Switch on User Button Choice`

#### Node: Lookup User Wallet
- **Type / role:** `dataTable` (get) — fetch wallet mapping for callback user.
- **Configuration:**
  - Filter: `chatid = {{ $json.callback_query.message.chat.id.toString() }}`
  - Target table: “User Table”
- **Connections:**
  - Output → `Switch on User Button Choice`
- **Failure modes:**
  - If user never ran `/start`, no row exists → downstream calls will fail (no guard here)
- **Edge cases:**
  - Type mismatches (`chat.id` numeric vs stored string) mitigated by `.toString()`

#### Node: Switch on User Button Choice
- **Type / role:** `switch` — routes based on callback data.
- **Configuration:**
  - Reads: `$('Telegram Trigger').item.json.callback_query.data`
  - Rules:
    1. equals `create-token` → token deployment flow
    2. equals `send-tokens` → token sending flow
    3. equals `list-tokens` → list tokens flow
- **Connections:**
  - create-token → `Get the name of the token`
  - send-tokens → `Which Token?`
  - list-tokens → `Get User's Tokens`
- **Edge cases:**
  - Any other callback_data is ignored (no default branch)

---

### 2.4 Deploy ERC‑20 Token (Token Factory)
**Overview:** Collects token metadata from the user (name, symbol, max supply), submits a deployment transaction via 1Shot Token Factory contract method, and then records the deployed token in a “Token Table”.  
**Nodes involved:** `Get the name of the token`, `Get the token Symbol`, `Max Supply`, `Creation Confirmation`, `Deploy User Token`, `Record New Token`, `Reply with Token Address`, `Try Again`

#### Node: Get the name of the token
- **Type / role:** `telegram` (sendAndWait) — asks user for token name.
- **Configuration:**
  - Chat ID: `callback_query.message.chat.id`
  - Prompt: “What do you want to call the token?”
  - Response type: `freeText`
- **Connections:**
  - Output → `Get the token Symbol`
- **Failure modes / edge cases:**
  - User never responds → node can time out depending on n8n/telegram wait behavior
  - User sends invalid content (empty, too long)

#### Node: Get the token Symbol
- **Type / role:** `telegram` (sendAndWait) — asks for token ticker/symbol.
- **Configuration:**
  - Response type: `freeText`
- **Connections:**
  - Output → `Max Supply`
- **Edge cases:**
  - No validation of length/charset (ERC-20 symbol typically short)

#### Node: Max Supply
- **Type / role:** `telegram` (sendAndWait) — asks for max supply (numeric).
- **Configuration:**
  - Response type: `freeText` (no numeric validation)
- **Connections:**
  - Output → `Creation Confirmation`
- **Edge cases:**
  - Non-numeric input will later break contract call formatting or revert

#### Node: Creation Confirmation
- **Type / role:** `telegram` — informs user deployment is starting.
- **Configuration:**
  - Text: `Creating {tokenName} ({tokenSymbol})`
  - Uses:
    - Token name from `Get the name of the token`.data.text
    - Token symbol from current `$json.data.text` (coming from “Get the token Symbol” chain, via Max Supply → this node receives Max Supply response; however expression uses `$json.data.text` which at this step is max supply, not symbol. The node’s text currently mixes sources and likely is incorrect.)
- **Connections:**
  - Output → `Deploy User Token`
- **Potential issue (important):**
  - The message is likely wrong because `$json.data.text` at this node is “Max Supply” response, not symbol. Should reference `$('Get the token Symbol').item.json.data.text`.

#### Node: Deploy User Token
- **Type / role:** `1shot.oneShotSynch` — synchronous contract method call to deploy ERC‑20 via Token Factory.
- **Configuration:**
  - Contract method id: `d1459dee-e529-4e7c-b27d-4cdbddb6a72f` (expected to be `deployToken` from 1Shot Token Factory)
  - Params JSON (string-built):
    - `admin`: user wallet address from `Lookup User Wallet.wallet`
    - `name`: token name
    - `ticker`: token symbol
    - `premint`: `maxSupply + "000000000000000000"` (assumes **18 decimals**)
  - Additional fields:
    - `walletId`: from `Lookup User Wallet.walletid`
    - `memo`: `chatid-{chatid}-{tokenName}`
- **Connections:**
  - Success output (index 0) → `Record New Token`
  - Failure output (index 1) → `Try Again`
- **Failure modes / edge cases:**
  - Contract method not imported into 1Shot account
  - Gas insufficient (explicitly handled)
  - Wrong chain / wrong contract deployment / method mismatch
  - Premint formatting assumes 18 decimals and integer input; decimals or commas will break

#### Node: Try Again
- **Type / role:** `telegram` — failure message for token deployment.
- **Configuration:**
  - Text: “Token launch failed, please check your wallet has gas funds and try again.”
- **Connections:** none

#### Node: Record New Token
- **Type / role:** `dataTable` — stores deployed token metadata in “Token Table”.
- **Configuration:**
  - Columns written:
    - `name` = token name
    - `Wallet` = user wallet address from `Lookup User Wallet.wallet`
    - `chatid` = callback chat id
    - `token` = **hardcoded placeholder** `YOUR_CREDENTIAL_HERE`
- **Connections:**
  - Output → `Reply with Token Address`
- **Critical issue:**
  - `token` should be set to the deployed contract address but is not. The next node *does* read an address from `Deploy User Token` logs; you likely want to write that same address into this field.
- **Edge cases:**
  - Table column is named `Wallet` (capital W), while other parts refer to wallet; keep consistent.

#### Node: Reply with Token Address
- **Type / role:** `telegram` — sends deployed token address to user.
- **Configuration:**
  - Text: `{tokenName} deployed at `{Deploy User Token.logs[2].args[0]}`
- **Connections:** none
- **Edge cases / fragility:**
  - Assumes deployment event is always at `logs[2]` and args layout fixed. This can change depending on contract/events; safer to search logs by event name/signature.

---

### 2.5 Send/Mint Tokens (contract method call)
**Overview:** Asks the user for token contract address, amount, and recipient address, then calls a 1Shot contract method (expected to be `mint`) with wallet and contract overrides.  
**Nodes involved:** `Which Token?`, `How Many?`, `To What Address?`, `Send Tokens`, `Reply with TX Hash`, `Failure to Send Tokens`

#### Node: Which Token?
- **Type / role:** `telegram` (sendAndWait) — asks for token address to send.
- **Configuration:** freeText response (expects contract address)
- **Connections:** → `How Many?`
- **Edge cases:** no validation of address format

#### Node: How Many?
- **Type / role:** `telegram` (sendAndWait) — asks token amount.
- **Configuration:** freeText response
- **Connections:** → `To What Address?`
- **Edge cases:** amount parsing; decimals unsupported in the later concatenation logic

#### Node: To What Address?
- **Type / role:** `telegram` (sendAndWait) — asks recipient address.
- **Connections:** → `Send Tokens`

#### Node: Send Tokens
- **Type / role:** `1shot.oneShotSynch` — synchronous contract call for token mint/send.
- **Configuration:**
  - Contract method id: `6369ad5e-1ea8-416a-9e14-3ae07f0a0bd1` (expected to be `mint`)
  - `contractAddress`: from `Which Token?.data.text`
  - `walletId`: from `Lookup User Wallet.walletid`
  - Params JSON:
    - `to`: from current step `$json.data.text` (this node receives response from “To What Address?”)
    - `amount`: `How Many?.data.text + "000000000000000000"` (assumes 18 decimals)
  - Memo: `chatid-{chatid}: sending tokens`
- **Connections:**
  - Success → `Reply with TX Hash`
  - Failure → `Failure to Send Tokens`
- **Failure modes / edge cases:**
  - Insufficient gas / wallet balance
  - Contract doesn’t implement expected method/signature
  - Amount concatenation breaks if input includes decimals, spaces, or non-numeric chars
  - Wrong chain vs contract address

#### Node: Reply with TX Hash
- **Type / role:** `telegram` — confirms send with transaction hash.
- **Configuration:** `Tokens sent, tx-hash: {transactionHash}`
- **Edge cases:** if response shape differs (hash nested elsewhere)

#### Node: Failure to Send Tokens
- **Type / role:** `telegram` — failure message for send flow.

---

### 2.6 List User Tokens (Data Table → formatted Telegram message)
**Overview:** Fetches all tokens associated with the user chat id from the “Token Table”, formats them into a single message, and sends to Telegram.  
**Nodes involved:** `Get User's Tokens`, `Generate Message`, `Send Token List`

#### Node: Get User's Tokens
- **Type / role:** `dataTable` (get) — read tokens for user.
- **Configuration:**
  - Filter: `chatid ILIKE {callback chat id as string}`
  - Table: “Token Table”
  - `alwaysOutputData: true`
- **Connections:** → `Generate Message`
- **Edge cases:**
  - If `token` values are placeholders (as currently), message will be useless
  - ILIKE may match partial strings unintentionally

#### Node: Generate Message
- **Type / role:** `code` — aggregates tokens into one formatted message.
- **Logic:**
  - Collects all input items’ `item.json.token` into backticked list
  - Creates: `"Your token addresses:\n" + addresses.join("\n")`
  - Writes message into `$input.all()[0].json.message`
  - Returns all items (first item carries the message used later)
- **Connections:** → `Send Token List`
- **Edge cases / failure modes:**
  - If zero items truly arrive (depends on Data Table behavior), `$input.all()[0]` would be undefined
  - If token column missing, outputs `` `undefined` `` entries

#### Node: Send Token List
- **Type / role:** `telegram` — sends the aggregated message.
- **Configuration:**
  - Text: `{{$input.first().json.message}}`
  - `executeOnce: true` (prevents multiple sends if multiple items)
- **Edge cases:**
  - If `message` not set (code node didn’t run as expected), blank message

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Trigger | Telegram Trigger | Entry point for Telegram messages and callback queries | — | Callback or Message | ## Telegram Entrypoint  Trigger the workflows on either a message from a user in a DM or a callback query from a inline keyboard  IMPORTANT - for this template, the chatid is used at the immutable id link between a user and their 1Shot API wallet. So in your bot settings, be sure to disable Groups to prevent creating wallets shared between group members. You could change the workflow to use the Telegram username instead, but usernames can be changed and Telegram users are also NOT guaranteed to have a username set. |
| Callback or Message | Switch | Routes message vs callback_query updates | Telegram Trigger | Lookup User Wallet; Only respond to /start Messages | ## Telegram Entrypoint  Trigger the workflows on either a message from a user in a DM or a callback query from a inline keyboard  IMPORTANT - for this template, the chatid is used at the immutable id link between a user and their 1Shot API wallet. So in your bot settings, be sure to disable Groups to prevent creating wallets shared between group members. You could change the workflow to use the Telegram username instead, but usernames can be changed and Telegram users are also NOT guaranteed to have a username set. |
| Only respond to /start Messages | IF | Accepts only `/start` command | Callback or Message | Check for Existing User | ## Main User Entrypoint  The user can only trigger the bot by sending the `/start` command; customize this for your purposes.   ### Wallets  The first time a user triggers your bot, a wallet will be created for them in 1Shot API & the user logged in a datatable. When the user sends the `/start` command again, the bot will send the users wallet address and its current balance.   ### Data Tables  This workflow needs two data tables: `User Table` and `Token Table`.   The `User Table` should have `chatid`, `wallet` and `walletid` as columns. The `Token Table` should have `wallet`, `token`, `name` and `chatid` as columns. |
| Check for Existing User | Data Table | Lookup user wallet mapping by chatid | Only respond to /start Messages | Does the User Have A Wallet | ## Main User Entrypoint  The user can only trigger the bot by sending the `/start` command; customize this for your purposes.   ### Wallets  The first time a user triggers your bot, a wallet will be created for them in 1Shot API & the user logged in a datatable. When the user sends the `/start` command again, the bot will send the users wallet address and its current balance.   ### Data Tables  This workflow needs two data tables: `User Table` and `Token Table`.   The `User Table` should have `chatid`, `wallet` and `walletid` as columns. The `Token Table` should have `wallet`, `token`, `name` and `chatid` as columns. |
| Does the User Have A Wallet | IF | Branch: existing wallet vs create new | Check for Existing User | Get User Wallet Balance; Create a User Wallet | ## Main User Entrypoint  The user can only trigger the bot by sending the `/start` command; customize this for your purposes.   ### Wallets  The first time a user triggers your bot, a wallet will be created for them in 1Shot API & the user logged in a datatable. When the user sends the `/start` command again, the bot will send the users wallet address and its current balance.   ### Data Tables  This workflow needs two data tables: `User Table` and `Token Table`.   The `User Table` should have `chatid`, `wallet` and `walletid` as columns. The `Token Table` should have `wallet`, `token`, `name` and `chatid` as columns. |
| Create a User Wallet | 1Shot (oneShot) | Create a new managed wallet for user | Does the User Have A Wallet | Record User Wallet | ## Main User Entrypoint  The user can only trigger the bot by sending the `/start` command; customize this for your purposes.   ### Wallets  The first time a user triggers your bot, a wallet will be created for them in 1Shot API & the user logged in a datatable. When the user sends the `/start` command again, the bot will send the users wallet address and its current balance.   ### Data Tables  This workflow needs two data tables: `User Table` and `Token Table`.   The `User Table` should have `chatid`, `wallet` and `walletid` as columns. The `Token Table` should have `wallet`, `token`, `name` and `chatid` as columns. |
| Record User Wallet | Data Table | Persist chatid → wallet/walletid mapping | Create a User Wallet | Send User Wallet Address | ## Main User Entrypoint  The user can only trigger the bot by sending the `/start` command; customize this for your purposes.   ### Wallets  The first time a user triggers your bot, a wallet will be created for them in 1Shot API & the user logged in a datatable. When the user sends the `/start` command again, the bot will send the users wallet address and its current balance.   ### Data Tables  This workflow needs two data tables: `User Table` and `Token Table`.   The `User Table` should have `chatid`, `wallet` and `walletid` as columns. The `Token Table` should have `wallet`, `token`, `name` and `chatid` as columns. |
| Send User Wallet Address | Telegram | Send newly created wallet address | Record User Wallet | — | ## Main User Entrypoint  The user can only trigger the bot by sending the `/start` command; customize this for your purposes.   ### Wallets  The first time a user triggers your bot, a wallet will be created for them in 1Shot API & the user logged in a datatable. When the user sends the `/start` command again, the bot will send the users wallet address and its current balance.   ### Data Tables  This workflow needs two data tables: `User Table` and `Token Table`.   The `User Table` should have `chatid`, `wallet` and `walletid` as columns. The `Token Table` should have `wallet`, `token`, `name` and `chatid` as columns. |
| Get User Wallet Balance | 1Shot (oneShot) | Fetch wallet info/balance | Does the User Have A Wallet | Send Action Keyboard Options | ## Main User Entrypoint  The user can only trigger the bot by sending the `/start` command; customize this for your purposes.   ### Wallets  The first time a user triggers your bot, a wallet will be created for them in 1Shot API & the user logged in a datatable. When the user sends the `/start` command again, the bot will send the users wallet address and its current balance.   ### Data Tables  This workflow needs two data tables: `User Table` and `Token Table`.   The `User Table` should have `chatid`, `wallet` and `walletid` as columns. The `Token Table` should have `wallet`, `token`, `name` and `chatid` as columns. |
| Send Action Keyboard Options | Telegram | Show wallet + balance + inline keyboard | Get User Wallet Balance | — | ## Main User Entrypoint  The user can only trigger the bot by sending the `/start` command; customize this for your purposes.   ### Wallets  The first time a user triggers your bot, a wallet will be created for them in 1Shot API & the user logged in a datatable. When the user sends the `/start` command again, the bot will send the users wallet address and its current balance.   ### Data Tables  This workflow needs two data tables: `User Table` and `Token Table`.   The `User Table` should have `chatid`, `wallet` and `walletid` as columns. The `Token Table` should have `wallet`, `token`, `name` and `chatid` as columns. |
| Lookup User Wallet | Data Table | Lookup wallet mapping for callback user | Callback or Message | Switch on User Button Choice | ## Handle Keyboard Options  The keyboard callback options are set in "Send Action Keyboard Options". This node switches on the user selections.  The branch should only be triggered once the user has a wallet. |
| Switch on User Button Choice | Switch | Route callback action: create/send/list | Lookup User Wallet | Get the name of the token; Which Token?; Get User's Tokens | ## Handle Keyboard Options  The keyboard callback options are set in "Send Action Keyboard Options". This node switches on the user selections.  The branch should only be triggered once the user has a wallet. |
| Get the name of the token | Telegram | Prompt for token name | Switch on User Button Choice | Get the token Symbol | ## Deploy ERC-20 Tokens with the 1Shot Token Factory  This template uses the [1Shot Token Factory](https://github.com/1Shot-API/1Shot-Token-Factory) to deploy standard ERC-20 tokens. You can clone this repo, customize the smart contracts, and redeploy it on any EVM network to launch tokens for free.   When deploying a token, you must specify its name, symbol, admin address (which has the permissions to mint tokens), and the max supply cap. |
| Get the token Symbol | Telegram | Prompt for token symbol | Get the name of the token | Max Supply | ## Deploy ERC-20 Tokens with the 1Shot Token Factory  This template uses the [1Shot Token Factory](https://github.com/1Shot-API/1Shot-Token-Factory) to deploy standard ERC-20 tokens. You can clone this repo, customize the smart contracts, and redeploy it on any EVM network to launch tokens for free.   When deploying a token, you must specify its name, symbol, admin address (which has the permissions to mint tokens), and the max supply cap. |
| Max Supply | Telegram | Prompt for max supply | Get the token Symbol | Creation Confirmation | ## Deploy ERC-20 Tokens with the 1Shot Token Factory  This template uses the [1Shot Token Factory](https://github.com/1Shot-API/1Shot-Token-Factory) to deploy standard ERC-20 tokens. You can clone this repo, customize the smart contracts, and redeploy it on any EVM network to launch tokens for free.   When deploying a token, you must specify its name, symbol, admin address (which has the permissions to mint tokens), and the max supply cap. |
| Creation Confirmation | Telegram | Inform user deployment is starting | Max Supply | Deploy User Token | ## Deploy ERC-20 Tokens with the 1Shot Token Factory  This template uses the [1Shot Token Factory](https://github.com/1Shot-API/1Shot-Token-Factory) to deploy standard ERC-20 tokens. You can clone this repo, customize the smart contracts, and redeploy it on any EVM network to launch tokens for free.   When deploying a token, you must specify its name, symbol, admin address (which has the permissions to mint tokens), and the max supply cap. |
| Deploy User Token | 1Shot (oneShotSynch) | Deploy ERC-20 via Token Factory method | Creation Confirmation | Record New Token; Try Again | ## Deploy ERC-20 Tokens with the 1Shot Token Factory  This template uses the [1Shot Token Factory](https://github.com/1Shot-API/1Shot-Token-Factory) to deploy standard ERC-20 tokens. You can clone this repo, customize the smart contracts, and redeploy it on any EVM network to launch tokens for free.   When deploying a token, you must specify its name, symbol, admin address (which has the permissions to mint tokens), and the max supply cap. |
| Record New Token | Data Table | Store deployed token record | Deploy User Token | Reply with Token Address | ## Deploy ERC-20 Tokens with the 1Shot Token Factory  This template uses the [1Shot Token Factory](https://github.com/1Shot-API/1Shot-Token-Factory) to deploy standard ERC-20 tokens. You can clone this repo, customize the smart contracts, and redeploy it on any EVM network to launch tokens for free.   When deploying a token, you must specify its name, symbol, admin address (which has the permissions to mint tokens), and the max supply cap. |
| Reply with Token Address | Telegram | Send deployed token address | Record New Token | — | ## Deploy ERC-20 Tokens with the 1Shot Token Factory  This template uses the [1Shot Token Factory](https://github.com/1Shot-API/1Shot-Token-Factory) to deploy standard ERC-20 tokens. You can clone this repo, customize the smart contracts, and redeploy it on any EVM network to launch tokens for free.   When deploying a token, you must specify its name, symbol, admin address (which has the permissions to mint tokens), and the max supply cap. |
| Try Again | Telegram | Deployment failure message | Deploy User Token (failure) | — | ## Deploy ERC-20 Tokens with the 1Shot Token Factory  This template uses the [1Shot Token Factory](https://github.com/1Shot-API/1Shot-Token-Factory) to deploy standard ERC-20 tokens. You can clone this repo, customize the smart contracts, and redeploy it on any EVM network to launch tokens for free.   When deploying a token, you must specify its name, symbol, admin address (which has the permissions to mint tokens), and the max supply cap. |
| Which Token? | Telegram | Prompt for token contract address | Switch on User Button Choice | How Many? | ## Send Tokens Using the User's Wallet  This branch is triggered when the user asks to send tokens using the inline keyboard. The transaction will be submitted from their wallet against their token using wallet and contract override options. |
| How Many? | Telegram | Prompt for amount | Which Token? | To What Address? | ## Send Tokens Using the User's Wallet  This branch is triggered when the user asks to send tokens using the inline keyboard. The transaction will be submitted from their wallet against their token using wallet and contract override options. |
| To What Address? | Telegram | Prompt for recipient address | How Many? | Send Tokens | ## Send Tokens Using the User's Wallet  This branch is triggered when the user asks to send tokens using the inline keyboard. The transaction will be submitted from their wallet against their token using wallet and contract override options. |
| Send Tokens | 1Shot (oneShotSynch) | Call contract method to mint/send tokens | To What Address? | Reply with TX Hash; Failure to Send Tokens | ## Send Tokens Using the User's Wallet  This branch is triggered when the user asks to send tokens using the inline keyboard. The transaction will be submitted from their wallet against their token using wallet and contract override options. |
| Reply with TX Hash | Telegram | Confirm tx hash | Send Tokens (success) | — | ## Send Tokens Using the User's Wallet  This branch is triggered when the user asks to send tokens using the inline keyboard. The transaction will be submitted from their wallet against their token using wallet and contract override options. |
| Failure to Send Tokens | Telegram | Send flow failure message | Send Tokens (failure) | — | ## Send Tokens Using the User's Wallet  This branch is triggered when the user asks to send tokens using the inline keyboard. The transaction will be submitted from their wallet against their token using wallet and contract override options. |
| Get User's Tokens | Data Table | Query token list for user | Switch on User Button Choice | Generate Message | ## Send the User's Currently Deployed Token Addresses  This branch is triggered when the user clicks the inline keyboard option `See My Tokens`. |
| Generate Message | Code | Build single message from multiple token rows | Get User's Tokens | Send Token List | ## Send the User's Currently Deployed Token Addresses  This branch is triggered when the user clicks the inline keyboard option `See My Tokens`. |
| Send Token List | Telegram | Send formatted token list | Generate Message | — | ## Send the User's Currently Deployed Token Addresses  This branch is triggered when the user clicks the inline keyboard option `See My Tokens`. |
| Sticky Note | Sticky Note | Comment | — | — | ## Telegram Entrypoint  Trigger the workflows on either a message from a user in a DM or a callback query from a inline keyboard  IMPORTANT - for this template, the chatid is used at the immutable id link between a user and their 1Shot API wallet. So in your bot settings, be sure to disable Groups to prevent creating wallets shared between group members. You could change the workflow to use the Telegram username instead, but usernames can be changed and Telegram users are also NOT guaranteed to have a username set. |
| Sticky Note1 | Sticky Note | Comment | — | — | ## Send the User's Currently Deployed Token Addresses  This branch is triggered when the user clicks the inline keyboard option `See My Tokens`. |
| Sticky Note2 | Sticky Note | Comment | — | — | ## Handle Keyboard Options  The keyboard callback options are set in "Send Action Keyboard Options". This node switches on the user selections.  The branch should only be triggered once the user has a wallet. |
| Sticky Note3 | Sticky Note | Comment | — | — | ## Send Tokens Using the User's Wallet  This branch is triggered when the user asks to send tokens using the inline keyboard. The transaction will be submitted from their wallet against their token using wallet and contract override options. |
| Sticky Note5 | Sticky Note | Comment | — | — | ## Deploy ERC-20 Tokens with the 1Shot Token Factory  This template uses the [1Shot Token Factory](https://github.com/1Shot-API/1Shot-Token-Factory) to deploy standard ERC-20 tokens. You can clone this repo, customize the smart contracts, and redeploy it on any EVM network to launch tokens for free.   When deploying a token, you must specify its name, symbol, admin address (which has the permissions to mint tokens), and the max supply cap. |
| Sticky Note6 | Sticky Note | Comment | — | — | # Create Your Own Telegram Crypto Bot This workflow template can be used as a starting point for your own Telegram bot that creates wallets for your users that they can use for making onchain transactions.  ## How It Works  This template connects to a Telegram bot that you create using the Telegram BotFather bot. Users can then interact with your bot to create a personal EVM wallet and submit transactions from their wallet via inline keyboard options when interacting with your bot.  [1Shot API](https://1shotapi.com) nodes are used for creating user wallets and managing transaction lifecycle and history.   ## Setup Steps  1. Create a Telegram bot with the [BotFather](https://t.me/BotFather) bot and use its access token to authenticate the Telegram nodes.  2. Create two data tables: `User Table` and `Token Table`.  3. Create a free [1Shot API](https://1shotapi.com) account and generate an API key & secret to authenticate the 1Shot API nodes.  4. Import the [Token Factory](https://github.com/1Shot-API/1Shot-Token-Factory) methods to your 1Shot API account and point the `Deploy User Token` and `Send Token` nodes at the `deployToken` and `mint` methods respectively.   ## Watch the Complete Walkthrough Tutorial Watch the [YouTube tutorial](https://youtu.be/bKRvt2DuVKA) for a complete walkthrough. @[youtube](bKRvt2DuVKA) |
| Sticky Note7 | Sticky Note | Comment | — | — | ## Main User Entrypoint  The user can only trigger the bot by sending the `/start` command; customize this for your purposes.   ### Wallets  The first time a user triggers your bot, a wallet will be created for them in 1Shot API & the user logged in a datatable. When the user sends the `/start` command again, the bot will send the users wallet address and its current balance.   ### Data Tables  This workflow needs two data tables: `User Table` and `Token Table`.   The `User Table` should have `chatid`, `wallet` and `walletid` as columns. The `Token Table` should have `wallet`, `token`, `name` and `chatid` as columns. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Telegram bot + credentials**
1. Create a bot via **BotFather**: https://t.me/BotFather  
2. In n8n, create **Telegram API** credentials using the bot token.
3. In BotFather settings, **disable Groups** (or otherwise ensure you only accept DMs) if you rely on `chat.id` as a unique user key.

2) **Create 1Shot API account + credentials**
4. Create a 1Shot API account: https://1shotapi.com  
5. Create OAuth2/API credentials in n8n for **1Shot** nodes (as required by the installed `n8n-nodes-1shot` package).

3) **Prepare Data Tables**
6. Create n8n **Data Table**: `User Table` with columns:
   - `chatid` (string)
   - `wallet` (string)
   - `walletid` (string)
7. Create n8n **Data Table**: `Token Table` with columns:
   - `wallet` (string) or `Wallet` (string) — pick one and keep consistent
   - `token` (string)
   - `name` (string)
   - `chatid` (string)

4) **Import / configure 1Shot contract methods**
8. Import Token Factory methods from: https://github.com/1Shot-API/1Shot-Token-Factory  
9. In 1Shot, ensure you have:
   - A `deployToken` method (used by **Deploy User Token**)  
   - A `mint` (or equivalent) method (used by **Send Tokens**)  
10. Note each method’s **contractMethodId** and configure the two 1ShotSynch nodes with the correct IDs.

5) **Build nodes and connect them**

**A. Entry + routing**
11. Add **Telegram Trigger**:
   - Updates: `message`, `callback_query`
12. Add **Switch** node “Callback or Message”:
   - Rule: `callback_query exists` → output 1
   - Rule: `message exists` → output 2
13. Connect: `Telegram Trigger → Callback or Message`

**B. `/start` flow**
14. Add **IF** node “Only respond to /start Messages”:
   - Condition: `Telegram Trigger.message.text equals "/start"`
15. Connect: `Callback or Message (message branch) → Only respond to /start Messages`
16. Add **Data Table (Get)** node “Check for Existing User”:
   - Table: `User Table`
   - Filter: `chatid = {{$json.message.chat.id}}`
   - Enable “Always output data”
17. Connect: `Only respond to /start Messages (true) → Check for Existing User`
18. Add **IF** node “Does the User Have A Wallet”:
   - Condition: `$json.wallet exists`
19. Connect: `Check for Existing User → Does the User Have A Wallet`

**B1. If no wallet: create + record + reply**
20. Add **1Shot** node “Create a User Wallet”:
   - Resource: wallets, Operation: create
   - chainId: `97`
   - name: `chatid-{{ Telegram Trigger.message.chat.id }}`
   - description: `{{ first_name }}, {{ last_name }}, {{ chat.id }}`
21. Connect: `Does the User Have A Wallet (false) → Create a User Wallet`
22. Add **Data Table (Insert/Create)** node “Record User Wallet”:
   - Table: `User Table`
   - chatid = `Telegram Trigger.message.chat.id`
   - wallet = `Create a User Wallet.accountAddress`
   - walletid = `Create a User Wallet.id`
23. Connect: `Create a User Wallet → Record User Wallet`
24. Add **Telegram** node “Send User Wallet Address”:
   - chatId: `Telegram Trigger.message.chat.id`
   - text: `You're wallet address is: {{ Create a User Wallet.accountAddress }}`
25. Connect: `Record User Wallet → Send User Wallet Address`

**B2. If wallet exists: get balance + show keyboard**
26. Add **1Shot** node “Get User Wallet Balance”:
   - Resource: wallets (read)
   - chainId: `97`
   - identifier/name: set to match how your 1Shot node locates the wallet (in the provided workflow it uses `{{$json.chatid}}`)
27. Connect: `Does the User Have A Wallet (true) → Get User Wallet Balance`
28. Add **Telegram** node “Send Action Keyboard Options”:
   - chatId: `Telegram Trigger.message.chat.id`
   - text includes wallet and BNB balance from 1Shot response
   - replyMarkup: inline keyboard with callback_data values:
     - `create-token`, `send-tokens`, `list-tokens`
29. Connect: `Get User Wallet Balance → Send Action Keyboard Options`

**C. Callback path: wallet lookup + action switch**
30. Add **Data Table (Get)** node “Lookup User Wallet”:
   - Table: `User Table`
   - Filter: `chatid = {{ Telegram Trigger.callback_query.message.chat.id.toString() }}`
31. Connect: `Callback or Message (callback branch) → Lookup User Wallet`
32. Add **Switch** node “Switch on User Button Choice”:
   - left: `Telegram Trigger.callback_query.data`
   - equals `create-token`, `send-tokens`, `list-tokens`
33. Connect: `Lookup User Wallet → Switch on User Button Choice`

**D. Create-token branch**
34. Add **Telegram (sendAndWait)** “Get the name of the token” (freeText)
35. Add **Telegram (sendAndWait)** “Get the token Symbol” (freeText)
36. Add **Telegram (sendAndWait)** “Max Supply” (freeText)
37. Add **Telegram** “Creation Confirmation”
38. Add **1ShotSynch** “Deploy User Token”:
   - contractMethodId: your `deployToken`
   - walletId: `Lookup User Wallet.walletid`
   - params: admin/name/ticker/premint (premint: append 18 zeros if using 18 decimals)
39. Add **Telegram** “Try Again” on failure output.
40. Add **Data Table (Insert/Create)** “Record New Token”:
   - Table: `Token Table`
   - name: token name
   - token: **set to deployed token address** (fix the placeholder)
   - wallet: user wallet
   - chatid: callback chat id
41. Add **Telegram** “Reply with Token Address”
42. Connect sequentially:
   - Switch(create-token) → Get name → Get symbol → Max supply → Confirmation → Deploy User Token
   - Deploy success → Record New Token → Reply with Token Address
   - Deploy failure → Try Again

**E. Send-tokens branch**
43. Add **Telegram (sendAndWait)** “Which Token?” (expects contract address)
44. Add **Telegram (sendAndWait)** “How Many?” (amount)
45. Add **Telegram (sendAndWait)** “To What Address?” (recipient)
46. Add **1ShotSynch** “Send Tokens”:
   - contractMethodId: your `mint` (or transfer) method id
   - contractAddress: from “Which Token?”
   - walletId: `Lookup User Wallet.walletid`
   - params: `to` and `amount` (append 18 zeros if using integer input)
47. Add **Telegram** “Reply with TX Hash” on success
48. Add **Telegram** “Failure to Send Tokens” on failure
49. Connect:
   - Switch(send-tokens) → Which Token → How Many → To What Address → Send Tokens → (success/failure)

**F. List-tokens branch**
50. Add **Data Table (Get)** “Get User’s Tokens”:
   - Table: Token Table
   - Filter: chatid matches callback chat id (string)
51. Add **Code** “Generate Message” to aggregate addresses into a single message
52. Add **Telegram** “Send Token List”
53. Connect:
   - Switch(list-tokens) → Get User’s Tokens → Generate Message → Send Token List

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| 1Shot API website | https://1shotapi.com |
| Token Factory repo referenced by the workflow | https://github.com/1Shot-API/1Shot-Token-Factory |
| BotFather (create/manage Telegram bots) | https://t.me/BotFather |
| Video link included in the workflow note | https://youtu.be/bKRvt2DuVKA |
| Key design constraint: using Telegram `chat.id` as immutable user key; disable groups to avoid shared wallets | Mentioned in “Telegram Entrypoint” sticky note |

