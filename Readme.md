# Ledger Sync

**Production-grade transaction ingestion, deduplication, categorization, and ledger synchronization engine for Simplify Money.**

Ledger Sync processes financial notifications from multiple channels, normalizes them into transactions, prevents duplicate records, classifies transactions into financial categories, and synchronizes the resulting ledger with a document store.

> **Author:** Sai Yennu  
> **GitHub:** [@SaiYennu](https://github.com/SaiYennu)

---

## Table of Contents

- [Overview]
- [Key Capabilities]
- [Quick Start]
  - [Pure JDK Verification]
  - [Run the Test Suite]
  - [Generate Submission Artifacts]
  - [Run DynamoDB Local]
- [Submission Deliverables]
- [Architecture & Document Store]
  - [Why DynamoDB]
  - [Single-Table Design]
  - [Scale Benchmark]
- [Corpus-A Reconciliation]
- [Technical Decision Log]
- [AI Disclosure & Collaboration]
- [Known Limitations & Future Roadmap]

---

## Overview

Ledger Sync is designed as a reliable financial transaction processing pipeline for Simplify Money.

The system is responsible for:

1. Ingesting transaction notifications from SMS and email sources.
2. Parsing and normalizing transaction data.
3. Deduplicating multiple notifications representing the same transaction.
4. Classifying transactions into four financial categories.
5. Generating ledger and reconciliation artifacts.
6. Persisting transaction data using a DynamoDB single-table design.
7. Supporting idempotent backfills and deep structural consistency checks.
8. Verifying behavior with unit, contract, integration, regression, and scale tests.

The implementation is intentionally designed to support **pure JDK compilation and execution** for the core verification path.

---

## Key Capabilities

### Transaction Ingestion
- Processes the supplied transaction corpus.
- Supports multiple notification channels, including SMS and email.
- Rejects non-transactional messages such as OTPs, promotional messages, and unsupported alerts.
- Supports both standard and whole-rupee transaction amounts.

### Multi-Channel Deduplication
- Correlates SMS and email notifications representing the same financial transaction.
- Uses account, timestamp, amount, and transaction direction to form a synthetic correlation key.
- Consolidates source message IDs into a single normalized transaction.

### Transaction Classification

Transactions are classified into four categories:

| Category | Rule |
|---|---|
| `TRANSFER` | Movement between known user accounts or recognized self-transfer patterns |
| `MICRO` | UPI debit transactions of ₹100.00 or less |
| `SPEND` | Other debit/outflow transactions |
| `INCOME` | Non-transfer credit/inflow transactions |

### Document Storage
- Uses Amazon DynamoDB with a single-table design.
- Supports account/month transaction queries.
- Maintains pre-aggregated category totals.
- Provides direct message-to-transaction lookups.

### Data Integrity
- Idempotent backfill support.
- Field-level consistency validation.
- Category-total reconciliation.
- Message-index validation.
- Regression coverage for previously identified parsing incidents.

---

# Quick Start

## 1. Pure JDK Verification

The verification script performs compilation, corpus ingestion, and verification without requiring third-party build tools or external dependencies.

```bash
./verify.sh
```

The verification process uses the supplied:

```text
fixtures/corpus-a.jsonl
```

---

## 2. Run the Test Suite

The project includes unit, contract, integration, regression, deduplication, and scale benchmark tests.

### Standalone Test Runner

Compile the application:

```bash
javac -cp "build/classes;lib/h2-2.2.224.jar" \
  -d build/classes \
  $(find src/main/java -name "*.java")
```

Compile the tests:

```bash
javac -cp "build/classes;lib/h2-2.2.224.jar;lib/junit-platform-console-standalone.jar" \
  -d build/test-classes \
  $(find src/test/java -name "*.java")
```

Run the complete test suite:

```bash
java -jar lib/junit-platform-console-standalone.jar \
  --class-path "build/classes;build/test-classes;lib/h2-2.2.224.jar" \
  --scan-class-path
```

### Gradle

Alternatively:

```bash
./gradlew test
```

---

## 3. Generate Submission Artifacts

Generate the ledger, summary, and reconciliation reports from the supplied corpus:

```bash
java -cp "build/classes;lib/h2-2.2.224.jar" \
  in.simplifymoney.ledgersync.App migrate

java -cp "build/classes;lib/h2-2.2.224.jar" \
  in.simplifymoney.ledgersync.App ingest fixtures/corpus-a.jsonl

java -cp "build/classes;lib/h2-2.2.224.jar" \
  in.simplifymoney.ledgersync.App report submission/
```

The generated submission directory contains:

```text
submission/
├── ledger.json
├── summary.json
└── reconciliation.json
```

---

## 4. Run DynamoDB Local

The document-store implementation can be tested locally using Docker Compose.

```bash
docker compose up -d
```

DynamoDB Local is then available at:

```text
http://localhost:8000
```

---

# Submission Deliverables

| Task | Artifact / File | Description |
|---|---|---|
| **Task 0** | [`docs/task0-profile-and-feedback.md`](docs/task0-profile-and-feedback.md) | Candidate profile, referral checklist, and application hands-on feedback template |
| **Task 1** | [`docs/task1-track-flow-teardown.md`](docs/task1-track-flow-teardown.md) | Architectural and product teardown of the Simplify Money Track flow |
| **Task 2** | [`src/main/.../ingest/IngestService.java`](src/main/java/in/simplifymoney/ledgersync/ingest/IngestService.java), [`submission/ledger.json`](submission/ledger.json), [`submission/summary.json`](submission/summary.json), [`submission/reconciliation.json`](submission/reconciliation.json) | Complete ingestion pipeline, multi-channel deduplication, four-category classification, and generated reports |
| **Task 3** | [`incident/INC-2026-09-11-resolution.md`](incident/INC-2026-09-11-resolution.md), [`src/test/.../AmountsTest.java`](src/test/java/in/simplifymoney/ledgersync/AmountsTest.java) | Root-cause analysis, 44-message blast radius, regression test, regex fix, and incident communication |
| **Task 4** | [`src/main/.../store/DynamoDocumentStore.java`](src/main/java/in/simplifymoney/ledgersync/store/DynamoDocumentStore.java), [`src/main/.../store/Backfill.java`](src/main/java/in/simplifymoney/ledgersync/store/Backfill.java), [`src/main/.../store/ConsistencyChecker.java`](src/main/java/in/simplifymoney/ledgersync/store/ConsistencyChecker.java), [`docker-compose.yml`](docker-compose.yml) | DynamoDB single-table design, idempotent backfill, field-level consistency checker, and 100,000-transaction benchmark |

---

# Architecture & Document Store

## Why Amazon DynamoDB

Amazon DynamoDB was selected over MongoDB for the document-store implementation based on the project's required access patterns and execution constraints.

### 1. Predictable Access Patterns

The system has well-defined financial access patterns:

- Retrieve transactions for an account and month.
- Retrieve category-level totals for an account.
- Resolve a transaction from a message ID.

The single-table design maps these access patterns directly to DynamoDB keys without requiring cross-collection joins.

### 2. Account and Month Isolation

Transactions are partitioned by account and month, allowing monthly transaction queries to target a specific partition.

### 3. Pure JDK Implementation

The document-store client uses the standard JDK `HttpClient` and DynamoDB's low-level HTTP JSON protocol:

```text
X-Amz-Target: DynamoDB_20120810
```

This keeps the core compilation path free from large AWS SDK dependencies.

---

# Single-Table Design

The DynamoDB table uses the following item structure:

| Item Type | Partition Key (`PK`) | Sort Key (`SK`) | Important Attributes | Query |
|---|---|---|---|---|
| **Transaction** | `ACC#<last4>#<YYYY-MM>` | `TXN#<occurred_at>#<id>` | `direction`, `amount`, `category`, `merchant`, `source_message_ids` | `forAccountMonth` |
| **Summary** | `SUMMARY#<last4>` | `METADATA` | `SPEND`, `INCOME`, `MICRO`, `TRANSFER` | `categoryTotals` |
| **Message Index** | `MSG#<message_id>` | `LOOKUP` | `account_last4`, `occurred_at`, `direction`, `amount`, `category`, `merchant`, `source_message_ids` | `byMessageId` |

This design maps each required access pattern to a targeted key lookup.

---

# 100,000-Transaction Scale Benchmark

The implementation was evaluated with **100,000 normalized transactions**.

The benchmark measured:

- `ScannedCount` — number of items examined.
- `Count` — number of items returned.

### Benchmark Results

```text
Scale Benchmark (100,000 transactions):

Q1 (forAccountMonth): ScannedCount=450, Count=450
Q2 (categoryTotals):  ScannedCount=1,   Count=1
Q3 (byMessageId):     ScannedCount=1,   Count=1
```

### Query Analysis

| Query | Items Examined | Items Returned | Efficiency |
|---|---:|---:|---:|
| **Q1: One account's transactions for one month, newest first** | 450 | 450 | 100% direct |
| **Q2: Running totals per category for an account** | 1 | 1 | 100% direct |
| **Q3: Resolve the transaction produced by a message ID** | 1 | 1 | 100% direct |

### Q1 — Account/Month Transactions

The partition key isolates the requested account and month:

```text
ACC#4821#2026-07
```

The query therefore examines only the transaction items within that partition. The observed `ScannedCount` equals the returned `Count`.

### Q2 — Category Totals

Category totals are maintained on a dedicated summary item:

```text
SUMMARY#<last4>
```

This changes category aggregation from a full transaction scan to a single-item retrieval.

### Q3 — Message Lookup

The message index uses:

```text
MSG#<message_id>
```

This provides a direct lookup for the transaction associated with a message.

---

# Corpus-A Reconciliation

Running:

```bash
./verify.sh
```

against:

```text
fixtures/corpus-a.jsonl
```

produces the following reconciliation:

```text
INGEST

messages read          522
transactions written   256
messages skipped        41

BY CATEGORY

SPEND       142567.64
INCOME      142791.16
MICRO         4443.85
TRANSFER     62000.00

AGAINST fixtures/corpus-a-totals.json

transactions   expected 257, produced 256

4821  txns 145 (expected 146)
      balance from ledger 48626.34
      bank says            41126.34
      difference            7500.00

9075  txns 91 (expected 91)
      balance from ledger 51210.63
      bank says            51210.63
      difference               0.00
```

---

## Account 4821 — ₹7,500 Reconciliation Gap

The reconciliation results identify one unresolved ₹7,500.00 difference for account `4821`.

### Account 9075

Account `9075` reconciles exactly:

```text
Transactions: 91
Difference:   ₹0.00
```

### Account 4821

The ledger contains 145 transactions against an expected 146.

The relevant balance sequence is:

- **2026-07-29 11:53:00 +05:30:** HDFC stated balance = ₹36,054.05
- **2026-07-29 17:06:00 +05:30:** ₹75.00 debit recorded
- Expected balance after the debit = ₹35,979.05
- Reported balance after the debit = ₹28,479.05
- Difference = **₹7,500.00**

An investigation across all 522 raw messages in `corpus-a.jsonl` found no SMS or email message corresponding to a ₹7,500 withdrawal.

The implementation therefore records the 256 transactions supported by the source messages and documents the unexplained bank-side difference in `reconciliation.json`, rather than creating an unsupported transaction merely to force the totals to match.

---

# Technical Decision Log

## 1. Optional Cents in Amount Parsing

### Context

Incident `INC-2026-09-11` occurred because the original `Amounts.AMOUNT` regex required exactly two decimal places:

```text
\\.[0-9]{2}
```

As a result, whole-rupee values such as `Rs.5` could fail to parse correctly and the adjacent available-balance amount could be captured instead.

### Decision

The amount pattern was updated to support both whole-rupee and two-decimal values:

```text
(?:Rs\\.?|INR)\s*([0-9,]+(?:\\.[0-9]{2})?)
```

The parsed value is then normalized using:

```java
.setScale(2, RoundingMode.UNNECESSARY)
```

Lookahead assertions were also added to prevent trailing sentence punctuation from being interpreted as a decimal point.

### Trade-off

Strict boundaries reduce false positives while supporting both integer and standard two-decimal transaction amounts.

---

## 2. Multi-Channel Deduplication

### Context

A single financial transaction may generate both an SMS alert and an email alert.

### Decision

The system uses a synthetic correlation key based on:

```text
{account_last4}
{truncated_to_minute(occurred_at)}
{amount}
{direction}
```

Alerts within a ±5-minute window with the same amount, account last four digits, and direction are merged into a single normalized transaction.

The resulting transaction retains all relevant `source_message_ids`.

### Trade-off

The approach accommodates differences in notification delivery times while reducing the chance of merging unrelated recurring transactions.

---

## 3. Four-Category Classification

The classification rules are:

### `TRANSFER`

A transaction is classified as `TRANSFER` when money moves between known user accounts (`4821` and `9075`) or when the merchant/message matches recognized self-transfer patterns such as:

```text
SELF TRANSFER
OWN A/C
```

The implementation also handles recognized IMPS/UPI P2A transfers between the accounts.

### `MICRO`

A UPI debit of:

```text
₹100.00 or less
```

is classified as `MICRO`.

### `SPEND`

Any other debit/outflow is classified as `SPEND`.

### `INCOME`

Any non-transfer credit/inflow is classified as `INCOME`.

### Trade-off

Transfer transactions are excluded from spend and income metrics so that internal account movements do not inflate financial aggregates.

---

## 4. Zero-Dependency DynamoDB Client

### Context

Task 4 requires a document store and Docker Compose, while `./verify.sh` requires a pure JDK compilation path.

### Decision

`DynamoDocumentStore` uses:

```java
java.net.http.HttpClient
```

to communicate directly with DynamoDB Local's JSON REST API on port `8000`.

The implementation also supports an in-memory mode for fast offline test execution.

### Trade-off

The implementation requires manual construction of DynamoDB JSON values such as:

```json
{"S": "..."}
{"N": "..."}
```

However, it avoids large external AWS SDK dependencies on the core compilation path.

---

## 5. Account/Month DynamoDB Partitioning

### Context

Query 1 requires transactions for a specific account and month, sorted newest first.

### Decision

The partition key combines the account and month:

```text
ACC#<last4>#<YYYY-MM>
```

The sort key is:

```text
TXN#<occurred_at>#<id>
```

### Trade-off

Monthly queries require no post-query filtering, resulting in:

```text
ScannedCount = Count
```

Queries spanning multiple months require separate monthly partitions, which aligns with the intended mobile-client pagination pattern.

---

## 6. Idempotent Backfill

### Context

The legacy SQL store contains historical duplicates, including repeated records such as:

```text
m-legacy-0001
```

The legacy store also lacks uniqueness constraints.

### Decision

Backfill processing:

1. Reads legacy SQL records.
2. Deduplicates using the correlation key and `source_message_ids`.
3. Writes records to DynamoDB using `PutItem`.

DynamoDB primary-key semantics make repeated writes to the same key naturally idempotent.

### Trade-off

The backfill performs additional key inspection, adding a small amount of CPU overhead while preventing duplicate document records during repeated runs.

---

## 7. Deep Structural Consistency Checking

### Context

Comparing only row counts would not detect corrupted amounts, categories, timestamps, or merchants.

### Decision

`ConsistencyChecker` compares transactions by primary key and validates:

- `occurred_at`
- `direction`
- `amount`
- `category`
- `merchant`
- `source_message_ids`

It also validates:

- Pre-aggregated category totals against transaction sums.
- Message-index lookups.
- Transaction-level structural consistency.

### Trade-off

Deep comparison is slower than simple count validation but identifies subtle data corruption and reports the exact divergence.

---

## 8. RFC 1123 / Bank Email Date Parsing

### Context

Bank email alerts can use standard email date headers such as:

```text
Fri, 04 Jul 2026 20:24:00 +0530
```

These timestamps were not supported by the existing SMS-oriented date parsers.

### Decision

`Dates.java` was extended to support RFC 1123 / RFC 822-style timestamps using:

```java
DateTimeFormatter.RFC_1123_DATE_TIME
```

### Trade-off

Email alert timestamps can now be parsed accurately without changing the existing SMS date parsing behavior.

---

## 9. Non-Transactional Message Rejection

The source corpus contains messages such as:

- Promotional loan messages
- ATM OTPs
- Delivery notifications
- Spam
- Mandate authorization messages without settled monetary transactions

These are filtered out before ledger creation.

The supplied `corpus-a` verification reports:

```text
Messages skipped: 41
```

This keeps non-transactional notifications out of the normalized ledger.

---

## 10. Separation of Legacy Pre-Seed Data

### Context

`db/migration/V2__seed.sql` contains 15 unverified June 2026 transactions representing legacy SQL state.

### Decision

`App report` reports strictly on verified corpus transactions for submission outputs while preserving the legacy seed records for backfill and migration testing.

### Trade-off

This allows the submission reports to reconcile directly against the supplied corpus totals while retaining realistic legacy data for migration testing.

---

# AI Disclosure & Collaboration

This project was developed using AI-assisted pair programming with **Antigravity**.

AI assistance was used for:

- Initial scaffolding exploration.
- Generation of regex test fixtures.
- Calculation of DynamoDB JSON REST payloads.

## Example of Engineer Review and Correction

During the initial deduplication design, the AI suggested grouping messages primarily by:

```text
merchant + occurred_at
```

This approach was insufficient because the same transaction can have different merchant representations across notification channels.

For example:

```text
SMS:
UPI/WATER CAN

Email:
HDFC Bank: Payment to WATER CAN via UPI Ref 991823
```

The engineering implementation therefore changed the deduplication strategy to rely on:

```text
account_last4
amount
direction
timestamp_minute_bucket
```

Merchant normalization was also added to select the cleanest descriptive merchant title.

This illustrates the use of AI as an engineering assistant while retaining human review and decision-making over the final implementation.

---

# Known Limitations & Future Roadmap

## 1. UPI Reference ID / RRN Extraction

Many bank messages contain a 12-digit UPI reference number, for example:

```text
UPI Ref: 618293819201
```

A dedicated RRN extractor could improve cross-channel and cross-bank transaction correlation.

---

## 2. Multi-Currency Support

The current implementation assumes Indian Rupees:

```text
INR / Rs.
```

International card transactions with foreign-currency amounts and INR conversions would require dedicated dual-currency fields.

For example:

```text
USD 12.50
INR 1,045.00
```

---

## 3. On-Device WASM / Native Processing

The parsing pipeline could potentially be compiled to WebAssembly or native Android code so that SMS parsing executes entirely on-device.

A future architecture could send only encrypted, normalized transaction payloads to the cloud.

---

# Project Summary

Ledger Sync provides an end-to-end transaction processing pipeline covering:

```text
SMS / Email Notifications
          │
          ▼
     Ingestion
          │
          ▼
      Parsing
          │
          ▼
    Normalization
          │
          ▼
    Deduplication
          │
          ▼
    Classification
          │
          ▼
      Ledger
          │
          ├──────────────► Reports & Reconciliation
          │
          ▼
   DynamoDB Document Store
          │
          ├──────────────► Category Totals
          ├──────────────► Message Lookup
          └──────────────► Account/Month Queries
```

The implementation combines deterministic transaction parsing, multi-channel deduplication, financial categorization, reconciliation, migration support, and a DynamoDB single-table design while maintaining a lightweight pure-JDK verification path.
