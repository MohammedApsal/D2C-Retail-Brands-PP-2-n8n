📦 AI-Powered WISMO Support Automation for D2C & Retail

A production-ready n8n automation architecture designed to solve one of the most repetitive customer-support problems for D2C and retail businesses:

“Where Is My Order?”

Customers can ask for order status through WhatsApp or Instagram, while the system automatically identifies the order, retrieves shipment information, checks courier data when required, and returns a controlled tracking response.

The system also runs a background shipment-sync engine that keeps shipment information updated and sends proactive notifications for important delivery milestones.

🎯 Business Pain Point

D2C and retail brands frequently receive repetitive messages such as:

“Where is my order?”
“Has my order shipped?”
“Where is my parcel?”
“Can you send the tracking link?”
“When will my order arrive?”

Handling these questions manually consumes support-team time and creates unnecessary repetitive work.

This automation is designed to turn that process into an automated customer-tracking system.

🏗️ Architecture

The solution is intentionally built around two core workflows.

                    ┌──────────────────────────┐
                    │   Customer WhatsApp /    │
                    │       Instagram          │
                    └────────────┬─────────────┘
                                 │
                                 ▼
              ┌───────────────────────────────────┐
              │ WF-1 — Customer WISMO Tracking    │
              │                                   │
              │ Webhook → Verify → Normalize      │
              │ → Identify Customer → AI Intent    │
              │ → Order Lookup → Shipment Lookup  │
              │ → Courier API → Response           │
              └─────────────────┬─────────────────┘
                                │
                                ▼
                       Customer receives
                       tracking information


              ┌───────────────────────────────────┐
              │ WF-2 — Shipment Status Sync       │
              │                                   │
              │ Schedule → Claim Shipments        │
              │ → Courier API → Normalize Status  │
              │ → Compare State → Update DB       │
              │ → Important Event?                │
              │ → WhatsApp Notification            │
              └───────────────────────────────────┘
1️⃣ WF-1 — Customer WISMO Tracking

This workflow handles incoming customer tracking requests from WhatsApp and Instagram.

The workflow includes webhook verification for Meta, message normalization, idempotency handling, customer identification, AI intent classification, customer-scoped order lookup, shipment retrieval, courier checks, and response delivery.

Flow
WhatsApp / Instagram Webhook
        ↓
Webhook Verification
        ↓
Signature Validation
        ↓
Normalize Incoming Message
        ↓
Validate Message
        ↓
Idempotency Check
        ↓
Identify Customer
        ↓
Load Conversation State
        ↓
AI Intent Classification
        ↓
Validate AI Output
        ↓
Find Relevant Order(s)
        ↓
Retrieve Shipment
        ↓
Check Shipment Freshness
        ↓
Courier API (when required)
        ↓
Normalize Shipment Data
        ↓
Build Customer Response
        ↓
WhatsApp / Instagram
AI Responsibility

The AI layer is deliberately limited to understanding the customer's message, rather than deciding shipment facts.

It can classify requests as:

TRACK_ORDER
NON_TRACKING
UNCLEAR

and extract:

order_number
tracking_number
confidence

The AI instructions explicitly prohibit inventing order numbers, AWBs, or tracking statuses.

2️⃣ WF-2 — Shipment Status Sync

WF-2 operates as an autonomous background synchronization engine.

It runs every 15 minutes, claims eligible shipments using database row locking and leases, queries supported courier APIs, normalizes their responses, compares them with the previous shipment state, and updates PostgreSQL.

Supported Courier Integrations
Shiprocket
Delhivery

Courier routing determines which adapter is used for each shipment.

Flow
Schedule Trigger
      ↓
Claim Active Shipments
      ↓
Lease-Based Concurrency Control
      ↓
Determine Courier
      ↓
Courier Router
      ├── Shiprocket
      ├── Delhivery
      └── Unsupported Courier
      ↓
Normalize Courier Response
      ↓
Compare With Previous State
      ↓
Shipment Changed?
      ↓
Atomic Database Update
      ↓
Important Event?
      ↓
Notification Eligibility
      ↓
WhatsApp Notification

The system also maintains retry handling for failed notifications and prevents outdated milestones from being resent when a newer shipment state has already been recorded.

🚚 Shipment Lifecycle

Courier-specific statuses are normalized into a common internal state model.

pending
confirmed
picked_up
in_transit
out_for_delivery
delivered
delivery_exception
cancelled
rto
lost
unknown

This allows the rest of the automation to work with a consistent shipment model even when courier APIs return different status formats.

🔔 Proactive Customer Notifications

The system can proactively notify customers when important shipment milestones occur.

