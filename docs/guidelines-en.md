# 📘 Single Owner Anti-Pattern Guidelines PHP 8.4^

## 🎯 Purpose
This document is a guideline for building a pipeline system in PHP 8.4^ based on the **single owner** principle. The principle ensures that every piece of data has **only one owner** (refcount ~ 1 → 0), every variable must be **moved** once it is used, and no data is allowed to remain alive without an owner. All output must go through a **verified frame mechanism** with the pattern `START–END + token + length (+ checksum)`, which guarantees that **echo statements or debug noise will never leak into STDOUT**.

---

## 🧩 Core Ideas

1. **Linear Flow**
    - **Goal:** Ensure all data flows through a single linear execution path. Each stage consumes the data from the previous stage and produces new data for the next stage.
    - **Setup:** Once a variable is consumed, it must be replaced by a new result. No reuse or branching of data is allowed.

2. **Ephemeral Data**
    - **Goal:** Treat all variables as temporary and short-lived, with no long-term caching inside the process.
    - **Setup:** Any data that needs to persist must be stored outside the process (e.g., Redis or temporary files).

3. **Move Semantics**
    - **Goal:** Enforce that every assignment is treated as a transfer of ownership, not a duplication.
    - **Setup:** Once a variable is moved, it must not be accessed again. Refcount always drops to zero after the move.

4. **Framed Contract**
    - **Goal:** Guarantee that all output data is wrapped in a frame with clear metadata to validate integrity.
    - **Setup:** Every payload must be wrapped with `START–END` markers, including `token`, `length`, and checksum/HMAC.

5. **Anti-Caching / Anti-Sharing**
    - **Goal:** Prevent internal caching or variable sharing inside the PHP process.
    - **Setup:** All data must be single-use. If caching is required, it must live outside the process and be immutable.

6. **Memory as a Transit Area**
    - **Goal:** Use memory only as a transit zone, not as storage.
    - **Setup:** Data must be processed and discarded immediately. No data should remain idle in memory.

7. **Deterministic Execution**
    - **Goal:** Guarantee that the same input always produces the same output.
    - **Setup:** Normalize data (e.g., sorted associative keys, consistent float formatting) so results are deterministic.

8. **Trade-offs**
    - **Goal:** Understand that the single owner principle increases CPU overhead (validation, recomputation) but significantly reduces memory usage.
    - **Setup:** Benchmark with large payloads to measure the balance between CPU cost and memory efficiency.

---

## ⚠️ Pitfalls and Fixes

- **Third-party libraries using echo or global state.**
    - *Pitfall:* Echo noise or global state can break single-owner discipline.
    - *Fix:* Drop foreign OB levels, audit superglobals, and enforce all output through framing.

- **Expensive recomputation.**
    - *Pitfall:* Without internal caching, recomputation can be costly.
    - *Fix:* Allow external caching (Redis, temp files) with short TTL as new sources.

- **Copy-on-Write in PHP.**
    - *Pitfall:* Small mutations on large arrays can trigger expensive copies.
    - *Fix:* Use streaming approaches, minimize mutations, and avoid string concatenation in loops.

- **Ambiguous length in UTF-8.**
    - *Pitfall:* Character length differs from byte length.
    - *Fix:* Always use byte length and record encoding in frame headers.

- **Weak CRC32 checksum.**
    - *Pitfall:* CRC32 is collision-prone.
    - *Fix:* Add HMAC-SHA256 for critical data integrity checks.

- **Regex parsing on large buffers.**
    - *Pitfall:* Regex becomes CPU heavy at scale.
    - *Fix:* Replace with a linear state-machine parser.

- **No caching increases latency.**
    - *Pitfall:* Always recomputing data slows execution.
    - *Fix:* Allow external immutable caching as sources.

- **Memory fragmentation.**
    - *Pitfall:* Long-running processes can fragment memory.
    - *Fix:* Recycle processes after a certain number of frames.

- **Non-deterministic serialization.**
    - *Pitfall:* Key ordering, floats, and locale can cause inconsistent output.
    - *Fix:* Normalize arrays, enforce float formatting, and fix locale.

- **High Time-To-First-Byte (TTFB).**
    - *Pitfall:* Flushing only at the end increases latency.
    - *Fix:* Use chunked flushing (64–256 KB) with per-chunk validation.

- **STDOUT blocking with slow consumers.**
    - *Pitfall:* Slow consumers can cause blocking or deadlock.
    - *Fix:* Apply backpressure policies and timeouts.

- **Weak or predictable tokens.**
    - *Pitfall:* Predictable tokens can be exploited.
    - *Fix:* Generate random tokens from CSPRNG with short lifetimes.

---

## 🧪 Experiment List

### A. Validation and Integrity
1. **Smoke test with valid payload and echo noise.**
    - *Goal:* Ensure only framed payloads reach STDOUT, while echo/debug noise is dropped.
    - *Setup:* Run with one valid payload and three noise echoes (before, inside, and after the frame).

2. **Frame with incorrect length or checksum.**
    - *Goal:* Ensure corrupted frames are rejected.
    - *Setup:* Send three frames (two valid, one with mismatched length/CRC).

3. **Nested frames and multiple tokens.**
    - *Goal:* Ensure parser processes only the intended token.
    - *Setup:* Insert foreign START/END tokens inside payloads.

