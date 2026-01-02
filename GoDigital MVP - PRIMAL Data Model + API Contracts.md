# GoDigital MVP — PRIMAL Data Model + API Contracts (as implemented)

> Extraction source: `godigital-platform-main/GoDigitalBack` (Express + Mongoose + MongoDB).
> Architecture pattern: **System DB (registry + auth + email ingestion core)** + **Tenant DB (per-entity operational data)**.

---

## Domain model (entities, attributes, relationships)

### 0) High-level multi-tenant partitioning

**Two physical storage partitions:**

1. **SYSTEM DB** (fixed database, e.g. `mongodb://.../system`)

* Tenant registry (workspaces)
* User identities
* Membership + roles
* TenantDetail (database-per-tenant detail)
* Gmail integration tracking + raw email ingest staging
* Email forwarding (routing) configuration

2. **TENANT DB** (tenant detail–selected database, e.g. `mongodb://.../<TenantDetail.dbName>`)

* Business operational data: Accounts, Transactions per Account
* Email/Transaction RAW & reconciliation staging (partially implemented)

**Tenant selection mechanism:**

* JWT contains `tenantId`.
* Active tenant database detail selected by header `x-tenant-detail-id`.
* Middleware attaches `req.tenantDB` (mongoose Connection) + `req.tenantDetailId`.

### 1) SYSTEM DOMAIN

#### 1.1 Entity: User (System Identity)

**Collection:** `system.users` (explicitly set)

**Purpose:** login credentials + profile.

**JSON shape**

```json
{
  "_id": "ObjectId",
  "email": "user@domain.com",
  "passwordHash": "bcrypt_hash",
  "fullName": "Full Name",
  "status": "active | invited | suspended",
  "createdAt": "2026-01-02T00:00:00.000Z",
  "updatedAt": "2026-01-02T00:00:00.000Z"
}
```

**Constraints & indexes**

* `email` unique.

#### 1.2 Entity: Tenant (Workspace)

**Collection:** inferred by Mongoose model name `Tenant` (system DB) (default pluralization applies)

**Purpose:** logical workspace grouping multiple tenant databases (via TenantDetail list).

**JSON shape**

```json
{
  "_id": "ObjectId",
  "name": "Workspace of <FullName>",
  "ownerEmail": "owner@domain.com",
  "dbList": ["ObjectId(TenantDetail)", "ObjectId(TenantDetail)"] ,
  "metadata": {"any": "mixed"},
  "createdAt": "...",
  "updatedAt": "..."
}
```

**Relationships**

* `Tenant (1) ── (N) TenantDetail` via `dbList[]`.
* `Tenant (1) ── (N) Member`.

#### 1.3 Entity: TenantDetail (Physical database detail)

**Collection:** inferred by Mongoose model name `TenantDetail` (system DB)

**Purpose:** defines a **physical MongoDB database** associated to a tenant; can support multiple per tenant.

**JSON shape**

```json
{
  "_id": "ObjectId",
  "tenantId": "ObjectId(Tenant)",
  "dbName": "GoDigital_<timestamp>_<random>",
  "country": "PE",
  "entityType": "natural | legal",
  "taxId": "string (unique)",
  "businessEmail": "string | null",
  "domain": "string | null",
  "metadata": {"any": "mixed"},
  "createdAt": "...",
  "updatedAt": "..."
}
```

**Constraints & indexes**

* `dbName` unique
* `taxId` unique
* index `{ tenantId: 1, dbName: 1 }`
* `tenantId` indexed

**Relationship role**

* This is the pivot for multi-tenancy: selects **tenant DB = TenantDetail.dbName**.

#### 1.4 Entity: Member (User membership in a Tenant)

**Collection:** `system.members` (explicitly set)

**Purpose:** role-based access for a user inside a tenant.

**JSON shape**

```json
{
  "_id": "ObjectId",
  "tenantId": "ObjectId(Tenant)",
  "userId": "ObjectId(User)",
  "role": "superadmin | admin | member | viewer",
  "status": "active | invited | suspended",
  "invitedBy": "ObjectId(User) | null",
  "createdAt": "...",
  "updatedAt": "..."
}
```

**Constraints & indexes**

* unique compound index `{ tenantId: 1, userId: 1 }`
* `tenantId`, `userId` indexed

#### 1.5 Entity: EmailForwardingConfig (Declarative routing rules)

**Collection:** `system.email_forwarding_config` (explicit)

**Purpose:** maps a sender email address to one or more **tenant Accounts** (by Account `_id` in tenant DB) to support routing/enrichment.

**JSON shape**

```json
{
  "_id": "ObjectId",
  "entityId": "ObjectId(TenantDetail)",
  "forwardingData": [
    {
      "email": "pepito@zapatero.com",
      "accounts": ["ObjectId(Account)", "ObjectId(Account)"]
    }
  ],
  "active": true,
  "createdAt": "...",
  "updatedAt": "..."
}
```

**Constraints & indexes**

