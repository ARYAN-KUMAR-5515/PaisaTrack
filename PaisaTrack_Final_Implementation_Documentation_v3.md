# PaisaTrack — Final Implementation Specification

## 0. Document Status and Authority

**Project:** PaisaTrack — Smart Expense Manager  
**Version:** 3.0 — Audited, Corrected, and Consolidated  
**Purpose:** Single source of truth for implementation by a coding agent such as Claude.

This document consolidates the product requirements, technology choices, database design, API contract, business rules, user flows, UI requirements, testing requirements, and acceptance criteria.

### Authority rule

This document is internally consolidated. There are no earlier competing schemas, routes, or business rules that an implementation agent should choose between.

If a lower-level implementation detail is not explicitly specified here, choose the simplest implementation consistent with this document. Do not add new product features.

### Explicit MVP boundary

PaisaTrack is a local-first personal finance application. It must not require:

- Docker
- Redis
- PostgreSQL
- Kafka
- Kubernetes
- cloud infrastructure
- paid APIs
- OpenAI/Anthropic APIs
- external financial APIs
- bank scraping
- SMS/email parsing
- multi-currency support
- OAuth/social login
- MFA
- push/email/SMS notifications
- PDF export
- bank-statement import
- account-to-account transfers
- credit-card payment transfers
- LLM-powered insights
- offline synchronization/PWA functionality

"Local-first" means the application runs completely on the user's machine. It does not mean that an offline synchronization engine is required.

---

# 1. Product Requirements Document

## 1.1 Project Purpose

PaisaTrack is a modern, privacy-focused personal finance and expense tracking web application tailored to the Indian market.

It helps users answer:

- Where did my money go?
- How much did I spend this month?
- How much money do I have left?
- Am I exceeding my budgets?
- How much am I saving?
- What spending patterns should I pay attention to?

The application is intentionally scoped for students, young adults, salaried users, and secondarily freelancers.

## 1.2 Target Users

### Primary — Students / Young Adults

Need:

- fast transaction entry
- category-wise spending visibility
- monthly budgets
- balance tracking
- warnings when approaching limits

### Primary — Salaried Individuals

Need:

- monthly income tracking
- recurring rent/subscriptions
- savings rate
- month-over-month comparison
- account tracking

### Secondary — Freelancers / Variable-Income Users

Need:

- irregular income tracking
- expense tracking
- account management
- cash-flow visibility

Advanced freelancer accounting and taxation are outside MVP.

## 1.3 Key Features

- Registration, login, logout, protected routes
- Dashboard
- Transaction CRUD
- Server-side pagination, filtering, search, and sorting
- CSV transaction export
- Accounts
- Default and custom categories
- Monthly category budgets
- Automatic retroactive recurring transactions
- Deterministic analytics
- Deterministic spending insights
- Light/Dark/System theme
- Responsive desktop/tablet/mobile UI
- Strong ownership and security checks
- Automated tests

---

# 2. Canonical Technology Stack

## Frontend

- React 18+
- TypeScript 5+
- Vite 5+
- Tailwind CSS
- TanStack React Query v5
- Recharts
- React Router
- Lucide React

## Backend

- Node.js 22+ LTS
- Express 4+
- TypeScript 5+
- Zod
- cookie-parser
- cors
- a maintained CSRF implementation compatible with the chosen Express version

## Database

- SQLite
- Prisma ORM

## Authentication

- jsonwebtoken
- bcrypt
- HttpOnly JWT cookie
- CSRF protection

## Testing

- Vitest
- Supertest
- React Testing Library
- @testing-library/jest-dom

## Development tooling

- tsx
- ESLint
- Prettier
- concurrently

No external paid service is required.

---

# 3. Localization and Money Rules

PaisaTrack is India-focused.

## 3.1 Currency

Canonical currency:

```text
INR
₹
```

All monetary values persisted in the database and sent in JSON API request/response bodies are represented as **integer paise**.

Examples:

```text
₹1       = 100 paise
₹10.50   = 1050 paise
₹1,25,000 = 12500000 paise
```

Never store monetary values as Float or Double.

Never use floating-point arithmetic for persisted money calculations.

Frontend formatting uses:

```text
Intl.NumberFormat("en-IN", {
  style: "currency",
  currency: "INR"
})
```

Display at most two decimal places.

For JSON API monetary fields, the unit is always paise unless an endpoint explicitly states that it is returning a formatted display string. CSV export is the one intentional exception: its Amount column is a numeric rupee value without the currency symbol.

## 3.2 Dates and Timezone

Canonical user timezone:

```text
Asia/Kolkata
```

Canonical display format:

```text
DD/MM/YYYY
```

Database timestamps may be stored as UTC `DateTime`.

Transaction dates and recurring occurrence dates are **date-only business values**, not user-entered times. Normalize them to the start of the corresponding Asia/Kolkata calendar day before persistence. The application must not depend on the browser's local timezone.

For a transaction's date-only meaning:

- user input `25/09/2026` means the calendar date 25 September 2026 in Asia/Kolkata;
- date-only comparisons use Asia/Kolkata calendar boundaries;
- the stored timestamp represents that local calendar date consistently in UTC;
- the frontend always displays the transaction in DD/MM/YYYY;
- backend analytics and monthly budgets use Asia/Kolkata calendar months.

All date calculations must use one shared date utility rather than ad-hoc timezone conversion.

For every transaction date:
- `startDate`/`endDate` filters are inclusive calendar dates.
- `startDate <= endDate` is required.
- `endDate` cannot be later than today's Asia/Kolkata date.
- Transaction dates are persisted at a normalized calendar-day value so two transactions on the same date compare consistently.

Future transaction dates are not allowed in MVP.

---

# 4. Authentication and Security

## 4.1 Registration

Fields:

```text
name
email
password
```

Password policy:

- minimum 8 characters
- at least one uppercase letter
- at least one lowercase letter
- at least one number
- at least one special character

Email is normalized to lowercase.

Passwords are hashed with bcrypt before persistence.

## 4.2 JWT

JWT contains the authenticated user identity.

Rules:

- expiration: 24 hours
- stored only in an HttpOnly cookie
- JavaScript must never receive or store the JWT
- never use localStorage/sessionStorage for authentication tokens
- cookie `SameSite=Lax`
- cookie `Secure=true` in HTTPS production
- local HTTP development may use `Secure=false`
- logout clears the cookie
- expired JWT requires login again
- no refresh-token system in MVP

## 4.3 CSRF

Because authentication uses cookies, all state-changing requests must have CSRF protection:

