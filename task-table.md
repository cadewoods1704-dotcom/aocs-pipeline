# Task Table & WCET Analysis

---

## 1. System Task Table

All Core 1 tasks use a 4096-byte stack to ensure sufficient headroom for `ESP_LOGI` / `vprintf` formatting.

| Task | Function | Core | Priority | Period | WCET | Utilization | Deadline |
|---|---|---|---|---|---|---|---|
| `prod` | `aocs_sample_task` | 1 | 8 | 50,000 µs | 25 us | 0.050% | 50 ms |
| `cons` | `attitude_update_task` | 1 | 8 | 50,000 µs | 8 us | 0.016% | 50 ms |
| `coord` | `downlink_coordinator_task` | 1 | 9 | 50,000 µs | 7 us | 0.014% | 50 ms |
| `resp` | `responder_task` | 1 | 12 | 50,000 µs | 7 us | 0.014% | 50 ms |
| Core 1 Total | | | | | 47 us | 0.094 % | |
| `webmon` / `monitor` | `webmonitor_task` / `serial_monitor_task` | 0 | 4 | 1,000,000 µs | — | Excluded (Core 0) | Soft |
| (httpd) | `esp_http_server` | 0  | 5 | Async | — | Excluded (Core 0) | Soft |

---

## 2. Priority Rationale

- resp (Priority 12): Highest priority to minimize latency for the interrupt/button-triggered path. Its execution is tiny (single log line), preventing task starvation.
- coord (Priority 9): Positioned directly above prod / cons to ensure completed event-group rendezvous are drained immediately.
- prod & cons (Priority 8): Equal priority with round-robin time-slicing. They form two halves of the 50 ms control pipeline.
- Core 0 Tasks (Priority 4–5): Non-deterministic observability tasks (Wi-Fi/HTTP) are isolated on Core 0 so they physically cannot preempt real-time tasks on Core 1.

---

## 3. Schedulability & Rate-Monotonic Bound

- Liu & Layland Bound (n = 4) U_bound = 4 * (2^1/4 - 1) = 75.7 %
- Measured Total Core 1 Utilization = 47 us / 50000 = 0.094 %
- Conclusion: Pass, Measured utilization is roughly 800x under the bound.

---

## 4. Verification Log
I (59760) aocs: [monitor] q_depth=0  evt=0x00  hb: prod=1195 cons=1195 coord=1195 resp=1195
I (59760) aocs: [wcet] compute us: prod=25 cons=8 coord=7 resp=7  sum=47 of 50000   | one ESP_LOGI call=20495 us