* `entityId` unique (1 config per TenantDetail)
* index on `forwardingData.email`
* index `{ active: 1, entityId: 1 }`
* pre-save validation: forwarding emails must be unique within the document

**Cross-database relationship**

* `forwardingData.accounts[]` are ObjectIds of **tenant DB Accounts** (tenant DB reference).

#### 1.6 Entity: GmailWatch (Gmail Push subscription tracking)

**Collection:** inferred by model name `GmailWatch` (system DB)

**Purpose:** stores OAuth tokens + watch state to receive Pub/Sub notifications.

**JSON shape**

```json
{
  "_id": "ObjectId",
  "tenantDetailId": "ObjectId(TenantDetail)",
  "email": "user@gmail.com",
  "historyId": "string",
  "expiration": "2026-01-09T00:00:00.000Z",
  "topicName": "projects/<project>/topics/<topic>",
  "accessToken": "string",
  "refreshToken": "string",
  "status": "active | expired | error",
  "lastError": "string | null",
  "createdAt": "...",
  "updatedAt": "..."
}
```

**Constraints & indexes**

* `tenantDetailId` unique
* `expiration` indexed

#### 1.7 Entity: SystemEmailRaw (System-level Gmail ingest staging)

**Collection:** `system.Transaction_Raw_Gmail_System` (explicit)

**Purpose:** stores raw Gmail message content + extracted variables + routing enrichment.

**JSON shape**

```json
{
  "_id": "ObjectId",

  "gmailId": "string (unique)",
  "threadId": "string",
  "historyId": "string",
  "messageId": "string",

  "from": "string",
  "subject": "string",
  "receivedAt": "Date",

  "html": "string | null",
  "textBody": "string | null",
  "labels": ["INBOX", "..."],

  "routing": {
    "entityId": "ObjectId(TenantDetail) | null",
    "bank": "string | null",
    "accountNumber": "string | null"
  },

  "transactionVariables": {
    "originAccount": "string | null",
    "destinationAccount": "string | null",
    "amount": 123.45,
    "currency": "PEN | USD | ...",
    "operationDate": "Date | null",
    "operationNumber": "string | null"
  },

  "transactionType": "string | null",

  "processed": false,
  "processedAt": "Date | null",
  "error": "string | null",

  "createdAt": "...",
  "updatedAt": "..."
}
```

**Indexes (selected)**

* `gmailId` unique + indexed
* `threadId` indexed
* `messageId` indexed
* `from` indexed
* `receivedAt` indexed
* compound `{ processed: 1, receivedAt: -1 }`
* `{ routing.entityId: 1, routing.bank: 1 }`

### 2) TENANT DOMAIN (per TenantDetail.dbName)

> All tenant entities live in the **active tenant DB** selected by `TenantDetail` and connected via `req.tenantDB`.

#### 2.1 Entity: Account (Bank account registry)

**Collection:** `accounts` (explicitly created during provisioning; Mongoose model name `Account` typically maps to `accounts`)

**Purpose:** registers bank accounts to ingest/hold transactions.

**JSON shape**

```json
{
  "_id": "ObjectId",

  "alias": "string | null",
  "bank_name": "string",
  "account_holder": "string",
  "bank_account_type": "string",
  "account_number": "string",

  "currency": "string | null",
  "account_type": "string | null",

  "tx_count": 0,
  "oldest": "Date | null",
  "newest": "Date | null",

  "createdAt": "...",
  "updatedAt": "..."
}
```

**Behavioral relationship**

* Creating an Account triggers creation of a dedicated transaction collection:

  * `Transaction_Raw_<sanitizedAccountNumber>`

#### 2.2 Entity: Transaction (Web-ingested transaction rows)

**Collection:** **dynamic per account number**

* `Transaction_Raw_<sanitizedAccountNumber>`

**Purpose:** stores transactions imported/entered via web flows for a specific bank account.

**JSON shape**

```json
{
  "_id": "ObjectId",

  "accountId": "ObjectId(Account)",
  "uuid": "string | null",

  "descripcion": "string",

  "fecha_hora": "Date | null",
  "fecha_hora_raw": "string | null",

  "monto": 0,
  "amount": 0,
  "balance": 0,

  "currency": "string | null",
  "currency_raw": "string | null",

  "operation_date": "string | null",
  "process_date": "string | null",
  "operation_number": "string | null",

  "movement": "string | null",
  "channel": "string | null",

  "metadata": {
    "yapero": "string | null",
    "origen": "string | null",
    "nombreBenef": "string | null",
    "cuentaBenef": "string | null",
    "celularBenef": "string | null",
    "comision": "string | null",
    "banco": "string | null",
    "...": "any"
  },

  "createdAt": "...",
  "updatedAt": "..."
}
```

**Constraints & indexes**

* unique sparse compound index `{ accountId: 1, uuid: 1 }`
* metadata indexes:

  * `metadata.banco`
  * `metadata.yapero`
  * `metadata.nombreBenef`