```text
POST
PUT
PATCH
DELETE
```

GET requests do not require CSRF tokens.

Use a maintained double-submit-token or equivalent CSRF implementation compatible with the chosen Express version.

Canonical SPA flow:

```text
GET /api/auth/csrf
        ↓
server creates/returns a CSRF token
        ↓
frontend keeps the token in memory
        ↓
frontend sends X-CSRF-Token on every POST/PUT/PATCH/DELETE
        ↓
backend validates the token before the route handler
```

`GET /api/auth/csrf` is intentionally available before authentication so the login and registration forms can obtain a token.

The CSRF token must not be stored in localStorage/sessionStorage.

CSRF failures return:

```text
403 CSRF_INVALID
```

Do not disable CSRF checks to make tests pass.

## 4.4 CORS and Cookie Credentials

The Vite development frontend and Express backend run on different localhost origins.

Backend CORS must:
- allow the exact configured frontend origin, such as `http://localhost:5173`;
- allow credentials;
- never use `Access-Control-Allow-Origin: *` with credentials.

Frontend API requests must use credentials so the HttpOnly authentication cookie is sent.

Production must use the actual configured frontend origin rather than a wildcard.

## 4.5 Authorization and Ownership

Every user-owned query must be scoped by the authenticated `userId`.

The backend must never trust a client-supplied `userId`.

For every referenced resource:

- account ownership must be verified
- category ownership/system-category access must be verified
- transaction ownership must be verified
- budget ownership must be verified
- recurring-rule ownership must be verified

A user must never be able to access another user's financial data by changing an ID.

---

# 5. Database Design

The following Prisma schema is authoritative.

```prisma
datasource db {
  provider = "sqlite"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

enum TransactionType {
  INCOME
  EXPENSE
}

enum AccountType {
  CASH
  BANK_ACCOUNT
  CREDIT_CARD
  UPI_WALLET
}

enum Frequency {
  DAILY
  WEEKLY
  MONTHLY
  YEARLY
}

model User {
  id            String                @id @default(uuid())
  email         String                @unique
  passwordHash  String
  name          String
  createdAt     DateTime              @default(now())
  updatedAt     DateTime              @updatedAt

  accounts      Account[]
  categories    Category[]
  transactions  Transaction[]
  budgets       Budget[]
  recurringTxs  RecurringTransaction[]

  @@index([email])
}

model Account {
  id             String                @id @default(uuid())
  userId         String
  user           User                  @relation(fields: [userId], references: [id], onDelete: Cascade)
  name           String
  type           AccountType
  balance        Int                   @default(0)
  isDefault      Boolean               @default(false)
  createdAt      DateTime              @default(now())
  updatedAt      DateTime              @updatedAt

  transactions   Transaction[]
  recurringTxs   RecurringTransaction[]

  @@unique([userId, name])
  @@index([userId])
}

model Category {
  id             String                @id @default(uuid())
  userId         String?
  user           User?                 @relation(fields: [userId], references: [id], onDelete: Cascade)
  name           String
  type           TransactionType       @default(EXPENSE)
  isDefault      Boolean               @default(false)
  createdAt      DateTime              @default(now())
  updatedAt      DateTime              @updatedAt

  transactions   Transaction[]
  budgets        Budget[]
  recurringTxs   RecurringTransaction[]

  @@unique([userId, name])
  @@index([userId])
}

model Transaction {
  id             String                 @id @default(uuid())
  userId         String
  user           User                  @relation(fields: [userId], references: [id], onDelete: Cascade)
  accountId      String
  account        Account               @relation(fields: [accountId], references: [id], onDelete: Restrict)
  categoryId     String
  category       Category              @relation(fields: [categoryId], references: [id], onDelete: Restrict)
  amount         Int
  type           TransactionType
  description    String
  date           DateTime
  paymentMethod  String?
  notes          String?
  recurringId    String?
  recurringTx    RecurringTransaction? @relation(fields: [recurringId], references: [id], onDelete: SetNull)
  recurringOccurrenceKey String?       @unique
  createdAt      DateTime              @default(now())
  updatedAt      DateTime              @updatedAt

  @@index([userId, date])
  @@index([userId, categoryId])
  @@index([userId, accountId])
  @@index([recurringId])
}

model Budget {
  id          String    @id @default(uuid())
  userId      String
  user        User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  categoryId  String
  category    Category  @relation(fields: [categoryId], references: [id], onDelete: Restrict)
  amount      Int
  month       Int
  year        Int
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt

  @@unique([userId, categoryId, month, year])
  @@index([userId, year, month])
}

model RecurringTransaction {
  id             String         @id @default(uuid())
  userId         String
  user           User           @relation(fields: [userId], references: [id], onDelete: Cascade)
  accountId      String
  account        Account        @relation(fields: [accountId], references: [id], onDelete: Restrict)
  categoryId     String
  category       Category       @relation(fields: [categoryId], references: [id], onDelete: Restrict)
  amount         Int
  type           TransactionType
  description    String
  frequency      Frequency
  nextOccurrence DateTime
  isActive       Boolean        @default(true)
  createdAt      DateTime       @default(now())
  updatedAt      DateTime       @updatedAt

  transactions   Transaction[]

  @@index([userId, isActive, nextOccurrence])
}
```

## 5.1 Category Ownership

Default categories:

```text
userId = null
isDefault = true
```

Custom categories:

```text
userId = authenticated user
isDefault = false
```

Default categories are visible to every user.

A user may only create, modify, or delete their own custom categories.

Default categories cannot be deleted or modified.

Category names are normalized by trimming whitespace and compared case-insensitively.

A custom category name must not collide with another custom category belonging to the same user, and must not collide with a default category of the same type. This prevents ambiguous duplicate labels such as `Food` and `food`.

Because SQLite's nullable composite unique behavior does not provide the desired global uniqueness for system categories, application-level validation must also prevent duplicate default-category definitions.

## 5.2 Canonical Categories

Default expense categories:

```text
Food
Travel
Shopping
Bills
Entertainment
Education
Health
Rent
Subscriptions
Other
```

Default income categories:

```text
Salary
Freelance
Other Income
```

The `Category.type` determines whether a category is suitable for an income or expense transaction.

Users may create custom categories of either type.

## 5.3 Account Balance — Single Source of Truth

`Account.balance` is the authoritative current balance.

It represents:

```text
initial account balance
+ income transactions
- expense transactions
```

The balance must be updated transactionally whenever a transaction is created, edited, or deleted.

