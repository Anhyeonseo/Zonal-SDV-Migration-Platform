# Verification

## 1. Baseline acceptance status

| Area | Test | Result |
|---|---|---|
| Build | STM32CubeIDE ARM Clean Build | PASS — 0 errors, 0 warnings |
| Clock | 24 MHz HSE to 170 MHz SYSCLK | PASS |
| ADC | Potentiometer covers low, middle and 4095 full-scale regions | PASS |
| Input | Switch raw state and 20 ms debounce | PASS |
| Distance sensor | VL53L0X initialization and changing distance values | PASS |
| CAN controller | CAN FD internal loopback | PASS |
| CAN RX path | FIFO0 interrupt callback and RX counter | PASS |
| Protocol | Status and command encode/decode tests | PASS |
| Application | State, output, timeout and sensor fault tests | PASS |
| Refactoring | InputService and CanService extraction board regression | PASS |
| Timing fix | No false CAN timeout after interrupt timestamp fix | PASS |

## 2. Host unit tests

| Test source | Scope | Expected output |
|---|---|---|
| `Tests/can_protocol_test.c` | Byte layout, round trip and invalid payload | `can_protocol tests: PASS` |
| `Tests/central_sim_test.c` | 200 ms scheduling, stale data, warning hysteresis | `central_sim tests: PASS` |
| `Tests/front_zonal_app_test.c` | ECU states, command timeout, safe output, fault injection | `front_zonal_app tests: PASS` |
| `Tests/input_service_test.c` | 10 ms ADC period, conversion, error count, debounce | `input_service tests: PASS` |
| `Tests/time_utils_test.c` | Normal elapsed time, future ISR timestamp, tick rollover | `time_utils tests: PASS` |

These tests exercise hardware-independent logic. HAL/FDCAN integration is verified on
the NUCLEO board, not emulated by the current host tests.

## 3. Board regression checklist

After a structural firmware change:

1. Terminate the old debug session so counters start from reset values.
2. Clean and build the `Debug` configuration.
3. Start a new debug session and immediately resume execution.
4. Let the board run for at least 10 seconds without intermediate breakpoints.
5. Pause once and inspect each expression separately.

Expected CAN service baseline:

```text
can_status.loopback_result                       = 1
can_service.front_status_rx_timeout_active       = 0
can_status.front_status_rx_timeout_count         = 0
can_status.last_error_reason                     = 0
uwTick - can_status.last_front_status_rx_time_ms = approximately 0..100 ms
```

Additional expectations:

- `front_status_tx_count` and `front_status_rx_count` continue increasing.
- TX may lead RX by one when the debugger pauses between enqueue and interrupt handling.
- Status and command sequence-error counters remain zero during uninterrupted execution.
- Distance changes update `last_front_status.distance_mm`.
- Moving the potentiometer changes `adc_raw` and `position_permille`.
- A valid close obstacle turns on the warning LED; moving past the OFF threshold turns it off.

## 4. Debugging case: false RX timeout

### Symptom

CAN frames were being received every few milliseconds, but the timeout counter increased
and `last_error_reason` reported reason 5.

### Root cause

The superloop sampled `current_time` before the FDCAN RX interrupt updated the last-RX
timestamp. Unsigned subtraction then interpreted a timestamp 1 ms in the future as a
very large elapsed time and produced a false timeout.

### Fix

`TimeUtils_ElapsedMs()` clamps an interrupt-updated future timestamp to zero while still
supporting normal 32-bit tick rollover. CAN service, Central Simulator and Front Zonal
Application now use the same helper.

### Regression evidence

- Host time utility test: PASS
- Central Simulator test: PASS
- Front Zonal Application test: PASS
- ARM Clean Build: 0 errors, 0 warnings
- Board result after fresh reset: loopback normal and timeout/error counters remain zero

## 5. Evidence still to collect

The following artifacts should be added when available:

- Annotated hardware photograph
- CubeIDE expression screenshot for the baseline counters
- Logic analyzer capture after external transceiver integration
- `candump` trace from the Raspberry Pi
- Cable disconnect/reconnect fault trace
- Long-duration traffic and error-counter report
- ADC/I2C blocking-time measurement under nominal and fault conditions
  (see [ADR-0001](adr/0001-scheduling-model-for-edge-nodes.md))
- Status transmission jitter distribution and deadline miss count

Performance evidence must report the sample count together with p50, p95, p99, p99.9 and
max, plus the deadline miss count. Averages alone are not sufficient. Raw data and the
command used to produce it are stored alongside any summary chart.

## 6. Known limitations

- The physical transceiver and CANH/CANL bus are not part of the current result.
- Debugger pauses can intentionally create a real timeout and must not be treated as a
  runtime failure.
- The host tests do not currently have a committed one-command runner or CI workflow.
- Sensor timeout fault injection is partly debugger-driven rather than automated HIL.

