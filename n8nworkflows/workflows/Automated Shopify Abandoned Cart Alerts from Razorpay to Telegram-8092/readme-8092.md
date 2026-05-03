Automated Shopify Abandoned Cart Alerts from Razorpay to Telegram

https://n8nworkflows.xyz/workflows/automated-shopify-abandoned-cart-alerts-from-razorpay-to-telegram-8092


# Automated Shopify Abandoned Cart Alerts from Razorpay to Telegram

### 1. Workflow Overview

This workflow automates the retrieval and notification of abandoned Shopify carts, specifically via Razorpay order data, sending alerts to a Telegram chat. It targets e-commerce store operators who want timely notifications about pending or abandoned orders to improve follow-up and conversion. The logical flow is segmented into these blocks:

- **1.1 Scheduled Trigger**: A cron job triggers the workflow every 6 hours.
- **1.2 Time Window Calculation**: Computes a 2-hour time window for querying recent orders.
- **1.3 Razorpay Orders Retrieval**: Fetches orders from Razorpay API within the time window.
- **1.4 Orders Filtering & Formatting**: Filters orders with status "created" or "orderpending" and formats the order details into a message.
- **1.5 Telegram Notification**: Sends the formatted alert message to a specified Telegram chat.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:** This block schedules the workflow to run automatically every six hours.
- **Nodes Involved:**  
  - Cron (every 6h)
  - Sticky Note (You can set your timer here)

- **Node Details:**

  **Cron (every 6h)**  
  - Type: Cron Trigger  
  - Role: Initiates workflow execution every 6 hours.  
  - Configuration: Uses “everyX” mode with default 6-hour interval.  
  - Inputs: None (trigger node)  
  - Outputs: Passes empty data to next node  
  - Version: 1  
  - Potential Failures: None typical; misconfiguration could cause wrong timing.  
  - Sticky Note: "You can set you timer here" — guides user to change the frequency.

#### 1.2 Time Window Calculation

- **Overview:** Calculates the timestamp range for the last two hours to filter recent orders.
- **Nodes Involved:**  
  - Window (last 2h)  
  - Sticky Note1 (Here you can customise the time window that razorpay picks up)

- **Node Details:**

  **Window (last 2h)**  
  - Type: Code (JavaScript)  
  - Role: Outputs JSON object with `from` and `to` keys representing UNIX timestamps for now minus 2 hours and now, respectively.  
  - Configuration: Inline JS code uses `Date.now()` to calculate current Unix time in seconds, subtracts 7200 seconds (2 hours).  
  - Inputs: Trigger from Cron  
  - Outputs: Object `{ json: { from: <timestamp>, to: <timestamp> } }`  
  - Version: 2  
  - Edge Cases: Timezone independent as it uses Unix timestamp; no direct failures unless JS error occurs.  
  - Sticky Note: "Here you can customise the time window that razorpay picks up" — user can adjust the 2-hour window in code.

#### 1.3 Razorpay Orders Retrieval

- **Overview:** Calls Razorpay API to fetch orders, authenticating via HTTP Basic Auth.
- **Nodes Involved:**  
  - HTTP → Razorpay Orders  
  - Sticky Note2 (Add your razorpay creds here)

- **Node Details:**

  **HTTP → Razorpay Orders**  
  - Type: HTTP Request  
  - Role: Retrieves orders from Razorpay `/v1/orders` endpoint.  
  - Configuration:  
    - URL: `https://api.razorpay.com/v1/orders`  
    - Auth: HTTP Basic Auth (credentials stored in n8n credentials manager)  
  - Inputs: Receives time window data (though not directly used in URL parameters)  
  - Outputs: Raw JSON response of Razorpay orders list  
  - Version: 3  
  - Potential Failures:  
    - Authentication errors if credentials invalid  
    - Network timeouts or Razorpay API rate limits  
    - No query parameters filtering orders by date — may fetch all orders (note: could be optimized)  
  - Sticky Note: "Add your razorpay creds here" — user must provide Razorpay HTTP Basic credentials.

#### 1.4 Orders Filtering & Formatting

- **Overview:** Filters orders with status "created" or "orderpending" (interpreted as abandoned or pending), formats the data into a human-readable message for Telegram.
- **Nodes Involved:**  
  - Filter status=created + Format  
  - Sticky Note3 (Avoid customising this. This basically sorts and formats the data to send)