For a transaction mutation:

- create expense → subtract amount
- create income → add amount
- delete expense → add amount back
- delete income → subtract amount back
- edit transaction → reverse old effect, then apply new effect

Example:

```text
Old expense = ₹500
New expense = ₹700

Balance adjustment = -₹200
```

All balance mutation and transaction persistence must occur in one database transaction.

Do not silently recalculate and overwrite account balances from unrelated records.

A dashboard "total balance" is the sum of the authenticated user's account balances.

For `CREDIT_CARD` accounts, the balance uses the same signed convention:
- positive balance = net credit/available value;
- negative balance = amount owed.

MVP does not implement account-to-account transfers or credit-card payment transfers. Do not model a transfer as an income/expense transaction because that would distort income, expense, and savings analytics.

## 5.4 Account Deletion

Accounts with transactions or recurring transactions cannot be deleted.

Return:

```text
ACCOUNT_IN_USE
```

The user must keep the account or remove/reassign dependent records first.

This avoids silently destroying financial history.

---

# 6. Business Rules

## 6.1 Transactions

- Amount must be strictly greater than zero.
- Amount is stored as integer paise.
- `INCOME` increases account balance.
- `EXPENSE` decreases account balance.
- The category type must match the transaction type.
- Account must belong to the authenticated user.
- Category must be either a system category of the correct type or a custom category belonging to the user.
- Description is required and maximum 100 characters.
- Notes are optional and maximum 500 characters.
- Payment method is optional and is trimmed if supplied.
- Future transaction dates are not allowed.
- Transaction date is interpreted in Asia/Kolkata.
- The account and category must both be owned/accessible by the authenticated user.
- A transaction update may move a transaction between accounts, categories, types, dates, and amounts; the old account/category/type effect must be reversed before the new effect is applied, atomically.

## 6.2 Budgets

A budget belongs to:

```text
user
category
calendar month
calendar year
```

There can be only one budget per:

```text
user + category + month + year
```

Budgets use strict calendar-month boundaries.

No rollover:

- surplus does not carry forward
- deficit does not carry forward

Historical budgets remain unchanged and viewable.

Budget amount must be greater than zero.

Budgets may only target expense categories.

Budget `spent` is the sum of `EXPENSE` transactions whose category matches the budget category and whose transaction date falls within the budget's exact Asia/Kolkata calendar month.

Budget remaining:

```text
remaining = budget - spent
```

Budget utilization:

```text
spent / budget * 100
```

The progress bar width is capped at 100%, but the displayed percentage may exceed 100%.

Example:

```text
budget = ₹10,000
spent = ₹12,500
display = 125%
bar width = 100%
```

Thresholds:

```text
0–69%    NORMAL
70–89%   WARNING
90–99%   CRITICAL
>=100%   EXCEEDED
```

Visual indicators must not rely on color alone. Include text/icon/status labels.

## 6.3 Recurring Transactions

A recurring rule contains:

```text
amount
type
category
account
description
frequency
nextOccurrence
isActive
```

Recurring rules use the same amount, category/type compatibility, description, and account-ownership validation as normal transactions.

`nextOccurrence` is a normalized Asia/Kolkata calendar date.

Supported frequencies:

```text
DAILY
WEEKLY
MONTHLY
YEARLY
```

### Automatic generation

Missed occurrences are generated automatically when an authenticated user's session is established and also when recurring data is requested if needed.

Only active recurring rules generate transactions.

For every active rule, compare `nextOccurrence` with today's Asia/Kolkata calendar date.

```text
while nextOccurrence <= today:
    occurrenceKey = recurringRuleId + ":" + nextOccurrence(YYYY-MM-DD)

    if occurrenceKey does not already exist:
        create transaction dated nextOccurrence
        store occurrenceKey

    nextOccurrence = calculateNextOccurrence(originalSchedule, nextOccurrence)
```

`today` means the current Asia/Kolkata calendar date, not the current UTC timestamp.

The recurrence calculation preserves the rule's original schedule anchor. For example, a monthly rule anchored to day 31 produces:

```text
31 Jan
28 Feb (or 29 Feb in a leap year)
31 Mar
30 Apr
31 May
...
```

It does not permanently change its anchor to day 28 after February.

The generated transaction:

- uses the exact scheduled occurrence date
- links to the recurring rule through `recurringId`
- affects account balance
- is created only once

The entire catch-up operation must be transactional.

### Duplicate prevention

A recurring occurrence is uniquely identified by the persisted:

```text
recurringOccurrenceKey = recurringRuleId + ":" + YYYY-MM-DD
```

The `recurringOccurrenceKey` field is unique in the database.

Before creating an occurrence, derive the key from the recurring rule ID and normalized scheduled date.

If it already exists:
- do not create another transaction;
- continue advancing `nextOccurrence` until it is the next future occurrence.

The implementation must be safe if login/session initialization happens multiple times or two requests attempt catch-up concurrently. Catch-up must be protected by a database transaction and the unique occurrence key.

### Example

A monthly rule:

```text
₹649
day = 5
```

If the user returns on 10 October after missing August, September and October:

Create:

```text
05/08
05/09
05/10
```

Then set:

```text
nextOccurrence = 05/11
```

Do not use login date as the transaction date.

### Month-end date handling

If a monthly recurrence is anchored to a day that does not exist in a month, use the last valid calendar day of that month for that occurrence while preserving the original anchor.

Example:

```text
31 January
→ February occurrence = 28 February
→ leap year February = 29 February
→ March occurrence = 31 March
```

For yearly February 29 rules:
- leap years use February 29;
- non-leap years use February 28;
- the original February 29 anchor is preserved for future leap years.

### Inactive rule

Inactive recurring rules never generate occurrences.

## 6.4 Category Deletion

- Default categories cannot be deleted.
- Custom categories with dependent transactions cannot be deleted.
- Custom categories with dependent budgets or recurring rules also cannot be deleted.
- Return `CATEGORY_IN_USE`.
- Do not silently delete or reassign financial records.

## 6.5 Analytics Ownership

All analytics include only authenticated user's data.

Never aggregate another user's data.

---

# 7. Analytics and Exact Formulas

Default dashboard analytics use the current calendar month in Asia/Kolkata.

For a selected date range, use the supplied inclusive range. The range must be valid and must not extend beyond today's Asia/Kolkata date.

## Total Income

```text
I = sum(transaction.amount where type = INCOME)
```

## Total Expenses

```text
E = sum(transaction.amount where type = EXPENSE)
```

## Savings

