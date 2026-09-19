# MerchantPulse 🚀
**Track 03:** Subscribing to Notifications  
**Powered by:** Fastn Embedded Integration Platform & MCP Agent  

---

## 1. Project Overview
MerchantPulse is a multi-tenant ecommerce notification SaaS designed for online merchants. It allows store owners to connect their own Shopify stores and Slack workspaces via Fastn embedded widgets, set dynamic alert rules (e.g., high-value orders, low-inventory warnings, unfulfilled order reminders), and receive instant notifications while eliminating notification fatigue through deterministic deduplication.

---

## 2. Problem
Ecommerce merchants are bombarded with dozens or hundreds of store notifications each day:
* Every order, regardless of value, generates an alert.
* Critical high-value orders get lost in the noise.
* Inventory shortages go unnoticed until stockouts occur.
* Webhook retries from ecommerce platforms result in duplicated notifications across team channels.

---

## 3. Solution
MerchantPulse filters the noise by letting each merchant define their own notification criteria:
1. **Dynamic Thresholds:** Notify only when order value exceeds merchant-selected thresholds (e.g., $300 or $500).
2. **Proactive Inventory Warnings:** Immediate alerts when stock levels drop below restock thresholds.
3. **Scheduled Reminders:** Automated reminders for orders awaiting fulfillment over 24 hours.
4. **Zero-Fatigue Idempotency:** State-backed deduplication ensures repeated webhook events never create duplicate alerts.

---

## 4. Challenge Track
**Track 03 — Subscribing to Notifications**  
* Focus: Connecting external event producers (Shopify), applying customer-specific filters/thresholds, and publishing high-signal notifications to communication destinations (Slack).

---

## 5. Architecture
```
[Shopify Store] ──(Webhooks/Events)──> [Fastn Inbound Trigger]
                                              │
                                              ▼
                                 [Fastn Workflow Sandbox]
                                 ├─ Resolve Tenant
                                 ├─ Normalize Event
                                 ├─ Check Idempotency (fastn.state)
                                 ├─ Evaluate Rule Threshold
                                 └─ Format Alert
                                              │
                                              ▼
[Customer Slack Workspace] <───── [Fastn Slack Connector]
```

---

## 6. Customer Journey
1. Merchant signs into MerchantPulse.
2. Merchant connects Shopify and Slack via Fastn's embedded integration widget.
3. Merchant sets a high-value order threshold (e.g. 500 USD or 300 PKR) and selects their preferred Slack channel.
4. A customer places an order on the merchant's Shopify store.
5. Fastn ingests the order webhook, verifies tenant isolation, evaluates the threshold, and checks idempotency.
6. The merchant's Slack channel instantly receives a formatted alert.

---

## 7. Fastn Architecture
* **V8 Sandbox Execution:** Workflows run inside Fastn's isolated V8 runtime with sub-millisecond connector invocations.
* **State Machine (`fastn.state`):** Durable, org-scoped key-value storage used to maintain idempotency keys and per-merchant preferences across runs.
* **Connector Manifest:** Manifest set to `MULTI_TENANT`, resolving individual credentials per tenant via `x-end-org-id`.
* **Connector Retries:** Automated backoff policy retrying network failures and HTTP `429, 500, 502, 503, 504`.

---

## 8. Fastn MCP Platform Agent Usage
The entire integration infrastructure was built and verified programmatically via Fastn's Platform Agent and MCP tools:
* **Discovery:** Inspected Shopify and Slack connector schemas, auth configurations, and events via `listConnectors`, `listActions`, and `listConnectorEvents`.
* **Widget & Embed:** Provisioned `wgt_80cc3b9975db` and created setup links via `createSetupLink`.
* **Multi-Tenant Scoping:** Configured child orgs `merchant_001` and `merchant_002` via `createTenant` and tuned manifests with `setWorkflowConnectorScope`.
* **Workflow Authoring:** Generated and published three workflows (`createWorkflow`, `publishWorkflow`).
* **Testing & Diagnostics:** Executed real-time sandbox tests, debugged execution logs via `getWorkflowExecution`, and recorded full-suite validation with `saveWorkflowValidation`.

---

## 9. Key Features
* 🔴 **High-Value Order Alerts:** Instantly notifies merchants of large sales.
* 🟠 **Low-Inventory Alerts:** Warns when SKUs drop below safety stock.
* ⏳ **Unfulfilled Order Reminders:** Automated hourly scheduler alerting on aging orders.
* 🛡️ **Built-in Deduplication:** Eliminates repeated webhook alerts.
* 🏢 **Multi-Tenant Isolation:** Complete data and credential separation per customer.

