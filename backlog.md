## **Epic 1 – Project Setup & Infrastructure**

**Goal:** Establish the technical foundation for backend (FastAPI), frontend (React + Tailwind), database (ZeroDB with vectors), and integrations.

### **User Story 1.1 – Backend Scaffold**

* **As a** developer
* **I want** a FastAPI application skeleton with environment config, database connection, and initial routes
* **So that** we have a working base for building features.
  **Acceptance Criteria:**
* `.env` file supports ZeroDB, ZeroInvoice, Stripe, Twilio/SendGrid, GA IDs.
* `/health` endpoint returns `"status": "ok"`.
* pgvector extension enabled in ZeroDB.

### **User Story 1.2 – Frontend Scaffold**

* **As a** developer
* **I want** a React app with Tailwind configured
* **So that** we can quickly build responsive UI components.
  **Acceptance Criteria:**
* TailwindCSS working with dark/light modes.
* Base layout with header, nav, and placeholder content.

### **User Story 1.3 – CI/CD & Dockerization**

* **As a** developer
* **I want** backend and frontend containerized with Docker and CI pipeline set up
* **So that** deployments and local dev are consistent.
  **Acceptance Criteria:**
* `docker-compose up` starts API + frontend + DB.
* GitHub Actions runs tests on push to `main`.

---

## **Epic 2 – Customer Management (CRM)**

**Goal:** Create and manage customer & lead records with vector search for segmentation.

### **User Story 2.1 – Add Customer**

* **As a** staff user
* **I want** to add a new customer or lead to the system
* **So that** I can store their contact details and service history.
  **Acceptance Criteria:**
* Required fields: name, type (customer/lead).
* Optional: email, phone, address, tags.
* Customer stored in ZeroDB with vector embedding generated.

### **User Story 2.2 – Edit & Archive Customer**

* **As a** staff user
* **I want** to update or archive a customer record
* **So that** records stay accurate without losing historical data.
  **Acceptance Criteria:**
* Updates save without breaking vector embedding.
* Archive flag hides customer from active lists.

### **User Story 2.3 – Search Customers (Vector + Filters)**

* **As a** staff user
* **I want** to search customers by name, tags, and semantic similarity
* **So that** I can segment and target them for campaigns.
  **Acceptance Criteria:**
* Vector search returns customers with cosine similarity > threshold.
* Filters by zip, type, or tags.

---

## **Epic 3 – Invoicing via ZeroInvoice API**

**Goal:** Create, send, and track invoices through ZeroInvoice with Stripe payment links.

### **User Story 3.1 – Create Invoice**

* **As a** staff user
* **I want** to create an invoice linked to a customer
* **So that** they can be billed for services.
  **Acceptance Criteria:**
* Backend calls ZeroInvoice API.
* Invoice stored in `zeroinvoice_invoices_cache`.
* Stripe payment link returned.

### **User Story 3.2 – Send Invoice (Email/SMS)**

* **As a** staff user
* **I want** to send an invoice to the customer via email and/or SMS
* **So that** they can pay promptly.
  **Acceptance Criteria:**
* Option to select delivery method.
* Delivery status saved in cache table.

### **User Story 3.3 – Invoice Webhook Sync**

* **As a** system
* **I want** to update invoice/payment status via webhook
* **So that** the app reflects real-time payment info.
  **Acceptance Criteria:**
* Paid invoices marked as paid within 5 seconds of webhook.
* Payment details saved in `payments` table.

---

## **Epic 4 – Email & SMS Marketing**

**Goal:** Build and send targeted campaigns to customers and leads.

### **User Story 4.1 – Create Template**

* **As a** staff user
* **I want** to design an email or SMS message with placeholders
* **So that** I can reuse it in multiple campaigns.
  **Acceptance Criteria:**
* Templates stored with metadata (subject, channel, content).
* Placeholders supported (e.g., `{{customer.name}}`).

### **User Story 4.2 – Create Campaign**

* **As a** staff user
* **I want** to create a campaign targeting a specific segment
* **So that** I can send promotions to the right audience.
  **Acceptance Criteria:**
* Select template, segment, and schedule.
* Save campaign as draft until launched.

### **User Story 4.3 – Send Campaign & Track**

* **As a** system
* **I want** to send campaign messages via Twilio/SendGrid
* **So that** recipients receive the promotion.
  **Acceptance Criteria:**
* Messages created in `messages` table.
* Open/click/bounce events stored in `message_events`.

---

## **Epic 5 – Conversion Tracking & Analytics**

**Goal:** Track and attribute campaign conversions to revenue.

### **User Story 5.1 – Attribute Conversions**

* **As a** system
* **I want** to link invoice payments to the last marketing touch
* **So that** I can measure campaign ROI.
  **Acceptance Criteria:**
* Payment webhook triggers attribution lookup.
* Store `campaign_run_id`, `message_id`, and `revenue_cents`.

### **User Story 5.2 – Google Analytics Integration**

* **As a** system
* **I want** to send page and event data to GA
* **So that** I can track off-platform behavior.
  **Acceptance Criteria:**
* UTM params from campaigns persist to GA events.

### **User Story 5.3 – Dashboard KPIs**

* **As a** staff user
* **I want** to see performance metrics in one place
* **So that** I can make data-driven decisions.
  **Acceptance Criteria:**
* Revenue from campaigns, open/click rates, payment rates visible.
* Date range filters.

---

## **Epic 6 – Security, Permissions & Compliance**

**Goal:** Enforce org-based data access, protect privacy, and support marketing opt-outs.

### **User Story 6.1 – JWT Authentication**

* **As a** system
* **I want** to authenticate API calls with JWT tokens
* **So that** only authorized users access data.
  **Acceptance Criteria:**
* JWT required for all org endpoints.
* Token expiration handled gracefully.

### **User Story 6.2 – Org Data Isolation**

* **As a** system
* **I want** to scope all queries to `organization_id`
* **So that** no org can access another’s data.
  **Acceptance Criteria:**
* All queries include `organization_id` filter.

### **User Story 6.3 – Marketing Opt-Out**

* **As a** customer
* **I want** to unsubscribe from marketing messages
* **So that** I don’t receive unwanted communication.
  **Acceptance Criteria:**
* `/unsubscribe` endpoint updates `subscriptions` table.
* Opt-outs respected in send pipelines.

---

## **Epic 7 – Hackathon Demo Prep**

**Goal:** Ensure smooth end-to-end flow for presentation.

### **User Story 7.1 – Seed Demo Data**

* **As a** developer
* **I want** pre-populated demo data for customers, invoices, campaigns
* **So that** I can run a live demo without manual setup.
  **Acceptance Criteria:**
* Script generates at least 10 customers, 3 invoices, 2 campaigns.

### **User Story 7.2 – E2E Flow Test**

* **As a** developer
* **I want** to run through the complete flow from customer creation to campaign ROI reporting
* **So that** I can verify everything works.
  **Acceptance Criteria:**
* Create customer → send invoice → pay invoice → send campaign → track conversion.

---
