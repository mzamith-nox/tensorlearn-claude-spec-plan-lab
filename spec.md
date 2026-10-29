# Specification

## Requirements

- **R1 (I-1) Release gate:** a patient is notified only after a doctor has reviewed and released the result; no notification is sent for a result that has not been released.
- **R2 (I-2) Content-free notification:** the notification says only that a result is ready and links to the authenticated portal. Clinical content stays out of notifications and logs.
- **R3 (I-2) Log redaction:** operational logs record event IDs and delivery states only, never patient identifiers or clinical payloads.
- **R4 (I-3) Reuse the portal route:** the notification link uses the existing `resultRoute` (`/portal/results/:resultId`) from `src/portal/result-route.mjs`, and the portal itself is not changed.
- **R5 (I-4) Idempotency key:** the idempotency key is the released result version, so one released result version is delivered at most once.
- **R6 (I-4) Bounded retries:** provider failures are retried a bounded number of times with the same idempotency key, so a retry never creates a duplicate delivery.
- **R7 (I-4) Provider limits:** sending stays within the provider contract of at most 50 requests per second and at most 20 messages per batch.

## Design

- The trigger is the doctor's release event for a result version; nothing earlier in the result lifecycle starts a notification.
- New file `src/service/release-notifier.mjs` (proposed, not yet created) checks the release gate, builds the job and hands it to `enqueueNotification` from the existing `src/service/queue.mjs`.
- The job holds only the event ID, the released result version (the idempotency key), a recipient reference resolved at send time, and a link built from `resultRoute` in `src/portal/result-route.mjs`; it has no clinical fields.
- A dedupe check on the released result version runs before each send, and retries reuse that key with a fixed maximum number of attempts.
- A throttle and batcher in front of the provider limits sends to 50 requests per second and groups at most 20 messages per batch.
- A redacting logger writes only event IDs and delivery states.
- The design is channel-agnostic until OPEN-1 is decided, and sends to patients only (OPEN-2 is out of scope).
- Rejected archived advice: notifying on upload is rejected, the archived `sendAllResults` helper is rejected and not used (it does not exist), putting result details in messages is rejected, and the portal redesign is rejected.

## Open decisions and owners

- **OPEN-1 Channel choice (SMS, email or push):** the owner is product with privacy approval. It is still open, so the design does not fix a channel.
- **OPEN-2 Referring-doctor notifications:** the owner is clinical with privacy approval. They are not in scope and are not built until approved.

## Traceability

| Intent | Requirement | Test |
| --- | --- | --- |
| I-1 Notify after clinical review and release | R1 | `release_gate_blocks_unreleased`: an unreleased result produces no job; a released one produces exactly one |
| I-2 Clinical content only in the portal, never in notifications or logs | R2, R3 | `notification_has_no_clinical_content` and `logs_contain_only_event_ids_and_states` |
| I-3 Reuse the existing portal route; portal redesign rejected | R4 | `link_uses_existing_result_route`: the link matches `/portal/results/:resultId` |
| I-4 No duplicate per released version; handle provider failure within its contract | R5, R6, R7 | `retry_same_version_delivers_once` and `throughput_boundary_50_per_second_20_per_batch` |
| OPEN-1 Channel choice | Owner: product + privacy | No test until decided |
| OPEN-2 Referring-doctor notifications | Owner: clinical + privacy; out of scope | No test until approved |