---

## 10. Workflow Details
1. **`shopify-smart-alert-router` (`wf_0d303f6ca53c`, Published v1)**
   * Inbound webhook: `3237dd61-cd14-4833-a125-f2f40bb18bb8`
2. **`shopify-low-inventory-alert` (`wf_17fd5a5e16c5`, Published v1)**
   * Inventory webhook & direct execution endpoint
3. **`shopify-unfulfilled-orders-reminder` (`wf_fb10d8da2938`, Published v1)**
   * Cron scheduler: `9a057da9-3eb2-4bbe-bae9-242faa9ea3ea` (`0 * * * *`)

---

## 11. Deduplication Strategy
Deterministic idempotency keys are stored in `fastn.state`:
* **Orders:** `${tenant_id}:order_${order_id}:high_value_order`
* **Inventory:** `${tenant_id}:inventory_${item_id}:low_stock` (dedupes against identical remaining count)
* **Unfulfilled Orders:** `${tenant_id}:unfulfilled_order_${order_id}:reminder` (24-hour cooldown)

---

## 12. Error Handling & Retries
* Malformed payloads safely return `status: "failed"` without crashing the engine.
* Transient Slack errors automatically retry up to 3 times with exponential backoff (`initialDelayMs: 1000`, `backoffMultiplier: 2`).

---

## 13. Tenant Isolation
* Every request carries `x-end-org-id`.
* The connector manifest is set to `MULTI_TENANT`, dynamically resolving the calling tenant's connection while clearing global pool pins.
* Keys in `fastn.state` are prefixed by tenant ID, preventing cross-tenant collision.

---

## 14. Setup & Installation
1. Clone this repository.
2. Set up environment variables in `.env` (see below).
3. In Shopify Admin → **Settings** → **Notifications** → **Webhooks**:
   * Add Order Creation webhook pointing to your Fastn Webhook URL.
4. Authorize your Slack workspace via the Fastn connection experience.

---

## 15. Environment Variables
* `FASTN_API_URL`
* `FASTN_API_KEY`
* `FASTN_ORG_ID`
* `FASTN_WIDGET_ID`
* `FASTN_ORDER_WEBHOOK_URL`
* `FASTN_ENVIRONMENT`

---

## 16. Running Locally
```bash
npm install
npm run dev
```
Open `http://localhost:3000` to access the MerchantPulse dashboard.

---

## 17. Testing
Workflows were tested in Fastn's live V8 sandbox with 100% test case pass rates:
* **Order > Threshold:** Delivered to Slack (`ts: 1789804981.261449`).
* **Order < Threshold:** Safely suppressed (`no_notification_required`).
* **Duplicate Event:** Rejected by state check (`duplicate: true`).
* **Low Inventory:** Delivered to Slack (`ts: 1789802818.543759`).
* **Unfulfilled Reminder:** Delivered to Slack (`ts: 1789802998.492169`).

---

## 18. Screenshots Needed for Submission
1. **Shopify Webhooks Page:** Showing webhook URL pointing to Fastn.
2. **Slack Live Alerts:** The Fastn app bot delivering Order `#9999` (404.95 PKR) and Low Stock alerts.
3. **Fastn Execution Monitor:** Showing completed execution `exec_956f2de8fcc2` and `exec_552435adf937` (duplicate blocked).
4. **Fastn Workflow Editor:** Showing the published `shopify-smart-alert-router`.

---

## 19. Demo Instructions (2 Minutes)
1. **Introduction (0:00 - 0:20):** Explain notification fatigue.
2. **Fastn Setup (0:20 - 0:45):** Show the Fastn widget and Slack connected.
3. **Live Trigger (0:45 - 1:15):** Trigger "Send test notification" in Shopify Admin.
4. **Alert Arrival (1:15 - 1:35):** Show the formatted alert pop up in Slack with order total and line items.
5. **Deduplication Proof (1:35 - 1:50):** Trigger it again and show the Fastn execution log blocking the duplicate.
6. **Wrap Up (1:50 - 2:00):** Highlight Fastn MCP tooling and multi-tenant capabilities.

---

## 20. Known Limitations
* Shopify webhook test notifications always send a fixed test payload (`#9999`, 404.95 PKR) rather than arbitrary dynamic orders.
* Slack OAuth requires public channel bot membership or direct messaging.

---

## 21. Future Improvements
* Optional email fallback via Gmail connector if Slack delivery encounters extended downtime.
* Merchant configurable quiet hours (e.g. 10 PM - 8 AM) with morning digest delivery.
