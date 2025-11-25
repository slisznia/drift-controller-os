# **drift-controller-os-overview.md**

## Drift Controller OS Overview

### Purpose

Drift Controller OS is a lightweight, soft real-time control runtime designed to demonstrate Drift’s capabilities on bare-metal or microcontroller-class hardware. It is not a general-purpose kernel. Instead, it provides a deterministic execution environment for periodic sensor and GPIO tasks, proving that Drift can function as a safe systems language without raw pointers or an underlying OS.

### Goals

* Run Drift code directly on hardware with minimal C/assembly bootstrap.
* Provide soft real-time scheduling for sensor reading, actuator control, and GPIO tasks.
* Showcase Drift’s language features: interfaces, virtual threads, plugins, signed modules, and structured error handling.
* Maintain predictable timing without complex kernel mechanisms.
* Support safe field updates using signed DMP controller modules.

---

## Architecture Overview

### **Layer 1: Bootstrap (C/ASM)**

* Initializes CPU mode and minimal hardware.
* Sets up a bump allocator or static memory region.
* Exposes low-level functions to Drift via `extern "C"`.

### **Layer 2: Drift Runtime Core**

* Cooperative virtual-thread executor.
* Timer tick counter updated by hardware timer interrupt.
* Priority-driven soft real-time scheduler.
* Panic handler with structured `Error` context.
* Safe abstractions for memory, timing, and tasking.

### **Layer 3: Hardware Abstraction Layer**

Defines strongly typed interfaces:

```drift
interface AnalogInput {
	fn read() returns Int32
}

interface DigitalInput {
	fn read() returns Bool
}

interface DigitalOutput {
	fn set(on: Bool) returns Void
}

interface PwmOutput {
	fn set_duty(percent: Float64) returns Void
}
```

Concrete implementations call into the unsafe C shim but remain exposed as safe, typed Drift APIs.

### **Layer 4: Controller Modules (DMP-signed)**

* Drift modules implementing actual controller logic.
* Signature-verified at boot.
* Replaceable/upgradeable in the field.
* Encapsulate:

  * sensor aggregation
  * control rules
  * GPIO actuation
  * logging/diagnostics

---

## Soft Real-Time Execution Model

### **Tick-Based Time Base**

* Hardware timer ISR increments a global tick counter (e.g., 1 ms).
* All scheduling and control loops derive from this monotonic clock.

### **Cooperative Scheduler**

* Static priority per task.
* Tasks must `yield()` or `sleep_until(tick)` to relinquish control.
* Deterministic execution if tasks remain short and predictable.

### **Scheduling Example**

* High-priority tasks: 1–10 ms periods (sensors, closed-loop control).
* Medium: 50–200 ms (status, filtering, slow IO).
* Low: >500 ms (logging, telemetry, housekeeping).

---

## Programming Model

### **Periodic Task Helper**

Drift API for specifying soft real-time loops:

```drift
std.controller.every(
	PeriodicTaskConfig(period_ticks = 10, priority = Priority.High),
	fn(now: Tick) returns Void {
		run_loop(now)
	}
)
```

### **Typical Sensor + GPIO Loop**

```drift
fn run_loop(now: Tick,
            sensor: AnalogInput,
            pump: DigitalOutput) returns Void {

	static var last = 0

	val raw = sensor.read()
	last = (last * 3 + raw) / 4   // low-pass filter

	if last < 2000 {
		pump.set(true)
	} else {
		pump.set(false)
	}
}
```

Features:

* No dynamic allocations.
* No blocking operations.
* Deterministic execution time.

---

## Safety Model

* **No raw pointers** in Drift code.
* All unsafe access hidden behind sealed `@unsafe` implementations.
* **Exceptions include full context**, making debugging on-device far easier.
* **Panic safe-state:**

  * Turn off outputs.
  * Log via serial/UART.
  * Optional reboot.

---

## Update / Deployment Workflow

1. Controller logic compiled into a signed DMP module.
2. MiniOS verifies signature on boot.
3. If invalid, fallback to last-known-good module.
4. Enables safe field updates for embedded control systems.

---

## Future Expansion

* Inter-task message channels.
* EEPROM/flash-backed configuration variables.
* Multi-rate loops with jitter measurement.
* Scoped watchdog integration.
* Optional hot-swapping of controller modules.

---

## Summary

Drift Controller OS is a focused, real-world demonstration that Drift can serve as a safe, expressive systems programming language for embedded control. It proves:

* determinism
* modularity
* safe hardware access
* field-updatable logic
* robust error introspection
* and operation without an underlying OS

---
