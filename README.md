# Doclast Architecture Review

## Insights from SidebarVulcan — Applied to Document Lifecycle Management

**Prepared for:** Masiha  
**Date:** June 2026  
**Scope:** Backend architecture recommendations for Doclast's document workflow engine

---snowball

## Executive Summary

SidebarVulcan is a production-grade newsletter platform built by Sacha Greif on Meteor/Vulcan.js. On the surface it manages blog posts. Architecturally it is a lifecycle orchestration system — and the patterns it uses to move a `Post` through `draft → scheduled → published` map almost directly onto Doclast's `Document` moving through `draft → sent → viewed → reviewing → approved → signed`.

The core insight is this: **Sidebar treats its central object as a lifecycle entity, not just a data record.** Every layer of the system — schema, callbacks, emails, cron — is organized around that lifecycle rather than around technical concerns. This is the pattern worth adopting wholesale, independent of any framework choice.

Three specific files contain directly transferable ideas. One file (`callbacks.js`) contains a pattern worth adapting with corrections. The Vulcan.js framework wiring is not portable and should be left behind.

---

## Part 1 — What Sidebar Actually Does

### The `Post` Object as a Lifecycle Entity

Sidebar's `Post` is not a passive data bag. It carries its full history as timestamped fields:

```js
// From schema.js (simplified)
{
  status: 'draft' | 'scheduled' | 'published',
  scheduledAt: Date,   // when it entered the scheduled state
  postedAt: Date,      // when it was published
  paidAt: Date,        // when a sponsorship was confirmed
}
```

This is the first architectural decision worth noting. The status field tells you **where** the document is. The timestamp fields tell you **how it got there and when**. Both are required for a useful audit trail and for analytics ("what is the average time between sent and signed?").

### The `callbacks.js` Pattern

Sidebar registers named, pure functions that run on mutations:

```js
// From callbacks.js (simplified)
addCallback("posts.new.after", checkForDuplicates);
addCallback("posts.edit.after", setPostPaidAt);
addCallback("posts.new.after", sendSponsorshipPaymentNotification);
```

Each callback is a focused function with a single responsibility. They are composable — you can add a new one without touching existing ones. They are testable — pure functions that accept document state and return mutations or fire side effects.

The weak point: some callbacks branch on `context.currentUser` or `context.event` strings. This creates invisible coupling — a callback that checks `if (context.event === 'stripe.process.sync')` is secretly dependent on an external system. For Doclast, named event types should be explicit constants, not magic strings.

### The `emails.js` Pattern

This is the most directly portable idea in the codebase. Each email is a **self-contained declaration**:

```js
// From emails.js (simplified)
{
  sponsorPaymentConfirmed: {
    template: 'sponsorPaymentConfirmed',
    subject: (data) => `Payment confirmed for ${data.post.title}`,
    to: (data) => data.post.sponsorEmail,
    query: `
      query PostEmailData($documentId: String) {
        post(input: { selector: { _id: $documentId } }) {
          result { title sponsorEmail postedAt }
        }
      }
    `,
    testVariables: { documentId: 'testId' }
  }
}
```

The email knows what data it needs. The caller only needs to say "send `sponsorPaymentConfirmed` for document `xyz`". No data preparation, no template building at the call site. This decouples the email sending trigger from the email rendering entirely.

### The `cron.js` Pattern

A dedicated file, run on a schedule, with focused job functions:

```js
// From cron.js (simplified)
SyncedCron.add({
  name: "Check for scheduled posts",
  schedule: (parser) => parser.text("every 1 hour"),
  job: () => checkForScheduledPosts(),
});
```

The principle: scheduled work is isolated from mutation logic. Nothing that happens due to a user action should also happen because a scheduler ran — they are different code paths.

---

## Part 2 — What Transfers to Doclast

### Direct transfers (no adaptation needed)

**1. Lifecycle timestamps on the Document schema**

Every status transition should leave a timestamp. This is a data modeling decision that costs nothing and pays dividends in analytics, debugging, and audit trails.

**2. Declarative email registry**

The self-contained email object pattern translates directly. Each email in Doclast's system should declare its own data query and recipient resolution. No scattered `sendEmail()` calls.

**3. Dedicated `cron.js` for time-based automation**

Signature reminders, expiry enforcement, and abandoned draft cleanup cannot be triggered by user actions. They need a scheduler. Isolating this in a single file is the correct separation.

### Patterns worth adapting (with corrections)

**4. Named callbacks on lifecycle transitions**

Adopt the pattern but replace magic context strings with explicit event constants:

