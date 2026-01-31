## 4.1 Scenario Response

**To the Reviewer:**

I selected **PR #115 (Support for Compacted Topics)** because it addresses a fundamental distributed systems challenge resilience in the face of data loss that I have personally encountered while building event-driven architectures.

**Selection Rationale & Technical Background**
My decision was heavily influenced by my recent experience building a **Distributed SOC Monitoring System** using Apache Kafka. In that project, I dealt extensively with log retention policies and the complexities of maintaining consumer state. I understand the specific pain point this PR addresses: when a Kafka broker compacts (deletes) old data, a naive consumer holding an old offset will crash because it tries to read a message that no longer exists.

I found this PR comprehensible because it maps directly to the "OffsetOutOfRange" errors I’ve debugged in the past. Furthermore, since I have experience with **Python's `asyncio**` and backend development (FastAPI/Django), I felt comfortable analyzing how `aiokafka` manages its event loop. The shift from a "crash-on-error" model to a "check-and-reset" model is a pattern I recognize and value in production systems.

**Anticipated Implementation Challenges**
The biggest challenge in implementing this is **testing reliability**. Simulating real Kafka log compaction in a test environment is notoriously slow and flaky because it relies on broker background threads that we cannot easily control. Writing a test that waits for compaction might make the CI/CD pipeline hang or fail randomly.

Another challenge is **concurrency**. Since `aiokafka` is asynchronous, there is a risk of a race condition: if we reset the offset in the middle of a fetch loop while another concurrent task is also trying to read or commit offsets, we could end up with an inconsistent state.

**Overcoming These Challenges**
To handle the testing challenge, I would avoid waiting for real compaction. Instead, I would **mock the broker's response** to force an `OffsetOutOfRange` error and verify that the client calls `list_offsets` in response. This ensures we test the *logic* without relying on the broker's timing. For the concurrency issue, I would ensure the reset logic is atomic or guarded by a lock, ensuring no other fetch requests are sent until the offset is successfully re-established.
