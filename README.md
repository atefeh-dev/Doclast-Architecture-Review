bash

# Doclast — Backend Architecture Report

## Document Lifecycle Management: Design Recommendations for CTO Review

**Version:** 2.0  
**Date:** June 2026  
**Prepared by:** Engineering Review  
**Based on:** SidebarVulcan analysis + Doclast product requirements

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [What We Learned from Sidebar](#2-what-we-learned-from-sidebar)
3. [Database Schema Recommendations](#3-database-schema-recommendations)
4. [Lifecycle State Design](#4-lifecycle-state-design)
5. [Event-Driven Architecture](#5-event-driven-architecture)
6. [Email Automation System](#6-email-automation-system)
7. [Workflow Automation and Scheduled Jobs](#7-workflow-automation-and-scheduled-jobs)
8. [Audit Trail and Compliance](#8-audit-trail-and-compliance)
9. [Missing Components and Services](#9-missing-components-and-services)
10. [Scalability Considerations](#10-scalability-considerations)
11. [Architecture Diagrams](#11-architecture-diagrams)
12. [Implementation Roadmap](#12-implementation-roadmap)
13. [What Not to Build](#13-what-not-to-build)

---

## 1. Executive Summary

Doclast's core product — generate → send → review → approve → sign — is a document lifecycle system. The database schema, event architecture, and automation layer must be designed around that lifecycle from the start, not retrofitted once the product grows.

SidebarVulcan demonstrates what this looks like at production scale for a simpler object (a blog post). Its patterns translate almost directly to Doclast's more complex document workflow, with three additions Sidebar doesn't need: cryptographic signature verification, a tamper-evident audit trail for legal compliance, and multi-party signing flows.

The five decisions that will most affect Doclast's architecture at scale:

1. **Store lifecycle timestamps as first-class schema fields**, not derived from an event log or computed from status history.
2. **Route all state transitions through a single `DocumentService.transition()` function** — no exceptions, no bypasses.
3. **Use an append-only event log as the authoritative audit trail**, separate from the mutable document record.
4. **Keep document generation (PDF/DOCX rendering) behind a queue**, not in the request path.
5. **Design the signature model to be verifiable without Doclast's servers** — users must be able to validate a signed document independently.

---

## 2. What We Learned from Sidebar

### 2.1 The Central Insight

Sidebar organizes every layer of its system around a single question: _where is this object in its lifecycle?_ The schema, callbacks, emails, and cron jobs are all answers to sub-questions of that question.

For Doclast this means: every engineering decision should start with the document lifecycle, not with the database table or the API endpoint.

### 2.2 Directly Transferable Patterns

**Lifecycle timestamps on the schema**  
Sidebar stores `scheduledAt`, `postedAt`, `paidAt` alongside `status`. This is not redundancy — `status` tells you the current state, timestamps tell you the history. Both are needed. Sidebar's approach eliminates the need to reconstruct history from an event log for common queries, while still allowing an event log for compliance.

**Declarative email registry**  
Each email in Sidebar is a self-contained object that declares its own data requirements. The caller provides only the document ID and the email key. This decouples trigger logic from rendering logic and makes the full email surface of the system discoverable from a single file.

**Dedicated cron layer**  
Sidebar's scheduler runs independently of the mutation layer. Scheduled work (reminders, expiry) has its own code path, its own error handling, and its own observability. This is the correct separation: time-based triggers and user-triggered mutations should never share implementation code.

**Named callbacks on business events**  
Sidebar registers callbacks per mutation type. Doclast should register them per business event. The event name carries intent (`document.signed`) that a CRUD operation name (`documents.update.after`) does not.

### 2.3 Patterns to Improve On

**Magic strings in callbacks**  
Sidebar's callbacks check `context.event === 'some-string'` internally. This creates silent coupling that TypeScript cannot catch. Doclast should use a typed event union from day one.

**No multi-party model**  
Sidebar emails go to one recipient. Doclast needs a signing model that supports multiple signers in a defined sequence, each with their own status, timestamp, and reminder chain.

**No document storage abstraction**  
Sidebar stores content in MongoDB. Doclast's documents are generated files — they need a storage model that supports multiple backends (Drive, Dropbox, Box) without the lifecycle logic knowing which backend is active.

---

## 3. Database Schema Recommendations

### 3.1 The Document Table

The document record is the mutable source of truth for current state. It is not the audit trail.

```ts
// documents table
interface Document {
  // ── Identity ────────────────────────────────────────────────────────────
  id: string; // UUID v4
  ownerId: string; // FK → users.id
  workspaceId: string; // FK → workspaces.id (for team plans)
  title: string;
  templateId: TemplateId; // 'NDA' | 'collaboration' | 'MOU' | custom

  // ── Current lifecycle state ──────────────────────────────────────────────
  status: DocumentStatus;

  // ── Lifecycle timestamps ─────────────────────────────────────────────────
  // Rule: always updated atomically with status.
  // Rule: never overwritten — use separate fields for re-sends.
  createdAt: Date;
  sentAt: Date | null;
  firstViewedAt: Date | null; // when the signer first opened the link
  reviewingAt: Date | null;
  approvedAt: Date | null;
  signedAt: Date | null;
  expiredAt: Date | null;
  revokedAt: Date | null;
  resentAt: Date | null; // if owner re-sent — does NOT overwrite sentAt

  // ── Expiry configuration ─────────────────────────────────────────────────
  expiresAt: Date | null; // deadline set by owner at creation
  reminderSentAt: Date | null; // tracks last reminder to avoid duplicates

  // ── Document fields (filled form data) ──────────────────────────────────
  fields: Record<string, string>;

  // ── Storage ──────────────────────────────────────────────────────────────
  // The generated file is stored externally. This record tracks where.
  storageProvider: StorageProvider | null; // 'drive' | 'dropbox' | 'box' | 'internal'
  storagePath: string | null;
  storageFileId: string | null;

  // ── Signature ────────────────────────────────────────────────────────────
  signatureHash: string | null; // SHA-256 of document content + signerEmail + signedAt
  signatureMethod: "otp_email" | "docusign" | null;

  // ── Metadata ─────────────────────────────────────────────────────────────
  updatedAt: Date;
  schemaVersion: number; // for schema migration safety
}

type DocumentStatus =
  | "draft" // fields being filled, not yet sent
  | "sent" // email delivered to signer
  | "viewed" // signer opened the link
  | "reviewing" // signer is actively reviewing
  | "approved" // signer accepted, pending signature
  | "signed" // signature verified and stored
  | "expired" // expiresAt passed before signing
  | "revoked"; // owner cancelled before signing
```

### 3.2 The Signers Table

The current schema treats a document as having one signer. This breaks the moment a customer asks for multi-party signing (NDA between three parties, employment contract requiring HR + candidate + legal). Design for it now.

```ts
// document_signers table
interface DocumentSigner {
  id: string;
  documentId: string; // FK → documents.id
  email: string;
  name: string | null;
  role: string | null; // 'party_a', 'party_b', 'witness', etc.
  order: number; // signing sequence (1 = first, 2 = second, ...)
  status: SignerStatus;

  // Per-signer lifecycle timestamps
  notifiedAt: Date | null; // when their signing email was sent
  viewedAt: Date | null;
  signedAt: Date | null;
  reminderSentAt: Date | null;

  // Per-signer signature data
  signatureHash: string | null;
  ipAddress: string | null; // for legal audit trail
  userAgent: string | null;
}

type SignerStatus = "pending" | "notified" | "viewed" | "signed" | "declined";
```

**Signing sequence rule:** A signer with `order: 2` should not be notified until all signers with `order: 1` have status `signed`. The cron job handles this check — not the document mutation handler.

### 3.3 The Document Events Table (Append-Only Audit Log)

This is a separate table from the document record. It is never updated and never deleted. It is the legal source of truth.

```ts
// document_events table
interface DocumentEvent {
  id: string;
  documentId: string; // FK → documents.id
  event: DocumentEventType;
  actorType: "user" | "signer" | "system" | "cron";
  actorId: string | null; // userId, signerEmail, 'system', or cronJobName
  occurredAt: Date;
  metadata: Record<string, unknown>; // event-specific data (IP, user agent, etc.)

  // Integrity
  // Each event includes a hash of (previousEventId + eventData).
  // This creates a tamper-evident chain — any modification of a past event
  // breaks the hash of every subsequent event.
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
  | "document.storage_saved"
  | "signer.added"
  | "signer.notified"
  | "signer.viewed"
  | "signer.signed"
  | "signer.declined";
```

### 3.4 Indexing Strategy

```sql
-- Core queries every request will hit
CREATE INDEX idx_documents_owner    ON documents(owner_id, status, created_at DESC);
CREATE INDEX idx_documents_status   ON documents(status, expires_at) WHERE status IN ('sent', 'viewing', 'reviewing');

-- Cron job queries (time-based scans)
CREATE INDEX idx_documents_expiry   ON documents(expires_at) WHERE status NOT IN ('signed', 'expired', 'revoked');
CREATE INDEX idx_documents_reminder ON documents(sent_at, reminder_sent_at) WHERE status = 'sent';

-- Audit log queries
CREATE INDEX idx_events_document    ON document_events(document_id, occurred_at DESC);
CREATE INDEX idx_events_integrity   ON document_events(previous_event_id);

-- Signer queries
CREATE INDEX idx_signers_document   ON document_signers(document_id, "order");
CREATE INDEX idx_signers_email      ON document_signers(email, status);
```

---

## 4. Lifecycle State Design

### 4.1 Valid Transitions

Not every state can transition to every other state. Encoding valid transitions explicitly prevents bugs that are otherwise invisible until production.

```ts
const VALID_TRANSITIONS: Record<DocumentStatus, DocumentStatus[]> = {
  draft: ["sent", "revoked"],
  sent: ["viewed", "expired", "revoked", "sent"], // 'sent' allows resend
  viewed: ["reviewing", "expired", "revoked"],
  reviewing: ["approved", "expired", "revoked"],
  approved: ["signed", "expired", "revoked"],
  signed: [], // terminal — no transitions out
  expired: [], // terminal
  revoked: [], // terminal
};

function assertValidTransition(from: DocumentStatus, to: DocumentStatus): void {
  if (!VALID_TRANSITIONS[from].includes(to)) {
    throw new InvalidTransitionError(
      `Cannot transition document from '${from}' to '${to}'`,
    );
  }
}
```

### 4.2 The Single Transition Function

All state changes — from API handlers, cron jobs, webhooks, admin tools — must pass through this function. There is no valid reason to bypass it.

```ts
// document-service.ts
async function transition(
  documentId: string,
  event: DocumentEventType,
  actor: EventActor,
  metadata: Record<string, unknown> = {},
): Promise<Document> {
  return await db.transaction(async (tx) => {
    // 1. Load current document with a row-level lock
    const doc = await tx.documents.findByIdForUpdate(documentId);

    // 2. Derive the target status from the event type
    const targetStatus = eventToStatus(event);

    // 3. Validate the transition before touching anything
    assertValidTransition(doc.status, targetStatus);

    // 4. Prepare the update atomically: status + timestamp + updatedAt
    const timestamp = timestampFieldForStatus(targetStatus);
    const now = new Date();

    const updatedDoc = await tx.documents.update(documentId, {
      status: targetStatus,
      [timestamp]: now,
      updatedAt: now,
    });

    // 5. Append an immutable event to the audit log
    await appendEvent(tx, {
      documentId,
      event,
      actor,
      occurredAt: now,
      metadata,
    });

    // 6. Run registered callbacks (outside the DB transaction — see note below)
    await runCallbacks(event, updatedDoc, actor);

    return updatedDoc;
  });
}
```

> **Note on callbacks and transactions:** Callbacks that send emails or call external APIs must run _after_ the transaction commits, not inside it. A transaction that holds a DB lock while waiting for an external HTTP call is a deadlock waiting to happen. The pattern: commit the transaction, then run callbacks. If a callback fails, log and retry — do not roll back the document state.

### 4.3 Transition Map (Text Diagram)

```
                        ┌─────────────────────────────┐
                        │          DOCUMENT            │
                        └─────────────────────────────┘
                                      │
                              [owner fills form]
                                      │
                                  ┌───▼───┐
                            ┌────►│ draft │◄─────────────────────┐
                            │     └───┬───┘                      │
                       [resend]   [owner sends]             [re-draft? future]
                            │         │
                        ┌───┴───┐   ┌─▼────┐
                        │       │   │ sent │──────────────────────────────┐
                        │       │   └──┬───┘                              │
                        │       │ [signer opens link]                     │
                        │       │      │                                  │
                        │       │  ┌───▼────┐                             │
                        │       │  │ viewed │                             │
                        │       │  └───┬────┘                             │
                        │       │ [starts reading]                        │
                        │       │      │                              [expires_at
                        │       │ ┌────▼──────┐                       reached]
                        │       │ │ reviewing │                           │
                        │       │ └────┬──────┘                          │
                        │       │ [accepts]                               │
                        │       │      │                                  │
                        │       │ ┌────▼────┐                             │
                        │       │ │approved │                             │
                        │       │ └────┬────┘                             │
                        │       │ [signs]                                 │
                        │       │      │                                  │
                        │       │ ┌────▼────┐                          ┌──▼──────┐
                        │       │ │ signed  │ ← terminal               │ expired │ ← terminal
                        │       │ └─────────┘                          └─────────┘
                        │       │
                    ┌───┴───────┴──┐
                    │   revoked    │ ← terminal (owner action from any non-terminal state)
                    └──────────────┘
```

---

## 5. Event-Driven Architecture

### 5.1 The Event Bus

Doclast needs a lightweight internal event bus. For early stage, a simple in-process emitter is sufficient. It must be replaceable with a proper message queue (BullMQ, Redis Streams) without changing the callback signatures.

```ts
// events/bus.ts
type EventHandler<T = unknown> = (payload: T, meta: EventMeta) => Promise<void>;

interface EventBus {
  emit(event: DocumentEventType, payload: DocumentEventPayload): Promise<void>;
  on(event: DocumentEventType, handler: EventHandler): void;
  off(event: DocumentEventType, handler: EventHandler): void;
}

// In development / early production: synchronous in-process emitter
// In production at scale: swap the implementation to BullMQ without
// changing any call sites — the interface stays identical.
```

### 5.2 Callback Registration

```ts
// callbacks/index.ts
// This file is the complete registry of all business logic in the system.
// A new engineer can read this file and understand every side effect
// that any document event triggers. Nothing is hidden.

bus.on("document.sent", triggerDocumentSentEmail);
bus.on("document.sent", startSignerReminderTracking);
bus.on("document.viewed", stampFirstViewedAt);
bus.on("document.signed", verifyAndStoreSignatureHash);
bus.on("document.signed", triggerDocumentSignedEmail);
bus.on("document.signed", saveToConnectedStorage);
bus.on("document.expired", triggerDocumentExpiredEmail);
bus.on("document.expired", blockSigningLink);
bus.on("document.revoked", triggerDocumentRevokedEmail);
bus.on("document.revoked", blockSigningLink);
```

### 5.3 Callback Design Rules

Each callback must follow four rules without exception:

**Rule 1: Single responsibility.** `triggerDocumentSentEmail` triggers an email and does nothing else. It does not stamp a timestamp. It does not validate the document.

**Rule 2: Idempotent.** If the same event is emitted twice (duplicate delivery from a queue), the callback must produce the same result without side effects. For emails: check if the email was already sent before sending. For storage: check if the file already exists before saving.

**Rule 3: No direct DB writes.** Callbacks that need to update document state must call `DocumentService.transition()` — never write to the documents table directly. This prevents bypassing the transition guard.

**Rule 4: Fail in isolation.** A failing `saveToConnectedStorage` callback must not prevent `triggerDocumentSignedEmail` from running. Log the failure, retry with backoff, alert if persistent. Never let one callback's failure cascade.

---

## 6. Email Automation System

### 6.1 The Email Registry Pattern

```ts
// emails/registry.ts

interface EmailDefinition {
  template: string;
  subject: (data: EmailData) => string;
  to: (data: EmailData) => string | string[];
  cc?: (data: EmailData) => string[];
  getData: (documentId: string) => Promise<EmailData>;
  testFixture?: EmailData; // for preview routes and tests
}

export const emailRegistry: Record<string, EmailDefinition> = {
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
    cc: (d) => d.signers.map((s) => s.email), // all signers get a copy
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
    subject: (d) => `"${d.document.title}" has expired unsigned`,
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
};

// The entire public API for sending any Doclast email:
export async function sendDocumentEmail(
  key: keyof typeof emailRegistry,
  documentId: string,
): Promise<void> {
  const definition = emailRegistry[key];
  const data = await definition.getData(documentId);
  const to = definition.to(data);
  const subject = definition.subject(data);

  await emailProvider.send({
    template: definition.template,
    to,
    cc: definition.cc?.(data),
    subject,
    data,
  });
}
```

### 6.2 Email Provider Abstraction

The email registry must not depend on a specific email provider. Wrap the provider behind an interface so switching from SendGrid to Postmark to Resend requires changing one file.

```ts
// emails/provider.ts
interface EmailProvider {
  send(options: SendOptions): Promise<void>;
}

// Production
export const emailProvider: EmailProvider = new ResendProvider(
  process.env.RESEND_API_KEY,
);

// Tests
export const emailProvider: EmailProvider = new InMemoryEmailProvider();
```

### 6.3 Deduplication

Before sending any email, check if the same email type was already sent for this document within a deduplication window. This prevents duplicate sends from retry logic or bug-induced double emissions.

```ts
async function sendDocumentEmail(
  key: string,
  documentId: string,
): Promise<void> {
  const dedupKey = `email:${key}:${documentId}`;

  // Check Redis or DB for recent sends of this email type for this document
  const alreadySent = await dedup.check(dedupKey, { window: "24h" });
  if (alreadySent) {
    logger.warn(`Duplicate email suppressed: ${dedupKey}`);
    return;
  }

  await dedup.mark(dedupKey);
  // ... rest of send logic
}
```

---

## 7. Workflow Automation and Scheduled Jobs

### 7.1 Job Registry

```ts
// cron/jobs.ts

export const cronJobs = [
  {
    name: "enforceDocumentExpiry",
    schedule: "0 * * * *", // every hour
    handler: async () => {
      // Find all documents where expiresAt has passed and status is not terminal
      const overdue = await db.documents.findMany({
        where: {
          expiresAt: { lte: new Date() },
          status: { notIn: ["signed", "expired", "revoked"] },
        },
      });
      for (const doc of overdue) {
        await DocumentService.transition(doc.id, "document.expired", {
          actorType: "cron",
          actorId: "enforceDocumentExpiry",
        });
      }
      logger.info(`Expired ${overdue.length} documents`);
    },
  },

  {
    name: "sendSignatureReminders",
    schedule: "0 */6 * * *", // every 6 hours
    handler: async () => {
      const REMINDER_THRESHOLD_HOURS = 48;
      const REMINDER_COOLDOWN_HOURS = 24; // don't re-remind within 24h

      const stale = await db.documents.findMany({
        where: {
          status: { in: ["sent", "viewed"] },
          sentAt: { lte: subHours(new Date(), REMINDER_THRESHOLD_HOURS) },
          reminderSentAt: {
            OR: [
              { equals: null },
              { lte: subHours(new Date(), REMINDER_COOLDOWN_HOURS) },
            ],
          },
          expiresAt: { OR: [{ equals: null }, { gte: new Date() }] }, // not expired
        },
      });

      for (const doc of stale) {
        await sendDocumentEmail("signatureReminder", doc.id);
        await db.documents.update(doc.id, { reminderSentAt: new Date() });
      }
    },
  },

  {
    name: "sendExpiryWarnings",
    schedule: "0 8 * * *", // daily at 8am
    handler: async () => {
      const WARNING_WINDOW_HOURS = 48;

      // Find documents expiring in the next 48 hours that haven't been warned
      const approaching = await db.documents.findMany({
        where: {
          expiresAt: {
            gte: new Date(),
            lte: addHours(new Date(), WARNING_WINDOW_HOURS),
          },
          status: { notIn: ["signed", "expired", "revoked"] },
          expiryWarnedAt: { equals: null }, // add this field to schema
        },
      });

      for (const doc of approaching) {
        await sendDocumentEmail("expiryWarning", doc.id);
        await db.documents.update(doc.id, { expiryWarnedAt: new Date() });
      }
    },
  },

  {
    name: "advanceSigningQueue",
    schedule: "*/15 * * * *", // every 15 minutes
    handler: async () => {
      // Find documents where all signers of order N have signed,
      // but signers of order N+1 have not yet been notified.
      const ready = await SignerService.findReadyForNextSequence();
      for (const { documentId, nextSigners } of ready) {
        await SignerService.notifySigners(documentId, nextSigners);
      }
    },
  },

  {
    name: "archiveAbandonedDrafts",
    schedule: "0 3 * * *", // daily at 3am
    handler: async () => {
      await db.documents.updateMany({
        where: {
          status: "draft",
          updatedAt: { lte: subDays(new Date(), 30) },
        },
        data: {
          status: "revoked",
          revokedAt: new Date(),
        },
      });
    },
  },
];
```

### 7.2 Job Observability

Every cron job must emit structured logs and metrics. At minimum:

```ts
// cron/runner.ts
async function runJob(job: CronJob): Promise<void> {
  const startTime = Date.now();
  logger.info({ job: job.name, event: "cron.job.started" });

  try {
    await job.handler();
    metrics.increment("cron.job.success", { job: job.name });
    metrics.timing("cron.job.duration", Date.now() - startTime, {
      job: job.name,
    });
    logger.info({
      job: job.name,
      event: "cron.job.completed",
      durationMs: Date.now() - startTime,
    });
  } catch (err) {
    metrics.increment("cron.job.failure", { job: job.name });
    logger.error({
      job: job.name,
      event: "cron.job.failed",
      error: err.message,
    });
    alerting.notify(`Cron job failed: ${job.name}`, err);
  }
}
```

---

## 8. Audit Trail and Compliance

### 8.1 Why This Matters for Doclast

A signed contract carries legal weight. In any dispute, Doclast must be able to produce evidence that:

- The correct document was sent to the correct party
- The signer opened the document and had access to it for sufficient time
- The signature was collected from the expected person (verified email)
- The document content was not altered after signing
- The audit log itself has not been tampered with

Without an intentional audit design, Doclast cannot provide this evidence. The architecture must be designed for it from day one.

### 8.2 The Append-Only Event Chain

As specified in Section 3.3, every state change appends a record to `document_events`. The integrity chain makes retrospective tampering detectable:

```ts
async function appendEvent(
  tx: Transaction,
  params: {
    documentId: string;
    event: DocumentEventType;
    actor: EventActor;
    occurredAt: Date;
    metadata: Record<string, unknown>;
  },
): Promise<DocumentEvent> {
  // Get the previous event to chain integrity hashes
  const previousEvent = await tx.documentEvents.findLatest(params.documentId);

  const eventData = {
    ...params,
    previousEventId: previousEvent?.id ?? null,
  };

  // Hash = SHA-256 of (previousHash + eventType + actorId + occurredAt + metadata)
  const integrityHash = computeIntegrityHash(
    previousEvent?.integrityHash,
    eventData,
  );

  return tx.documentEvents.create({ ...eventData, integrityHash });
}
```

### 8.3 The Signature Hash

When a document is signed, store a hash that allows independent verification:

```ts
// Input to hash: document content + signer email + timestamp + document ID
// Output: a hex string stored on the document record

function computeSignatureHash(params: {
  documentContent: Buffer; // the exact bytes of the PDF that was signed
  signerEmail: string;
  signedAt: Date;
  documentId: string;
}): string {
  return createHash("sha256")
    .update(params.documentContent)
    .update(params.signerEmail)
    .update(params.signedAt.toISOString())
    .update(params.documentId)
    .digest("hex");
}
```

The signer receives this hash in their confirmation email. Any party can recompute the hash from the original PDF and verify it matches — without trusting Doclast's servers.

### 8.4 Audit Trail API

Expose a read-only endpoint that returns the full, verified event chain for a document. This is what legal proceedings require.

```ts
// GET /api/documents/:id/audit-trail

interface AuditTrailResponse {
  documentId: string;
  verified: boolean; // true if all integrity hashes in the chain are valid
  events: Array<{
    id: string;
    event: DocumentEventType;
    actorType: string;
    actorId: string | null;
    occurredAt: Date;
    metadata: Record<string, unknown>;
    integrityHash: string;
    chainValid: boolean; // true if this event's hash is valid against the previous
  }>;
}
```

### 8.5 Data Retention Policy

Define this before launch — retrofitting it is expensive:

| Data type                  | Retention                    | Reason                                      |
| -------------------------- | ---------------------------- | ------------------------------------------- |
| Signed documents           | 7 years                      | Standard contract law in most jurisdictions |
| Audit event log            | 7 years                      | Same — legal evidence                       |
| Draft documents (unsigned) | 90 days after abandonment    | No legal obligation, storage cost           |
| Email delivery logs        | 1 year                       | Dispute resolution                          |
| OTP codes                  | Delete immediately after use | Security                                    |
| IP addresses in audit log  | Anonymize after 2 years      | GDPR                                        |

---

## 9. Missing Components and Services

These are services that Doclast's architecture will require but that are not addressed in the current design.

### 9.1 Document Generation Service

Document generation (filling a template with form data → rendering a PDF) must be behind a queue, not in the API request path. PDF rendering is CPU-intensive and can take 500ms–3s. Blocking an HTTP request for that duration degrades the entire API.

```
Owner submits form
      │
      ▼
API creates Document record (status: 'draft')
API enqueues generation job
API returns 202 Accepted immediately
      │
      ▼ (async, via BullMQ or similar)
Generation worker picks up job
Renders PDF/DOCX from template + fields
Stores file to owner's connected storage
Emits 'document.generated' event
      │
      ▼
Document status → 'ready_to_send'
Owner gets notified (push/email) that document is ready
```

This pattern also enables retry logic if rendering fails, progress indicators in the UI, and decoupled scaling of the rendering workers.

### 9.2 Signing Link Service

The link sent to signers must be:

- Unique per document + signer
- Time-limited (expires when the document does)
- Single-use after signing is complete
- Verifiable without a database lookup (JWT with embedded claims)

```ts
// signing-links/service.ts

function generateSigningLink(params: {
  documentId: string;
  signerEmail: string;
  expiresAt: Date | null;
}): string {
  const token = jwt.sign(
    {
      sub: params.signerEmail,
      doc: params.documentId,
      type: "signing",
    },
    process.env.SIGNING_LINK_SECRET,
    {
      expiresIn: params.expiresAt
        ? Math.floor((params.expiresAt.getTime() - Date.now()) / 1000)
        : "30d",
    },
  );

  return `${process.env.BASE_URL}/sign/${token}`;
}
```

### 9.3 Storage Abstraction Layer

Doclast supports Drive, Dropbox, and Box. The lifecycle logic must not know which provider is active. Define a storage interface and implement it per provider:

```ts
interface StorageProvider {
  save(params: {
    content: Buffer;
    filename: string;
    mimeType: string;
  }): Promise<StoredFile>;
  getDownloadUrl(fileId: string): Promise<string>;
  delete(fileId: string): Promise<void>;
}

// Implementations: GoogleDriveProvider, DropboxProvider, BoxProvider, InternalProvider
// The document record stores: storageProvider, storagePath, storageFileId
// The lifecycle layer calls: StorageFactory.forOwner(owner).save(...)
```

### 9.4 OTP / Verification Service

The current flow uses OTP email for signer verification. This service needs its own module:

- Generate a 6-digit code
- Store the code with a TTL of 10 minutes (Redis, not the documents DB)
- Rate-limit attempts per email address (max 5 attempts before lockout)
- Delete the code immediately after successful use
- Never log the code value itself

### 9.5 Webhook Service (Future)

When Doclast adds integrations, customers will need webhooks. Design the event log to double as a webhook event source from the start — every `document_events` entry is a candidate webhook payload. Adding webhooks later becomes a subscription model on top of an existing event stream, not a new system.

---

## 10. Scalability Considerations

### 10.1 Database

The mutable `documents` table and the append-only `document_events` table have different read/write patterns and should be optimized independently.

`documents` is read-heavy and written on every state transition. It benefits from read replicas for dashboard queries and row-level locking on writes (already covered by the transaction pattern in Section 4.2).

`document_events` is write-heavy and append-only. It should never be updated or deleted. Consider a separate table partition per month or per year once volume warrants it. For compliance, this table should live in a separate schema with restricted write access — only the event-appending service account can insert rows.

### 10.2 The Generation Queue

Document rendering workers should scale horizontally, independent of the API servers. Use a job queue (BullMQ on Redis, or a managed service like Inngest) with:

- Concurrency limits per worker to prevent CPU saturation
- Dead letter queue for failed jobs with alerting
- Job prioritization (Pro plan documents before Free plan)

### 10.3 Cron Jobs at Scale

A single cron runner is sufficient until Doclast has tens of thousands of active documents. At that point, batch jobs that iterate over all documents become slow. The solution is not to rewrite them — it is to ensure the cron queries are index-backed (covered in Section 3.4) and the iteration uses cursor-based pagination:

```ts
// Instead of: findMany({ where: { status: 'sent' } })
// Use cursor pagination to avoid loading 50,000 rows at once:

async function* iterateSentDocuments() {
  let cursor: string | undefined;
  do {
    const batch = await db.documents.findMany({
      where: { status: "sent" },
      take: 100,
      cursor: cursor ? { id: cursor } : undefined,
      orderBy: { id: "asc" },
    });
    yield batch;
    cursor = batch.at(-1)?.id;
  } while (cursor);
}
```

### 10.4 Multi-Tenancy

Doclast will eventually serve teams and organizations, not just individual users. Design the schema for it now:

- `workspaceId` on all documents (present in Section 3.1)
- Row-level access policies enforced at the service layer, not the API layer
- Workspace-level rate limits on document creation, email sends, and signing link generation
- Separate billing records per workspace for addon usage tracking

---

## 11. Architecture Diagrams

### 11.1 System Layers

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                              │
│   Landing page (static HTML)    Contract flow (Vue 3 iframe)    │
└────────────────────────────┬────────────────────────────────────┘
                             │  HTTPS
┌────────────────────────────▼────────────────────────────────────┐
│                         API LAYER                                │
│   POST /documents         GET /documents/:id                     │
│   POST /documents/:id/send   POST /sign/:token                  │
│   GET  /documents/:id/audit-trail                               │
└──────┬──────────────┬──────────────────┬───────────────────────┘
       │              │                  │
┌──────▼──────┐ ┌─────▼──────┐  ┌───────▼────────┐
│  Document   │ │  Signing   │  │  Audit Trail   │
│  Service   │ │  Service   │  │  Service       │
│  (FSM +    │ │  (JWT +    │  │  (read-only,   │
│  callbacks)│ │  OTP)      │  │  verified)     │
└──────┬──────┘ └─────┬──────┘  └────────────────┘
       │              │
┌──────▼──────────────▼─────────────────────────────────────────┐
│                      EVENT BUS                                  │
│  document.sent → [triggerEmail, startReminderTracking]          │
│  document.signed → [verifyHash, triggerEmail, saveToStorage]    │
│  document.expired → [triggerEmail, blockSigningLink]            │
└───────────────┬────────────────────────────────────────────────┘
                │
     ┌──────────┼──────────────┐
     │          │              │
┌────▼────┐ ┌──▼──────┐ ┌─────▼──────────┐
│  Email  │ │ Storage │ │   Document     │
│ Service │ │Abstraction│ │  Event Log    │
│(Resend/ │ │(Drive/  │ │(append-only,  │
│SendGrid)│ │Dropbox/ │ │hashed chain)  │
└─────────┘ │Box)     │ └───────────────┘
            └─────────┘

           ┌─────────────────────────────────────┐
           │            CRON LAYER               │
           │  enforceExpiry (hourly)             │
           │  sendReminders (every 6h)           │
           │  sendExpiryWarnings (daily)         │
           │  advanceSigningQueue (every 15min)  │
           │  archiveAbandonedDrafts (daily)     │
           └──────────────┬──────────────────────┘
                          │
                   calls DocumentService.transition()
                   and sendDocumentEmail()
                   — same interfaces as user actions
```

### 11.2 Document Lifecycle with Actors

```
OWNER                    SYSTEM                    SIGNER
  │                         │                         │
  │── fills form ──────────►│                         │
  │                    creates draft                  │
  │                         │                         │
  │── clicks Send ─────────►│                         │
  │                  generates PDF                    │
  │                  stores to Drive                  │
  │                  status → sent ────────email──────►│
  │                  appends event                    │
  │                         │                         │
  │                         │◄──── opens link ────────│
  │                    status → viewed                │
  │                    appends event                  │
  │                         │                         │
  │◄── viewed notification ─│                         │
  │                         │                         │
  │                         │◄──── enters OTP ────────│
  │                    verifies OTP                   │
  │                    computes hash                  │
  │                    status → signed ───email────────►│
  │◄── signed notification ─│                         │
  │                    appends event                  │
  │                    saves final PDF                │
  │                         │                         │


                    CRON (independent)
                         │
              every hour: check expiresAt
              if passed and not signed:
                status → expired ──email──► OWNER
                         │        ──email──► SIGNER
                    appends event
```

### 11.3 Multi-Signer Sequence

```
  Document created with 3 signers (order: 1, 2, 3)

  ┌──────────────────────────────────────────────────────┐
  │ Signer A (order:1)    Signer B (order:2)    Signer C (order:3) │
  │                                                       │
  │  ◄── notified ──                                      │
  │       signs                                           │
  │                                                       │
  │  [cron: advanceSigningQueue detects A signed]         │
  │                   ◄── notified ──                     │
  │                        signs                          │
  │                                                       │
  │  [cron: advanceSigningQueue detects B signed]         │
  │                                       ◄── notified ── │
  │                                            signs      │
  │                                                       │
  │  [all signers done → document.signed event emitted]   │
  │  [owner notified]                                     │
  └──────────────────────────────────────────────────────┘
```

---

## 12. Implementation Roadmap

### Phase 1 — Foundation (Weeks 1–4)

These decisions are expensive to change later. Do them first.

- [ ] Implement the full `Document` schema with lifecycle timestamps
- [ ] Build `document_events` append-only table with integrity hashing
- [ ] Implement `DocumentService.transition()` as the single state-change entry point
- [ ] Set up the event bus (in-process emitter, swappable interface)
- [ ] Build the email registry with provider abstraction
- [ ] Set up cron runner with structured logging

### Phase 2 — Core Workflow (Weeks 5–8)

- [ ] Implement document generation queue (BullMQ)
- [ ] Build signing link service (JWT-based, time-limited)
- [ ] Implement OTP verification service with rate limiting
- [ ] Build all six email templates
- [ ] Implement `enforceDocumentExpiry` and `sendSignatureReminders` cron jobs
- [ ] Expose `/audit-trail` endpoint with chain verification

### Phase 3 — Storage and Multi-Party (Weeks 9–12)

- [ ] Build storage abstraction layer
- [ ] Implement Google Drive, Dropbox, Box providers
- [ ] Build `document_signers` table and multi-party signing flow
- [ ] Implement `advanceSigningQueue` cron job
- [ ] Add per-signer reminder tracking

### Phase 4 — Scalability and Compliance (Weeks 13–16)

- [ ] Add read replicas for dashboard queries
- [ ] Implement cursor-based pagination in all cron jobs
- [ ] Add data retention job (expiry warnings + GDPR anonymization)
- [ ] Build workspace/team model
- [ ] Add webhook service (subscribes to event log)

---

## 13. What Not to Build

**Do not build a custom state machine library.** The valid transitions map in Section 4.1 is all you need. A state machine library adds an abstraction layer that makes the code harder to read without adding meaningful safety that TypeScript's type system does not already provide.

**Do not put document generation in the API request path.** The temptation is strong — it is simpler to render and respond in one request. It is also a reliability and latency problem at any meaningful scale.

**Do not use document `status` as the only lifecycle signal.** Many time-aware features — reminders, expiry, analytics, compliance reporting — need the timestamp fields. Building without them means retrofitting later, which is expensive in a live production system.

**Do not mix the audit log with the application log.** Application logs (errors, performance traces) are operational data. The audit event log is legal evidence. They have different retention policies, different access controls, and different compliance requirements. They are different tables.

**Do not expose the raw event log to the client.** The audit trail endpoint should return a verified, human-readable representation of events — not raw database rows. The internal event format is an implementation detail.

---

## Appendix A — Files Reviewed from SidebarVulcan

| File                                                 | Key finding                                                                                   |
| ---------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| `packages/sidebar2020/lib/modules/posts/schema.js`   | Lifecycle timestamps as first-class schema fields alongside status                            |
| `packages/sidebar2020/lib/server/posts/callbacks.js` | Named composable callbacks per mutation; magic context strings are the anti-pattern to avoid  |
| `packages/sidebar2020/lib/server/emails/emails.js`   | Self-contained declarative email objects — the most directly portable pattern in the codebase |
| `packages/sidebar2020/lib/server/cron.js`            | Isolated scheduled job registry; clean separation from mutation and callback layers           |
| `README.md`                                          | Architecture overview: Vulcan.js on Meteor, GraphQL, MongoDB                                  |

## Appendix B — Technology Recommendations

These are recommendations, not requirements. The architecture above is framework-agnostic.

| Layer         | Recommendation          | Rationale                                                                |
| ------------- | ----------------------- | ------------------------------------------------------------------------ |
| Runtime       | Node.js + TypeScript    | Typed event constants, enforced at compile time                          |
| Database      | PostgreSQL              | Row-level locking, ACID transactions, strong indexing                    |
| ORM           | Prisma or Drizzle       | Type-safe queries, migration tooling                                     |
| Queue         | BullMQ (Redis)          | Production-grade, observable, retries built in                           |
| Email         | Resend                  | Modern API, good deliverability, template support                        |
| Cron          | node-cron or Inngest    | node-cron for simplicity; Inngest if you want observability from day one |
| Signing links | JWT (jsonwebtoken)      | Self-verifying, no DB lookup required                                    |
| Hashing       | Node.js `crypto` module | No dependency, SHA-256 is sufficient                                     |