- **Node Details:**

  **Filter status=created + Format**  
  - Type: Code (JavaScript)  
  - Role:  
    - Filters orders based on status.  
    - Calculates order totals considering line items, COD fee, shipping fee, and promotions.  
    - Extracts customer name, email, contact, order creation time (formatted to IST timezone).  
    - Constructs a multi-line message block per order and joins all into a single message.  
  - Configuration:  
    - Defines helper functions for currency conversion, summing promotions, and picking customer info.  
    - Limits message to 40 orders max.  
    - Converts timestamps to localized string in IST.  
    - Status "created" shown as "abandoned".  
  - Inputs: Raw Razorpay orders JSON  
  - Outputs: Single JSON object with key `msg` containing the formatted message string.  
  - Version: 2  
  - Edge Cases:  
    - Missing customer details fall back to "Unknown" or `-`.  
    - Orders array empty or missing keys handled gracefully.  
    - Large orders list truncated to 40 to avoid message overload.  
    - Timezone formatting depends on environment support.  
  - Sticky Note: "Avoid customising this. This basically sorts and formats the data to send"

#### 1.5 Telegram Notification

- **Overview:** Sends the formatted order alert to a specified Telegram chat.
- **Nodes Involved:**  
  - Telegram → Me  
  - Sticky Note4 (Set your telegram creds here)

- **Node Details:**

  **Telegram → Me**  
  - Type: Telegram node  
  - Role: Sends a text message to a Telegram chat ID.  
  - Configuration:  
    - Text: Uses expression `{{$json.msg}}` to send formatted message from previous node.  
    - Chat ID: User must set their Telegram chat ID here.  
    - Credentials: Telegram API credentials configured separately.  
  - Inputs: Message string JSON from previous node  
  - Outputs: None (terminal node)  
  - Version: 1  
  - Edge Cases:  
    - Telegram API errors if credentials or chat ID incorrect.  
    - Message length limit (Telegram max message length ~4096 chars) — large messages may get truncated or fail.  
  - Sticky Note: "Set your telegram creds here"

---

### 3. Summary Table

| Node Name                | Node Type          | Functional Role                      | Input Node(s)          | Output Node(s)          | Sticky Note                                           |
|--------------------------|--------------------|------------------------------------|-----------------------|-------------------------|-------------------------------------------------------|
| Cron (every 6h)          | Cron Trigger       | Scheduled trigger every 6 hours     | —                     | Window (last 2h)         | You can set you timer here                            |
| Window (last 2h)          | Code (JavaScript)  | Calculate last 2-hour time window   | Cron (every 6h)        | HTTP → Razorpay Orders   | Here you can customise the time window that razorpay picks up |
| HTTP → Razorpay Orders    | HTTP Request       | Retrieve Razorpay orders via API    | Window (last 2h)       | Filter status=created + Format | Add your razorpay creds here                         |
| Filter status=created + Format | Code (JavaScript)  | Filter relevant orders and format message | HTTP → Razorpay Orders | Telegram → Me            | Avoid customising this. This basically sorts and formats the data to send |
| Telegram → Me            | Telegram           | Send formatted message to Telegram  | Filter status=created + Format | —                     | Set your telegram creds here                          |
| Sticky Note              | Sticky Note        | User guidance on timer setting      | —                     | —                       | You can set you timer here                            |
| Sticky Note1             | Sticky Note        | Guidance on time window customization | —                     | —                       | Here you can customise the time window that razorpay picks up |
| Sticky Note2             | Sticky Note        | Guidance on Razorpay credentials    | —                     | —                       | Add your razorpay creds here                          |
| Sticky Note3             | Sticky Note        | Warning not to customize formatting | —                     | —                       | Avoid customising this. This basically sorts and formats the data to send |
| Sticky Note4             | Sticky Note        | Guidance on Telegram credentials     | —                     | —                       | Set your telegram creds here                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cron Trigger Node**  
   - Type: Cron Trigger  
   - Name: "Cron (every 6h)"  
   - Set Mode to “Every X” with interval 6 hours (default).  
   - Connect no inputs; this is the entry node.

2. **Create Code Node for Time Window**  
   - Type: Code (JavaScript)  
   - Name: "Window (last 2h)"  
   - Add JS Code:
     ```javascript
     const now = Math.floor(Date.now()/1000);
     return [{json: {from: now - 2*3600, to: now}}];
     ```
   - Connect input from Cron node.