#### 2.3 Entity: Transaction_Raw (Tenant-side reconciliation staging)

**Collection:** `Transaction_Raw`

**Purpose:** a tenant-scoped raw record designed to connect:

* system Gmail raw (`Transaction_Raw_Gmail_System`)
* IMAP raw (`Transaction_Raw_IMAP`)
* plus match/conciliation status

**JSON shape**

```json
{
  "_id": "ObjectId",

  "gmailId": "string",
  "threadId": "string",
  "historyId": "string",
  "messageId": "string",

  "from": "string",
  "subject": "string",
  "receivedAt": "Date",

  "html": "string | null",
  "textBody": "string | null",
  "labels": ["string"],

  "routing": {
    "entityId": "ObjectId(TenantDetail) | null",
    "bank": "string | null",
    "accountNumber": "string | null"
  },

  "transactionVariables": {
    "originAccount": "string | null",
    "destinationAccount": "string | null",
    "amount": 123.45,
    "currency": "PEN | USD | ...",
    "operationDate": "Date | null",
    "operationNumber": "string | null"
  },

  "transactionType": "string | null",

  "systemRawId": "ObjectId(Transaction_Raw_Gmail_System) | null",
  "imapRawId": "ObjectId(Transaction_Raw_IMAP) | null",
  "matchStatus": false,
  "matchAt": "Date | null",

  "processed": false,
  "processedAt": "Date | null",
  "error": "string | null",

  "createdAt": "...",
  "updatedAt": "..."
}
```

**Constraints & indexes**

* unique index on `messageId`
* index `{ matchStatus: 1, receivedAt: -1 }`
* index `{ routing.entityId: 1 }`

#### 2.4 Entity: Transaction_Raw_IMAP (IMAP ingestion raw)

**Collection:** `Transaction_Raw_IMAP`

**Purpose:** stores IMAP-fetched emails (raw bodies + PDFs) for later extraction/reconciliation.

**JSON shape**

```json
{
  "_id": "ObjectId",

  "uid": 123,
  "message_id": "string",
  "from": "string",
  "subject": "string",
  "date": "Date",

  "html_body": "string | null",
  "text_body": "string | null",

  "pdfs": [
    {
      "filename": "string",
      "mimeType": "application/pdf",
      "size": 123456,
      "storage": {"any": "mixed"}
    }
  ],

  "fetched_at": "Date",
  "source": "string"
}
```

**Schema notes**

* `strict: false` (tolerates extra fields)
* no timestamps

#### 2.5 Entity: EmailLog (Tenant-side ingestion logging)

**Collection:** inferred from model `EmailLog` (likely `emaillogs` unless overridden; not overridden)

**Purpose:** track processing of emails → transactions created.

**JSON shape**

```json
{
  "_id": "ObjectId",

  "accountId": "ObjectId(Account) | null",

  "messageId": "string (unique)",
  "threadId": "string | null",
  "historyId": "string",

  "from": "string",
  "subject": "string",
  "receivedDate": "Date",

  "processed": false,
  "processedAt": "Date | null",
  "transactionsCreated": 0,

  "rawBody": "string | null",
  "attachments": [
    {
      "filename": "string",
      "mimeType": "string",
      "size": 1234,
      "processed": false
    }
  ],

  "error": "string | null",

  "createdAt": "...",
  "updatedAt": "..."
}
```

**Indexes**

* `{ accountId: 1, receivedDate: -1 }`
* `{ processed: 1, createdAt: 1 }`

#### 2.6 (Legacy / transitional) Entities: Transaction_Raw_Gmail + Transaction_Raw_Processed

> These exist as tenant models but are currently defined using the **default mongoose connection** (`model(...)`) instead of `connection.model(...)`. They should be treated as **legacy/transitional** shapes.

**Transaction_Raw_Gmail** (collection likely `transaction_raw_gmails` by default, unless explicitly set elsewhere)

```json
{
  "_id": "ObjectId",
  "gmailMessageId": "string",
  "threadId": "string | null",
  "from": "string | null",
  "to": "string | null",
  "subject": "string | null",
  "date": "Date | null",
  "bodyText": "string | null",
  "bodyHtml": "string | null",
  "attachments": [{
    "filename": "string",
    "mimeType": "string",
    "attachmentId": "string",
    "size": 1234
  }],
  "parsed": false,
  "createdAt": "...",
  "updatedAt": "..."
}
```

**Transaction_Raw_Processed**

```json
{
  "_id": "ObjectId",
  "rawGmailId": "ObjectId(Transaction_Raw_Gmail)",
  "amount": 0,
  "currency": "string",
  "date": "Date",
  "description": "string",
  "bank": "string",
  "accountHint": "string",
  "confidence": 0.0,
  "createdAt": "...",
  "updatedAt": "..."
}
```

### 3) Relationship diagrams (ASCII)

#### 3.1 High-level relationship map (ASCII)