```text
S = I - E
```

## Savings Rate

```text
if I > 0:
    SR = (S / I) * 100
else:
    SR = 0
```

Savings rate may be negative if expenses exceed income.

## Daily Average Spending

For the current month:

```text
E / elapsed calendar days in the current month
```

For a completed/custom range:

```text
E / number of calendar days in the selected range
```

Never divide by zero.

## Month-over-Month Expense Change

For the current dashboard month:

```text
if previousExpense > 0:
    ((currentExpense - previousExpense) / previousExpense) * 100
else:
    0
```

When previous expense is zero, do not report an infinite percentage.

For analytics over a selected range, return one monthly data point per calendar month intersecting the range. Each monthly point contains:

```text
month
income
expenses
savings
```

Also return the previous comparable calendar period needed for the displayed comparison. Do not invent a comparison when no previous period exists.

## Category Spending

```text
sum(EXPENSE amounts grouped by category)
```

## Budget Utilization

```text
spent / budget * 100
```

If a budget is zero, the budget is invalid and must not be persisted.

## Recurring Expense Total

For display, report both:
- the raw scheduled amount/frequency of each active recurring expense rule;
- a normalized monthly-equivalent total.

Monthly-equivalent normalization:

```text
DAILY   = amount * 365 / 12
WEEKLY  = amount * 52 / 12
MONTHLY = amount
YEARLY  = amount / 12
```

Round the final monthly-equivalent amount to the nearest paise.

The normalized monthly-equivalent total is used by the recurring-burden insight.

This represents scheduled recurring expense obligations, not necessarily already-generated transactions.

---

# 8. Deterministic Spending Insights

No LLM is used.

The backend evaluates these exact rules.

## Insight 1 — Higher Spending

Trigger when:

```text
currentMonthExpenses > previousMonthExpenses * 1.15
```

and previous-month expenses are greater than zero.

## Insight 2 — Category Spike

Trigger for each category where:

```text
categoryExpense / totalExpense > 0.40
```

and total expenses are greater than zero.

## Insight 3 — Budget Warning

Trigger when an active budget reaches:

```text
70% <= utilization < 100%
```

## Insight 4 — Budget Exceeded

Trigger when an active budget reaches:

```text
>= 100%
```

For a budget at or above 100%, emit only the EXCEEDED insight, not both WARNING and EXCEEDED.

## Insight 5 — Recurring Burden

Trigger when:

```text
recurringExpenseTotal / currentMonthIncome > 0.50
```

and current-month income is greater than zero.

Each insight must contain:

```text
type
title
message
severity
metadata
```

Canonical insight types:

```text
HIGHER_SPENDING
CATEGORY_SPIKE
BUDGET_WARNING
BUDGET_EXCEEDED
RECURRING_BURDEN
```

Canonical severities:

```text
INFO
WARNING
CRITICAL
```

Use:
- HIGHER_SPENDING → WARNING
- CATEGORY_SPIKE → INFO
- BUDGET_WARNING → WARNING
- BUDGET_EXCEEDED → CRITICAL
- RECURRING_BURDEN → WARNING

No duplicate identical insight should be emitted repeatedly within one API response.

---

# 9. API Contract

All protected endpoints require a valid JWT cookie.

Success:

```json
{
  "success": true,
  "data": {}
}
```

Error:

```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable message"
  }
}
```

Never expose stack traces, SQL details, JWT secrets, password hashes, or sensitive internals to clients.

## 9.1 Authentication

### Register

```text
POST /api/auth/register
```

Body:

```json
{
  "name": "Aryan Kumar",
  "email": "aryan@example.com",
  "password": "StrongPassword@123"
}
```

Returns `201`.

### Login

```text
POST /api/auth/login
```

Body:

```json
{
  "email": "aryan@example.com",
  "password": "StrongPassword@123"
}
```

Returns `200` and sets HttpOnly JWT cookie.

Invalid credentials:

```text
401 AUTHENTICATION_FAILED
```

### Logout

```text
POST /api/auth/logout
```

Clears authentication cookie.

### CSRF Token

```text
GET /api/auth/csrf
```

Returns the CSRF token required for state-changing requests.

This endpoint does not require authentication.

### Current User

```text
GET /api/auth/me
```

Returns current authenticated user.

Unauthenticated:

```text
401 UNAUTHORIZED
```

## 9.2 Profile

```text
GET /api/users/profile
PUT /api/users/profile
PUT /api/users/password
```

Profile update:

```json
{
  "name": "New Name",
  "email": "new@example.com"
}
```

Password change:

```json
{
  "currentPassword": "OldPassword@123",
  "newPassword": "NewPassword@123"
}
```

The backend must verify the current password before changing it.

## 9.3 Accounts

```text
GET    /api/accounts
POST   /api/accounts
PUT    /api/accounts/:id
DELETE /api/accounts/:id
```

Create:

```json
{
  "name": "HDFC Bank",
  "type": "BANK_ACCOUNT",
  "balance": 500000
}
```

`balance` is integer paise and is the opening/current balance at account creation.

For account creation only, the supplied `balance` establishes the opening balance.

`PUT /api/accounts/:id` may update only:
```text
name
type
```

The current `balance` must not be directly edited through the API because it is maintained by transaction mutations. Account names are trimmed and compared case-insensitively within the authenticated user's accounts.

Delete an account with dependent records:

```text
409 ACCOUNT_IN_USE
```

## 9.4 Categories

```text
GET    /api/categories
POST   /api/categories
PUT    /api/categories/:id
DELETE /api/categories/:id
```

Create:

```json
{
  "name": "Gym",
  "type": "EXPENSE"
}
```

Default categories are read-only.

Custom category update:

```json
{
  "name": "Gym & Fitness",
  "type": "EXPENSE"
}
```

The update endpoint must reject changes that would collide with another custom category or a default category of the same type.

## 9.5 Transactions

### List

```text
GET /api/transactions
```

Query parameters:

```text
page
limit
search
startDate
endDate
type
categoryId
accountId
minAmount
maxAmount
sortBy
sortOrder
```

Allowed:

```text
limit = 25 | 50 | 100
sortBy = date | amount
sortOrder = asc | desc
type = INCOME | EXPENSE
```

Default:

```text
page = 1
limit = 25
sortBy = date
sortOrder = desc
```

Search is case-insensitive and matches transaction description, notes, payment method, category name, and account name.

Amount filters are integer paise.

Date filters are inclusive calendar dates in Asia/Kolkata.

Pagination metadata is calculated after all filters/search are applied.

