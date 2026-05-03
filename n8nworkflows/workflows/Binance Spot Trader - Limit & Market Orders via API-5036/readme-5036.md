Binance Spot Trader - Limit & Market Orders via API

https://n8nworkflows.xyz/workflows/binance-spot-trader---limit---market-orders-via-api-5036


# Binance Spot Trader - Limit & Market Orders via API

### 1. Workflow Overview

This workflow is designed to interact with the Binance Spot trading API, enabling automated trading actions such as placing limit and market buy/sell orders, retrieving account information, managing open orders, and cancelling all open orders on the Binance Spot market. It is intended for users who want to automate trading operations on Binance via API calls within n8n.

The workflow is logically divided into the following blocks:

- **1.1 Initialization & Credential Setup**: Manual trigger and credential configuration for Binance API access.
- **1.2 Account Information Retrieval**: Querying account details using signed requests.
- **1.3 Limit Order Handling**: Preparing, signing, and executing limit buy and sell orders.
- **1.4 Market Order Handling**: Preparing, signing, and executing market buy and sell orders.
- **1.5 Open Orders Management**: Retrieving currently open orders.
- **1.6 Order Cancellation**: Cancelling all open orders for a specified trading symbol.

Each block involves preparing query parameters, generating HMAC SHA256 signatures for Binance API authentication, and sending HTTP requests with proper headers.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization & Credential Setup

- **Overview**: The workflow starts with a manual trigger to initiate the process and sets up Binance API credentials required for all subsequent API calls.
- **Nodes Involved**:
  - When clicking ‘Execute workflow’
  - Set Credentials
  - Set Account Query

- **Node Details**:

  1. **When clicking ‘Execute workflow’**
     - Type: Manual Trigger
     - Role: Entry point to start the workflow manually.
     - Config: Default manual trigger with no parameters.
     - Input: None
     - Output: Triggers the next node.
     - Failures: None expected.

  2. **Set Credentials**
     - Type: Set
     - Role: Holds Binance API credentials (api_key, api_secret) for reuse.
     - Config: Stores two string fields `api_key` and `api_secret`. No values set here; user must create Binance API credentials in n8n credential manager.
     - Notes: Instructional note in Thai explaining how to create Binance API credentials in n8n and naming convention.
     - Input: Manual trigger output.
     - Output: Passes credentials JSON to next node.
     - Failures: None, but missing or invalid credentials will cause API authentication errors downstream.

  3. **Set Account Query**
     - Type: Set
     - Role: Creates a JSON object with a `timestamp` field containing current time in milliseconds to be used as query parameter.
     - Config: Outputs JSON with `timestamp` = current time.
     - Input: Credentials node output (to ensure proper sequence).
     - Output: JSON with timestamp for account info query.
     - Failures: Minimal; timezone or system clock issues could affect timestamp accuracy.

#### 2.2 Account Information Retrieval

- **Overview**: Prepares and signs the query string for the account info API call, then fetches account details.
- **Nodes Involved**:
  - Account Info Query
  - Signature Get Account
  - Get Account Info

- **Node Details**:

  1. **Account Info Query**
     - Type: Code (JavaScript)
     - Role: Converts timestamp JSON into URL query string format.
     - Config: Takes JSON `{timestamp: <ms>}`, outputs `queryString` like `timestamp=1234567890`.
     - Input: From Set Account Query node.
     - Output: JSON with `queryString` and original data.
     - Failures: Code errors if input JSON malformed; unlikely here.

  2. **Signature Get Account**
     - Type: Crypto Node
     - Role: Generates HMAC SHA256 signature of query string using stored Binance API secret.
     - Config: Uses HMAC-SHA256 with secret from credentials and input queryString.
     - Input: Takes `queryString` from Account Info Query.
     - Output: Adds `signature` field to JSON.
     - Failures: Missing or invalid secret causes signature failure.

  3. **Get Account Info**
     - Type: HTTP Request
     - Role: Sends authenticated GET request to Binance API `/api/v3/account` to retrieve account info.
     - Config:
       - URL: `https://api.binance.com/api/v3/account`
       - Method: GET
       - Query parameters: `timestamp`, `signature`
       - Headers: `X-MBX-APIKEY` with API key from credentials
     - Input: Signed query string + signature.
     - Output: Binance API response with account data.
     - Failures: Possible 401 Unauthorized (bad API key/secret), 400 Bad Request (timestamp issues), network errors.

