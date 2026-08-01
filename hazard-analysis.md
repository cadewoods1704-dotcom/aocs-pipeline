# Hazard analysis and standard mapping

## Hazard table

| # | Hazard | Root Cause | Code Mitigation | Observability / Detection |
|---|---|---|---|---|
| H1 | Sample backlog | Consumer delayed > 50 ms | 5 ms bounded send timeout + drop-oldest eviction policy | q_depth approaches 8; logs back pressure: dropped oldest sample |
| H2 | Silent task death | Task hangs, starves, or crashes | Per-task heartbeat counters incremented per loop. Detection only, nothing acts on them | Single heartbeat stalls while others increment |
| H3 | Network Failure | Wi-Fi/HTTP stack fails | USE_WEBSERVER=0 switch compiles out network stack to run over UART | Telemetry streams over serial instead of HTTP |
| H4 | Torn read | Unlocked concurrent access to item_l | taskENTER_CRITICAL(&monitor_mux) around reads and writes | Prevented by spinlock construction |
| H5 | Stale / Lost Rendezvous | Event bits unconsumed or cleared incorrectly | xWaitForAllBits(pdTRUE, pdTRUE) enforces atomic read-and-clear | Monitor evt field remains stuck non-zero |
| H6 | Responder Monopolizes Core 1 | Task priority 12 runs excessively | Responder body measured at 7 us of compute, so it cannot starve lower tasks. Nothing enforces this if the body grows | hb_prod stalls while hb_resp climbs |
| H7 | Stale Attitude State | Queue empty on consumer read | xQueueReceive uses 100 ms timeout instead of portMAX_DELAY | Logs no sample within 100 ms timeout |
| H8 | Stack Overflow | ESP_LOGI / vprintf formatting headroom | Allocated 4096-byte stacks. Prevents the overflow rather than detecting it | Prevents stack canary trip or Guru Meditation crash |
| H9 | Observability Preemption | HTTP server competes with control loop | Monitor pinned to Core 0 (Priority 4/5); control tasks isolated on Core 1 | Core affinity guarantees zero preemption |
| H10 | Priority Inversion | Task blocked on shared resource | System uses zero mutexes/semaphores on data path; state protected by spinlock | N/A (Prevented by design) |
| H11 | Deadlock | Circular resource dependency | No task acquires multiple resources simultaneously | All 4 heartbeats freeze simultaneously |