```text
                         ┌──────────────────────────────────────────┐
                         │                 SYSTEM DB                 │
                         │   (registry, auth, gmail ingest core)     │
                         └───────────────────────┬──────────────────┘
                                                 │
                                                 │
         ┌───────────────────────────────┐       │       ┌───────────────────────────────┐
         │             User              │       │       │            Tenant              │
         │   system.users (identity)     │       │       │   (workspace / org unit)       │
         └───────────────┬──────────────┘       │       └───────────────┬──────────────┘
                         │                      │                       │
                         │ 1..N                 │                       │ 1..N (dbList)
                         │                      │                       │
                ┌────────▼────────┐             │             ┌─────────▼─────────┐
                │     Member      │             │             │    TenantDetail    │
                │ system.members  │             │             │ (physical tenant DB│
                │ RBAC per tenant │             │             │  descriptor)        │
                └────────┬────────┘             │             └─────────┬─────────┘
                         │                      │                       │
                         │ (tenantId,userId)    │                       │ selects dbName
                         │                      │                       │
                         └───────────────┬──────┘                       │
                                         │                              │
                                         │                              │
                                         │                              │
                                         │                      ┌───────▼──────────────────────────┐
                                         │                      │     EmailForwardingConfig        │
                                         │                      │ system.email_forwarding_config   │
                                         │                      │ entityId = TenantDetail._id      │
                                         │                      │ forwardingData[email→accounts[]] │
                                         │                      └───────┬──────────────────────────┘
                                         │                              │
                                         │                              │ routing enrichment
                                         │                              │
                                         │                      ┌───────▼──────────────────────────┐
                                         │                      │            GmailWatch             │
                                         │                      │ (oauth + watch state per tenant) │
                                         │                      └───────┬──────────────────────────┘
                                         │                              │
                                         │                              │ Pub/Sub watch events
                                         │                              │
                                         │                      ┌───────▼──────────────────────────┐
                                         │                      │          SystemEmailRaw           │
                                         │                      │ system.Transaction_Raw_Gmail_System│
                                         │                      │ gmailId(unique) + bodies + vars   │
                                         │                      │ routing.{entityId,bank,accountNo} │
                                         │                      └───────┬──────────────────────────┘
                                         │                              │
                                         │                              │ cross-db linkage (optional)
                                         │                              │
┌────────────────────────────────────────▼──────────────────────────────────────────────────────────────┐
│                                           TENANT DB                                                  │
│                            (dbName selected by TenantDetail)                                         │
└────────────────────────────────────────┬──────────────────────────────────────────────────────────────┘
                                         │
                                         │
                           ┌─────────────▼─────────────┐
                           │           Account          │
                           │ tenant.accounts            │
                           │ bank/account registry      │
                           └─────────────┬─────────────┘
                                         │
                              1 Account  │  N Transactions (dynamic collection)
                                         │
                ┌────────────────────────▼─────────────────────────┐
                │        Transaction_Raw_<accountNumber>            │
                │  (dynamic per account; web-ingested transactions) │
                └────────────────────────┬─────────────────────────┘
                                         │
                                         │
                                         │
                           ┌─────────────▼─────────────┐
                           │          EmailLog           │
                           │ tenant.emaillogs (ingest)   │
                           │ messageId unique            │
                           └─────────────┬─────────────┘
                                         │
                                         │
                                         │
                           ┌─────────────▼─────────────┐
                           │     Transaction_Raw_IMAP    │
                           │ tenant.Transaction_Raw_IMAP  │
                           │ raw IMAP emails + PDFs       │
                           └─────────────┬─────────────┘
                                         │
                                         │ optional link (imapRawId)
                                         │
                           ┌─────────────▼─────────────┐
                           │       Transaction_Raw       │
                           │ tenant.Transaction_Raw       │
                           │ reconciliation staging       │
                           │ systemRawId → SystemEmailRaw │
                           │ imapRawId   → Raw_IMAP       │
                           │ matchStatus / processed      │
                           └─────────────────────────────┘


NOTES
-----
A) Cross-DB references exist:
   - Transaction_Raw.systemRawId  → SystemEmailRaw._id (SYSTEM DB)
   - EmailForwardingConfig.forwardingData.accounts[] → Account._id (TENANT DB)

B) Some processed/raw endpoints reference tenant collections by name:
   - transaction_raw_processed
   - transaction_raw_gmail
   - emails
   These are part of the evolving email-reconciliation strategy.

C) /api/gmail/* is mounted without tenantContext in MVP (intentional during strategy development).
```



#### 3.2 System membership + multi-tenant DB list

```text
User (system.users)
  | 1
  |\
  | \ N
  |  \
Member (system.members) --------------------> Tenant (system.tenants)
   (tenantId, userId, role)                    (dbList[])
                                                |
                                                | 1..N
                                                v
                                         TenantDetail (system.tenantdetails)
                                           (dbName = physical tenant DB)
```

#### 3.3 Email ingestion routing (system)

