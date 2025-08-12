# ERD (conceptual overview)

**Tenancy & Access**

* `organizations` 1—\* `memberships` \*—1 `users`
* `roles` via `memberships.role` (simple RBAC)

**Core CRM**

* `customers` 1—\* `addresses` (optional)
* `customers` *—* `tags` via `customer_tags`

**Invoicing (via ZeroInvoice)**

* `zeroinvoice_invoices_cache` 1—\* `zeroinvoice_invoice_items_cache`
* `payments` (Stripe/ZeroInvoice webhooks) → `zeroinvoice_invoices_cache.external_invoice_id`

**Marketing**

* `templates` (email/sms)
* `segments` define reusable audience logic (filters/vector)
* `campaigns` 1—\* `campaign_runs` 1—\* `campaign_audience` (snapshot of targets)
* `messages` (outbound sends)
* `message_events` (delivered, opened, clicked, bounced, spam, unsubscribed)
* `subscriptions` (per‑channel opt‑in/out)

**Analytics & Attribution**

* `events` (normalized product analytics)
* `attributions` (link campaign/message to conversion)
* `ga_sessions` (GA client/user mapping)

**Ops & Integrations**

* `integration_accounts` (Twilio/SendGrid/ZeroInvoice credentials)
* `webhooks_inbound` (all providers)
* `audit_log`

---

# Logical schema (tables & fields)

> Types assume **Postgres** (ZeroDB). Vector columns use `pgvector` (`vector(1536)` as default; adjust to your embedding size).

### 1) Tenancy & RBAC

**organizations**

* id UUID PK
* name text not null
* created\_at timestamptz default now()

**users**

* id UUID PK
* email citext unique not null
* name text
* phone text
* created\_at timestamptz default now()

**memberships**

* id UUID PK
* organization\_id UUID FK→organizations(id) on delete cascade
* user\_id UUID FK→users(id) on delete cascade
* role text check (role in (‘owner’,‘admin’,‘staff’)) default ‘staff’
* created\_at timestamptz default now()
* unique (organization\_id, user\_id)

### 2) CRM & Catalog

**customers**

* id UUID PK
* organization\_id UUID FK→organizations on delete cascade
* type text check (type in (‘customer’,‘lead’)) not null
* name text not null
* email citext
* phone text
* address\_line1 text
* address\_line2 text
* city text
* state text
* postal\_code text
* country text default ‘US’
* tags jsonb default ‘\[]’
* notes text
* external\_zeroinvoice\_customer\_id text  -- if mirrored in ZeroInvoice
* embedding vector(1536)                 -- for semantic search/segmentation
* created\_at timestamptz default now()
* updated\_at timestamptz default now()

**tags**

* id UUID PK
* organization\_id UUID FK→organizations on delete cascade
* name citext
* unique (organization\_id, name)

**customer\_tags** (m2m)

* customer\_id UUID FK→customers on delete cascade
* tag\_id UUID FK→tags on delete cascade
* primary key (customer\_id, tag\_id)

*(Optional normalized addresses table if you prefer separate shipping/billing.)*

### 3) ZeroInvoice cache & payments (read‑optimized, not source‑of‑truth)

**zeroinvoice\_invoices\_cache**

* id UUID PK
* organization\_id UUID FK→organizations on delete cascade
* external\_invoice\_id text not null          -- ZeroInvoice invoice ID
* external\_customer\_id text                  -- ZeroInvoice customer ID
* customer\_id UUID FK→customers null         -- local join convenience
* status text check (status in (‘draft’,‘sent’,‘paid’,‘overdue’,‘void’))
* currency text default ‘usd’
* subtotal\_cents int
* tax\_cents int
* total\_cents int
* amount\_due\_cents int
* due\_date date
* sent\_via text\[] default ‘{}’               -- \[‘email’, ‘sms’]
* stripe\_payment\_link text
* snapshot jsonb not null                    -- raw ZeroInvoice payload snapshot
* created\_at timestamptz default now()
* updated\_at timestamptz default now()
* unique (organization\_id, external\_invoice\_id)

**zeroinvoice\_invoice\_items\_cache**

* id UUID PK
* invoice\_id UUID FK→zeroinvoice\_invoices\_cache(id) on delete cascade
* name text not null
* description text
* quantity numeric(12,2) default 1
* unit\_price\_cents int not null
* total\_cents int not null
* position int

**payments**