Examples include:

🚚 Out for Delivery
✅ Delivered
⚠️ Delivery Exception
⚠️ RTO
⚠️ Lost Shipment

Notifications are protected by:

deterministic notification keys
database-based locking
notification eligibility checks
retry handling
attempt limits
milestone validation

WhatsApp notifications are sent through the Meta Graph API.

🔐 Reliability & Security

A major focus of this project was making the workflows suitable for real-world failure conditions, rather than building a simple demo automation.

Webhook Security

Meta webhook requests are validated using:

Verify-token validation
HMAC SHA-256 signature verification
Raw request-body verification

Idempotency

Inbound messages are stored using deterministic event keys so duplicate webhook deliveries do not create duplicate processing. Failed or abandoned events can also be reclaimed.

Customer Data Isolation

Order and shipment lookups are scoped to the authenticated customer rather than allowing arbitrary customer-number searches.

Concurrency Control

Shipment processing uses PostgreSQL row locking with:

FOR UPDATE SKIP LOCKED

and durable leases to reduce concurrent execution collisions.

🔄 Failure Recovery

The system includes recovery mechanisms for both inbound events and proactive notifications.

Inbound Recovery

Abandoned or failed inbound events can be reclaimed using:

processing + expired lease
            OR
failed + retry available

with bounded attempts.

Notification Recovery

Failed or expired notification jobs are periodically claimed and retried, with attempt limits and shipment-state validation.

🗄️ Data Layer

The automation uses PostgreSQL as the source of truth for customer, order, shipment, idempotency, notification, and shipment-event data.

Key entities used by the workflow include:

customers
orders
shipments
shipment_events
channel_identities
idempotency_events
notification_events
🛠️ Tech Stack
Technology	Purpose
n8n	Workflow orchestration
PostgreSQL	Persistent data + state management
AI Agent	Intent classification + entity extraction
WhatsApp Cloud API	Customer messaging
Instagram	Customer messaging
Shiprocket API	Shipment tracking
Delhivery API	Shipment tracking
JavaScript	Validation + normalization + business logic
📈 Business Impact

This automation is designed to help D2C and retail brands:

Reduce repetitive WISMO support conversations
Provide faster shipment information
Automate routine tracking requests
Keep shipment data synchronized
Proactively notify customers about important delivery milestones
Reduce manual support workload
Create a consistent tracking experience across messaging channels
🔧 Configuration Requirements

Before deployment, configure the required credentials and environment variables for:

PostgreSQL
META_VERIFY_TOKEN
META_APP_SECRET
INSTAGRAM_APP_SECRET
META_SYSTEM_USER_TOKEN
WHATSAPP_PHONE_NUMBER_ID
SHIPROCKET_JWT
DELHIVERY_AUTH_TOKEN

The workflow should be tested with real webhook payloads, courier responses, database records, and Meta messaging credentials before being considered operational in a live business environment.

⚠️ Delivery Semantics

The workflow is designed for robust at-least-once processing.

For outbound WhatsApp notifications, there is an unavoidable edge case between an external API accepting a message and PostgreSQL recording the sent state. The workflow mitigates this with deterministic keys, leases, and retry logic, but the architecture does not claim mathematically guaranteed exactly-once external delivery.

📂 Workflow Structure
D2C & Retail WISMO Automation
│
├── WF-1 — Customer WISMO Tracking
│   ├── WhatsApp Webhook
│   ├── Instagram Webhook
│   ├── Meta Verification
│   ├── Signature Validation
│   ├── Idempotency
│   ├── Customer Identification
│   ├── AI Intent Classification
│   ├── Order Lookup
│   ├── Shipment Lookup
│   ├── Courier Tracking
│   └── Customer Response
│
└── WF-2 — Shipment Status Sync
    ├── Scheduled Polling
    ├── Shipment Claiming
    ├── Lease Management
    ├── Shiprocket Adapter
    ├── Delhivery Adapter
    ├── Status Normalization
    ├── State Comparison
    ├── Atomic Database Update
    ├── Notification Eligibility
    └── Notification Retry
🚀 Project Summary

Project: AI-Powered WISMO Support Automation
Industry: D2C & Retail
Problem: Repetitive “Where Is My Order?” support requests
Platform: n8n
Database: PostgreSQL
Channels: WhatsApp + Instagram
Courier APIs: Shiprocket + Delhivery
AI Role: Intent classification + reference extraction
Architecture: 2-workflow deterministic tracking system

The goal was not just to connect APIs. It was to design a reliable automation that can handle real customer messages, shipment synchronization, duplicate events, courier failures, retries, and proactive delivery notifications.