```text
EmailForwardingConfig (system.email_forwarding_config)
  entityId = TenantDetail._id
  forwardingData[email -> accounts[]]

SystemEmailRaw (system.Transaction_Raw_Gmail_System)
  routing.entityId  -> TenantDetail._id
  routing.accountNumber -> string
  transactionVariables -> extracted
```

#### 3.4 Tenant operational data model

```text
Tenant DB (dbName from TenantDetail)

Account (accounts)
  _id
  account_number
  ...

Account 1 ---- N Transaction_Raw_<accountNumber>
               (web ingested transactions)

Tenant-side reconciliation (staging)
Transaction_Raw (Transaction_Raw)
  systemRawId -> SystemEmailRaw._id  (cross-db reference)
  imapRawId   -> Transaction_Raw_IMAP._id
  matchStatus/matchAt

Transaction_Raw_IMAP (Transaction_Raw_IMAP)
  raw IMAP emails + PDFs
```

---

## API endpoints and contracts

### Conventions

* Base URL: `http(s)://<host>:<port>/api`
* Auth cookie: `session_token` (HTTP-only) is set on login/signup.
* Alternate auth header supported: `Authorization: Bearer <token>`
* Tenant DB selection header: `x-tenant-detail-id: <TenantDetail._id>`

### Authentication & Workspace APIs (`/api/*`)

#### 1) Login / Signup (single handler)

`POST /api/users`

**Request (YAML)**

```yaml
action: login | signup
email: string
password: string
fullName: string # required when action=signup
```

**Response — login (YAML)**

```yaml
success: true
user:
  email: string
  fullName: string
  role: superadmin | admin | member | viewer
  token: string
workspaces:
  - tenantId: string
    name: string
    role: string
    hasDatabase: boolean
    databases:
      - id: string           # TenantDetail._id
        dbName: string       # physical DB name
        country: string
        entityType: natural | legal
```

**Response — signup (YAML)**

```yaml
success: true
user:
  email: string
  fullName: string
  role: superadmin
  token: string
workspaces:
  - tenantId: string
    name: string
    role: superadmin
    databases: []
    hasDatabase: false
```

**Errors**

* 400: missing action/email/password, missing fullName on signup
* 401: invalid credentials
* 403: no active workspace
* 409: user already exists

#### 2) Logout

`POST /api/logout` or `GET /api/logout`

**Response**

```yaml
success: true
```

#### 3) Get Tenant (workspace)

`GET /api/tenants/:id`

**Response**

```yaml
id: string
name: string
ownerEmail: string
metadata: object
createdAt: string
updatedAt: string
databases:
  - id: string
    dbName: string
    country: string
    entityType: natural | legal
    taxId: string
    businessEmail: string | null
    domain: string | null
    createdAt: string
```

#### 4) Update Tenant

`PUT /api/tenants/:id`

**Allowed fields**

```yaml
name: string
metadata: object
```

**Response**

```yaml
success: true
message: Tenant updated successfully
tenant:
  id: string
  name: string
  ownerEmail: string
  metadata: object
```

#### 5) Provision Tenant Database (creates TenantDetail + physical tenant DB)

`POST /api/tenants/:id/provision`

**Request**

```yaml
country: string
entityType: natural | legal
taxId: string
businessEmail: string | null
domain: string | null
metadata: object
```

**Response**

```yaml
success: true
message: Database provisioned successfully
detail:
  id: string
  tenantId: string
  dbName: string
  country: string
  entityType: natural | legal
  taxId: string
  businessEmail: string | null
  domain: string | null
```

**Behavioral side effects**

* Creates `TenantDetail` in system DB.
* Pushes `TenantDetail._id` into `Tenant.dbList`.
* Creates physical tenant DB + collection `accounts`.

#### 6) Get TenantDetail

`GET /api/tenant-details/:detailId`

**Response**

```yaml
id: string
tenantId: string
dbName: string
country: string
entityType: natural | legal
taxId: string
businessEmail: string | null
domain: string | null
metadata: object
createdAt: string
updatedAt: string
```

#### 7) Update TenantDetail

`PUT /api/tenant-details/:detailId`

**Updatable fields**

```yaml
country: string
entityType: natural | legal
taxId: string
businessEmail: string | null
domain: string | null
metadata: object
```

**Constraints**

* `dbName` cannot be changed.
* taxId must remain unique.

**Response**

```yaml
success: true
message: Tenant detail updated successfully
detail:
  id: string
  tenantId: string
  dbName: string
  country: string
  entityType: natural | legal
  taxId: string
  businessEmail: string | null
  domain: string | null
```

#### 8) Get Tenant list with details (single-tenant, role gate)

`GET /api/tenants/details/:id`

**Response**

```yaml
tenantId: string
name: string
code: string | null
role: string
details:
  - detailId: string
    dbName: string
    createdAt: string
    status: string # default "ready" when missing
    entityType: natural | legal
    taxId: string
```

### Account APIs (`/api/accounts*`) — require tenantContext + provisioned tenant DB

