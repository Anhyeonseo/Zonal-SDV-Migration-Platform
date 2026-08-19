# Development Log

This document records engineering decisions and verified milestones rather than a
day-by-day activity diary.

## Milestone 1 — MCU baseline

### Goal

Establish a known-good board, clock and GPIO baseline before adding communication.

### Result

- NUCLEO-G474RE flash/debug through ST-LINK
- 24 MHz HSE and 170 MHz SYSCLK
- On-board and external LED output verification
- Switch input and debounce verification

### Why it matters

A known-good clock and GPIO baseline prevents peripheral configuration problems from
being confused with basic board or debugger problems.

## Milestone 2 — FDCAN controller bring-up

### Goal

Verify the MCU CAN FD controller without involving transceiver wiring.

### Result

- CAN FD with BRS configuration
- 500 kbit/s arbitration and 2 Mbit/s data phase
- Internal loopback TX/RX
- FIFO0 new-message interrupt callback
- Periodic status transmission and alive counter observation

### Gate

TX/RX counters increase, alive counters remain sequential and communication-error counters
remain zero during uninterrupted execution.

## Milestone 3 — Local inputs and distance sensor

### Goal

Turn simple peripheral experiments into reusable ECU services.

### Result

- ADC calibration and 10 ms sampling service
- Raw ADC to millivolt, percent and permille conversion
- 20 ms switch debounce and press counter
- VL53L0X initialization and measurement state machine
- Distance validity, range status, error and timeout counters

## Milestone 4 — Vehicle-style application contract

### Goal

Separate sensor acquisition, CAN transport and vehicle policy.

### Result

- `FrontZonalStatus` and `CentralControlCommand` contracts
- ECU states: `INIT`, `NORMAL`, `DEGRADED`, `SAFE`
- 500 ms command timeout and deterministic LED-off safe output
- 200/250 mm distance-warning hysteresis
- Sensor fault injection path
- Temporary Central Computer simulator for closed-loop testing

## Milestone 5 — Main reduction and module extraction

### Problem

Peripheral logic, protocol fields, timeout checks and diagnostic variables accumulated in
`main.c`, making later FreeRTOS migration difficult to reason about.

### Decision

Move reusable logic into modules with explicit input, output and status structures:

- `InputService`
- `DistanceSensorService`
- `CanProtocol`
- `CanService`
- `CentralSimulator`
- `FrontZonalApp`

`main.c` remains the composition root that connects modules to HAL and physical outputs.

### Verification

Run host unit tests first, then repeat the board regression checklist. The refactoring is
complete only when the external behavior remains unchanged.

## Milestone 6 — Interrupt timestamp race diagnosis

### Symptom

The CAN timeout count increased even though the last RX age observed in the debugger was
only a few milliseconds.

### Investigation

Independent diagnostic counters showed:

- no encode error
- no HAL TX error
- no payload error
- repeated reason 5, Front Status RX timeout
- valid and continuously changing RX timestamp

### Root cause

An RX interrupt could store a timestamp slightly newer than the `current_time` value that
the superloop had sampled earlier. Unsigned subtraction underflowed and looked like a very
large elapsed interval.

### Resolution

Add a shared elapsed-time helper that handles this interrupt ordering and normal tick
rollover, then apply it to every command/status freshness calculation.

### Evidence

Host tests and ARM build passed, and a fresh board debug session maintained normal loopback
with zero timeout/error count.

## Current baseline and next gate

Current baseline:

```text
STM32 superloop
  + modular application/services
  + host-tested pure logic
  + board-verified FDCAN internal loopback
  + ADC, switch and VL53L0X inputs
  + timeout-based safe output
```

Next gate: establish a reproducible CMake/CI build, then measure the actual blocking
bounds of the ADC and I2C paths under both nominal and fault conditions.

Code inspection has already shown that the current loop is not fully non-blocking:
`HAL_ADC_PollForConversion()` can block for up to 10 ms and each VL53L0X I2C access can
block for up to 100 ms, which is the same order as the 100 ms status transmission period.

The execution model for the edge nodes — keeping the superloop, moving I2C to
interrupt/DMA, or introducing an RTOS — is therefore treated as an open question to be
settled by measurement, not by assumption. The decision and its evidence are recorded in
[ADR-0001](adr/0001-scheduling-model-for-edge-nodes.md).