#### 2.3 Limit Order Handling

- **Overview**: Prepares parameters for limit buy and limit sell orders, signs requests, and executes them on Binance.
- **Nodes Involved**:
  - LimitBuy Parmeter
  - Order Query
  - Signature Limit BUY
  - Execute Limit BUY
  - LimitSale Parameter
  - Sale Query
  - Signature Limit SELL
  - Execute Limit SELL

- **Node Details**:

  1. **LimitBuy Parmeter**
     - Type: Set
     - Role: Creates JSON for limit buy order with fixed parameters: symbol BTCUSDT, side BUY, type LIMIT, price 98000.20, quantity 0.0001, timeInForce GTC, current timestamp.
     - Input: None (static).
     - Output: JSON with order parameters.
     - Failures: Hardcoded values limit flexibility; timestamp must be current.

  2. **Order Query**
     - Type: Code
     - Role: Converts limit buy JSON into query string.
     - Input: LimitBuy Parmeter JSON.
     - Output: `queryString` with order parameters.
     - Failures: Minimal; depends on input integrity.

  3. **Signature Limit BUY**
     - Type: Crypto
     - Role: Signs the limit buy query string with HMAC SHA256.
     - Input: Query string from Order Query.
     - Output: Adds signature.
     - Failures: As per other signature nodes.

  4. **Execute Limit BUY**
     - Type: HTTP Request
     - Role: Sends POST request to Binance `/api/v3/order` to place limit buy order.
     - Config:
       - Body: `queryString` + `signature`
       - Content-Type: form-urlencoded
       - Headers: API key
       - Method: POST
     - Input: Signed query string.
     - Output: Binance API response.
     - Failures: API errors (invalid params, insufficient balance), network errors.

  5. **LimitSale Parameter**
     - Type: Set
     - Role: Creates JSON for limit sell order, similar to buy but side SELL.
     - Input: None (static).
     - Output: JSON with sell order parameters.
     - Failures: Same as LimitBuy Parmeter.

  6. **Sale Query**
     - Type: Code
     - Role: Converts limit sell JSON to query string.
     - Input: LimitSale Parameter output.
     - Output: `queryString` for sell order.
     - Failures: Minimal.

  7. **Signature Limit SELL**
     - Type: Crypto
     - Role: Signs the sell order query string.
     - Input: Sale Query output.
     - Output: Adds signature.
     - Failures: As above.

  8. **Execute Limit SELL**
     - Type: HTTP Request
     - Role: Sends POST request to place limit sell order.
     - Config: Same as Execute Limit BUY but for sell.
     - Failures: Same as Execute Limit BUY.

#### 2.4 Market Order Handling

- **Overview**: Prepares, signs, and executes market buy and sell orders.
- **Nodes Involved**:
  - MarketBuy Parameter
  - Market Buy Query
  - Signature MarketBuy
  - Execute MarketBuy
  - MarketSell Parameter
  - Market Sell Query
  - Signature MarketSell
  - Execute MarketSell

- **Node Details**:

  1. **MarketBuy Parameter**
     - Type: Set
     - Role: JSON for market buy order: symbol BTCUSDT, side BUY, type MARKET, quantity 0.0001, timestamp current.
     - Input: None.
     - Output: JSON order params.
     - Failures: Hardcoded values limit flexibility.

  2. **Market Buy Query**
     - Type: Code
     - Role: Converts order JSON to query string.
     - Input: MarketBuy Parameter.
     - Output: `queryString`.
     - Failures: Minimal.

  3. **Signature MarketBuy**
     - Type: Crypto
     - Role: Signs query string.
     - Failures: As above.

  4. **Execute MarketBuy**
     - Type: HTTP Request
     - Role: Sends POST to execute market buy.
     - Failures: API errors, insufficient funds.

  5. **MarketSell Parameter**
     - Type: Set
     - Role: JSON for market sell order, similar to buy but side SELL.
     - Input: None.
     - Failures: Same as MarketBuy Parameter.

  6. **Market Sell Query**
     - Type: Code
     - Role: Converts JSON to query string.
     - Failures: Minimal.

  7. **Signature MarketSell**
     - Type: Crypto
     - Role: Signs query string.
     - Failures: Same as above.

  8. **Execute MarketSell**
     - Type: HTTP Request
     - Role: Sends POST to execute market sell.
     - Failures: API errors, network errors.