#### 9) List accounts

`GET /api/accounts`

**Response** (array)

```yaml
- id: string
  alias: string | null
  bank_name: string
  account_holder: string
  bank_account_type: string
  account_number: string
  currency: string | null
  account_type: string | null
  tx_count: number
  oldest: string | null
  newest: string | null
  createdAt: string
```

#### 10) Create account

`POST /api/accounts`

**Request**

* Either sends the account object as the body, or as `body.account`.

```yaml
alias: string | null
bank_name: string
account_holder: string
bank_account_type: string
account_number: string
currency: string | null
account_type: string | null
```

**Response**

* If body used `account`: `{ ok: true, saved: <full_doc> }`
* Otherwise: returns created document plus:

```yaml
transactionCollection: "Transaction_Raw_<account_number>" # note: returned value not sanitized
```

**Side effects**

* Creates transaction collection for account: `Transaction_Raw_<sanitizedAccountNumber>`.

#### 11) Get account by id

`GET /api/accounts/:id`

**Response**

```yaml
id: string
alias: string | null
bank_name: string
account_holder: string
bank_account_type: string
account_number: string
currency: string | null
account_type: string | null
tx_count: number
oldest: string | null
newest: string | null
```

#### 12) Update account

`PUT /api/accounts/:id`

**Request**: arbitrary fields allowed (`strict:false` in update)

**Response**

```yaml
id: string
alias: string | null
bank_name: string
account_holder: string
bank_account_type: string
account_number: string
currency: string | null
account_type: string | null
tx_count: number
oldest: string | null
newest: string | null
createdAt: string
updatedAt: string
```

#### 13) Delete account

`DELETE /api/accounts/:id`

**Response**

```yaml
ok: true
message: Account deleted successfully
```

### Transaction APIs (web transaction sets per account)

#### 14) Get transactions for an account

`GET /api/accounts/:id/transactions`

**Response**

* Array of documents from collection `Transaction_Raw_<accountNumber>` filtered by `accountId`.

#### 15) Replace transactions for an account

`POST /api/accounts/:id/transactions`

**Request**

```yaml
transactions:
  - # transaction fields (see tenant Transaction entity)
    uuid: string | null
    descripcion: string
    fecha_hora: string
    monto: number
    currency: string
    # ... other optional fields
```

**Behavior**

* Deletes all existing transactions for the account (`deleteMany({accountId:id})`).
* Inserts the new list (adds `accountId`).
* Updates Account `tx_count`, `oldest`, `newest` from inserted `fecha_hora`.

**Response**

```yaml
ok: true
inserted: number
collection: "Transaction_Raw_Web_<account_number>" # returned label (note: actual collection created by model is Transaction_Raw_<accountNumber>)
```

### Transaction APIs (processed/raw; tenant-detail addressed)

> These endpoints select the tenant DB by `tenantDetailId` parameter (they do not depend on `req.tenantDB`).

#### 16) Get processed transactions (with lookups)

`GET /api/transactions/processed/:tenantDetailId`

**Query params**

```yaml
bank: string | null
startDate: string | null # ISO
endDate: string | null   # ISO
limit: number | default 50
```

**Response**

```yaml
total: number
transactions:
  - _id: string
    bank: string
    amount: number
    currency: string
    date: string
    description: string
    reference: string | null
    accountHint: string | null
    confidence: number
    createdAt: string
    raw:
      id: string | null
      rawText: string | null
      status: string | null
    email:
      id: string | null
      from: string | null
      subject: string | null
      receivedAt: string | null
      gmailId: string | null
stats:
  - _id: string # bank
    count: number
    totalAmount: number
    avgConfidence: number
```

**Storage dependency (tenant DB collections referenced)**

* `transaction_raw_processed`
* `transaction_raw_gmail`
* `emails`

#### 17) Get processed transaction detail

`GET /api/transactions/detail/:tenantDetailId/:transactionId`

**Response**

* Aggregated single document from `transaction_raw_processed` with lookups to `transaction_raw_gmail` and `emails`.

#### 18) Get raw transactions

`GET /api/transactions/raw/:tenantDetailId`

**Query params**

```yaml
status: string | default parsed
```

**Response**

```yaml
total: number
transactions: [] # from tenant collection transaction_raw_gmail
```

### Gmail & Forwarding Config APIs (`/api/gmail/*`)

> Mounted at `/api/gmail` and explicitly **without tenantContext**.

#### 19) OAuth start

`GET /api/gmail/auth`

#### 20) OAuth callback

`GET /api/gmail/callback`

#### 21) Pub/Sub webhook

`POST /api/gmail/webhook`

* Protected by custom middleware (`validatePubSubToken` or `validatePubSubJWT`) and `rateLimitWebhook`.

#### 22) Gmail integration status

`GET /api/gmail/status/:tenantDetailId`

#### 23) Disconnect

`DELETE /api/gmail/disconnect/:tenantDetailId`