* id UUID PK
* organization\_id UUID FK→organizations on delete cascade
* external\_payment\_id text not null          -- from ZeroInvoice/Stripe
* external\_invoice\_id text not null
* amount\_cents int not null
* currency text default ‘usd’
* status text check (status in (‘succeeded’,‘failed’,‘pending’))
* method text                                 -- ‘card’, ‘ach’, etc
* processed\_at timestamptz
* raw\_payload jsonb not null
* created\_at timestamptz default now()
* unique (organization\_id, external\_payment\_id)

### 4) Messaging & Campaigns

**templates**

* id UUID PK
* organization\_id UUID FK→organizations on delete cascade
* name text not null
* channel text check (channel in (‘email’,‘sms’)) not null
* subject text                                 -- email only
* content text not null                         -- HTML for email / text for SMS
* metadata jsonb default ‘{}’                   -- Handlebars vars, etc.
* embedding vector(1536)                        -- semantic retrieval
* created\_by UUID FK→users
* created\_at timestamptz default now()
* updated\_at timestamptz default now()

**segments**

* id UUID PK
* organization\_id UUID FK→organizations on delete cascade
* name text not null
* type text check (type in (‘rule’,‘vector’,‘hybrid’)) not null
* rule\_sql text                                 -- serialized SQL WHERE
* vector\_query vector(1536)                     -- query embedding
* vector\_threshold float                         -- cosine similarity cutoff
* metadata jsonb default ‘{}’
* created\_at timestamptz default now()
* updated\_at timestamptz default now()

**campaigns**

* id UUID PK
* organization\_id UUID FK→organizations on delete cascade
* name text not null
* objective text check (objective in (‘promotion’,‘upsell’,‘winback’,‘nurture’))
* channel text check (channel in (‘email’,‘sms’)) not null
* template\_id UUID FK→templates
* segment\_id UUID FK→segments
* status text check (status in (‘draft’,‘scheduled’,‘running’,‘paused’,‘completed’)) default ‘draft’
* schedule\_at timestamptz
* utm\_source text default ‘greentrack’
* utm\_medium text default ‘email’
* utm\_campaign text
* metadata jsonb default ‘{}’
* created\_at timestamptz default now()
* updated\_at timestamptz default now()

**campaign\_runs**  *(each schedule/batch execution)*

* id UUID PK
* campaign\_id UUID FK→campaigns on delete cascade
* started\_at timestamptz
* completed\_at timestamptz
* stats jsonb default ‘{}’  -- {sent, delivered, opened, clicked, bounced, unsubscribed, conversions, revenue\_cents}

**campaign\_audience** *(frozen list of targets per run)*

* id UUID PK
* campaign\_run\_id UUID FK→campaign\_runs on delete cascade
* customer\_id UUID FK→customers on delete cascade
* channel text check (channel in (‘email’,‘sms’))
* address text                -- email or phone at time of send
* included\_reason text        -- matched rules/nearest neighbors
* created\_at timestamptz default now()
* unique (campaign\_run\_id, customer\_id)

**subscriptions** *(per‑channel consent)*

* id UUID PK
* organization\_id UUID FK→organizations on delete cascade
* customer\_id UUID FK→customers on delete cascade
* channel text check (channel in (‘email’,‘sms’)) not null
* status text check (status in (‘subscribed’,‘unsubscribed’,‘bounced’,‘complained’)) not null
* updated\_at timestamptz default now()
* unique (organization\_id, customer\_id, channel)

**messages** *(one outbound message = one row)*

* id UUID PK
* organization\_id UUID FK→organizations on delete cascade
* campaign\_run\_id UUID FK→campaign\_runs
* customer\_id UUID FK→customers
* channel text check (channel in (‘email’,‘sms’)) not null
* to\_address text not null
* provider text               -- ‘twilio’, ‘sendgrid’, ‘postmark’, or ‘zeroinvoice’
* provider\_message\_id text
* template\_id UUID FK→templates
* subject text                -- email only
* body text                   -- resolved content after templating
* utm\_source text
* utm\_medium text
* utm\_campaign text
* status text check (status in (‘queued’,‘sent’,‘delivered’,‘bounced’,‘failed’)) default ‘queued’
* sent\_at timestamptz
* created\_at timestamptz default now()
* unique (organization\_id, provider, provider\_message\_id)

**message\_events**

* id UUID PK
* organization\_id UUID FK→organizations on delete cascade
* message\_id UUID FK→messages(id) on delete cascade
* type text check (type in (‘delivered’,‘open’,‘click’,‘bounce’,‘spam’,‘unsubscribe’)) not null
* occurred\_at timestamptz not null
* metadata jsonb default ‘{}’
* unique (message\_id, type, occurred\_at)