```js
// ❌ Sidebar's approach — implicit context
if (context.event === 'stripe.process.sync') { ... }

// ✅ Doclast's approach — explicit event types
const DOCUMENT_EVENTS = {
  SENT:     'document.sent',
  VIEWED:   'document.viewed',
  SIGNED:   'document.signed',
  EXPIRED:  'document.expired',
};
```

**5. Callback composition**

Register callbacks per event type, not per mutation type. Sidebar registers per CRUD operation (`posts.new.after`). Doclast should register per business event (`document.sent`), which is more semantically meaningful and avoids callback proliferation.

### What does not transfer

**6. Vulcan.js framework wiring**

`addCallback`, `updateMutator`, `Connectors` — these are Meteor/Vulcan primitives with no equivalent in a modern Node.js stack. The pattern transfers; the implementation does not.

**7. GraphQL for internal data fetching in emails**

Sidebar uses GraphQL queries inside email definitions because GraphQL is its only data layer. If Doclast uses tRPC or a direct ORM, adapt the pattern: each email should have a `getData(documentId)` function rather than a query string.

---

## Part 3 — Recommended Architecture for Doclast

### The Document Schema

```ts
interface Document {
  // Identity
  id: string;
  ownerId: string;
  title: string;
  templateId: "NDA" | "collaboration" | "MOU";

  // Lifecycle state
  status: DocumentStatus;

  // Lifecycle timestamps — one per state transition
  createdAt: Date;
  sentAt: Date | null;
  viewedAt: Date | null;
  reviewingAt: Date | null;
  approvedAt: Date | null;
  signedAt: Date | null;
  expiredAt: Date | null;

  // Configuration
  expiresAt: Date | null; // deadline set by owner at creation

  // Signer
  signerEmail: string;
  signatureHash: string | null; // stored after signing, for verification

  // Metadata
  fields: Record<string, string>; // filled form data
}

type DocumentStatus =
  | "draft"
  | "sent"
  | "viewed"
  | "reviewing"
  | "approved"
  | "signed"
  | "expired";
```

The critical rule: **status and its corresponding timestamp must always be updated atomically.** When status changes to `signed`, `signedAt` is set in the same write. Never update one without the other.

### The Callback Layer

```ts
// callbacks/index.ts
// Single registration point — all business logic hooks in one place.
// Each callback is a named, pure function imported from its own file.

import { validateRequiredFields } from "./onCreate";
import { stampLifecycleTimestamp } from "./onStatusChange";
import { triggerDocumentSentEmail } from "./onSent";
import { verifySignatureAndStampHash } from "./onSign";
import { blockFurtherActions } from "./onExpiry";

export const documentCallbacks = {
  [DOCUMENT_EVENTS.CREATED]: [validateRequiredFields],
  [DOCUMENT_EVENTS.SENT]: [stampLifecycleTimestamp, triggerDocumentSentEmail],
  [DOCUMENT_EVENTS.VIEWED]: [stampLifecycleTimestamp],
  [DOCUMENT_EVENTS.REVIEWING]: [stampLifecycleTimestamp],
  [DOCUMENT_EVENTS.APPROVED]: [stampLifecycleTimestamp],
  [DOCUMENT_EVENTS.SIGNED]: [
    stampLifecycleTimestamp,
    verifySignatureAndStampHash,
  ],
  [DOCUMENT_EVENTS.EXPIRED]: [stampLifecycleTimestamp, blockFurtherActions],
};
```

Each callback module is a focused file that does exactly one thing. `stampLifecycleTimestamp` does not know about email. `triggerDocumentSentEmail` does not know about signature verification.

### The Email Registry

```ts
// emails/index.ts
// Declarative registry. Each email knows its own data requirements.
// The caller only provides the documentId and event type.

export const documentEmails = {
  documentSent: {
    template: "documentSent",
    subject: (d) => `Please review and sign: ${d.document.title}`,
    to: (d) => d.document.signerEmail,
    getData: (id) => DocumentService.getEmailData(id),
  },

  signatureReminder: {
    template: "signatureReminder",
    subject: (d) =>
      `Reminder: ${d.document.title} is waiting for your signature`,
    to: (d) => d.document.signerEmail,
    getData: (id) => DocumentService.getEmailData(id),
  },

  documentSigned: {
    template: "documentSigned",
    subject: (d) => `Signed: ${d.document.title}`,
    to: (d) => d.document.ownerEmail,
    getData: (id) => DocumentService.getEmailData(id),
  },

  expiryWarning: {
    template: "expiryWarning",
    subject: (d) => `Action required: ${d.document.title} expires in 48 hours`,
    to: (d) => d.document.signerEmail,
    getData: (id) => DocumentService.getEmailData(id),
  },

  documentExpired: {
    template: "documentExpired",
    subject: (d) => `Expired: ${d.document.title}`,
    to: (d) => d.document.ownerEmail,
    getData: (id) => DocumentService.getEmailData(id),
  },
};

// Sending a document email from anywhere in the codebase:
// await sendDocumentEmail('documentSent', documentId);
// That is the entire API surface.
```