#### 24) List emails by sender

`GET /api/gmail/emails/:tenantDetailId`

#### 25) Process emails by sender

`POST /api/gmail/process-emails/:tenantDetailId`

#### 26) Create/Update forwarding config

`POST /api/gmail/`

**Request**

```yaml
entityId: string # TenantDetail._id
forwardingData:
  - email: string
    accounts:
      - string # Account._id (tenant DB)
```

**Response**

```yaml
success: true
message: Forwarding config created/updated
config:
  id: string
  entityId: string
  forwardingData:
    - email: string
      accounts: [string]
  active: boolean
```

#### 27) Get forwarding config for entity

`GET /api/gmail/:entityId`

#### 28) List all forwarding configs

`GET /api/gmail/`

#### 29) Toggle config active

`PATCH /api/gmail/:entityId/toggle`

#### 30) Delete config

`DELETE /api/gmail/:entityId`

#### 31) Test match (no persistence)

`POST /api/gmail/test-match`

**Request**

```yaml
from: string
subject: string | null
```

**Response**

```yaml
success: true
input:
  from: string
  subject: string
result:
  matched: boolean
  entityId: string | null
  bank: string | null
  accountNumber: string | null
```

#### 32) List raw emails (system ingest store)

`GET /api/gmail/emails/raw`

**Query params**

```yaml
limit: number | default 20
entityId: string | null
matched: "true" | "false" | null
```

**Response**

```yaml
success: true
count: number
emails:
  - id: string
    gmailId: string
    from: string
    subject: string
    receivedAt: string
    routing: object | null
    processed: boolean
    error: string | null
    createdAt: string
```

#### 33) Raw email detail

`GET /api/gmail/emails/raw/:gmailId`

**Response**

```yaml
success: true
email:
  # full SystemEmailRaw document
  textBodyPreview: string | null
  htmlPreview: string | null
```

#### 34) Matching stats

`GET /api/gmail/stats/matching`

**Response**

```yaml
success: true
stats:
  total: number
  matched: number
  unmatched: number
  processed: number
  withErrors: number
  matchRate: string
  byBank: [{ _id: string, count: number }]
  byEntity: [{ _id: string, count: number }]
```

#### 35) Manual fetch emails (batch)

`POST /api/gmail/fetch-emails`

**Request**

```yaml
idFetching: string # EmailForwardingConfig._id
maxResults: number | default 100000
```

---

## Data dictionary

### System DB — fields dictionary

#### User

* `email`: unique login identifier
* `passwordHash`: bcrypt hash
* `fullName`: display name
* `status`: lifecycle state
* timestamps

#### Tenant

* `name`: workspace label
* `ownerEmail`: owner
* `dbList[]`: list of TenantDetail references
* `metadata`: mixed
* timestamps

#### TenantDetail

* `tenantId`: owning Tenant
* `dbName`: physical tenant database name
* `country`: country code
* `entityType`: natural/legal
* `taxId`: unique tax identifier
* `businessEmail`, `domain`: optional
* `metadata`: mixed
* timestamps

#### Member

* `tenantId`, `userId`: link user↔tenant
* `role`: RBAC
* `status`: lifecycle
* `invitedBy`: optional inviter
* timestamps

#### EmailForwardingConfig

* `entityId`: TenantDetail id (routing scope)
* `forwardingData[]`:

  * `email`: sender address normalized lower+trim
  * `accounts[]`: Account ObjectIds (tenant DB)
* `active`: enable/disable
* timestamps

#### GmailWatch

* `tenantDetailId`: scope
* `email`: gmail account
* `historyId`: last processed
* `expiration`: watch expiry
* `topicName`: pubsub topic
* `accessToken`, `refreshToken`: OAuth creds
* `status`, `lastError`
* timestamps

#### SystemEmailRaw

* identifiers: `gmailId`(unique), `threadId`, `historyId`, `messageId`
* headers: `from`, `subject`, `receivedAt`
* bodies: `html`, `textBody`
* `labels[]`
* `routing`: { entityId, bank, accountNumber }
* `transactionVariables`: extracted transaction fields
* `transactionType`: classifier
* processing flags: `processed`, `processedAt`, `error`
* timestamps

### Tenant DB — fields dictionary

#### Account

* `alias`: friendly label
* `bank_name`: bank identifier
* `account_holder`: owner name
* `bank_account_type`: checking/savings/etc (string)
* `account_number`: bank account number string
* `currency`, `account_type`: optional
* `tx_count`: cached count
* `oldest`, `newest`: cached range
* timestamps

#### Transaction (per-account dynamic collection)

* `accountId`: owning account
* `uuid`: dedupe key (sparse)
* `descripcion`: free text
* datetime fields: `fecha_hora`, `fecha_hora_raw`
* amount fields: `monto`, `amount`, `balance`
* currency fields: `currency`, `currency_raw`
* operational fields: `operation_date`, `process_date`, `operation_number`, `movement`, `channel`
* `metadata`: mixed, parser-enriched
* timestamps

