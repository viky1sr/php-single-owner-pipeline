# PHP Single Owner Pipeline

A reference implementation and guidelines for building a **CLI-based pipeline system** in PHP 8.4^ with the **Single Owner Anti-Pattern**.  
This repository provides both **documentation (EN/ID)** and **examples** to ensure data integrity, memory discipline, and echo-free output.

---

## 📖 Documentation

- [ID – Panduan Single Owner Anti-Pattern (Bahasa Indonesia)](docs/guidelines-id.md)
- [EN – Single Owner Anti-Pattern Guidelines (English)](docs/guidelines-en.md)

---

## 🚀 Features

- **Single Owner Principle** → all variables must be moved (no idle data, refcount ~ 1 → 0).
- **Framed Contract** → `START–END + token + length (+ checksum/HMAC)` ensures output integrity.
- **Ephemeral Data** → memory acts only as transit, no long-lived cache.
- **Deterministic Execution** → normalized arrays, consistent floats, schema versioning.
- **Noise Shield** → echo/debug statements are dropped, logs go only to STDERR.
- **Experiments Checklist** → 26 tests (validation, performance, determinism, security).

---

## 🧪 Experiments

Refer to the documentation for the full list of **26 experiments (A–I)**.  
Example categories include:
- Validation & integrity checks
- Throughput and memory usage
- Latency and backpressure
- Deterministic serialization
- Fault injection (truncated frames, noise storms)
- Security (token entropy, HMAC overhead)
- Concurrency (multi-producer pipelines, fork safety)

---

## 📂 Project Structure

```text
php-single-owner-pipeline/
├── docs/
│   ├── guidelines-id.md    # Full guidelines in Bahasa Indonesia
│   └── guidelines-en.md    # Full guidelines in English
│
├── src/                    # Core library (Owned, Framer, Parser, etc.)
│
├── example/
│   └── cli.php             # Example CLI pipeline usage with frames
│
└── README.md               # Project overview 
```


---

## 🔧 Requirements

- PHP **8.4^** (CLI mode)
- Extensions: `pcntl`, `posix`, (optional: `igbinary`, `msgpack`)
- Linux / WSL recommended for process forking and STDOUT backpressure tests

---

## 📊 Reporting Experiments

Use the included [report template](docs/guidelines-en.md#-experiment-report-template) or copy the following table:

| ID | Test Name | Setup | Goal | Metrics | Result | Status |
|----|-----------|-------|------|---------|--------|--------|
| 1  | Smoke test | 1 payload + 3 echoes | Only framed payload passes | STDOUT bytes | ✅ | Pass |

---

## 📝 License

MIT License – feel free to use and modify for your own pipelines.