### The Cron Layer

```ts
// cron/index.ts

// Every 6 hours: find documents sent but never viewed, queue reminders.
cron.schedule("checkUnviewedDocuments", "0 */6 * * *", async () => {
  const stale = await DocumentService.findSentUnviewed({
    sentBefore: subHours(now(), 48),
  });
  for (const doc of stale) {
    await sendDocumentEmail("signatureReminder", doc.id);
  }
});

// Every hour: find documents past their expiresAt, update status to expired.
cron.schedule("enforceDocumentExpiry", "0 * * * *", async () => {
  const overdue = await DocumentService.findExpired();
  for (const doc of overdue) {
    await DocumentService.transition(doc.id, DOCUMENT_EVENTS.EXPIRED);
  }
});

// Daily: archive drafts abandoned for more than 30 days.
cron.schedule("archiveAbandonedDrafts", "0 2 * * *", async () => {
  await DocumentService.archiveOlderThan({
    status: "draft",
    olderThan: subDays(now(), 30),
  });
});
```

---

## Part 4 — Key Architectural Decisions

### 1. Single transition function

All status changes, from any trigger (user action, cron job, webhook), must go through a single `DocumentService.transition(documentId, event)` function. This function stamps the timestamp, runs the registered callbacks, and persists atomically. There is no path to change document status that bypasses this function.

This is the most important decision. It prevents the state and timestamp from drifting apart, makes the callback system reliable, and gives you a single place to add logging, monitoring, or rate limiting later.

### 2. Events over CRUD

Callbacks are registered on **business events** (`document.signed`), not on **database operations** (`documents.update.after`). The event carries intent; the CRUD operation does not. A document update can mean a field was edited, a status changed, or a signature was added — these are different events that should trigger different callbacks.

### 3. Immutable audit trail

Never delete or overwrite lifecycle timestamps. If a document is re-sent, add a `resentAt` field rather than overwriting `sentAt`. The document's history is a product feature, not an implementation detail.

### 4. Email as a declarative registry, not imperative calls

Enforce a rule: no file outside `emails/index.ts` may construct or send an email directly. All email sending goes through `sendDocumentEmail(emailKey, documentId)`. This gives you a complete map of every email the system sends in one file, makes testing trivial, and prevents email spaghetti as the codebase grows.

---

## Part 5 — What Not to Do

### Avoid the Sidebar mistake: magic strings in callbacks

The most fragile pattern in `callbacks.js` is checking `context.event === 'some-string'` inside a callback. This creates a dependency that the type system cannot catch and that tests will not surface until production. Use typed event constants enforced by TypeScript from day one.

### Do not mix lifecycle logic with template rendering

Doclast generates PDF/DOCX documents. The generation logic (filling a template with field data) must be entirely separate from the lifecycle logic (what happens after the document is generated). A good test: the lifecycle layer should be completely testable without any PDF library installed.

### Do not use status as the only lifecycle signal

A document in `sent` status for 30 minutes is very different from one in `sent` status for 14 days. The timestamp fields enable this distinction. Any feature that requires time-awareness — reminders, expiry, analytics — will need them. Add them now.

---

## Appendix — Files Reviewed

| File           | Location                                  | Key finding                                                                       |
| -------------- | ----------------------------------------- | --------------------------------------------------------------------------------- |
| `schema.js`    | `packages/sidebar2020/lib/modules/posts/` | Lifecycle timestamps as first-class schema fields                                 |
| `callbacks.js` | `packages/sidebar2020/lib/server/posts/`  | Named composable callbacks per mutation; magic context strings are the weak point |
| `emails.js`    | `packages/sidebar2020/lib/server/emails/` | Self-contained declarative email objects with embedded data queries               |
| `cron.js`      | `packages/sidebar2020/lib/server/`        | Isolated scheduled job registry; clean separation from mutation layer             |
| `README.md`    | Repository root                           | High-level architecture: Vulcan.js on Meteor, GraphQL data layer, MongoDB         |

# Doclast-Architecture-Review
