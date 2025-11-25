# Drift Controller OS

**Drift Controller OS** is a lightweight, soft real-time runtime designed to run **Drift** programs directly on microcontroller-class hardware.
It provides a deterministic execution model for periodic sensor tasks, GPIO control, and small embedded applications — all without relying on raw pointers, POSIX, or a traditional operating system.

This project is not a full kernel. It’s a focused demonstration of **Drift as a systems language**.

---

## Why this exists

Drift needs a credible “metal-level” showcase:

* Can Drift run without an OS?
* Can hardware be accessed safely?
* Can real-world control logic be implemented deterministically?
* Can signed modules be loaded and verified at boot?
* Can exceptions and context-tracked errors work on bare metal?

**Drift Controller OS answers all of that.**
It’s a small but real environment that proves Drift can serve as a viable replacement for C/C++ in embedded control.

---

## High-level architecture

### 1. Bootstrap Layer (C/Assembly)

Minimal hardware bring-up:

* CPU initialization
* Stack, memory region, and static allocator
* Timer interrupt → global monotonic tick
* Hardware shims (ADC, GPIO, PWM, UART) via `extern "C"`

No libc. No dynamic runtime. As small as possible.

### 2. Drift Runtime Core

Written entirely in Drift:

* Cooperative **virtual-thread** executor
* **sleep_until()** / **yield()** for deterministic timing
* Static priority scheduler (High / Medium / Low)
* Panic handler with structured `Error` context frames
* Safe wrappers around unsafe hardware shims

### 3. Hardware Abstraction Layer (HAL)

Type-safe Drift interfaces:

```drift
interface AnalogInput  { fn read() returns Int32 }
interface DigitalInput { fn read() returns Bool }
interface DigitalOutput { fn set(on: Bool) returns Void }
interface PwmOutput     { fn set_duty(percent: Float64) returns Void }
```

Concrete implementations call into the C bootstrap.

### 4. Controller Modules (Signed)

User logic lives in signed **DMP modules**:

* loaded at boot
* signature-verified
* fall back to last-known-good if invalid
* implement real control loops:

  * sensors
  * filtering
  * GPIO actuation
  * telemetry/logging

This is the “application layer” of Drift Controller OS.

---

## Soft real-time model

The system uses a **tick-based soft RT scheduler**:

* 1 tick = 1 ms (or hardware-dependent)
* Timer ISR updates `CURRENT_TICK`
* Tasks run only when:

  * their deadline has arrived, and
  * no higher-priority task is pending

Tasks must not block and must explicitly yield.

A helper in `std.controller` simplifies creation of periodic tasks:

```drift
std.controller.every(
    PeriodicTaskConfig(period_ticks = 10, priority = Priority.High),
    fn(now: Tick) returns Void {
        run_loop(now)
    }
)
```

---

## Example control loop

```drift
fn run_loop(now: Tick,
            sensor: AnalogInput,
            pump: DigitalOutput) returns Void {

	static var smoothed = 0
	val raw = sensor.read()
	smoothed = (smoothed * 3 + raw) / 4

	if smoothed < 2000 {
		pump.set(true)
	} else {
		pump.set(false)
	}
}
```

Predictable. Small. Hard to misuse. Ideal for embedded settings.

---

## Safety model

* No raw pointers exposed to user code.
* All unsafe operations sealed inside HAL.
* Exceptions capture full context for on-device debugging.
* Panic enters **safe state**:

  * disable outputs
  * flush logs
  * optional auto-reboot

This model balances safety, transparency, and embedded practicality.

---

## Roadmap

### Phase 1 — Foundation

* [ ] C/ASM bootstrap
* [ ] Drift runtime initialization
* [ ] Timer + tick source
* [ ] Virtual-thread executor
* [ ] Priority scheduler

### Phase 2 — Hardware Abstraction

* [ ] GPIO read/write
* [ ] ADC channels
* [ ] PWM output
* [ ] UART logging

### Phase 3 — Drift Integration

* [ ] std.controller periodic task helper
* [ ] Structured panic handler
* [ ] DMP module loader + signature check

### Phase 4 — Demonstrations

* [ ] Water pump controller demo
* [ ] Multi-rate sensor aggregation demo
* [ ] GPIO blinking/status tasks
* [ ] Example signed update workflow

### Phase 5 — Optional Expansion

* [ ] Message channels between tasks
* [ ] Config storage in flash
* [ ] Hot-swappable modules
* [ ] Jitter measurement utilities

---

## Status

Early design stage.
Compiler and runtime integration still evolving.
The project will track Drift language features as they solidify.

---

## Related documents

* **drift-controller-os-overview.md** — high-level OS whitepaper
* (future) **chapter: controller runtime** — spec-level documentation

---