Sorting must be stable:
- `date` primary sort, then `createdAt`, then `id`;
- `amount` primary sort, then `date`, then `id`.

Response:

```json
{
  "success": true,
  "data": {
    "items": [],
    "pagination": {
      "page": 1,
      "limit": 25,
      "total": 142,
      "totalPages": 6,
      "hasNext": true,
      "hasPrevious": false
    }
  }
}
```

### Create

```text
POST /api/transactions
```

Body:

```json
{
  "accountId": "account-id",
  "categoryId": "category-id",
  "amount": 125000,
  "type": "EXPENSE",
  "description": "Groceries",
  "date": "25/09/2026",
  "paymentMethod": "UPI",
  "notes": "Weekly groceries"
}
```

### Get one

```text
GET /api/transactions/:id
```

### Update

```text
PUT /api/transactions/:id
```

### Delete

```text
DELETE /api/transactions/:id
```

All mutations update account balance transactionally.

## 9.6 CSV Export

```text
GET /api/transactions/export
```

Accepts the same filtering parameters as transaction listing.

Exports either:

- all user transactions, or
- transactions matching active filters.

Canonical columns:

```text
Date
Type
Amount
Category
Description
Account
Payment Method
Notes
```

Rules:

- UTF-8
- Date = DD/MM/YYYY
- Amount = numeric rupee value without `₹`
- filename = `transactions_YYYY-MM-DD.csv`
- no other user's data
- no PDF export

## 9.7 Budgets

```text
GET    /api/budgets?month=9&year=2026
POST   /api/budgets
PUT    /api/budgets/:id
DELETE /api/budgets/:id
```

Create:

```json
{
  "categoryId": "category-id",
  "amount": 1000000,
  "month": 9,
  "year": 2026
}
```

`amount` is integer paise.

Duplicate:

```text
409 BUDGET_ALREADY_EXISTS
```

## 9.8 Recurring Transactions

```text
GET    /api/recurring
POST   /api/recurring
PUT    /api/recurring/:id
DELETE /api/recurring/:id
```

Create:

```json
{
  "accountId": "account-id",
  "categoryId": "category-id",
  "amount": 64900,
  "type": "EXPENSE",
  "description": "Subscription",
  "frequency": "MONTHLY",
  "nextOccurrence": "05/10/2026",
  "isActive": true
}
```

Creating/editing a recurring rule does not retroactively generate dates before its specified `nextOccurrence`. Catch-up begins from that date.

When editing an active recurring rule, changing amount/type/category/account/description/frequency/nextOccurrence affects only future generated occurrences. Already-generated transactions remain unchanged.

## 9.9 Dashboard

```text
GET /api/dashboard
```

Returns:

- current total balance
- current-month income
- current-month expenses
- savings
- savings rate
- category spending
- recent 5 transactions
- budget statuses
- insights

## 9.10 Analytics

```text
GET /api/analytics?startDate=01/09/2026&endDate=25/09/2026
```

Returns:

- income
- expenses
- savings
- savings rate
- daily average
- category breakdown for the selected range
- monthly time-series data for income, expenses, and savings
- previous-period expense comparison
- budget utilization for budgets whose months intersect the selected range
- recurring expense rules plus normalized monthly-equivalent total

## 9.11 Insights

```text
GET /api/insights
```

Returns deterministic insight objects.

---

# 10. Validation Rules

Backend validation is authoritative.

## Common limits

```text
name: 1–100 characters after trimming
account name: 1–100 characters
category name: 1–50 characters
description: 1–100 characters
notes: 0–500 characters
payment method: 0–50 characters
email: valid email, normalized lowercase
```

Amounts:
- integer paise in API/database;
- strictly greater than zero;
- reject fractional paise;
- reject NaN, Infinity, negative values, and zero;
- reject values outside the application's safe integer monetary range.

Dates:
- exact `DD/MM/YYYY` in public JSON date fields;
- valid calendar date;
- no future transaction dates;
- `startDate <= endDate`;
- analytics/transaction ranges cannot extend beyond today's Asia/Kolkata date.

IDs must be valid resource identifiers.

Unknown enum values are rejected.

Category type must match transaction/recurring transaction type.

Budget:
- amount > 0;
- month = 1–12;
- year must be a valid four-digit calendar year.

Recurring:
- amount > 0;
- frequency must be one of DAILY/WEEKLY/MONTHLY/YEARLY;
- `nextOccurrence` must be a valid calendar date;
- account/category ownership and type compatibility are required.

Profile:
- email uniqueness is enforced;
- password changes must satisfy the registration password policy.

Validation errors return `400 VALIDATION_ERROR` with field-level details when useful.

---

# 10. HTTP Error Codes

Canonical codes:

```text
400 VALIDATION_ERROR
401 UNAUTHORIZED
401 AUTHENTICATION_FAILED
403 FORBIDDEN
404 NOT_FOUND
409 ACCOUNT_IN_USE
409 CATEGORY_IN_USE
409 BUDGET_ALREADY_EXISTS
409 DUPLICATE_RECURRING_OCCURRENCE
409 CONFLICT
403 CSRF_INVALID
500 INTERNAL_SERVER_ERROR
```

Use `403` only when the authenticated user lacks permission.

For another user's resource, `404 NOT_FOUND` is acceptable to avoid revealing that the resource exists.

---

# 11. Frontend Pages

Routes:

```text
/login
/register
/dashboard
/transactions
/accounts
/categories
/budgets
/recurring
/analytics
/profile
```

Protected routes:

```text
/dashboard
/transactions
/accounts
/categories
/budgets
/recurring
/analytics
/profile
```

## Dashboard

Show:

- balance
- income
- expenses
- savings
- savings rate
- category spending chart
- monthly comparison
- budget progress
- insights
- recent transactions

## Transactions

Show:

- search
- type filter
- category filter
- account filter
- date range
- amount range
- sort
- pagination
- CSV export
- add/edit/delete

Pagination:

```text
25 / 50 / 100
```

Show:

```text
Showing 51–75 of 1,240
```

Filters remain active when changing pages.

Changing a filter resets the page to 1.

## Accounts

Show account cards with:

- name
- type
- balance
- edit
- delete

## Categories

Show default and custom categories.

## Budgets

Show:

- month/year
- category
- limit
- spent
- remaining
- percentage
- threshold state

## Recurring

Show:

- description
- amount
- type
- category
- account
- frequency
- next occurrence
- active/inactive
- edit/delete

## Analytics

Use Recharts.