### 5) Analytics, Events & Attribution

**events** *(normalized product/behavioral events)*

* id UUID PK
* organization\_id UUID FK→organizations on delete cascade
* user\_id UUID FK→users null
* customer\_id UUID FK→customers null
* session\_id uuid null
* name text not null            -- ‘invoice\_sent’, ‘payment\_received’, ‘booking\_request’, etc
* source text                   -- ‘system’, ‘ga’, ‘webhook’, ‘app’
* ga\_client\_id text             -- tie to GA
* utm\_source text
* utm\_medium text
* utm\_campaign text
* referrer text
* properties jsonb default ‘{}’
* occurred\_at timestamptz not null
* created\_at timestamptz default now()
* index on (organization\_id, occurred\_at desc)

**attributions**

* id UUID PK
* organization\_id UUID FK→organizations on delete cascade
* customer\_id UUID FK→customers
* message\_id UUID FK→messages null
* campaign\_run\_id UUID FK→campaign\_runs null
* event\_id UUID FK→events not null        -- the conversion event
* conversion\_name text                    -- ‘payment’, ‘booking’, ‘invoice\_paid’
* revenue\_cents int default 0
* attributed\_model text check (in (‘last\_click’,‘first\_touch’,‘linear’,‘time\_decay’)) default ‘last\_click’
* created\_at timestamptz default now()

**ga\_sessions**

* id UUID PK
* organization\_id UUID FK→organizations on delete cascade
* customer\_id UUID FK→customers null
* ga\_client\_id text not null
* first\_seen\_at timestamptz
* last\_seen\_at timestamptz
* unique (organization\_id, ga\_client\_id)

### 6) Integrations, Webhooks & Audit

**integration\_accounts**

* id UUID PK
* organization\_id UUID FK→organizations on delete cascade
* provider text check (in (‘twilio’,‘sendgrid’,‘postmark’,‘zeroinvoice’,‘stripe’,‘google\_analytics’)) not null
* name text
* credentials jsonb not null     -- stored encrypted at rest
* status text check (in (‘active’,‘disabled’)) default ‘active’
* created\_at timestamptz default now()
* updated\_at timestamptz default now()
* unique (organization\_id, provider, name)

**webhooks\_inbound**

* id UUID PK
* organization\_id UUID FK→organizations on delete cascade
* provider text not null
* event\_type text
* headers jsonb
* payload jsonb not null
* received\_at timestamptz default now()
* processed\_at timestamptz
* status text check (in (‘pending’,‘processed’,‘failed’)) default ‘pending’
* error text

**audit\_log**

* id UUID PK
* organization\_id UUID FK→organizations on delete cascade
* actor\_type text check (in (‘user’,‘system’,‘webhook’)) not null
* actor\_id uuid null
* action text not null
* entity text not null
* entity\_id uuid null
* before jsonb
* after jsonb
* occurred\_at timestamptz default now()

---

# Postgres / ZeroDB DDL (selected core)