#### 2.5 Open Orders Management

- **Overview**: Retrieves open orders for the BTCUSDT symbol.
- **Nodes Involved**:
  - Open Orders Params
  - OpenOrder Query
  - Signature OpenOrder
  - Get Open Orders

- **Node Details**:

  1. **Open Orders Params**
     - Type: Set
     - Role: Creates JSON with `timestamp`, `symbol` BTCUSDT, and `recvWindow` 2000ms.
     - Input: None.
     - Output: JSON query parameters.
     - Failures: recvWindow too low might cause errors.

  2. **OpenOrder Query**
     - Type: Code
     - Role: Converts JSON to query string.
     - Failures: Minimal.

  3. **Signature OpenOrder**
     - Type: Crypto
     - Role: Signs query string.
     - Failures: As above.

  4. **Get Open Orders**
     - Type: HTTP Request
     - Role: Sends GET request to `/api/v3/openOrders` with query string and signature.
     - Config: URL composed with query string; includes API key header.
     - Failures: API errors, connectivity issues.

#### 2.6 Order Cancellation

- **Overview**: Cancels all open orders for BTCUSDT symbol.
- **Nodes Involved**:
  - Cancel All Order Params
  - CancelAllOrder Query
  - Signature Cancel All Order
  - Cancell All Order

- **Node Details**:

  1. **Cancel All Order Params**
     - Type: Set
     - Role: Creates JSON with `timestamp`, `symbol` BTCUSDT, and `recvWindow` 2000ms.
     - Input: None.
     - Output: JSON query params.
     - Failures: Same as Open Orders Params.

  2. **CancelAllOrder Query**
     - Type: Code
     - Role: Converts JSON to query string.
     - Failures: Minimal.

  3. **Signature Cancel All Order**
     - Type: Crypto
     - Role: Signs query string.
     - Failures: As above.

  4. **Cancell All Order**
     - Type: HTTP Request
     - Role: Sends DELETE request to `/api/v3/openOrders` with query string and signature to cancel all orders.
     - Config: Includes API key header.
     - Failures: API errors if no open orders, or permission issues.

---

### 3. Summary Table