Charts:

- monthly income vs expense using the analytics monthly time-series data
- category spending for the selected range
- savings trend using the analytics monthly time-series data
- budget utilization for the selected month/budget set
- recurring monthly-equivalent obligations as a summary, not as historical transaction spending

Do not show a recurring-vs-non-recurring historical spending chart unless the API provides a deterministic definition and data for it; it is not required for MVP.

## Profile

Show:

- name
- email
- account creation date
- change password
- CSV export
- theme preference

---

# 12. Dashboard UX and Design System

## 12.1 Theme

Canonical theme options:

```text
SYSTEM
LIGHT
DARK
```

First visit:

```text
SYSTEM
```

System mode follows OS `prefers-color-scheme`.

If user selects Light or Dark, persist the choice.

If user selects System, persist `SYSTEM`.

Use:

```text
localStorage key = paisatrack_theme
```

This is acceptable because theme preference is not an authentication secret.

## 12.2 Responsive Design

Desktop:

```text
>= 1024px
```

- sidebar
- multi-column dashboard
- full tables

Tablet:

```text
768–1023px
```

- collapsible sidebar
- two-column layout
- scrollable tables

Mobile:

```text
< 768px
```

- drawer/hamburger navigation
- single-column cards
- mobile-friendly transaction list
- touch-friendly controls

## 12.3 Budget Visual Semantics

```text
0–69%    NORMAL
70–89%   WARNING
90–99%   CRITICAL
>=100%   EXCEEDED
```

Use:

- green/neutral for normal
- amber for warning
- orange for critical
- red for exceeded

Also display textual status.

Do not rely only on color.

## 12.4 Loading, Empty, Error States

Every major page must support:

- loading state
- empty state
- error state
- retry action where appropriate

Mutating buttons show:

```text
Saving...
Deleting...
```

and become disabled while the request is running.

## 12.5 Destructive Actions

Require confirmation for:

- delete transaction
- delete account
- delete category
- delete budget
- delete recurring rule

## 12.6 Toasts

Success/error/warning notifications appear consistently.

Do not repeatedly show the same budget warning merely because the dashboard is opened again.

Budget warnings must primarily be visual dashboard/page indicators.

No browser push, SMS, or email notifications.

## 12.7 Accessibility

Use:

- semantic HTML
- labels for inputs
- keyboard navigation
- visible focus states
- accessible modal dialogs
- sufficient contrast
- text/icon status in addition to color
- descriptive button labels

---

# 13. Styling Guidelines

Use Tailwind CSS.

Design goals:

- modern
- professional
- clean
- finance-oriented
- information-dense without being cluttered
- subtle borders/shadows
- restrained animation
- no excessive gradients
- no heavy glassmorphism

Suggested neutral palette:

```text
Light background: #F8FAFC
Light surface:    #FFFFFF
Light border:     #E2E8F0
Dark background:  #0F172A
Dark surface:     #1E293B
Dark border:      #334155
Primary:          #2563EB
```

Semantic colors:

```text
Income/positive: green
Expense/negative: red
70–89% warning: amber
90–99% critical: orange
>=100% exceeded: red
```

Use Inter/system sans-serif typography.

Main content should use a sensible maximum width such as `max-w-7xl`.

Cards should use subtle borders and rounded corners.

Avoid animations that interfere with data entry or reading.

---

# 14. Project Structure

```text
paisatrack/
├── backend/
│   ├── prisma/
│   │   ├── schema.prisma
│   │   ├── migrations/
│   │   └── seed.ts
│   ├── src/
│   │   ├── config/
│   │   ├── controllers/
│   │   ├── middleware/
│   │   ├── routes/
│   │   ├── services/
│   │   ├── utils/
│   │   ├── types/
│   │   ├── app.ts
│   │   └── server.ts
│   ├── tests/
│   │   ├── unit/
│   │   ├── integration/
│   │   └── setup.ts
│   ├── .env.example
│   ├── .env.test
│   ├── package.json
│   └── tsconfig.json
├── frontend/
│   ├── public/
│   ├── src/
│   │   ├── api/
│   │   ├── components/
│   │   │   ├── common/
│   │   │   ├── dashboard/
│   │   │   ├── transactions/
│   │   │   ├── budgets/
│   │   │   ├── accounts/
│   │   │   ├── categories/
│   │   │   ├── recurring/
│   │   │   └── analytics/
│   │   ├── context/
│   │   ├── hooks/
│   │   ├── pages/
│   │   ├── utils/
│   │   ├── types/
│   │   ├── App.tsx
│   │   ├── main.tsx
│   │   └── index.css
│   ├── index.html
│   ├── package.json
│   ├── tailwind.config.js
│   ├── tsconfig.json
│   └── vite.config.ts
├── .gitignore
├── package.json
└── README.md
```

Keep the architecture understandable. Do not introduce unnecessary abstraction.

---

# 15. Local Development

The root project must provide simple orchestration scripts.

Canonical commands:

```bash
npm install
npm run db:migrate
npm run db:seed
npm run dev
npm test
npm run build
```

`npm run dev` must start both backend and frontend.

A root `concurrently` script may be used.

Example root scripts:

```json
{
  "scripts": {
    "dev": "concurrently \"npm --prefix backend run dev\" \"npm --prefix frontend run dev\"",
    "db:migrate": "npm --prefix backend exec prisma migrate dev",
    "db:seed": "npm --prefix backend run seed",
    "test": "npm --prefix backend test && npm --prefix frontend test",
    "build": "npm --prefix backend run build && npm --prefix frontend run build"
  }
}
```

The implementation may adjust exact command syntax as long as the canonical commands work.

## Environment

Backend `.env`:

```env
PORT=5000
DATABASE_URL="file:./dev.db"
JWT_SECRET="replace-with-a-long-random-local-secret"
NODE_ENV="development"
```

Do not commit real `.env`.

`DATABASE_URL` for tests must point to an isolated test database.

---

# 16. Seed Data

Seed data is development-only.

Demo credentials:

```text
Email: demo@paisatrack.local
Password: Password@123
```

Seed:

### Accounts

```text
HDFC Bank       BANK_ACCOUNT
Cash Wallet     CASH
SBI Credit Card CREDIT_CARD
Paytm UPI       UPI_WALLET
```

### Categories

All canonical default categories plus income categories.

### Transactions

Include:

- several income transactions
- food expenses
- rent
- shopping
- bills
- subscriptions
- education
- travel

Include transactions across more than one month so analytics and month-over-month comparisons work.

### Budgets