#### Transaction_Raw (tenant staging)

* email identifiers: `gmailId`, `threadId`, `historyId`, `messageId`
* `from`, `subject`, `receivedAt`, `html`, `textBody`, `labels[]`
* `routing` (same structure as system)
* `transactionVariables`
* `transactionType`
* reconciliation:

  * `systemRawId`: ref to system raw
  * `imapRawId`: ref to imap raw
  * `matchStatus`, `matchAt`
* processing:

  * `processed`, `processedAt`, `error`
* timestamps

#### Transaction_Raw_IMAP

* `uid`, `message_id`, `from`, `subject`, `date`
* `html_body`, `text_body`
* `pdfs[]`: mixed
* `fetched_at`
* `source`

#### EmailLog

* `accountId`
* `messageId` unique
* `threadId`, `historyId`
* `from`, `subject`, `receivedDate`
* processing flags + metrics
* attachments[]
* error + timestamps

---

## Event structures (if applicable)

### Implemented integration events (external)

#### Gmail Pub/Sub Notification (Webhook)

* Endpoint: `POST /api/gmail/webhook`
* Authentication: custom token/JWT validation middleware
* Payload: Google Pub/Sub push message (envelope)

**Canonical envelope (conceptual)**

```json
{
  "message": {
    "data": "base64(...)",
    "messageId": "...",
    "publishTime": "...",
    "attributes": {"...": "..."}
  },
  "subscription": "projects/.../subscriptions/..."
}
```

**Decoded Gmail watch data (conceptual)**

```json
{
  "emailAddress": "user@gmail.com",
  "historyId": "123456"
}
```

> Note: The codebase persists/uses `historyId` in `GmailWatch` and in `SystemEmailRaw`.

### Internal events

* No explicit event bus or domain-event collection is implemented in the current MVP.
* Side effects occur as synchronous writes within controllers/services.

---

## Storage model & schemas

### A) Physical databases

#### A.1 System DB

**Configured via** `SYSTEM_DB_URI` (defaults to `mongodb://127.0.0.1:27017/system`).

**Collections (implemented)**

* `users`
* `members`
* `tenants` (default pluralization for `Tenant` model)
* `tenantdetails` (default pluralization for `TenantDetail` model)
* `email_forwarding_config`
* `gmailwatches` (default pluralization for `GmailWatch` model)
* `Transaction_Raw_Gmail_System`

#### A.2 Tenant DB (per TenantDetail.dbName)

**Created during provisioning** with at least:

* `accounts`

**Collections (implemented/observed in code paths)**

* `accounts`
* `Transaction_Raw_<sanitizedAccountNumber>` (dynamic, per account)
* `Transaction_Raw` (tenant staging)
* `Transaction_Raw_IMAP`
* (referenced by aggregation endpoints)

  * `transaction_raw_gmail`
  * `transaction_raw_processed`
  * `emails`

### B) Schema-level constraints summary

#### System DB

* `users.email` unique
* `members.(tenantId,userId)` unique
* `tenantdetails.dbName` unique
* `tenantdetails.taxId` unique
* `email_forwarding_config.entityId` unique
* `Transaction_Raw_Gmail_System.gmailId` unique

#### Tenant DB

* `Transaction_Raw.messageId` unique
* `Transaction_Raw_<account>. (accountId, uuid)` unique sparse

### C) Multi-tenant access rules (as enforced)

#### Enforced by middleware (`tenantContext`) for `/api/*` (except `/api/gmail/*`)

* Requires valid token
* Sets:

  * `req.userId`
  * `req.tenantId`
  * `req.role`
  * `req.tenantDB` (if provisioned)
  * `req.tenantDetailId`
  * `req.tenantProvisioned`

#### Tenant DB selection precedence

1. `x-tenant-detail-id` header (must belong to Tenant.dbList)
2. default: first `Tenant.dbList[0]`

### D) Connection and model binding pattern

#### Preferred tenant-safe pattern

* Use `connection.model(...)` factories:

  * `getAccountModel(connection)`
  * `getTransactionModel(connection, accountNumber)`
  * `getEmailLogModel(connection)`

#### System-safe pattern

* Use `getSystemDB()` + `getOrCreateModel(systemDB, ...)` to avoid global default connection.

---

## Appendix — Operational primitives (MVP rules embedded in code)

### Account provisioning rule

* On `POST /api/accounts`, after saving Account:

  * create transaction collection `Transaction_Raw_<sanitizedAccountNumber>`

### Transaction replacement rule

* `POST /api/accounts/:id/transactions`

  * wipes all existing transactions for that accountId
  * inserts provided list
  * updates cached counters and date boundaries on Account

### Workspace provisioning rule

* `POST /api/tenants/:id/provision`

  * generates `dbName` = `GoDigital_<timestamp>_<random>`
  * creates TenantDetail
  * pushes into Tenant.dbList
  * creates physical tenant DB and collection `accounts`
