# **📘 Product Requirements Document**

**Project Name:** GreenTrack – Lawn & Gardening Service Manager
**Version:** 1.1
**Created On:** 2025-08-12
**Prepared For:** SMB Lawn & Gardening Businesses

---

## **1. Executive Summary**

GreenTrack is a responsive, cloud-based application for small and medium lawn and gardening businesses. It enables service providers to manage customers, send bills, receive payments, and engage in targeted email/SMS marketing.

The system will:

* **Use ZeroInvoice API** for invoice generation, delivery, and payment processing.
* **Use FastAPI** for backend services and API orchestration.
* Leverage **ZeroDB** for serverless relational and vector data services.
* Integrate with **Stripe** for payments (via ZeroInvoice where applicable).
* Track engagement and conversions through **Google Analytics** and in-app reporting.

---

## **2. Goals & Objectives**

| Goal                       | Objective                                                                          |
| -------------------------- | ---------------------------------------------------------------------------------- |
| 📋 Manage customers easily | Store, update, and search customer records using ZeroDB relational + vector search |
| 💳 Get paid faster         | Generate and send invoices using ZeroInvoice API                                   |
| 📢 Boost revenue           | Run targeted email/SMS marketing campaigns for customers and leads                 |
| 📊 Measure success         | Track campaign conversion rates and revenue impact                                 |
| 📈 Gain insights           | Integrate Google Analytics and in-app reporting                                    |
| 🖥️ Accessibility          | Fully responsive UI across desktop, tablet, and mobile                             |

---

## **3. User Personas**

### **Primary**

1. **Business Owner**

   * Wants a quick way to send invoices and get paid.
   * Needs marketing capabilities to increase repeat business.

2. **Office/Admin Staff**

   * Manages customer records, sends bills, and handles basic marketing campaigns.

### **Secondary**

1. **Customer**

   * Receives invoices via email/SMS.
   * Pays online using Stripe link embedded in ZeroInvoice-generated bill.

---

## **4. Core Features**

### **A. Customer Management**

* Create, update, archive customer records.
* Store contact details, address, and tags.
* Assign type: **Customer** or **Lead**.
* Vector search for segmentation (e.g., “customers with outstanding bills in zip 90210”).

### **B. Billing & Payments (via ZeroInvoice API)**

* Create invoices through **ZeroInvoice API**:

  * Itemized services and rates.
  * Auto-generate Stripe payment link.
  * Store invoice data in ZeroDB for local reference.
* Send invoices via **ZeroInvoice email/SMS delivery** or custom Twilio/SendGrid flow.
* Track payment status via **ZeroInvoice webhooks**.
* Automatic overdue reminders.

### **C. Email & SMS Marketing**

* Create promotional templates.
* Send to:

  * Customers
  * Leads
* Segment audiences via vector queries.
* Set up automated follow-up messages.
* Track opens, clicks, and conversions.

### **D. Conversion Tracking**

* Monitor if recipients:

  * Book a service
  * Pay an invoice
* Link campaign to customer action for revenue attribution.
* Store metrics in ZeroDB.

### **E. Analytics & Reporting**

* Google Analytics integration for website and campaign landing pages.
* Dashboard KPIs:

  * Revenue
  * Invoice payment rates
  * Campaign engagement
  * Conversion rates

### **F. Integrations**

* **ZeroInvoice API** – invoice generation, delivery, payment links, webhooks.
* **Stripe** – payments (via ZeroInvoice).
* **ZeroDB** – customer, campaign, and analytics data.
* **Google Analytics** – tracking and reporting.
* **Email/SMS Gateway** – Twilio, SendGrid, or Postmark.

---

## **5. Technical Requirements**

### **Frontend**

* **Responsive Design**: Mobile-first.
* Framework: React (Next.js optional).
* Tailwind CSS for styling.
* Stripe payment link handling from ZeroInvoice API.

### **Backend**

* **FastAPI** service layer.
* Orchestrates:

  * Customer CRUD (ZeroDB)
  * Invoice creation/delivery (ZeroInvoice API)
  * Marketing campaigns (Email/SMS APIs)
  * Analytics sync (Google Analytics API)
* Serverless deployment-ready.
* Stripe webhook listener via ZeroInvoice integration.

### **Data Model (ZeroDB)**

#### **Tables**

1. **Customers**

   * id (UUID)
   * name
   * email
   * phone
   * address
   * type (customer | lead)
   * tags (JSON)
   * created\_at / updated\_at
   * vector\_embedding

2. **Campaigns**

   * id (UUID)
   * name
   * channel (email | sms)
   * target\_segment
   * template\_id
   * sent\_count
   * open\_rate
   * click\_rate
   * conversion\_rate
   * revenue\_generated

3. **Templates**

   * id (UUID)
   * name
   * type (email | sms)
   * content (HTML/Text)

4. **Events**

   * id (UUID)
   * event\_type (invoice\_sent, payment\_received, campaign\_open, click, conversion)
   * entity\_id
   * metadata (JSON)
   * timestamp

*(Invoice storage handled primarily in ZeroInvoice; minimal local cache in ZeroDB for quick reporting.)*

---

## **6. User Flows**

**Send Invoice (ZeroInvoice API)**

1. Admin selects customer in app.
2. Enters service details.
3. Backend calls ZeroInvoice API → creates invoice.
4. ZeroInvoice sends invoice via email/SMS.
5. Payment received via Stripe link → webhook updates status.

**Run Marketing Campaign**

1. Admin filters audience (ZeroDB vector search).
2. Chooses template and schedule.
3. Messages sent via Email/SMS provider.
4. Engagement and conversions tracked in ZeroDB.

---

## **7. Analytics & KPIs**

* **Invoice Metrics:** Paid %, Avg Days to Pay.
* **Campaign Metrics:** Open, Click, Conversion rates.
* **Revenue Attribution:** Total revenue from campaigns.
* **Customer Growth:** New leads vs converted customers.

---

## **8. Security & Compliance**

* HTTPS across all endpoints.
* ZeroInvoice + Stripe PCI compliance.
* GDPR-compliant opt-outs for marketing.
* ZeroDB encryption at rest and in transit.

---

## **9. Future Enhancements**

* AI-driven customer upsell recommendations.
* Customer self-service booking & payment portal.
* Pre-built seasonal campaign templates.

---
