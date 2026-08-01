# AOCS Pipeline — Real Time Systems Final Capstone

A spacecraft attitude control system that samples, estimates, and downlinks telemetry at
20 Hz across two cores built to demonstrate deterministic scheduling and graceful
degradation for an embedded real time systems role.

## Demo
- Video: <YouTube link to be added later..>
- Live Wokwi: [https://wokwi.com/projects/471030717093156865](https://wokwi.com/projects/471030717093156865) (WOODS-FINAL-RTS26Summer)

## Architecture

[System architecture]

The producer samples attitude at 20 Hz and sends an aocs_t into data_q. The consumer
receives it and computes an estimate. Each one sets its own bit in evt_group when it
finishes. The coordinator waits for the atomic AND of both bits then notifies the
responder to package the downlink packet. A button on GPIO 18 is a second path into that
same responder through button_isr.

Core 0 reads Core 1 state and never writes it. The monitor reads queue depth, event bits,
and the four heartbeat counters. Nothing on Core 0 can preempt Core 1 because the tasks are
pinned so this is enforced by the scheduler.

## Tasks and timing (WCET evidence)

Full method and instrumentation: [docs/task-table.md][docs/hazard-analysis.md](https://github.com/cadewoods1704-dotcom/aocs-pipeline/blob/main/ddocs/task-table.md)

## Hazard analysis and standard mapping

Full table: [docs/hazard-analysis.md](https://github.com/cadewoods1704-dotcom/aocs-pipeline/blob/main/docs/hazard-analysis.md)


## Graceful degradation

Data path. The consumer falls behind, xQueueSendToBack fails after a bounded 5 ms wait,
and the producer evicts the oldest sample and queues the newest.

```c
if (xQueueSendToBack(data_q, &item, pdMS_TO_TICKS(5)) != pdTRUE) {
    aocs_t discard;
    xQueueReceive(data_q, &discard, 0);        /* evict oldest */
    if (xQueueSendToBack(data_q, &item, 0) != pdTRUE)
        ESP_LOGW(TAG, "queue still full after oldest out eviction, sample lost");
    else
        ESP_LOGW(TAG, "back pressure: dropped oldest sample");
}
```

The producer never blocks longer than one tenth of its period so it stays on 20 Hz no matter what the consumer does. Every drop is logged. Observability plane. Wi-Fi is unavailable or the network stack fails to come up. USE_WEBSERVER=0 compiles out esp_wifi, esp_netif, nvs_flash, and esp_http_server, and runs serial_monitor_task on Core 0 reporting the same fields over UART. The Core 1 pipeline is byte for byte identical in both modes, so losing the monitor is a degraded mode and not an outage.

## Build & run

Wokwi needs no hardware. Open the project and press play. It uses USE_WEBSERVER = 1.
Set it to 0 for the serial path in order to demo since it needs no Wi-Fi.

```bash
idf.py set-target esp32s3
idf.py build
idf.py -p /dev/ttyUSB0 flash monitor
```

Wire a normally open button between GPIO 18 and GND. The pin is configured with
GPIO_PULLUP_ENABLE and GPIO_INTR_NEGEDGE, so no external resistor is needed.

## Repository layout

```
├── docs/
│   ├── index.html           # GitHub Pages site
│   ├── architecture.svg     # system diagram
│   ├── task-table.md        # task table + WCET evidence
│   └── hazard-analysis.md   # hazards + standard mapping
├── firmware/
│   └── src/main.c
├── REFLECTION.md
├── LICENSE
└── README.md
```

## Tailored for

General embedded and real time systems work together. In practice that meant putting the effort into things that can be defended instead of things that look impressive. There is a schedule that can be checked with timing that can be reproduced, a hazard table where each mitigation maps to a line of code, and two degradation paths that can be triggered on demand. Those are the parts an interviewer can actually question on. Back pressure, drop policies, bounded waits, and observable liveness are the same four ideas at any role / scale. The one thing not optimized for is consumer facing polish. The web monitor is a functional readout and not a designed dashboard because the interesting engineering is on the other core.