At least:

```text
Food
Rent
Shopping
Entertainment
```

for the current month.

Include realistic utilization levels.

### Recurring Rules

Include:

- monthly rent
- monthly subscription
- monthly salary/income

The seed should demonstrate the recurring system without generating duplicates.

Seed data must be deterministic for tests. Test fixtures must not depend on wall-clock time. Development seed data may use the current date for presentation, but recurring rules must be initialized deliberately so running the seed does not unexpectedly create a different number of transactions each time.

---

# 17. Testing Specification

Testing is mandatory.

## 17.1 Backend Unit Tests

Test:

- savings formula
- savings rate
- zero-income handling
- daily average
- month-over-month calculation
- category aggregation
- budget utilization
- budget threshold classification
- insight rules
- recurring date calculation
- monthly end-of-month handling
- leap-year handling
- duplicate recurring occurrence prevention
- account balance adjustments

## 17.2 Backend Integration/API Tests

Test:

### Authentication

- registration
- duplicate email
- login
- invalid credentials
- logout
- `/api/auth/me`
- expired JWT
- protected routes

### Ownership

- user A cannot access user B transaction
- user A cannot edit user B account
- user A cannot delete user B budget
- user A cannot access user B recurring rule

### Transactions

- create
- read
- update
- delete
- validation
- filtering
- amount range
- search semantics
- pagination
- stable sorting/tie-breaking
- CSV export
- account balance effects

### Accounts

- CRUD
- balance updates
- account-in-use deletion rejection
- opening balance behavior
- account type/name update behavior

### Categories

- default categories visible
- custom category creation
- duplicate name rejection
- category ownership
- category type compatibility
- custom category update
- category-in-use deletion rejection

### Budgets

- create
- duplicate month/category rejection
- update
- delete
- utilization
- threshold states
- month isolation

### Recurring

- create
- update
- delete
- inactive rule
- missed occurrence generation
- multiple missed occurrences
- duplicate prevention using the unique occurrence key
- recurrence anchor preservation for day 31 and February 29
- month boundaries
- leap year
- account balance updates

## 17.3 Frontend Tests

Use React Testing Library.

Test:

- login form
- registration validation
- protected route behavior
- dashboard rendering
- transaction form
- transaction filters
- pagination controls
- budget progress states
- recurring form
- loading state
- empty state
- error state
- confirmation dialogs
- theme selection

## 17.4 Security Tests

Explicitly test:

- unauthenticated API access
- invalid JWT
- expired JWT
- cross-user access
- CSRF token retrieval
- CSRF failure on state-changing requests
- CORS credential behavior
- invalid input
- malformed IDs
- sensitive error suppression

## 17.5 Edge Cases

Must include:

```text
zero income
zero expenses
negative savings
exactly 70% budget
exactly 90% budget
exactly 100% budget
125% budget
empty transaction history
thousands of transactions
amount with 2 decimals
invalid amount
future date
invalid category
invalid account
month boundary
year boundary
February 28
February 29 leap year
monthly recurrence on day 31
multiple missed recurring occurrences
inactive recurring rule
duplicate recurring generation
editing an existing transaction
deleting an existing transaction
```

No tests may be skipped merely to obtain a passing test suite.

Do not weaken assertions to hide failures.

---

# 18. Performance Requirements

Transaction listing must use server-side pagination.

Default:

```text
25
```

Maximum:

```text
100
```

Supported:

```text
25
50
100
```

Filtering, searching, sorting, and pagination must happen in the backend/database.

Required indexes are already defined in the Prisma schema.

The frontend must not download thousands of transactions merely to filter them locally.

No virtualization or infinite scroll is required for MVP.

---

# 19. Data Portability

CSV export is the only data-export feature.

Export respects current transaction filters.

Canonical columns:

```text
Date
Type
Amount
Category
Description
Account
Payment Method
Notes
```

Encoding:

```text
UTF-8
```

Date:

```text
DD/MM/YYYY
```

Amount:

```text
numeric rupee amount without currency symbol
```

Filename:

```text
transactions_YYYY-MM-DD.csv
```

No CSV import, OFX import, PDF export, or bank statement import.

---

# 20. User Flows

## Registration

```text
Register
↓
Validate fields
↓
Create user
↓
Hash password
↓
Persist user
↓
Set authenticated cookie
↓
Dashboard
```

## Login

```text
Login
↓
Validate credentials
↓
Verify bcrypt password
↓
Issue 24h JWT cookie
↓
Dashboard
```

## Transaction Creation

```text
Add Transaction
↓
Select type
↓
Select category
↓
Select account
↓
Enter amount
↓
Enter date
↓
Submit
↓
Backend validation
↓
Database transaction
↓
Persist transaction
↓
Update account balance
↓
Refresh dashboard/list
```

## Transaction Edit

```text
Existing transaction
↓
Edit
↓
Reverse old balance effect
↓
Validate new values
↓
Apply new balance effect
↓
Persist changes atomically
```

## Budget

```text
Create budget
↓
Select category
↓
Select month/year
↓
Enter amount
↓
Validate uniqueness
↓
Persist
↓
Calculate utilization
↓
Display threshold
```

## Recurring

```text
Create rule
↓
Set next occurrence
↓
Rule becomes active
↓
User logs in later
↓
Generate all due occurrences
↓
Update nextOccurrence
↓
Update balances
↓
Show generated transactions
```

---

# 21. Canonical Enums

## TransactionType

```text
INCOME
EXPENSE
```

## AccountType

```text
CASH
BANK_ACCOUNT
CREDIT_CARD
UPI_WALLET
```

## Frequency

```text
DAILY
WEEKLY
MONTHLY
YEARLY
```

## Budget Status

Derived only:

```text
NORMAL
WARNING
CRITICAL
EXCEEDED
```

## Theme

```text
SYSTEM
LIGHT
DARK
```

---

# 22. Canonical Database Entities

```text
User
Account
Category
Transaction
Budget
RecurringTransaction
```

Relationships:

```text
User
 ├── Accounts
 ├── Categories
 ├── Transactions
 ├── Budgets
 └── RecurringTransactions

Account
 ├── Transactions
 └── RecurringTransactions

Category
 ├── Transactions
 ├── Budgets
 └── RecurringTransactions

RecurringTransaction
 └── Generated Transactions
```

---

# 23. Canonical API Routes