```sql
-- Enable pgvector
CREATE EXTENSION IF NOT EXISTS vector;

-- Organizations & Users
CREATE TABLE organizations (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name text NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE users (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  email citext UNIQUE NOT NULL,
  name text,
  phone text,
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE memberships (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  user_id uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  role text NOT NULL CHECK (role IN ('owner','admin','staff')) DEFAULT 'staff',
  created_at timestamptz NOT NULL DEFAULT now(),
  UNIQUE (organization_id, user_id)
);

-- Customers
CREATE TABLE customers (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  type text NOT NULL CHECK (type IN ('customer','lead')),
  name text NOT NULL,
  email citext,
  phone text,
  address_line1 text,
  address_line2 text,
  city text,
  state text,
  postal_code text,
  country text DEFAULT 'US',
  tags jsonb NOT NULL DEFAULT '[]',
  notes text,
  external_zeroinvoice_customer_id text,
  embedding vector(1536),
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX customers_org_idx ON customers (organization_id);
CREATE INDEX customers_email_idx ON customers (email);
CREATE INDEX customers_embedding_hnsw ON customers USING hnsw (embedding vector_cosine_ops);

-- ZeroInvoice cache
CREATE TABLE zeroinvoice_invoices_cache (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  external_invoice_id text NOT NULL,
  external_customer_id text,
  customer_id uuid REFERENCES customers(id) ON DELETE SET NULL,
  status text CHECK (status IN ('draft','sent','paid','overdue','void')),
  currency text DEFAULT 'usd',
  subtotal_cents int,
  tax_cents int,
  total_cents int,
  amount_due_cents int,
  due_date date,
  sent_via text[] NOT NULL DEFAULT '{}',
  stripe_payment_link text,
  snapshot jsonb NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  UNIQUE (organization_id, external_invoice_id)
);

CREATE TABLE zeroinvoice_invoice_items_cache (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  invoice_id uuid NOT NULL REFERENCES zeroinvoice_invoices_cache(id) ON DELETE CASCADE,
  name text NOT NULL,
  description text,
  quantity numeric(12,2) NOT NULL DEFAULT 1,
  unit_price_cents int NOT NULL,
  total_cents int NOT NULL,
  position int
);

-- Payments
CREATE TABLE payments (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  external_payment_id text NOT NULL,
  external_invoice_id text NOT NULL,
  amount_cents int NOT NULL,
  currency text DEFAULT 'usd',
  status text NOT NULL CHECK (status IN ('succeeded','failed','pending')),
  method text,
  processed_at timestamptz,
  raw_payload jsonb NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now(),
  UNIQUE (organization_id, external_payment_id)
);

-- Messaging & Campaigns
CREATE TABLE templates (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  name text NOT NULL,
  channel text NOT NULL CHECK (channel IN ('email','sms')),
  subject text,
  content text NOT NULL,
  metadata jsonb NOT NULL DEFAULT '{}',
  embedding vector(1536),
  created_by uuid REFERENCES users(id),
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE segments (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  name text NOT NULL,
  type text NOT NULL CHECK (type IN ('rule','vector','hybrid')),
  rule_sql text,
  vector_query vector(1536),
  vector_threshold float,
  metadata jsonb NOT NULL DEFAULT '{}',
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE campaigns (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  name text NOT NULL,
  objective text CHECK (objective IN ('promotion','upsell','winback','nurture')),
  channel text NOT NULL CHECK (channel IN ('email','sms')),
  template_id uuid REFERENCES templates(id),
  segment_id uuid REFERENCES segments(id),
  status text NOT NULL CHECK (status IN ('draft','scheduled','running','paused','completed')) DEFAULT 'draft',
  schedule_at timestamptz,
  utm_source text DEFAULT 'greentrack',
  utm_medium text DEFAULT 'email',
  utm_campaign text,
  metadata jsonb NOT NULL DEFAULT '{}',
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE campaign_runs (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  campaign_id uuid NOT NULL REFERENCES campaigns(id) ON DELETE CASCADE,
  started_at timestamptz,
  completed_at timestamptz,
  stats jsonb NOT NULL DEFAULT '{}'
);

CREATE TABLE campaign_audience (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  campaign_run_id uuid NOT NULL REFERENCES campaign_runs(id) ON DELETE CASCADE,
  customer_id uuid NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
  channel text NOT NULL CHECK (channel IN ('email','sms')),
  address text NOT NULL,
  included_reason text,
  created_at timestamptz NOT NULL DEFAULT now(),
  UNIQUE (campaign_run_id, customer_id)
);

CREATE TABLE subscriptions (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  customer_id uuid NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
  channel text NOT NULL CHECK (channel IN ('email','sms')),
  status text NOT NULL CHECK (status IN ('subscribed','unsubscribed','bounced','complained')),
  updated_at timestamptz NOT NULL DEFAULT now(),
  UNIQUE (organization_id, customer_id, channel)
);

CREATE TABLE messages (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  campaign_run_id uuid REFERENCES campaign_runs(id),
  customer_id uuid REFERENCES customers(id),
  channel text NOT NULL CHECK (channel IN ('email','sms')),
  to_address text NOT NULL,
  provider text,
  provider_message_id text,
  template_id uuid REFERENCES templates(id),
  subject text,
  body text NOT NULL,
  utm_source text,
  utm_medium text,
  utm_campaign text,
  status text NOT NULL CHECK (status IN ('queued','sent','delivered','bounced','failed')) DEFAULT 'queued',
  sent_at timestamptz,
  created_at timestamptz NOT NULL DEFAULT now(),
  UNIQUE (organization_id, provider, provider_message_id)
);

CREATE TABLE message_events (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  message_id uuid NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
  type text NOT NULL CHECK (type IN ('delivered','open','click','bounce','spam','unsubscribe')),
  occurred_at timestamptz NOT NULL,
  metadata jsonb NOT NULL DEFAULT '{}',
  UNIQUE (message_id, type, occurred_at)
);

-- Analytics & Attribution
CREATE TABLE events (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  user_id uuid REFERENCES users(id),
  customer_id uuid REFERENCES customers(id),
  session_id uuid,
  name text NOT NULL,
  source text,
  ga_client_id text,
  utm_source text,
  utm_medium text,
  utm_campaign text,
  referrer text,
  properties jsonb NOT NULL DEFAULT '{}',
  occurred_at timestamptz NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX events_org_occurred_idx ON events (organization_id, occurred_at DESC);

CREATE TABLE attributions (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  customer_id uuid NOT NULL REFERENCES customers(id),
  message_id uuid REFERENCES messages(id),
  campaign_run_id uuid REFERENCES campaign_runs(id),
  event_id uuid NOT NULL REFERENCES events(id),
  conversion_name text NOT NULL,
  revenue_cents int NOT NULL DEFAULT 0,
  attributed_model text NOT NULL CHECK (attributed_model IN ('last_click','first_touch','linear','time_decay')) DEFAULT 'last_click',
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE ga_sessions (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  customer_id uuid REFERENCES customers(id),
  ga_client_id text NOT NULL,
  first_seen_at timestamptz,
  last_seen_at timestamptz,
  UNIQUE (organization_id, ga_client_id)
);

-- Integrations & Webhooks
CREATE TABLE integration_accounts (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  provider text NOT NULL CHECK (provider IN ('twilio','sendgrid','postmark','zeroinvoice','stripe','google_analytics')),
  name text,
  credentials jsonb NOT NULL,
  status text NOT NULL CHECK (status IN ('active','disabled')) DEFAULT 'active',
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  UNIQUE (organization_id, provider, name)
);

CREATE TABLE webhooks_inbound (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  provider text NOT NULL,
  event_type text,
  headers jsonb,
  payload jsonb NOT NULL,
  received_at timestamptz NOT NULL DEFAULT now(),
  processed_at timestamptz,
  status text NOT NULL CHECK (status IN ('pending','processed','failed')) DEFAULT 'pending',
  error text
);

CREATE TABLE audit_log (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  actor_type text NOT NULL CHECK (actor_type IN ('user','system','webhook')),
  actor_id uuid,
  action text NOT NULL,
  entity text NOT NULL,
  entity_id uuid,
  before jsonb,
  after jsonb,
  occurred_at timestamptz NOT NULL DEFAULT now()
);
```

