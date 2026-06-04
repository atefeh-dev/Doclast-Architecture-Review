# Doclast Backend Architecture Report

### Document Lifecycle Management — Engineering Design for CTO Review

**Version:** 3.0  
**Date:** June 2026  
**Status:** Final

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Lessons from SidebarVulcan](#2-lessons-from-sidebarvulcan)
3. [Core Architecture Principles](#3-core-architecture-principles)
4. [Database Schema](#4-database-schema)
5. [Lifecycle State Machine](#5-lifecycle-state-machine)
6. [Event-Driven Architecture](#6-event-driven-architecture)
7. [Email Automation](#7-email-automation)
8. [Scheduled Jobs and Workflow Automation](#8-scheduled-jobs-and-workflow-automation)
9. [Audit Trail and Legal Compliance](#9-audit-trail-and-legal-compliance)
10. [Missing Services and Gaps](#10-missing-services-and-gaps)
11. [Scalability and Multi-Tenancy](#11-scalability-and-multi-tenancy)
12. [System Architecture Diagrams](#12-system-architecture-diagrams)
13. [Implementation Roadmap](#13-implementation-roadmap)
14. [What Not to Build](#14-what-not-to-build)
15. [Appendix](#15-appendix)

---

## 1. Executive Summary

Doclast's product is a **document lifecycle system**. Every engineering decision should flow from that fact. The generate → send → review → approve → sign workflow is not a feature sitting on top of an application — it _is_ the application. Every layer of the backend (schema, events, automation, emails) must be designed around this lifecycle from the start.

SidebarVulcan, a production newsletter platform by Sacha Greif, demonstrates exactly this pattern at scale — organized around a single central object (`Post`) moving through a defined lifecycle. Its architecture translates directly to Doclast, with three additions Sidebar doesn't require: cryptographic signature verification, a tamper-evident compliance trail, and sequential multi-party signing.

### Five Decisions That Define the Architecture

These are architectural commitments, not implementation choices. Getting them wrong early is expensive to correct later.

| #   | Decision                                                                             | Why It Matters                                                                                            |
| --- | ------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------- |
| 1   | Store lifecycle timestamps as first-class schema fields                              | Enables time-aware queries, analytics, reminders, and compliance without reconstructing history           |
| 2   | Route all state transitions through a single `DocumentService.transition()` function | Prevents state drift, enforces business rules, and guarantees the audit log is always complete            |
| 3   | Maintain an append-only event log separate from the mutable document record          | The event log is legal evidence; the document record is operational state — they serve different purposes |
| 4   | Run document generation (PDF/DOCX rendering) behind a job queue                      | Rendering is CPU-bound; it cannot share a process with HTTP request handling at any meaningful scale      |
| 5   | Design signatures to be verifiable without Doclast's servers                         | A signed contract must remain provable even if Doclast is unavailable or the customer leaves the platform |

---

## 2. Lessons from SidebarVulcan

### 2.1 The Central Insight

Sidebar organizes every layer of its system around one question: _where is this object in its lifecycle?_ The schema answers it with timestamps. The callbacks react to lifecycle transitions. The emails communicate them to stakeholders. The cron layer advances the lifecycle when no user action has occurred.

For Doclast, the equivalent question is: _where is this document in its journey from draft to signed?_

### 2.2 Patterns Worth Adopting Directly

**Lifecycle timestamps alongside status**  
Sidebar stores `scheduledAt`, `postedAt`, and `paidAt` as explicit fields, not derived values. This is deliberate design: `status` tells you _where_ the object is now; timestamps tell you _when it got there and how long it has been there_. Both dimensions are required for reminders, expiry enforcement, SLA tracking, and legal evidence. Doclast needs the same discipline.

**Declarative email registry**  
In Sidebar, each email is a self-contained object specifying its template, subject line, recipient resolution, and data requirements. The caller provides only the document ID and the email key — it never prepares data or constructs subjects. This design makes the full set of system emails discoverable from a single file and makes each email independently testable. It is the most directly portable pattern in the codebase.

**Isolated cron layer**  
Sidebar's scheduled work lives in a dedicated file with its own error handling and observability, completely separate from mutation logic. Time-based triggers (reminders, expiry enforcement) and user-triggered mutations are different code paths. They must never share implementation, because they have different failure modes, different retry semantics, and different observability requirements.

**Named callbacks on business events**  
Sidebar registers callbacks per CRUD operation (`posts.new.after`). Doclast should register them per _business event_ (`document.signed`). The event name carries intent that a database operation name does not, and it decouples the business logic layer from the persistence layer.

### 2.3 Anti-Patterns to Avoid

**Magic strings in callback context**  
Sidebar's callbacks check `context.event === 'some-string'` internally. This creates invisible coupling that TypeScript cannot catch at compile time and that tests will not surface until production. Doclast must use a typed event union enforced from day one.

**Single-recipient email model**  
Sidebar emails go to one person. Doclast needs sequential multi-party signing where each signer has their own notification chain, reminder schedule, and per-signer audit record. This cannot be retrofitted onto a single-signer model.

**Coupled storage**  
Sidebar stores content directly in its application database. Doclast generates files that must be saved to customer-controlled storage (Drive, Dropbox, Box). The lifecycle layer must be agnostic to which provider is active — storage is a plugin, not a dependency.

---

## 3. Core Architecture Principles

These principles govern every design decision in this report. When a specific implementation question arises, these are the tie-breakers.

**1. The document lifecycle is the system.**  
Every service, table, and callback exists to serve the document's journey from draft to signed. If a component cannot be explained in terms of its role in that journey, it does not belong in the core architecture.

**2. One transition function. No exceptions.**  
All state changes — whether triggered by an API call, a cron job, a webhook, or an admin command — pass through `DocumentService.transition()`. There is no valid shortcut. Any bypass is a bug, even if it is convenient.

**3. Status and timestamp are a pair.**  
Every time `status` changes, its corresponding timestamp is written in the same database transaction. They are never updated independently. Violating this invariant corrupts the audit trail and breaks time-aware features.

**4. Callbacks are side effects, not state managers.**  
A callback reacts to a lifecycle event. It sends an email, saves a file, or appends a metric. It does not own state transitions. If a callback needs to change document state, it calls `DocumentService.transition()` — it does not write to the database directly.

**5. The audit log is immutable.**  
The `document_events` table is append-only at the database level. No application code, no migration script, and no administrative tool may update or delete rows from it. Its integrity is what makes it legally defensible.

**6. Design for independent verifiability.**  
Every signed document must carry enough information for any party — including a court — to verify the signature without trusting or contacting Doclast. The signature hash is self-contained evidence.

---

## 4. Database Schema

### 4.1 The Documents Table

The `documents` table is the mutable operational record. It holds current state and the full lifecycle timestamp history. It is not the audit trail — that is the `document_events` table.

```ts
interface Document {
  // ── Identity ────────────────────────────────────────────────────────────
  id: string; // UUID v4
  ownerId: string; // FK → users.id
  workspaceId: string; // FK → workspaces.id (required from day one — see §11.3)
  title: string;
  templateId: TemplateId; // 'NDA' | 'collaboration' | 'MOU' | custom

  // ── Lifecycle ───────────────────────────────────────────────────────────
  // Invariant: status and its corresponding timestamp are always written together.
  // Invariant: timestamps are never overwritten. Re-sends use resentAt, not sentAt.
  status: DocumentStatus;
  createdAt: Date;
  sentAt: Date | null;
  firstViewedAt: Date | null;
  reviewingAt: Date | null;
  approvedAt: Date | null;
  signedAt: Date | null;
  expiredAt: Date | null;
  revokedAt: Date | null;
  resentAt: Date | null; // populated on resend; sentAt is preserved

  // ── Expiry and reminders ─────────────────────────────────────────────────
  expiresAt: Date | null; // deadline set by owner at creation
  expiryWarnedAt: Date | null; // set when 48h warning email is sent
  reminderSentAt: Date | null; // updated on each reminder send

  // ── Form data ────────────────────────────────────────────────────────────
  fields: Record<string, string>; // template-specific field values

  // ── Storage ──────────────────────────────────────────────────────────────
  // The generated file lives in the owner's connected storage, not on Doclast servers.
  storageProvider: "drive" | "dropbox" | "box" | "internal" | null;
  storagePath: string | null;
  storageFileId: string | null;

  // ── Signature ────────────────────────────────────────────────────────────
  signatureHash: string | null; // SHA-256(documentBytes + signerEmail + signedAt + documentId)
  signatureMethod: "otp_email" | "docusign" | null;

  // ── Schema versioning ────────────────────────────────────────────────────
  schemaVersion: number; // incremented on breaking schema changes
  updatedAt: Date;
}

type DocumentStatus =
  | "draft" // fields being filled; not yet sent
  | "sent" // email dispatched to signer(s)
  | "viewed" // at least one signer has opened the signing link
  | "reviewing" // signer is actively reading the document
  | "approved" // signer has accepted; awaiting signature
  | "signed" // all signers have signed; terminal
  | "expired" // expiresAt passed before completion; terminal
  | "revoked"; // owner cancelled; terminal
```

### 4.2 The Document Signers Table

Treating a document as having a single signer breaks the moment a customer requests multi-party signing. This is not a V2 concern — it affects the schema, the event model, and the reminder system. Design for it now.

```ts
interface DocumentSigner {
  id: string;
  documentId: string; // FK → documents.id
  email: string;
  name: string | null;
  role: string | null; // 'party_a' | 'party_b' | 'witness' | etc.

  // Signing order: signer with order 2 is not notified until all order-1 signers have signed.
  // Signers with the same order value sign in parallel.
  signingOrder: number;

  status: SignerStatus;

  // Per-signer lifecycle timestamps — mirror the document-level pattern
  notifiedAt: Date | null;
  viewedAt: Date | null;
  signedAt: Date | null;
  declinedAt: Date | null;
  reminderSentAt: Date | null;

  // Per-signer signature evidence
  signatureHash: string | null;
  ipAddress: string | null; // collected at signing time for audit trail
  userAgent: string | null;
}

type SignerStatus = "pending" | "notified" | "viewed" | "signed" | "declined";
```

### 4.3 The Document Events Table

This table is the legal source of truth. It records everything that happens to a document, in order, with a tamper-evident integrity chain. It is never updated and never deleted.

```ts
interface DocumentEvent {
  id: string; // UUID v4
  documentId: string; // FK → documents.id

  event: DocumentEventType;

  // Who caused this event
  actorType: "user" | "signer" | "system" | "cron";
  actorId: string | null; // user ID, signer email, job name, or null for system

  occurredAt: Date;

  // Structured event-specific data (IP address, user agent, template version, etc.)
  metadata: Record<string, unknown>;

  // Tamper-evident integrity chain.
  // integrityHash = SHA-256(previousEvent.integrityHash + eventType + actorId + occurredAt + metadata)
  // Any modification to a past event invalidates all subsequent hashes.
  previousEventId: string | null;
  integrityHash: string;
}

type DocumentEventType =
  | "document.created"
  | "document.fields_updated"
  | "document.generated"
  | "document.sent"
  | "document.resent"
  | "document.viewed"
  | "document.reviewing_started"
  | "document.approved"
  | "document.signed"
  | "document.expired"
  | "document.revoked"
  | "document.reminder_sent"
  | "document.expiry_warning_sent"
  | "document.storage_saved"
  | "signer.added"
  | "signer.notified"
  | "signer.viewed"
  | "signer.signed"
  | "signer.declined";
```

### 4.4 Index Strategy

Indexes are not an afterthought. The cron jobs and dashboard queries that run most frequently need covering indexes defined from the first migration.

```sql
-- Dashboard queries: owner views their document list
CREATE INDEX idx_documents_owner
  ON documents (owner_id, status, created_at DESC);

-- Partial index for active documents only — keeps it small
CREATE INDEX idx_documents_active_expiry
  ON documents (expires_at)
  WHERE status NOT IN ('signed', 'expired', 'revoked');

-- Cron: find documents due for signature reminders
CREATE INDEX idx_documents_reminder
  ON documents (sent_at, reminder_sent_at)
  WHERE status IN ('sent', 'viewed');

-- Audit log: reconstruct history for a given document
CREATE INDEX idx_events_document
  ON document_events (document_id, occurred_at DESC);

-- Audit log: verify integrity chain
CREATE INDEX idx_events_chain
  ON document_events (previous_event_id);

-- Signer queries: find signers for a document in signing order
CREATE INDEX idx_signers_document_order
  ON document_signers (document_id, signing_order);

-- Cron: find signers due for reminders
CREATE INDEX idx_signers_reminder
  ON document_signers (status, notified_at)
  WHERE status IN ('notified', 'viewed');
```

---

## 5. Lifecycle State Machine

### 5.1 Valid Transitions

Encoding valid transitions as an explicit map eliminates an entire class of bugs. An invalid transition attempt becomes a loud, immediate error — not a silent data corruption.

```ts
// The complete set of valid lifecycle transitions.
// Terminal states (signed, expired, revoked) have empty arrays — no exits.
const VALID_TRANSITIONS: Record<DocumentStatus, DocumentStatus[]> = {
  draft: ["sent", "revoked"],
  sent: ["viewed", "expired", "revoked", "sent"], // 'sent' → 'sent' permits resend
  viewed: ["reviewing", "expired", "revoked"],
  reviewing: ["approved", "expired", "revoked"],
  approved: ["signed", "expired", "revoked"],
  signed: [],
  expired: [],
  revoked: [],
};

class InvalidTransitionError extends Error {
  constructor(from: DocumentStatus, to: DocumentStatus) {
    super(`Cannot transition document from '${from}' to '${to}'`);
    this.name = "InvalidTransitionError";
  }
}
```

### 5.2 The Transition Function

Every state change in the system — from API handlers, cron jobs, webhooks, and admin tooling — passes through this function without exception. It is the system's single point of truth for lifecycle management.

```ts
// document-service.ts

async function transition(
  documentId: string,
  event: DocumentEventType,
  actor: EventActor,
  metadata: Record<string, unknown> = {},
): Promise<Document> {
  // All writes are atomic: status, timestamp, and audit event commit together
  // or not at all. A partial write is worse than no write.
  const updatedDocument = await db.transaction(async (tx) => {
    // SELECT FOR UPDATE prevents concurrent transitions on the same document.
    const doc = await tx.documents.findByIdForUpdate(documentId);

    const targetStatus = eventToStatus(event);
    assertValidTransition(doc.status, targetStatus);

    const now = new Date();
    const tsField = statusTimestampField(targetStatus); // e.g. 'signedAt' for 'signed'

    const updated = await tx.documents.update(documentId, {
      status: targetStatus,
      [tsField]: now,
      updatedAt: now,
    });

    // The event is appended inside the same transaction.
    // If the document update fails, no event is recorded — the log stays consistent.
    await appendAuditEvent(tx, {
      documentId,
      event,
      actor,
      occurredAt: now,
      metadata,
    });

    return updated;
  });

  // Callbacks execute after the transaction commits.
  // Rationale: a transaction holding a DB lock while awaiting an external HTTP call
  // (email send, storage upload) creates deadlock risk under load. The document
  // state is already committed; a callback failure should trigger a retry, not
  // a rollback. Callbacks are side effects — they do not own state.
  await runCallbacks(event, updatedDocument, actor);

  return updatedDocument;
}
```

### 5.3 State Transition Map

```
                          ┌──────────┐
                          │  draft   │ ◄─── owner fills form
                          └────┬─────┘
                               │ owner clicks Send
                               ▼
                          ┌──────────┐
                    ┌────►│   sent   │◄──── resend (sentAt preserved, resentAt set)
                    │     └────┬─────┘
                 [resend]      │ signer opens link
                    │          ▼
                    │     ┌──────────┐
                    │     │  viewed  │
                    │     └────┬─────┘
                    │          │ signer begins reading
                    │          ▼
                    │     ┌──────────────┐
                    │     │  reviewing   │
                    │     └────┬─────────┘
                    │          │ signer accepts changes
                    │          ▼
                    │     ┌──────────┐
                    │     │ approved │
                    │     └────┬─────┘
                    │          │ signer signs
                    │          ▼
                    │     ┌──────────┐
                    │     │  signed  │ ◄─── terminal
                    │     └──────────┘
                    │
                    │  From any non-terminal state:
                    │
                    ├──────────────────────► expired  ◄─── cron: expiresAt passed
                    │                        (terminal)
                    │
                    └──────────────────────► revoked  ◄─── owner action
                                             (terminal)
```

---

## 6. Event-Driven Architecture

### 6.1 The Event Bus Interface

The event bus is intentionally defined as an interface, not an implementation. In early development, an in-process synchronous emitter is sufficient and easier to debug. As volume grows, the same interface is satisfied by a BullMQ-backed async queue — no call sites change.

```ts
// events/bus.ts

type EventHandler = (
  payload: DocumentEventPayload,
  meta: EventMeta,
) => Promise<void>;

interface EventBus {
  emit(event: DocumentEventType, payload: DocumentEventPayload): Promise<void>;
  on(event: DocumentEventType, handler: EventHandler): void;
  off(event: DocumentEventType, handler: EventHandler): void;
}

// Swap implementation without touching any call site:
// Development:  new InProcessEventBus()
// Production:   new BullMQEventBus(redisConnection)
```

### 6.2 The Callback Registry

The callback registry is the single most important file for a new engineer joining the team. It answers the question: _what does the system do when this event occurs?_ Every side effect is visible here. Nothing is hidden in database triggers, middleware, or scattered service calls.

```ts
// callbacks/index.ts
// Complete registry of all business logic side effects.
// Every event → handler relationship in the system is declared here.

bus.on("document.sent", triggerDocumentSentEmail);
bus.on("document.sent", startSignerReminderTracking);
bus.on("document.viewed", stampFirstViewedAt);
bus.on("document.signed", verifyAndStoreSignatureHash);
bus.on("document.signed", triggerDocumentSignedEmail);
bus.on("document.signed", saveDocumentToStorage);
bus.on("document.expired", triggerDocumentExpiredEmail);
bus.on("document.expired", invalidateSigningLinks);
bus.on("document.revoked", triggerDocumentRevokedEmail);
bus.on("document.revoked", invalidateSigningLinks);
```

### 6.3 Callback Design Contracts

Every callback is a named, exported, independently testable function. It must satisfy four contracts:

**Single responsibility.** Each callback does exactly one thing. `triggerDocumentSentEmail` sends an email. It does not stamp a timestamp, validate document fields, or update any status.

**Idempotency.** If the same event is delivered twice — which happens with any retry-capable queue — the callback produces no additional side effects on the second delivery. Before sending an email, check whether it was already sent. Before saving a file, check whether it already exists.

**No direct state writes.** A callback that needs to change document state calls `DocumentService.transition()`. It never writes to the `documents` table directly. Bypassing the transition function bypasses the audit log.

**Independent failure.** If `saveDocumentToStorage` throws, `triggerDocumentSignedEmail` still runs. Callbacks are isolated. One failure logs and retries; it does not cancel the other handlers for that event.

---

## 7. Email Automation

### 7.1 The Declarative Email Registry

Each email is a self-contained definition. It knows its own template, how to resolve its recipient, how to construct its subject line, and how to fetch its own data. The caller provides only two things: the email key and the document ID.

```ts
// emails/registry.ts

interface EmailDefinition {
  template: string;
  subject: (data: EmailData) => string;
  to: (data: EmailData) => string | string[];
  cc?: (data: EmailData) => string[];
  getData: (documentId: string) => Promise<EmailData>;

  // Used by preview routes and test suites — no live sends required
  testFixture?: EmailData;
}

export const emailRegistry = {
  documentSent: {
    template: "document-sent",
    subject: (d) => `Please review and sign: ${d.document.title}`,
    to: (d) => d.signer.email,
    getData: (id) => EmailDataService.forDocument(id),
    testFixture: fixtures.documentSent,
  },

  signatureReminder: {
    template: "signature-reminder",
    subject: (d) =>
      `Reminder: ${d.document.title} is waiting for your signature`,
    to: (d) => d.signer.email,
    getData: (id) => EmailDataService.forDocument(id),
    testFixture: fixtures.signatureReminder,
  },

  documentSigned: {
    template: "document-signed",
    subject: (d) => `Signed: ${d.document.title}`,
    to: (d) => d.owner.email,
    cc: (d) => d.signers.map((s) => s.email), // all signers receive a confirmed copy
    getData: (id) => EmailDataService.forDocument(id),
    testFixture: fixtures.documentSigned,
  },

  expiryWarning: {
    template: "expiry-warning",
    subject: (d) =>
      `Action required: "${d.document.title}" expires in 48 hours`,
    to: (d) => d.signer.email,
    getData: (id) => EmailDataService.forDocument(id),
    testFixture: fixtures.expiryWarning,
  },

  documentExpired: {
    template: "document-expired",
    subject: (d) => `"${d.document.title}" expired before signing`,
    to: (d) => d.owner.email,
    getData: (id) => EmailDataService.forDocument(id),
    testFixture: fixtures.documentExpired,
  },

  documentRevoked: {
    template: "document-revoked",
    subject: (d) => `"${d.document.title}" has been cancelled`,
    to: (d) => d.signer.email,
    getData: (id) => EmailDataService.forDocument(id),
    testFixture: fixtures.documentRevoked,
  },
} satisfies Record<string, EmailDefinition>;

// The entire public API for sending any email Doclast sends:
export async function sendDocumentEmail(
  key: keyof typeof emailRegistry,
  documentId: string,
): Promise<void> {
  const def = emailRegistry[key];
  const data = await def.getData(documentId);

  await emailProvider.send({
    template: def.template,
    to: def.to(data),
    cc: def.cc?.(data),
    subject: def.subject(data),
    data,
  });
}
```

### 7.2 Provider Abstraction

The email registry must never depend on a specific provider. Switching from Resend to SendGrid to Postmark must require changing exactly one file.

```ts
// emails/provider.ts

interface EmailProvider {
  send(options: SendOptions): Promise<void>;
}

// Resolved at application startup:
export const emailProvider: EmailProvider =
  process.env.NODE_ENV === "test"
    ? new InMemoryEmailProvider()
    : new ResendProvider(process.env.RESEND_API_KEY);
```

### 7.3 Send Deduplication

Retry logic, duplicate event emissions, and race conditions can all cause the same email to be sent twice. Deduplicate at the send layer, not at the call sites.

```ts
export async function sendDocumentEmail(
  key: keyof typeof emailRegistry,
  documentId: string,
): Promise<void> {
  const dedupKey = `email:${key}:${documentId}`;

  // Use Redis with a TTL window, or a deduplicated DB record.
  // The dedup window should be longer than the longest possible retry interval.
  const alreadySent = await dedup.check(dedupKey, { ttlSeconds: 86400 });
  if (alreadySent) {
    logger.warn({ dedupKey }, "Duplicate email suppressed");
    return;
  }

  await dedup.mark(dedupKey);

  const def = emailRegistry[key];
  const data = await def.getData(documentId);

  await emailProvider.send({
    template: def.template,
    to: def.to(data),
    cc: def.cc?.(data),
    subject: def.subject(data),
    data,
  });
}
```

---

## 8. Scheduled Jobs and Workflow Automation

### 8.1 Design Rules for Cron Jobs

Before listing the jobs, three rules:

**Rule 1: Cron jobs use the same interfaces as API handlers.** A cron job that expires a document calls `DocumentService.transition()` exactly as an API handler would. It does not bypass state logic just because there is no HTTP request involved. This guarantees the audit log is always complete.

**Rule 2: Cron queries must be fully index-backed.** An unindexed scan of the documents table at 50,000 rows will degrade noticeably. At 500,000 rows it becomes an operational incident. Every `WHERE` clause in every cron query must have a corresponding index defined in the initial migration (see §4.4).

**Rule 3: Every job iterates with cursor-based pagination.** `findMany({ where: { status: 'sent' } })` without pagination loads every matching row into memory at once. At scale, this causes OOM errors. Use batch iteration from day one.

```ts
// Reusable cursor-based batch iterator
async function* batchQuery<T extends { id: string }>(
  queryFn: (cursor?: string) => Promise<T[]>,
  batchSize = 100,
): AsyncGenerator<T[]> {
  let cursor: string | undefined;
  do {
    const batch = await queryFn(cursor);
    if (!batch.length) return;
    yield batch;
    cursor = batch.at(-1)?.id;
  } while (cursor);
}
```

### 8.2 The Job Registry

```ts
// cron/jobs.ts

export const cronJobs: CronJobDefinition[] = [
  {
    name: "enforceDocumentExpiry",
    schedule: "0 * * * *", // every hour
    handler: async () => {
      for await (const batch of batchQuery((cursor) =>
        db.documents.findMany({
          where: {
            expiresAt: { lte: new Date() },
            status: { notIn: TERMINAL_STATUSES },
          },
          take: 100,
          cursor: cursor ? { id: cursor } : undefined,
          orderBy: { id: "asc" },
        }),
      )) {
        for (const doc of batch) {
          await DocumentService.transition(
            doc.id,
            "document.expired",
            CRON_ACTOR,
          );
        }
      }
    },
  },

  {
    name: "sendSignatureReminders",
    schedule: "0 */6 * * *", // every 6 hours
    handler: async () => {
      const REMINDER_AFTER_HOURS = 48;
      const REMINDER_COOLDOWN_HRS = 24;
      const cutoff = subHours(new Date(), REMINDER_AFTER_HOURS);
      const cooldown = subHours(new Date(), REMINDER_COOLDOWN_HRS);

      for await (const batch of batchQuery((cursor) =>
        db.documents.findMany({
          where: {
            status: { in: ["sent", "viewed"] },
            sentAt: { lte: cutoff },
            expiresAt: { OR: [{ equals: null }, { gt: new Date() }] },
            reminderSentAt: { OR: [{ equals: null }, { lte: cooldown }] },
          },
          take: 100,
          cursor: cursor ? { id: cursor } : undefined,
          orderBy: { id: "asc" },
        }),
      )) {
        for (const doc of batch) {
          await sendDocumentEmail("signatureReminder", doc.id);
          await db.documents.update(doc.id, { reminderSentAt: new Date() });
        }
      }
    },
  },

  {
    name: "sendExpiryWarnings",
    schedule: "0 8 * * *", // daily at 08:00 UTC
    handler: async () => {
      const WARNING_WINDOW_HRS = 48;
      const warningCutoff = addHours(new Date(), WARNING_WINDOW_HRS);

      for await (const batch of batchQuery((cursor) =>
        db.documents.findMany({
          where: {
            expiresAt: { gte: new Date(), lte: warningCutoff },
            status: { notIn: TERMINAL_STATUSES },
            expiryWarnedAt: { equals: null },
          },
          take: 100,
          cursor: cursor ? { id: cursor } : undefined,
          orderBy: { id: "asc" },
        }),
      )) {
        for (const doc of batch) {
          await sendDocumentEmail("expiryWarning", doc.id);
          await db.documents.update(doc.id, { expiryWarnedAt: new Date() });
        }
      }
    },
  },

  {
    name: "advanceSigningQueue",
    schedule: "*/15 * * * *", // every 15 minutes
    handler: async () => {
      // Find documents where all signers of order N have signed
      // but signers of order N+1 have not yet been notified.
      const ready = await SignerService.findReadyForNextSequence();
      for (const { documentId, nextSigners } of ready) {
        await SignerService.notifySigners(documentId, nextSigners);
      }
    },
  },

  {
    name: "archiveAbandonedDrafts",
    schedule: "0 3 * * *", // daily at 03:00 UTC
    handler: async () => {
      const threshold = subDays(new Date(), 30);
      for await (const batch of batchQuery((cursor) =>
        db.documents.findMany({
          where: { status: "draft", updatedAt: { lte: threshold } },
          take: 100,
          cursor: cursor ? { id: cursor } : undefined,
          orderBy: { id: "asc" },
        }),
      )) {
        for (const doc of batch) {
          await DocumentService.transition(
            doc.id,
            "document.revoked",
            CRON_ACTOR,
            { reason: "abandoned_draft_cleanup" },
          );
        }
      }
    },
  },
];
```

### 8.3 Job Observability

Every job emits structured logs and metrics. This is not optional — it is how you know whether the system is working in production.

```ts
// cron/runner.ts

async function runJob(job: CronJobDefinition): Promise<void> {
  const start = Date.now();
  logger.info({ job: job.name }, "cron job started");

  try {
    await job.handler();
    const duration = Date.now() - start;
    metrics.increment("cron.success", { job: job.name });
    metrics.timing("cron.duration_ms", duration, { job: job.name });
    logger.info({ job: job.name, durationMs: duration }, "cron job completed");
  } catch (error) {
    metrics.increment("cron.failure", { job: job.name });
    logger.error({ job: job.name, error }, "cron job failed");
    alerts.fire(`Cron job failed: ${job.name}`, { error, job: job.name });
  }
}
```

---

## 9. Audit Trail and Legal Compliance

### 9.1 The Legal Requirement

A signed Doclast contract is a legal instrument. In the event of a dispute, the following must be provable:

1. The specific document version that was sent (not a later revision)
2. That the signing link was delivered to the correct email address
3. That the signer opened the document and had meaningful access to it
4. That the signature was collected from the party who received the email
5. That the document content was not modified after signing
6. That the audit record itself has not been altered

Without deliberate architecture for each of these, Doclast cannot produce legally credible evidence. This is not a compliance checkbox — it is a core product guarantee.

### 9.2 Append-Only Audit Event Chain

Every call to `DocumentService.transition()` appends a record to `document_events` inside the same database transaction. The record contains a cryptographic hash chained from the previous event. Modifying any past event breaks the hash chain from that point forward, making tampering detectable.

```ts
async function appendAuditEvent(
  tx: Transaction,
  params: {
    documentId: string;
    event: DocumentEventType;
    actor: EventActor;
    occurredAt: Date;
    metadata: Record<string, unknown>;
  },
): Promise<DocumentEvent> {
  const previous = await tx.documentEvents.findLatest(params.documentId);

  // The hash input includes the previous hash, making this a chain.
  // Any modification of a past event breaks all subsequent hashes.
  const integrityHash = computeHash({
    previousHash: previous?.integrityHash ?? null,
    event: params.event,
    actorId: params.actor.id ?? null,
    occurredAt: params.occurredAt.toISOString(),
    metadata: params.metadata,
  });

  return tx.documentEvents.create({
    ...params,
    previousEventId: previous?.id ?? null,
    integrityHash,
  });
}
```

### 9.3 Signature Hash

The signature hash is the cornerstone of Doclast's legal credibility. It allows any party — including a court, without contacting Doclast — to verify that a specific PDF was signed by a specific person at a specific time.

```ts
function computeSignatureHash(params: {
  documentBytes: Buffer; // the exact bytes of the PDF presented for signing
  signerEmail: string;
  signedAt: Date;
  documentId: string;
}): string {
  return createHash("sha256")
    .update(params.documentBytes)
    .update(params.signerEmail)
    .update(params.signedAt.toISOString())
    .update(params.documentId)
    .digest("hex");
}
```

The hash is stored on the `document_signers` row. It is also embedded in the signed PDF (as a metadata field) and included in the confirmation email to the signer. The signer can verify it independently at any time by recomputing the hash from their copy of the PDF.

### 9.4 Audit Trail API

The audit trail endpoint is read-only and publicly accessible to document owners. It verifies the integrity chain on every request and returns a human-readable representation of the document's history.

```ts
// GET /api/documents/:id/audit-trail
// Response:

interface AuditTrailResponse {
  documentId: string;
  chainIntact: boolean; // false if any event hash fails verification
  events: Array<{
    id: string;
    event: DocumentEventType;
    actorType: string;
    actorLabel: string; // human-readable: "Owner", "Signer (alice@example.com)", "System"
    occurredAt: Date;
    description: string; // e.g. "Document sent to alice@example.com"
    metadata: Record<string, unknown>;
    hashValid: boolean;
  }>;
}
```

### 9.5 Database-Level Access Controls

The `document_events` table must be protected at the database level, not just the application level.

```sql
-- The application database user cannot INSERT, UPDATE, or DELETE on this table.
-- Only the dedicated audit_writer role can append rows.
-- This means a compromised application cannot silently alter the audit log.

REVOKE INSERT, UPDATE, DELETE ON document_events FROM app_user;
GRANT  INSERT                  ON document_events TO audit_writer;
GRANT  SELECT                  ON document_events TO app_user;
```

### 9.6 Data Retention Policy

Define retention rules before launch. Retrofitting them into a running system with live legal data is complex and risky.

| Data                              | Retention                                                       | Rationale                                           |
| --------------------------------- | --------------------------------------------------------------- | --------------------------------------------------- |
| Signed documents and audit logs   | 7 years                                                         | Standard contract law minimum in most jurisdictions |
| Unsigned draft documents          | 90 days after last activity                                     | No legal obligation; reduces storage cost           |
| OTP codes                         | Delete immediately on use; expire unused codes after 10 minutes | Security                                            |
| Email delivery receipts           | 1 year                                                          | Sufficient for dispute resolution                   |
| IP addresses in audit records     | Anonymize after 2 years                                         | GDPR compliance                                     |
| Session tokens and refresh tokens | Delete on logout; expire unused tokens after 30 days            | Security                                            |

---

## 10. Missing Services and Gaps

The following services are required for production but are not yet fully addressed in the current architecture. Each is a self-contained module that can be built independently.

### 10.1 Document Generation Service

PDF and DOCX rendering is CPU-bound work that can take 500ms to 3 seconds depending on template complexity. Keeping this in the API request path creates latency spikes, timeouts under load, and cascading failures when the renderer is slow.

The correct pattern: the API enqueues a generation job and returns `202 Accepted` immediately. A pool of worker processes picks up jobs, renders the document, stores it, and emits `document.generated`. The owner receives a notification when their document is ready to send.

This architecture also enables priority queues (Pro plan renders before Free plan), retry logic for failed renders, and horizontal scaling of rendering capacity independent of API capacity.

### 10.2 Signing Link Service

Signing links must satisfy four requirements: unique per document and signer, time-bound to match the document's `expiresAt`, single-use after the document is signed, and self-verifying without a database lookup.

JWT satisfies all four. The token embeds `documentId`, `signerEmail`, and expiry. Verification requires only the signing secret — no database round trip. After a document is signed, the server adds the token's `jti` (JWT ID) to a Redis blocklist. Any subsequent attempt with the same token is rejected.

### 10.3 OTP Verification Service

The current ContractFlow uses OTP for signer identity verification. This service needs explicit design:

- 6-digit codes generated with cryptographic randomness (not `Math.random()`)
- Stored in Redis with a 10-minute TTL — never in the application database
- Rate-limited to 5 attempts per email address per hour (a Redis counter with TTL)
- Immediately deleted from Redis after successful use
- The code value itself never appears in any log line
- A new code request invalidates any existing pending code for that email

### 10.4 Storage Abstraction Layer

Doclast stores documents in the owner's connected storage, not on Doclast's servers. The lifecycle layer must be completely agnostic to which provider is active.

```ts
interface StorageProvider {
  save(params: {
    content: Buffer;
    filename: string;
    mimeType: string;
    folder?: string;
  }): Promise<StoredFile>;

  getDownloadUrl(fileId: string, expiresInSeconds?: number): Promise<string>;
  delete(fileId: string): Promise<void>;
  exists(fileId: string): Promise<boolean>;
}

// Implementations: GoogleDriveProvider, DropboxProvider, BoxProvider
// Factory resolves the correct provider at runtime based on owner configuration:
// StorageFactory.forOwner(owner).save(...)
```

The document record stores `storageProvider`, `storagePath`, and `storageFileId`. No other layer in the system needs to know which provider is active.

### 10.5 Webhook Service

When Doclast expands to support integrations, customers will expect webhooks for events like `document.signed` and `document.expired`. The event log is already the perfect source for these.

Design the webhook system as a subscription model on top of the existing event stream: customers subscribe to specific event types, and a webhook dispatcher reads from `document_events` and delivers matching events to customer endpoints. The dispatcher handles retries, delivery receipts, and signature verification of outbound payloads.

Building the event log correctly now means adding webhooks later requires no schema changes — only the subscriber and dispatcher services.

### 10.6 Rate Limiting

Document creation, signing link generation, and email sends all need rate limits, applied at the workspace level rather than per-user:

| Action                  | Rate Limit                   |
| ----------------------- | ---------------------------- |
| Document creation       | 100 per workspace per hour   |
| Signing link generation | 20 per document              |
| Email sends (any type)  | 500 per workspace per day    |
| OTP requests            | 5 per email address per hour |
| Audit trail reads       | 60 per document per hour     |

---

## 11. Scalability and Multi-Tenancy

### 11.1 Database Scaling

The `documents` and `document_events` tables have fundamentally different characteristics and must be optimized independently.

`documents` is read-heavy and mutated on every transition. It benefits from a read replica for dashboard queries and analytics. Write contention is handled by the row-level lock in `transition()` (see §5.2) — only one transition can proceed on a given document at a time.

`document_events` is write-heavy and strictly append-only. It grows without bound and must be partitioned by time (monthly or yearly) once it exceeds manageable size. For compliance reasons, it must reside in a separate database schema with write access restricted to the audit writer service account (see §9.5).

### 11.2 Generation Queue Scaling

Rendering workers are stateless and CPU-bound. They scale horizontally independent of the API servers. Key configuration decisions:

- **Concurrency per worker:** limit to prevent CPU saturation (typically 2–4 concurrent renders per core)
- **Dead letter queue:** failed render jobs should be captured with full context for debugging and manual retry
- **Priority lanes:** implement separate queues for Pro and Free plan renders so high-value customers are never behind a burst of free-tier activity
- **Timeout:** render jobs that exceed 30 seconds are likely stuck; mark as failed and trigger a retry

### 11.3 Multi-Tenancy

Doclast will serve teams and organizations, not just individual users. The schema must reflect this from the start. `workspaceId` is already present on the `Document` interface (§4.1) — ensure it is also present on `document_signers`, `document_events`, and any future tables.

Row-level access policies are enforced at the service layer, never in SQL. This means:

- `DocumentService` always filters queries by `workspaceId` derived from the authenticated session
- No query across the documents table is issued without a `workspaceId` constraint
- Workspace isolation is tested explicitly as part of the security test suite

### 11.4 Observability Requirements

The following metrics must be captured and visible in a dashboard before launching to production:

| Metric                                                               | Why                                                           |
| -------------------------------------------------------------------- | ------------------------------------------------------------- |
| `documents.transitions_per_minute` by status                         | Detect activity spikes and stalled workflows                  |
| `documents.time_to_sign_p50_p95`                                     | Core SLA metric; identifies friction in the signing flow      |
| `emails.sent`, `emails.bounced`, `emails.opened`                     | Signer engagement; delivery health                            |
| `cron.job_duration_ms` per job                                       | Detect jobs growing slower as data volume increases           |
| `generation.queue_depth` and `generation.render_duration_ms`         | Rendering pipeline health                                     |
| `signing_links.valid`, `signing_links.expired`, `signing_links.used` | Security and expiry health                                    |
| `audit_events.chain_violations`                                      | Should always be zero; any non-zero value is a critical alert |

---

## 12. System Architecture Diagrams

### 12.1 System Layers

```
┌──────────────────────────────────────────────────────────────────┐
│                          CLIENT LAYER                             │
│   Landing page (static HTML)    ContractFlow (Vue 3, in iframe)  │
└──────────────────────────────┬───────────────────────────────────┘
                               │ HTTPS
┌──────────────────────────────▼───────────────────────────────────┐
│                           API LAYER                               │
│  POST /documents                 GET /documents/:id               │
│  POST /documents/:id/send        POST /sign/:token (JWT)         │
│  POST /documents/:id/revoke      GET  /documents/:id/audit-trail │
└───────┬──────────────┬───────────────────────┬───────────────────┘
        │              │                       │
┌───────▼──────┐ ┌─────▼──────────┐  ┌────────▼──────────────┐
│  Document    │ │  Signing Link  │  │  Audit Trail Service  │
│  Service     │ │  Service       │  │  (read-only,          │
│  (FSM +      │ │  (JWT + OTP)   │  │   chain-verified)     │
│  callbacks)  │ └────────────────┘  └───────────────────────┘
└───────┬──────┘
        │ emits after transaction commits
┌───────▼───────────────────────────────────────────────────────┐
│                        EVENT BUS                               │
│  document.sent    → [triggerEmail, startReminderTracking]      │
│  document.signed  → [verifyHash, triggerEmail, saveToStorage]  │
│  document.expired → [triggerEmail, invalidateLinks]            │
└───┬───────────────────┬───────────────────────────────────────┘
    │                   │
┌───▼────────────┐ ┌───▼──────────────┐ ┌──────────────────────┐
│ Email Service  │ │ Storage Layer    │ │ Document Event Log   │
│ (Resend)       │ │ (Drive/Dropbox/  │ │ (append-only,        │
│                │ │  Box/internal)   │ │  hashed chain,       │
└────────────────┘ └──────────────────┘ │  restricted writes)  │
                                        └──────────────────────┘

┌───────────────────────────────────────────────────────────────┐
│                       ASYNC LAYERS                             │
│                                                               │
│  GENERATION QUEUE (BullMQ)           CRON LAYER               │
│  ─────────────────────────          ────────────────────────  │
│  render PDF/DOCX from fields         enforceExpiry  (hourly)  │
│  store to owner storage              sendReminders  (6h)      │
│  emit document.generated             sendWarnings   (daily)   │
│                                      advanceQueue   (15min)   │
│                                      archiveDrafts  (daily)   │
│                                                               │
│  Both layers call DocumentService.transition() and            │
│  sendDocumentEmail() — same interfaces as the API layer.      │
└───────────────────────────────────────────────────────────────┘
```

### 12.2 Document Lifecycle with Actors

```
OWNER                     SYSTEM                      SIGNER
  │                          │                            │
  │─── fills form ──────────►│                            │
  │                   creates document (draft)            │
  │                          │                            │
  │─── clicks Send ─────────►│                            │
  │                   enqueues generation job             │
  │                          │ (async)                    │
  │                   renders PDF                         │
  │                   stores to Drive                     │
  │                   status → sent ──────── email ──────►│
  │                   appends audit event                 │
  │                          │                            │
  │◄── "document ready" ─────│                            │
  │                          │                            │
  │                          │◄──── opens signing link ───│
  │                   status → viewed                     │
  │                   appends audit event                 │
  │◄── "document viewed" ────│                            │
  │                          │                            │
  │                          │◄──── submits OTP ──────────│
  │                   verifies OTP                        │
  │                   computes signature hash             │
  │                   status → signed ──── email ─────────►│
  │◄── "document signed" ────│                            │
  │                   appends audit event                 │
  │                   saves signed PDF                    │
  │                          │                            │


                CRON (time-based, independent of user actions)
                          │
            every hour → check expiresAt
            if passed and not signed:
                status → expired
                appends audit event
                email ──────────────────────────────────► OWNER
                email ──────────────────────────────────► SIGNER
```

### 12.3 Sequential Multi-Party Signing

```
Document: NDA between Party A (order:1) and Party B (order:2)

  SYSTEM                  PARTY A (order:1)         PARTY B (order:2)
    │                           │                           │
    │── notifies ───────────────►│                           │
    │                      signs │                           │
    │◄── signed ────────────────│                           │
    │                                                        │
    │  [cron: advanceSigningQueue]                           │
    │  all order:1 signers have signed                       │
    │── notifies ────────────────────────────────────────────►│
    │                                                   signs │
    │◄── signed ─────────────────────────────────────────────│
    │                                                        │
    │  [all signers complete]                                │
    │  document.signed event emitted                         │
    │  owner notified                                        │
    │  final PDF saved to storage                            │
```

---

## 13. Implementation Roadmap

The roadmap is organized by risk, not by feature. The decisions that are most expensive to change later are scheduled first.

### Phase 1 — Foundation (Weeks 1–4)

_Goal: establish the architecture that everything else builds on. No features ship in this phase. The output is a correct foundation, not a working product._

- [ ] Implement full `Document` schema with lifecycle timestamp fields
- [ ] Create `document_signers` table with sequential signing order
- [ ] Build `document_events` table with integrity hash chain and database-level write restrictions
- [ ] Implement `DocumentService.transition()` with row-level locking and atomic writes
- [ ] Define typed `DocumentEventType` union — no magic strings anywhere
- [ ] Set up in-process event bus with swappable interface
- [ ] Build email registry with provider abstraction and send deduplication
- [ ] Set up cron runner with structured logging, metrics, and alerting
- [ ] Define all indexes from the initial migration
- [ ] Write test suite for: valid transitions, invalid transition errors, audit chain integrity

### Phase 2 — Core Document Workflow (Weeks 5–8)

_Goal: a document can be generated, sent, and signed end to end._

- [ ] Build document generation queue (BullMQ worker pool)
- [ ] Implement signing link service (JWT with jti blocklist)
- [ ] Build OTP verification service (Redis TTL + rate limiting)
- [ ] Implement all six email templates with preview routes
- [ ] Build `enforceDocumentExpiry` and `sendSignatureReminders` cron jobs
- [ ] Expose `/audit-trail` endpoint with chain verification
- [ ] Implement `saveDocumentToStorage` callback for at least one provider (Drive)
- [ ] Write integration tests covering the full document lifecycle

### Phase 3 — Multi-Party Signing and Storage (Weeks 9–12)

_Goal: sequential multi-party signing works; all three storage providers are connected._

- [ ] Implement `advanceSigningQueue` cron job
- [ ] Per-signer reminder tracking and notification emails
- [ ] Google Drive, Dropbox, and Box storage provider implementations
- [ ] `sendExpiryWarnings` cron job
- [ ] Rate limiting at the workspace level
- [ ] Document revocation flow with signer notification

### Phase 4 — Scalability and Compliance (Weeks 13–16)

_Goal: the system handles growth without architectural changes._

- [ ] PostgreSQL read replica for dashboard and analytics queries
- [ ] Cursor-based pagination in all cron jobs (replace any findMany without cursor)
- [ ] Data retention job: draft archival and GDPR IP anonymization
- [ ] Workspace and team model with row-level access enforcement
- [ ] Webhook service: subscription model over the event log
- [ ] Load testing: validate generation queue, cron jobs, and audit trail under production-scale data volumes

---

## 14. What Not to Build

These are deliberate constraints. Each represents a common engineering impulse that would add complexity without proportionate value at Doclast's current stage.

**Do not build a custom state machine library.**  
The valid transitions map in §5.1 is all you need. A state machine library adds an indirection layer that makes the transitions harder to read and debug, without adding safety that TypeScript's type system doesn't already provide.

**Do not put document generation in the API request path.**  
The short-term simplicity is real. The long-term cost — latency degradation, timeout risk, inability to scale rendering capacity independently — is larger. The queue adds one day of implementation complexity and eliminates an entire category of production incidents.

**Do not use `status` as the only lifecycle signal.**  
Dozens of features — reminder cooldown, expiry enforcement, analytics dashboards, compliance reports — require time-based reasoning. Without the timestamp fields they require reconstructing history from the event log on every request, which is expensive and fragile.

**Do not merge the audit log and the application log.**  
Application logs are operational data for engineers. The audit event log is legal evidence for contracts. They have different access controls, different retention policies, different query patterns, and in some jurisdictions different compliance obligations. They are different tables, managed by different service accounts.

**Do not expose raw database rows through the audit trail API.**  
The internal schema of `document_events` is an implementation detail. The API response is a product — it should be human-readable, chain-verified, and stable across schema changes. Return a transformed response, not a direct ORM result.

**Do not add signing integrations (DocuSign, HelloSign) in Phase 1.**  
OTP-based signing is sufficient for the early market. External signing integrations add OAuth complexity, webhook receivers, and third-party dependency risk before you have validated whether customers need it. Design the `signatureMethod` field on the schema to accommodate it later — build the integration only when customers ask for it explicitly.

---

## 15. Appendix

### Appendix A — SidebarVulcan Files Reviewed

| File                            | Key Finding                                                                                                         |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| `lib/modules/posts/schema.js`   | Lifecycle timestamps as first-class schema fields; `status` and timestamps are complementary, not redundant         |
| `lib/server/posts/callbacks.js` | Composable named callbacks per mutation; the anti-pattern to avoid is implicit context branching with magic strings |
| `lib/server/emails/emails.js`   | Declarative self-contained email objects — the most directly portable pattern in the codebase                       |
| `lib/server/cron.js`            | Isolated scheduled job registry with clean separation from the mutation layer                                       |
| `README.md`                     | System overview: Vulcan.js on Meteor, GraphQL data layer, MongoDB                                                   |

### Appendix B — Technology Recommendations

The architecture described in this report is framework-agnostic. These are recommendations based on production suitability at Doclast's stage.

| Layer          | Recommendation            | Rationale                                                                          |
| -------------- | ------------------------- | ---------------------------------------------------------------------------------- |
| Language       | TypeScript (strict mode)  | Typed event unions catch whole classes of lifecycle bugs at compile time           |
| Database       | PostgreSQL                | ACID transactions, row-level locking, partial indexes, strong ecosystem            |
| ORM            | Prisma or Drizzle         | Type-safe queries, migration tooling, good TypeScript integration                  |
| Job queue      | BullMQ on Redis           | Production-grade, observable, configurable retry and priority                      |
| Email provider | Resend                    | Modern REST API, template support, good deliverability                             |
| Cron scheduler | Inngest or node-cron      | Inngest provides out-of-the-box observability; node-cron if you prefer self-hosted |
| Signing links  | JWT (`jsonwebtoken`)      | Self-verifying tokens require no database lookup; built-in expiry                  |
| OTP storage    | Redis with TTL            | Fast, automatic expiry, no DB polling                                              |
| Hashing        | Node.js `crypto` built-in | No external dependency; SHA-256 is sufficient for all use cases                    |
| Logging        | Pino (structured JSON)    | Machine-readable, low overhead, works well with all log aggregation platforms      |

### Appendix C — Glossary

| Term                    | Definition in This Document                                                                              |
| ----------------------- | -------------------------------------------------------------------------------------------------------- |
| **Terminal state**      | A status (`signed`, `expired`, `revoked`) from which no further transitions are permitted                |
| **Lifecycle timestamp** | A nullable `Date` field on the document record that captures when a specific status was first reached    |
| **Transition function** | `DocumentService.transition()` — the single authorized path for changing document status                 |
| **Integrity hash**      | A SHA-256 hash chained from the previous event, stored on each audit event row                           |
| **Signing order**       | An integer on `document_signers` that controls which signers are notified first in multi-party workflows |
| **CRON_ACTOR**          | A constant `EventActor` object used when a cron job is the actor in a transition call                    |
| **Dead letter queue**   | A secondary queue where failed jobs are deposited for investigation and manual retry                     |
| **Cursor pagination**   | A query technique that fetches batches using the last record's ID as a cursor, avoiding full table scans |