| Node Name                 | Node Type          | Functional Role                         | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                           |
|---------------------------|--------------------|---------------------------------------|-------------------------------|--------------------------------|-----------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger     | Workflow entry point                   | None                          | Set Credentials                |                                                                                                     |
| Set Credentials           | Set                | Holds Binance API credentials          | When clicking ‘Execute workflow’ | Set Account Query             | กรุณาสร้าง Binance API Credentials ใน n8n: 1. ไปที่ Credentials > New 2. เลือก Binance API 3. ใส่ API Key และ Secret Key 4. ตั้งชื่อว่า 'Binance API' 5. Save 6. ทุก node จะใช้ credential เดียวกันนี้ |
| Set Account Query         | Set                | Prepare timestamp for account query    | Set Credentials               | Account Info Query             |                                                                                                     |
| Account Info Query        | Code               | Convert JSON to query string for account info | Set Account Query           | Signature Get Account          |                                                                                                     |
| Signature Get Account     | Crypto             | Generate HMAC SHA256 signature for account | Account Info Query           | Get Account Info              |                                                                                                     |
| Get Account Info          | HTTP Request       | Fetch Binance account information      | Signature Get Account         | None                         |                                                                                                     |
| LimitBuy Parmeter         | Set                | Prepare parameters for limit buy order | None                         | Order Query                   |                                                                                                     |
| Order Query               | Code               | Convert limit buy JSON to query string | LimitBuy Parmeter            | Signature Limit BUY           |                                                                                                     |
| Signature Limit BUY       | Crypto             | Sign limit buy order query string      | Order Query                  | Execute Limit BUY             |                                                                                                     |
| Execute Limit BUY         | HTTP Request       | Place limit buy order                   | Signature Limit BUY          | None                         |                                                                                                     |
| LimitSale Parameter       | Set                | Prepare parameters for limit sell order | None                         | Sale Query                   |                                                                                                     |
| Sale Query               | Code               | Convert limit sell JSON to query string | LimitSale Parameter          | Signature Limit SELL          |                                                                                                     |
| Signature Limit SELL      | Crypto             | Sign limit sell order query string     | Sale Query                   | Execute Limit SELL            |                                                                                                     |
| Execute Limit SELL        | HTTP Request       | Place limit sell order                  | Signature Limit SELL         | None                         |                                                                                                     |
| MarketBuy Parameter       | Set                | Prepare parameters for market buy order | None                         | Market Buy Query              |                                                                                                     |
| Market Buy Query          | Code               | Convert market buy JSON to query string | MarketBuy Parameter          | Signature MarketBuy           |                                                                                                     |
| Signature MarketBuy       | Crypto             | Sign market buy order query string     | Market Buy Query             | Execute MarketBuy             |                                                                                                     |
| Execute MarketBuy         | HTTP Request       | Place market buy order                  | Signature MarketBuy          | None                         |                                                                                                     |
| MarketSell Parameter      | Set                | Prepare parameters for market sell order | None                         | Market Sell Query             |                                                                                                     |
| Market Sell Query         | Code               | Convert market sell JSON to query string | MarketSell Parameter         | Signature MarketSell          |                                                                                                     |
| Signature MarketSell      | Crypto             | Sign market sell order query string    | Market Sell Query            | Execute MarketSell            |                                                                                                     |
| Execute MarketSell        | HTTP Request       | Place market sell order                 | Signature MarketSell         | None                         |                                                                                                     |
| Open Orders Params        | Set                | Prepare parameters for open orders retrieval | None                         | OpenOrder Query              |                                                                                                     |
| OpenOrder Query           | Code               | Convert open orders JSON to query string | Open Orders Params           | Signature OpenOrder           |                                                                                                     |
| Signature OpenOrder       | Crypto             | Sign open orders query string           | OpenOrder Query              | Get Open Orders              |                                                                                                     |
| Get Open Orders           | HTTP Request       | Retrieve open orders from Binance       | Signature OpenOrder          | None                         |                                                                                                     |
| Cancel All Order Params   | Set                | Prepare parameters for cancelling all orders | None                         | CancelAllOrder Query          |                                                                                                     |
| CancelAllOrder Query      | Code               | Convert cancel all orders JSON to query string | Cancel All Order Params      | Signature Cancel All Order    |                                                                                                     |
| Signature Cancel All Order| Crypto             | Sign cancel all orders query string     | CancelAllOrder Query         | Cancell All Order            |                                                                                                     |
| Cancell All Order         | HTTP Request       | Cancel all open orders on Binance       | Signature Cancel All Order   | None                         |                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**
   - Name: `When clicking ‘Execute workflow’`
   - Type: Manual Trigger
   - No parameters.

2. **Create Set Node for Credentials**
   - Name: `Set Credentials`
   - Type: Set
   - Add string fields: `api_key`, `api_secret`
   - Leave values empty; user must create Binance API credentials in n8n credential manager under Binance API type.
   - Connect from manual trigger.

3. **Create Set Node for Account Timestamp**
   - Name: `Set Account Query`
   - Type: Set
   - Mode: Raw
   - JSON Output:
     ```json
     {
       "timestamp": "{{ $now.toMillis() }}"
     }
     ```
   - Connect from `Set Credentials`.

4. **Create Code Node to Convert Account JSON to Query String**
   - Name: `Account Info Query`
   - Type: Code (JavaScript)
   - Code:
     ```javascript
     const data = items[0].json;
     const queryString = Object.keys(data).map(key => `${key}=${data[key]}`).join('&');
     return [{ json: { queryString, originalData: data } }];
     ```
   - Connect from `Set Account Query`.