3. **Create HTTP Request Node to Razorpay**  
   - Type: HTTP Request  
   - Name: "HTTP → Razorpay Orders"  
   - URL: `https://api.razorpay.com/v1/orders`  
   - Authentication: HTTP Basic Auth  
   - Configure credentials with Razorpay API key and secret in n8n credentials manager.  
   - Connect input from "Window (last 2h)".

4. **Create Code Node to Filter and Format Orders**  
   - Type: Code (JavaScript)  
   - Name: "Filter status=created + Format"  
   - Paste the following JS code:
     ```javascript
     function rupees(p) { return Math.round((p||0)/100); }
     function sumPromo(ps) { return (ps||[]).reduce((s,p)=> s + (p?.value||0), 0); }
     function pickName(o) { return o?.customer_details?.shipping_address?.name || o?.customer_details?.name || o?.notes?.name || 'Unknown'; }
     function pickContact(o) { return o?.customer_details?.contact || o?.customer_details?.shipping_address?.contact || o?.notes?.contact || '-'; }
     function pickEmail(o) { return o?.customer_details?.email || o?.notes?.email || '-'; }
     function fmtIST(sec) { return new Date((sec||Math.floor(Date.now()/1000))*1000).toLocaleString('en-IN',{timeZone:'Asia/Kolkata', hour12:false}); }

     const body = items[0]?.json;
     const orders = Array.isArray(body) ? body : (body?.items || [body]).filter(Boolean);
     const wanted = orders.filter(o => ['created','orderpending'].includes(String(o?.status||'').toLowerCase()));
     if (!wanted.length) return [];

     const blocks = wanted.slice(0, 40).map(o => {
       const line = (o?.line_items_total ?? o?.customer_details?.line_items_total ?? o?.amount ?? 0);
       const cod = o?.cod_fee || 0;
       const ship = o?.shipping_fee || 0;
       const promo = sumPromo(o?.promotions);
       const val = Math.max(0, line + cod + ship - promo);
       const name = pickName(o);
       const email = pickEmail(o);
       const contact = pickContact(o);
       const rawStatus = String(o?.status||'').toLowerCase();
       const statusLabel = rawStatus === 'created' ? 'abandoned' : rawStatus;
       const when = fmtIST(o?.created_at);
       return `name: ${name}\nvalue: ${rupees(val)}\nstatus: ${statusLabel}\ncontact: ${contact}\nemail: ${email}\ntime: ${when}`;
     }).join('\n\n');

     const msg = `🟡 abandoned via webhook\n\n${blocks}`;
     return [{ json: { msg } }];
     ```
   - Connect input from HTTP Request node.

5. **Create Telegram Node**  
   - Type: Telegram  
   - Name: "Telegram → Me"  
   - Credentials: Configure Telegram Bot API credentials in n8n.  
   - Text: Set to expression `{{$json.msg}}` to send the formatted message.  
   - Chat ID: Set your Telegram chat ID to receive alerts.  
   - Connect input from the filter/format code node.

6. **Add Sticky Notes (Optional for User Guidance)**  
   - Create Sticky Note nodes near relevant nodes to guide users on configuration:
     - For Cron: “You can set you timer here”
     - For Time Window Code: “Here you can customise the time window that razorpay picks up”
     - For HTTP Request: “Add your razorpay creds here”
     - For Filter/Format Code: “Avoid customising this. This basically sorts and formats the data to send”
     - For Telegram Node: “Set your telegram creds here”

---

### 5. General Notes & Resources

| Note Content                                                                              | Context or Link                                             |
|-------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| The workflow relies on Razorpay API returning all orders; consider adding query params for date filtering if supported to reduce data volume. | Razorpay API docs: https://razorpay.com/docs/api/orders/    |
| Telegram message length limit (~4096 characters) may truncate long alerts; consider pagination or limiting orders further. | Telegram Bot API docs: https://core.telegram.org/bots/api  |
| Time formatting uses IST timezone explicitly; adjust if targeting other regions.           | JavaScript Date.toLocaleString timeZone option              |
| Ensure Razorpay credentials have API access and Telegram Bot token has permissions to send messages to your chat. | Setup instructions on respective platforms                   |

---

**Disclaimer:** This documentation is based exclusively on the provided n8n workflow JSON. It complies fully with content policies and contains no illegal or offensive elements. All data handled is public and legal.