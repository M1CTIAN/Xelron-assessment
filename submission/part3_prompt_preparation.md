# Part 3: Prompt Preparation

**Selected PR:** PR #115 - Support for Compacted Topics  
**Repository:** aiokafka

## 3.1 Repository Context

**Repository:** `aiokafka` (AsyncIO Kafka Client)

The `aiokafka` repository is a Python client for Apache Kafka designed specifically for asynchronous applications. While the standard `kafka python` library is synchronous (blocking), `aiokafka` is built on top of Python's `asyncio` library. This is critical for modern, high throughput microservices—like those built with FastAPI or Sanic—where blocking the main thread for network I/O is unacceptable.

The intended users are backend engineers building real time streaming applications who need to produce or consume messages without halting their event loop. The problem domain is strictly distributed systems communication. Kafka acts as the central nervous system for data, and `aiokafka` ensures that Python applications can talk to this system efficiently.

Unlike standard clients that might wait for a server response before doing anything else, `aiokafka` uses a "Reactor" pattern. It sends requests (like "fetch me data") and immediately goes back to doing other work until the data arrives. It handles all the complex underlying Kafka protocols—partition assignment, group rebalancing, and protocol negotiation—abstracting them away so the developer can just `await consumer.getone()`. It is effectively the bridge between the synchronous world of Kafka protocols and the asynchronous world of modern Python.

---

## 3.2 Pull Request Description

**PR #115: Support for Compacted Topics**

This Pull Request addresses a specific fatal flaw when consuming from "Compacted Topics."

In Kafka, most topics are "retention based" (delete old data after X days). However, "compacted" topics work differently: they keep only the latest value for a specific key and delete older versions. This process creates "holes" or gaps in the offset sequence. For example, if offset `5` is an old update for user `Alice`, and offset `10` is a new update for `Alice`, the compaction process deletes offset `5`.

**The Problem:**
Previously, the `aiokafka` consumer was naive. If it tried to fetch offset `5` (because it remembered being there last time it ran), the Kafka broker would return an `OffsetOutOfRange` error because that message no longer exists. `aiokafka` didn't know how to handle this specific error context; it would just treat it as a generic failure, raise an exception, and crash the consumer application.

**The Fix:**
This PR introduces a "self healing" mechanism in the fetcher loop.

1. **Previous Behavior:** Receive `OffsetOutOfRange` -> Raise Exception -> Crash.
2. **New Behavior:** Receive `OffsetOutOfRange` -> Pause -> Ask Broker for current valid range -> Reset position to the nearest valid offset -> Continue.

The changes are needed because crashing on compaction gaps makes `aiokafka` unusable for KTable like applications or any system using Kafka as a persistent state store. The consumer needs to be smart enough to say, "Oh, that data is gone? I'll just skip to what's available" without waking up the on call engineer.

---

## 3.3 Acceptance Criteria

To consider this PR successfully implemented, the following criteria must be met:

1. ✓ **When** the consumer requests an offset that has been deleted by compaction, the system **should not** raise an `OffsetOutOfRangeError` exception to the user.

2. ✓ **When** the requested offset is smaller than the broker's `earliest_offset`, the consumer **should** automatically reset its position to the `earliest_offset` and continue fetching.

3. ✓ **When** the requested offset is larger than the broker's `latest_offset`, the consumer **should** reset its position to the `latest_offset`.

4. ✓ The implementation **must** emit a log warning indicating that a reset occurred, so operators know data was skipped.

5. ✓ **When** the consumer successfully resets, it **should** immediately attempt to fetch the next batch of records without requiring a manual restart.

---

## 3.4 Edge Cases

The model must account for these specific scenarios:

1. **Large Offset Gaps from Aggressive Compaction:**
   If compaction creates an offset gap exceeding 1 million offsets (for example, jumping from offset 1000 to 5,001,000), the consumer must not iterate through each missing offset checking availability. Instead, it should directly query the broker for the earliest available offset and seek to that position to prevent timeout or excessive API calls. The implementation should handle this in a single `list_offsets` request rather than making sequential checks.