5. **Create Crypto Node to Sign Account Query**
   - Name: `Signature Get Account`
   - Type: Crypto
   - Type: HMAC SHA256
   - Value: `={{ $json.queryString }}`
   - Secret: `={{ $('Set Credentials').item.json.api_secret }}`
   - Output property: `signature`
   - Connect from `Account Info Query`.

6. **Create HTTP Request Node to Get Account Info**
   - Name: `Get Account Info`
   - Type: HTTP Request
   - Method: GET
   - URL: `https://api.binance.com/api/v3/account`
   - Query Parameters:
     - `timestamp`: `={{ $('Set Account Query').item.json.timestamp }}`
     - `signature`: `={{ $json.signature }}`
   - Headers:
     - `X-MBX-APIKEY`: `={{ $('Set Credentials').item.json.api_key }}`
   - Connect from `Signature Get Account`.

---

7. **Create Limit Buy Parameter Node**
   - Name: `LimitBuy Parmeter`
   - Type: Set
   - JSON Output:
     ```json
     {
       "symbol": "BTCUSDT",
       "side": "BUY",
       "type": "LIMIT",
       "price": 98000.20,
       "quantity": 0.0001,
       "timeInForce": "GTC",
       "timestamp": "{{ $now.toMillis() }}"
     }
     ```
   - Connect from `Set Credentials` (or appropriate trigger).

8. **Create Code Node to Convert Limit Buy to Query**
   - Name: `Order Query`
   - Same code as `Account Info Query` node.
   - Connect from `LimitBuy Parmeter`.

9. **Create Crypto Node to Sign Limit Buy Query**
   - Name: `Signature Limit BUY`
   - Same config as `Signature Get Account`, but input from `Order Query`.
   - Connect from `Order Query`.

10. **Create HTTP Request Node to Execute Limit Buy**
    - Name: `Execute Limit BUY`
    - Method: POST
    - URL: `https://api.binance.com/api/v3/order`
    - Body: `={{ $json.queryString }}&signature={{ $json.signature }}`
    - Content-Type: `application/x-www-form-urlencoded`
    - Headers: `X-MBX-APIKEY` with API key
    - Connect from `Signature Limit BUY`.

---

11. **Create Limit Sell Parameter Node**
    - Name: `LimitSale Parameter`
    - Type: Set
    - JSON Output:
      ```json
      {
        "symbol": "BTCUSDT",
        "side": "SELL",
        "type": "LIMIT",
        "price": 98000.20,
        "quantity": 0.0001,
        "timeInForce": "GTC",
        "timestamp": "{{ $now.toMillis() }}"
      }
      ```
    - Connect from `Set Credentials`.

12. **Create Code Node to Convert Limit Sell to Query**
    - Name: `Sale Query`
    - Same code as previous code nodes.
    - Connect from `LimitSale Parameter`.

13. **Create Crypto Node to Sign Limit Sell Query**
    - Name: `Signature Limit SELL`
    - Connect from `Sale Query`.

14. **Create HTTP Request Node to Execute Limit Sell**
    - Name: `Execute Limit SELL`
    - Same config as `Execute Limit BUY`.
    - Connect from `Signature Limit SELL`.

---

15. **Create Market Buy Parameter Node**
    - Name: `MarketBuy Parameter`
    - Type: Set
    - JSON Output:
      ```json
      {
        "symbol": "BTCUSDT",
        "side": "BUY",
        "type": "MARKET",
        "quantity": 0.0001,
        "timestamp": "{{ $now.toMillis() }}"
      }
      ```
    - Connect from `Set Credentials`.

16. **Create Code Node to Convert Market Buy to Query**
    - Name: `Market Buy Query`
    - Same code as before.
    - Connect from `MarketBuy Parameter`.

17. **Create Crypto Node to Sign Market Buy Query**
    - Name: `Signature MarketBuy`
    - Connect from `Market Buy Query`.

18. **Create HTTP Request Node to Execute Market Buy**
    - Name: `Execute MarketBuy`
    - Same config as other HTTP requests.
    - Connect from `Signature MarketBuy`.

---