**Materialized views (recommended)**

* `mv_campaign_performance` (join `messages`, `message_events`, `attributions` → per campaign/run KPIs)
* `mv_revenue_by_campaign` (sum `revenue_cents` group by campaign\_run/campaign)

---

# FastAPI models (excerpt)

```python
# customers.py
from pydantic import BaseModel, EmailStr, Field
from typing import Optional, List
from uuid import UUID
from datetime import datetime

class CustomerCreate(BaseModel):
    type: str = Field(pattern="^(customer|lead)$")
    name: str
    email: Optional[EmailStr] = None
    phone: Optional[str] = None
    address_line1: Optional[str] = None
    address_line2: Optional[str] = None
    city: Optional[str] = None
    state: Optional[str] = None
    postal_code: Optional[str] = None
    country: str = "US"
    tags: List[str] = []

class CustomerOut(CustomerCreate):
    id: UUID
    organization_id: UUID
    created_at: datetime
    updated_at: datetime
```

```python
# campaigns.py
class SegmentRef(BaseModel):
    id: UUID

class CampaignCreate(BaseModel):
    name: str
    objective: str  # promotion|upsell|winback|nurture
    channel: str    # email|sms
    template_id: UUID
    segment_id: UUID
    schedule_at: Optional[datetime] = None
    utm_source: str = "greentrack"
    utm_medium: str = "email"
    utm_campaign: Optional[str] = None
```

```python
# zeroinvoice cache mapping
class ZeroInvoiceSnapshot(BaseModel):
    external_invoice_id: str
    external_customer_id: Optional[str]
    status: Optional[str]
    currency: str = "usd"
    subtotal_cents: Optional[int]
    tax_cents: Optional[int]
    total_cents: Optional[int]
    amount_due_cents: Optional[int]
    due_date: Optional[str]
    sent_via: List[str] = []
    stripe_payment_link: Optional[str]
    snapshot: dict
```

---