```text
POST   /api/auth/register
POST   /api/auth/login
POST   /api/auth/logout
GET    /api/auth/csrf
GET    /api/auth/me

GET    /api/users/profile
PUT    /api/users/profile
PUT    /api/users/password

GET    /api/accounts
POST   /api/accounts
PUT    /api/accounts/:id
DELETE /api/accounts/:id

GET    /api/categories
POST   /api/categories
PUT    /api/categories/:id
DELETE /api/categories/:id

GET    /api/transactions
POST   /api/transactions
GET    /api/transactions/export
GET    /api/transactions/:id
PUT    /api/transactions/:id
DELETE /api/transactions/:id

GET    /api/budgets
POST   /api/budgets
PUT    /api/budgets/:id
DELETE /api/budgets/:id

GET    /api/recurring
POST   /api/recurring
PUT    /api/recurring/:id
DELETE /api/recurring/:id

GET    /api/dashboard
GET    /api/analytics
GET    /api/insights
```

---

# 24. Implementation Principles

The coding agent must:

1. Read this entire specification before coding.
2. Treat this document as the source of truth.
3. Not invent additional product features.
4. Not replace SQLite with PostgreSQL.
5. Not introduce Docker or Redis.
6. Not require paid APIs.
7. Not require an LLM.
8. Store money as integer paise.
9. Keep account balance updates transactional.
10. Enforce ownership server-side.
11. Implement server-side transaction pagination.
12. Implement recurring catch-up exactly as specified.
13. Use Asia/Kolkata for user-facing calendar calculations.
14. Use the persisted recurring occurrence key to prevent duplicate catch-up transactions.
15. Use backend validation as authoritative.
16. Write automated tests for core business logic.
17. Actually run the tests.
18. Fix implementation errors instead of merely reporting them.
19. Never claim a feature works without testing it.
20. Avoid placeholder buttons and fake functionality.
21. Keep the application understandable and suitable for a single-session build.

---

# 25. Definition of Done

PaisaTrack is complete only when all of the following are true:

## Application

- [ ] frontend starts
- [ ] backend starts
- [ ] SQLite database initializes
- [ ] Prisma migration succeeds
- [ ] seed command succeeds
- [ ] README setup instructions work

## Authentication

- [ ] registration works
- [ ] password is bcrypt-hashed
- [ ] login works
- [ ] JWT is HttpOnly
- [ ] JWT expires after 24 hours
- [ ] logout clears cookie
- [ ] protected routes reject unauthenticated requests
- [ ] CSRF protection works
- [ ] CSRF token endpoint works
- [ ] CORS credentials work for the configured frontend origin

## Transactions

- [ ] create works
- [ ] edit works
- [ ] delete works
- [ ] list works
- [ ] search works
- [ ] filters work
- [ ] amount range filter works
- [ ] sorting works
- [ ] server pagination works
- [ ] CSV export works
- [ ] account balances update correctly

## Accounts

- [ ] all account types work
- [ ] balances are correct
- [ ] ownership is enforced
- [ ] account-in-use deletion is rejected

## Categories

- [ ] default categories exist
- [ ] income categories exist
- [ ] custom categories work
- [ ] custom category updates work
- [ ] ownership is enforced
- [ ] categories in use cannot be deleted

## Budgets

- [ ] monthly budgets work
- [ ] uniqueness works
- [ ] no rollover occurs
- [ ] utilization is correct
- [ ] 70% warning works
- [ ] 90% critical state works
- [ ] 100% exceeded state works
- [ ] historical months remain unchanged

## Recurring Transactions

- [ ] create works
- [ ] edit works
- [ ] delete works
- [ ] inactive rules do not generate
- [ ] missed occurrences are generated
- [ ] generated dates are scheduled dates
- [ ] duplicates are prevented
- [ ] nextOccurrence advances correctly
- [ ] monthly edge dates work
- [ ] leap-year handling works
- [ ] account balances update

## Analytics

- [ ] income correct
- [ ] expenses correct
- [ ] savings correct
- [ ] savings rate handles zero income
- [ ] daily average correct
- [ ] category spending correct
- [ ] month-over-month calculation correct
- [ ] budget utilization correct
- [ ] recurring expense total correct

## Insights

- [ ] all five deterministic rules work
- [ ] zero denominators are safe
- [ ] no LLM is required

## Frontend

- [ ] responsive
- [ ] light theme
- [ ] dark theme
- [ ] system theme
- [ ] loading states
- [ ] empty states
- [ ] error states
- [ ] confirmation dialogs
- [ ] toast feedback
- [ ] keyboard-accessible forms
- [ ] semantic status indicators

## Security

- [ ] cross-user access is blocked
- [ ] invalid JWT is rejected
- [ ] expired JWT is rejected
- [ ] state-changing requests require CSRF protection
- [ ] secrets are environment variables
- [ ] passwords and tokens are not logged
- [ ] sensitive server errors are not exposed

## Testing

- [ ] backend unit tests pass
- [ ] backend integration tests pass
- [ ] frontend component tests pass
- [ ] security tests pass
- [ ] edge-case tests pass
- [ ] no unexplained skipped tests

## Final quality

- [ ] frontend production build succeeds
- [ ] backend production build succeeds
- [ ] no required feature is a placeholder
- [ ] no API contract contradicts the database
- [ ] no database rule contradicts business rules
- [ ] no UI feature requires an undefined backend API
- [ ] README explains setup and demo credentials
- [ ] application can be demonstrated end-to-end from a clean local checkout

---

# 26. Golden-Path Demonstration

A reviewer should be able to perform this sequence:

```text
Start PaisaTrack
      ↓
Login with demo user
      ↓
View seeded dashboard
      ↓
Create an expense
      ↓
Observe account balance decrease
      ↓
Open Budgets
      ↓
Observe updated utilization
      ↓
Open Transactions
      ↓
Filter by category/date
      ↓
Paginate results
      ↓
Export CSV
      ↓
Open Recurring Transactions
      ↓
Inspect active monthly rule
      ↓
Run recurring catch-up test
      ↓
Observe generated historical occurrence
      ↓
Open Analytics
      ↓
Observe updated charts
      ↓
Open Insights
      ↓
Observe deterministic insight
      ↓
Switch Light / Dark / System theme
      ↓
Logout
      ↓
Protected route redirects to Login
```

The entire flow must work locally without paid services or external financial APIs.

---

# 27. Final Constraint

Do not expand this MVP into an enterprise finance platform.

If a feature is not required by this document, do not implement it unless it is necessary for the documented functionality to work.

The objective is a polished, secure, locally runnable, portfolio-worthy expense manager that can realistically be implemented and tested in one continuous coding session.