19. **Create Market Sell Parameter Node**
    - Name: `MarketSell Parameter`
    - Type: Set
    - JSON Output:
      ```json
      {
        "symbol": "BTCUSDT",
        "side": "SELL",
        "type": "MARKET",
        "quantity": 0.0001,
        "timestamp": "{{ $now.toMillis() }}"
      }
      ```
    - Connect from `Set Credentials`.

20. **Create Code Node to Convert Market Sell to Query**
    - Name: `Market Sell Query`
    - Same code.
    - Connect from `MarketSell Parameter`.

21. **Create Crypto Node to Sign Market Sell Query**
    - Name: `Signature MarketSell`
    - Connect from `Market Sell Query`.

22. **Create HTTP Request Node to Execute Market Sell**
    - Name: `Execute MarketSell`
    - Same config as others.
    - Connect from `Signature MarketSell`.

---

23. **Create Open Orders Parameter Node**
    - Name: `Open Orders Params`
    - Type: Set
    - JSON Output:
      ```json
      {
        "timestamp": "{{ $now.toMillis() }}",
        "symbol": "BTCUSDT",
        "recvWindow": 2000
      }
      ```
    - Connect from `Set Credentials`.

24. **Create Code Node to Convert Open Orders Params**
    - Name: `OpenOrder Query`
    - Same code.
    - Connect from `Open Orders Params`.

25. **Create Crypto Node to Sign Open Orders Query**
    - Name: `Signature OpenOrder`
    - Connect from `OpenOrder Query`.

26. **Create HTTP Request Node to Get Open Orders**
    - Name: `Get Open Orders`
    - Method: GET
    - URL: `=https://api.binance.com/api/v3/openOrders?{{ $json.queryString }}`
    - Query Parameters: `signature`
    - Headers: API key
    - Connect from `Signature OpenOrder`.

---

27. **Create Cancel All Orders Parameter Node**
    - Name: `Cancel All Order Params`
    - Type: Set
    - JSON Output:
      ```json
      {
        "timestamp": "{{ $now.toMillis() }}",
        "symbol": "BTCUSDT",
        "recvWindow": 2000
      }
      ```
    - Connect from `Set Credentials`.

28. **Create Code Node to Convert Cancel All Orders Params**
    - Name: `CancelAllOrder Query`
    - Same code.
    - Connect from `Cancel All Order Params`.

29. **Create Crypto Node to Sign Cancel All Orders Query**
    - Name: `Signature Cancel All Order`
    - Connect from `CancelAllOrder Query`.

30. **Create HTTP Request Node to Cancel All Orders**
    - Name: `Cancell All Order`
    - Method: DELETE
    - URL: `https://api.binance.com/api/v3/openOrders`
    - Body: query string
    - Query Parameters: `signature`
    - Headers: API key
    - Connect from `Signature Cancel All Order`.

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                                    |
|------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| กรุณาสร้าง Binance API Credentials ใน n8n: 1. ไปที่ Credentials > New 2. เลือก Binance API 3. ใส่ API Key และ Secret Key 4. ตั้งชื่อว่า 'Binance API' 5. Save 6. ทุก node จะใช้ credential เดียวกันนี้ | Instructional note in Thai inside the `Set Credentials` node for creating Binance API credentials in n8n.          |
| Binance API documentation: https://binance-docs.github.io/apidocs/spot/en/                                                               | Official API docs useful for understanding parameter details, error codes, and rate limits.                       |
| Binance API requires all signed endpoints to use HMAC SHA256 signature with query string including timestamp and recvWindow parameters. | Important for ensuring authentication correctness and avoiding request failures.                                   |
| Ensure system clock is synchronized to avoid timestamp-related request errors.                                                           | Common pitfall with Binance API is timestamp drift causing 400 errors.                                            |
| Quantity and price fields are hardcoded; adjust these nodes to accept dynamic input if needed.                                            | For production use, modify Set Parameter nodes to accept external input or variables.                               |
| Rate limits apply: excessive requests may cause IP bans or temporary block from Binance API.                                              | Consider adding rate limiting or delay nodes if integrating into fast loops.                                       |

---

**Disclaimer:** The provided text and workflow reflect an n8n automation setup strictly compliant with current content policies. No illegal, offensive, or protected data is included. All handled data is legal and public.