2. **Concurrent Compaction During Active Consumption:**
   If the Kafka broker runs compaction while the consumer is actively fetching data, and the specific offset being requested gets compacted before the fetch completes, the consumer must handle the `OffsetOutOfRangeException` gracefully. Rather than crashing, it should immediately seek to the earliest available offset in that partition and continue processing. This scenario is particularly common in high-throughput systems where compaction runs frequently.

3. **Fully Compacted Empty Partition:**
   In scenarios where all messages in a partition share the same key and the latest message is a delete tombstone (null value), the partition becomes completely empty after compaction. The consumer should recognize this state (where `earliest_offset` == `latest_offset` or both are undefined), not crash, and either wait for new messages or handle the empty state gracefully based on configuration. The error handling logic must distinguish between "no messages yet" and "all messages were compacted away."

---

## 3.5 Initial Prompt

**Role:** Senior Python Engineer  
**Task:** Implement robust error handling for Kafka Compacted Topics in `aiokafka`.

**Context:**
We are working on `aiokafka`, an asynchronous Python client for Apache Kafka. Currently, our `FetchRequest` loop is fragile when dealing with compacted topics. If a consumer requests an offset that has been garbage-collected by the broker (compaction), the broker returns an `OFFSET_OUT_OF_RANGE` error. Our current implementation simply propagates this error, causing the consumer to crash.

**Objective:**
Modify `aiokafka/consumer/fetcher.py` and `aiokafka/client.py` to handle this error gracefully. The consumer should detect when it is asking for a deleted offset and automatically "fast-forward" to the next valid offset without crashing.

**Technical Requirements:**

1. **Modify `aiokafka/client.py`:**
   * Expose two new helper methods: `fetch_earliest_offsets(partitions)` and `fetch_latest_offsets(partitions)`. These should send a `ListOffsetRequest` to the broker to find the current valid boundaries of the topic.

2. **Modify `aiokafka/consumer/fetcher.py`:**
   * Locate the `_proc_fetch_request` method (or the main fetch handling loop).
   * Wrap the fetch logic in a `try/except` block specifically catching `OffsetOutOfRangeError`.
   * **Inside the `except` block:**
     * Do not fail immediately.
     * Call the new client methods to get the valid range (`earliest`, `latest`) for the failing partition.
     * Compare the *requested* offset against these valid offsets.
     * **Logic:**
       * If `requested < earliest`: Reset position to `earliest`.
       * If `requested > latest`: Reset position to `latest`.
     * Update the internal subscription state with the new offset.
     * Log a warning: "Offset out of range, resetting to {new_offset}".

**Acceptance Criteria:**
Reference the 5 criteria defined in Section 3.3 above. Ensure all criteria are met, particularly that the consumer continues processing without manual intervention and logs appropriate warnings.

**Edge Cases to Consider:**
Pay special attention to the 3 edge cases defined in Section 3.4:
1. Large offset gaps (millions of missing offsets)
2. Concurrent compaction during active reads
3. Fully compacted empty partitions

**Constraints:**
* **Concurrency:** Since this is `asyncio`, ensure you are `await` ing the offset validation calls. Do not block the event loop.
* **Empty Topics:** Ensure the logic handles cases where the topic exists but has 0 messages.
* **Safety:** Only handle `OffsetOutOfRangeError`. Any other Kafka error (e.g., `TopicAuthorizationFailed`) should still raise an exception as normal.

**Testing Requirements:**
Write integration tests that:
1. Simulate a compacted topic state
2. Verify the consumer skips missing offsets
3. Confirm appropriate warning logs are emitted
4. Validate the consumer continues processing after reset

**Deliverables:**
* Modified `aiokafka/client.py` with new offset fetching methods
* Modified `aiokafka/consumer/fetcher.py` with error handling logic
* New or updated tests in `tests/test_compacted_topics.py`
* Updated documentation if necessary

---

## Integrity Declaration

I declare that all written content in this assessment is my own work, created without the use of AI language models or automated writing tools. All technical analysis and documentation reflects my personal understanding and has been written in my own words.
