### Workflow Inventory

1. **01 - WhatsApp Incoming Message Handler**  
   - **Trigger:** `Webhook` node (`/whatsapp/inbound`) receiving Facebook WhatsApp Cloud POST payloads.  
   - **Node Flow:** Webhook → Function normalize → IF skip non-text → Google Sheets lookup → (optional append new lead) → Function lead context → IF duplicate guard → OpenAI chat completion → HTTP WhatsApp reply → Google Sheets update → Switch intent router → (Execute Order Capture | Telegram handoff | Opt-out handling) → Respond Webhook.  
   - **External APIs:**  
     - WhatsApp Cloud API `POST /{phone-id}/messages` for outbound replies.  
     - OpenAI Chat Completions.  
     - Google Sheets API via service account (lookup/update/append).  
     - Telegram Bot API `sendMessage` for human handoff.  
   - **Data Contracts:** expects `ORDERS` sheet schema; AI response JSON includes `reply`, `intent`, `collect_field`, `field_value`, `currency_amount`. Dedupe using `Last Message ID`.

2. **02 - Order Capture Engine**  
   - **Trigger:** `Execute Workflow` call from Workflow 01 when AI intent=`placing_order`.  
   - **Node Flow:** Start → Function parse payload → Google Sheets lookup by phone → Merge row data → Determine next missing field → IF all fields captured → (history update → sheet status NEW_ORDER → summary message → Execute order notification) else (history update → sheet status QUALIFYING).  
   - **External APIs:** WhatsApp Cloud API for order summary, Google Sheets for state persistence.  
   - **State Machine:** Maintains `Status` transitions `QUALIFYING` → `NEW_ORDER_PENDING` → `NEW_ORDER`. Appends structured objects to `Order History JSON`.  
   - **Outputs:** Triggers Workflow 03 and surfaces `nextField` hint for AI context (stored in execution data for future iterations).

3. **03 - Order Notification Broadcast**  
   - **Trigger:** `Execute Workflow` call from Workflow 02 once order finalized.  
   - **Node Flow:** Start → Function prepare Telegram payload with inline buttons → HTTP POST to Telegram `sendMessage`.  
   - **Inline Buttons:** `dispatch:{orderId}`, `cancel:{orderId}` consumed by Workflow 04.  

4. **04 - Dispatch Rider Assignment**  
   - **Trigger:** Telegram Trigger node listening for callback queries.  
   - **Node Flow:** Telegram trigger → Parse callback data → Switch (dispatch/cancel).  
     - **Dispatch path:** Google Sheets lookup by `ID` → Build rider message (resolve `Assigned Rider` or default) → WhatsApp Cloud API message to rider → Sheet update `Status=DISPATCHED` → Telegram confirmation.  
     - **Cancel path:** Google Sheets update `Status=CANCELLED` → Telegram confirmation.  
   - **Assumptions:** Riders reply via WhatsApp numbers stored in sheet or env `DEFAULT_RIDER_WHATSAPP`.  

5. **05 - Rider Status Updates**  
   - **Trigger:** Webhook at `/whatsapp/rider` dedicated to rider replies. Configure WhatsApp Cloud webhook subscription for rider phone numbers.  
   - **Node Flow:** Webhook → Parse text → IF actionable keyword → Determine lookup column (order ID or assigned rider) → Google Sheets lookup → Prepare status update → Sheet update → Telegram notify owner → WhatsApp acknowledgement to rider → Respond to webhook. Non-text/unknown commands short-circuit with ack.  
   - **Supported Keywords:** `DELIVERED {orderId}`, `FAILED {orderId}` (orderId optional if only one active dispatch per rider).  

6. **99 - Global Error Logger**  
   - **Trigger:** n8n Error Trigger subscribed globally.  
   - **Node Flow:** Error Trigger → Format payload → Append to `ERROR_LOG` sheet → Telegram alert to owner.  
   - **Use:** Centralizes exceptions from all workflows; escalate critical failures via Telegram.

### Environment Variable Usage

| Variable | Purpose |
| --- | --- |
| `N8N_WHATSAPP_VERIFY_TOKEN` | Meta webhook verification handshake for `/whatsapp/inbound` and `/whatsapp/rider`. |
| `N8N_WHATSAPP_ACCESS_TOKEN` | Bearer token for all WhatsApp Graph API requests. |
| `N8N_WHATSAPP_PHONE_ID` | Path parameter for outbound WhatsApp messaging API. |
| `N8N_WHATSAPP_BUSINESS_ID` | Optional for logging/analytics. |
| `OPENAI_API_KEY`, `OPENAI_MODEL`, `OPENAI_TEMPERATURE` | Configure OpenAI Chat node. |
| `GOOGLE_SHEETS_SERVICE_ACCOUNT_EMAIL`, `GOOGLE_SHEETS_PRIVATE_KEY`, `GOOGLE_SHEETS_DOC_ID` | Google Sheets integration. |
| `TELEGRAM_BOT_TOKEN`, `TELEGRAM_OWNER_CHAT_ID`, `TELEGRAM_DISPATCH_RIDER_CHAT_ID` | Telegram trigger + notifications. |
| `DEFAULT_RIDER_WHATSAPP`, `DEFAULT_WAREHOUSE_ADDRESS` | Dispatch defaults when sheet lacks rider contact. |

### Suggested Naming & Folders (n8n)

- **Folder:** `WhatsApp Sales Automation`
  - `01 - WhatsApp Incoming Message Handler`
  - `02 - Order Capture Engine`
  - `03 - Order Notification Broadcast`
  - `04 - Dispatch Rider Assignment`
  - `05 - Rider Status Updates`
  - `99 - Global Error Logger`

### API Endpoints Summary

- **WhatsApp Cloud API**
  - `POST /{PHONE_ID}/messages` — send customer and rider messages.
  - Webhook subscription for `/whatsapp/inbound` and `/whatsapp/rider`.
- **OpenAI API**
  - `POST /v1/responses` (handled by n8n OpenAI node).
- **Google Sheets API**
  - `spreadsheets.values.get` (lookup), `spreadsheets.values.append`, `spreadsheets.values.update`.
- **Telegram Bot API**
  - `POST /bot{TOKEN}/sendMessage`
  - Webhook to `04` workflow for callback queries.

### Data Integrity Notes

- `Last Message ID` prevents duplicate processing of WhatsApp retry payloads.
- `Order History JSON` stores chronological events for audit and replays.
- Status transitions allowed: `QUALIFYING` → `NEW_ORDER_PENDING` → `NEW_ORDER` → `DISPATCHED` → (`DELIVERED` | `FAILED` | `CANCELLED`).
- All workflows push exceptions into `ERROR_LOG` sheet.
