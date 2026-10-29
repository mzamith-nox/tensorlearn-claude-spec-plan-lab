# Implementation plan

## Ordered steps

1. Define the release-event contract and the job shape: event ID, released result version as the idempotency key, and a link from `resultRoute` in `src/portal/result-route.mjs`, with no clinical fields (R1, R2, R4).
2. Add the release gate in new file `src/service/release-notifier.mjs` so only a doctor-released result version creates a job (R1).
3. Add a dedupe check on the released result version and bounded retries that reuse the same key (R5, R6).
4. Add a throttle and batcher capped at 50 requests per second and 20 messages per batch, feeding `enqueueNotification` from the existing `src/service/queue.mjs` (R7).
5. Add redacted logging of event IDs and delivery states only (R3).
6. Write the tests listed below and confirm that nothing depends on OPEN-1 (channel) or OPEN-2 (referring doctors).

## Paths and dependencies

- Existing `src/portal/result-route.mjs` is reused unchanged for the patient link (`/portal/results/:resultId`).
- Existing `src/service/queue.mjs` is reused unchanged; the notifier calls its `enqueueNotification(queue, job)`.
- New file `src/service/release-notifier.mjs` is proposed: it holds the release gate, dedupe, retries, throttle and redacted logging.
- The archived helper `src/service/sendAllResults.mjs` does not exist and is not used; the plan does not depend on it.
- Dependency order: step 1 comes before steps 2 to 5, steps 2 to 5 are done in order, and step 6 runs against all of them.
- No new dependencies are installed, and no portal file changes.

## Risks and verification

- **Notifying before release:** checked by `release_gate_blocks_unreleased`.
- **Clinical content or patient identifiers leaking into messages or logs:** checked by `notification_has_no_clinical_content` and `logs_contain_only_event_ids_and_states`.
- **Duplicate delivery on retry:** checked by `retry_same_version_delivers_once`, which retries the same released result version after a provider failure and expects exactly 1 delivery.
- **Throttling or rejection by the provider:** checked by `throughput_boundary_50_per_second_20_per_batch`, which sends exactly 50 requests in 1 second and 20 messages in a batch, then 1 more, and expects the overflow to be deferred, not dropped or duplicated.
- **Wrong link or portal change:** checked by `link_uses_existing_result_route`.
- **Scope creep on channel or recipients:** OPEN-1 and OPEN-2 stay with their owners, and no channel- or doctor-specific code is written.

| Requirement | Step | Existing path | Test |
| --- | --- | --- | --- |
| R1 Release gate | 2 | - | `release_gate_blocks_unreleased` |
| R2 No clinical content in notification | 1 | `src/portal/result-route.mjs` | `notification_has_no_clinical_content` |
| R3 Logs hold event IDs and states only | 5 | - | `logs_contain_only_event_ids_and_states` |
| R4 Reuse portal route | 1 | `src/portal/result-route.mjs` | `link_uses_existing_result_route` |
| R5 Idempotency key = released result version | 3 | `src/service/queue.mjs` | `retry_same_version_delivers_once` |
| R6 Bounded retries without duplicates | 3 | `src/service/queue.mjs` | `retry_same_version_delivers_once` |
| R7 50 requests/second, 20 messages/batch | 4 | `src/service/queue.mjs` | `throughput_boundary_50_per_second_20_per_batch` |