4. **UTF-8 normalization differences.**
    - *Goal:* Ensure length is based on bytes, not characters.
    - *Setup:* Send payloads with emojis or multibyte characters in NFC and NFD forms.

---

### B. Performance: Throughput and Memory
5. **Payload sizes from 1 KB to 100 MB.**
    - *Goal:* Measure scalability of performance.
    - *Setup:* Run tests with payloads of varying sizes.

6. **Single flush vs chunked flush.**
    - *Goal:* Measure TTFB, total time, and peak memory.
    - *Setup:* Compare flushing 50 MB once vs flushing in 64 KB chunks.

7. **String concatenation vs php://temp sink.**
    - *Goal:* Validate efficiency of stream-based sinks.
    - *Setup:* Process 10–50 MB payloads with both modes.

8. **Regex parser vs state machine parser.**
    - *Goal:* Compare CPU usage.
    - *Setup:* Parse large payloads with both implementations.

9. **JSON serializer vs igbinary/msgpack.**
    - *Goal:* Benchmark serialization speed and output size.
    - *Setup:* Serialize large arrays with JSON and alternatives.

---

### C. Latency, Backpressure, and Robustness
10. **STDOUT backpressure with slow consumer.**
    - *Goal:* Ensure pipeline doesn’t stall with slow readers.
    - *Setup:* Pipe output to a consumer that reads slowly with delays.

11. **Timeout per frame.**
    - *Goal:* Ensure stalled frames do not block indefinitely.
    - *Setup:* Inject artificial delays longer than SLA.

12. **Quota and process recycling.**
    - *Goal:* Prevent memory fragmentation.
    - *Setup:* Process 10,000 small frames, measure RSS and peak memory.

---

### D. Determinism and Normalization
13. **Associative arrays with random key order.**
    - *Goal:* Ensure consistent serialized output.
    - *Setup:* Serialize arrays with shuffled keys.

14. **Float and locale stability.**
    - *Goal:* Ensure consistent numeric formatting.
    - *Setup:* Serialize floats under different locales.

---

### E. Noise and Fault Injection
15. **Noise storm with echo spam.**
    - *Goal:* Ensure only valid frames pass through.
    - *Setup:* Send 10 MB of echo spam plus one valid frame.

16. **Truncated frames.**
    - *Goal:* Ensure incomplete frames are rejected.
    - *Setup:* Send frames with declared LEN larger than body.

17. **Header fuzzing.**
    - *Goal:* Ensure parser resists malformed headers.
    - *Setup:* Use oversized tokens or unknown schema versions.

---

### F. Security and Integrity
18. **Token uniqueness at scale.**
    - *Goal:* Ensure no collisions or predictability.
    - *Setup:* Generate 1 million tokens and test entropy.

19. **CRC32 vs HMAC-SHA256 integrity.**
    - *Goal:* Measure additional CPU overhead.
    - *Setup:* Serialize large payloads with both checksums.

---

### G. Concurrency and Process
20. **Fork safety with pcntl_fork.**
    - *Goal:* Ensure child processes do not overlap output.
    - *Setup:* Each child emits frames with its own token.

21. **Multi-producer to single-consumer pipeline.**
    - *Goal:* Ensure frame order and validity.
    - *Setup:* Multiple producers pipe into one consumer.

---

### H. Observability and Logging
22. **Logging discipline.**
    - *Goal:* Ensure logs go only to STDERR.
    - *Setup:* Spam warnings/errors during processing.

23. **Metrics sampling overhead.**
    - *Goal:* Keep monitoring overhead below 5%.
    - *Setup:* Record encode/flush metrics every N frames.

---

### I. Practical and Edge Cases
24. **Maximum frame size limit.**
    - *Goal:* Ensure oversized frames are dropped.
    - *Setup:* Send frames larger than 128 MB.

25. **Schema version mismatch.**
    - *Goal:* Ensure consumers gracefully reject or fallback.
    - *Setup:* Producer sends a new schema version, consumer expects old.

26. **GC and Copy-On-Write behavior.**
    - *Goal:* Ensure move semantics free memory quickly.
    - *Setup:* Pass large arrays through move-only functions and measure peak memory.

---

## 📊 Experiment Report Template

| ID | Test Name | Setup | Goal | Metrics | Result | Status |
|----|-----------|-------|------|---------|--------|--------|
| 1  | Smoke test | 1 valid + 3 noise echoes | Only valid payload passes | STDOUT bytes | ✅ clean payload | Pass |
| 5  | Large payload | 50 MB JSON | Measure throughput & memory | MB/s, peak usage | … | … |
| 6  | Chunked flush | 50 MB @64KB | Measure TTFB & memory | time, peak usage | … | … |
| …  | … | … | … | … | … | … |

---

## ✅ Summary
- **Core principle:** All data must follow single ownership. Variables are moved once and never reused.
- **Guard:** Every output must go through verified frames with metadata (token, length, checksum).
- **Pitfalls:** CPU overhead, echo noise, recomputation, regex parsing.
- **Fixes:** Use streaming, chunking, linear parsers, and optional HMAC for stronger integrity.
- **Experiments:** Run checklist A–I (1–26) to validate integrity, performance, robustness, security, and determinism.  
