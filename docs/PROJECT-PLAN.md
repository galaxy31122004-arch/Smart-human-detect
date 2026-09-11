# Smart Human Detect — Project Plan

## 1. Project objective

Build a **firmware-focused human-presence controller** using **Hi-Link LD2410 + optional PIR + MCU**.

The main goal is to develop and validate the **logic/code** that decides when an output should turn ON or OFF based on human presence.

Core behavior:

> Detect person → output ON → keep ON while person is present, including when stationary → when presence is lost, wait for configurable OFF delay → output OFF.

PIR is an **optional supporting sensor** and can be enabled or disabled by configuration.

The project intentionally keeps hardware complexity low. The main work is the **presence algorithm, state machine, timers, filtering, configuration, and reliable firmware behavior**.

## 2. System architecture

```text
                 ┌──────────────┐
                 │     PIR      │
                 │   OPTIONAL   │
                 └──────┬───────┘
                        │ GPIO
                        ▼
┌──────────────┐   ┌───────────────┐
│    LD2410    │──►│      MCU      │
│    UART      │   │               │
│              │   │ Presence Logic│
│ Moving       │   │ State Machine │
│ Static       │   │ Timer         │
│ Distance/Gate│   │ Configuration │
└──────────────┘   └───────┬───────┘
                           │
                           ▼
                      OUTPUT CONTROL
```

The MCU is the center of the system. Sensors only provide input; the MCU makes the final decision.

## 3. Sensor logic

### LD2410 — primary sensor

LD2410 is the main source of human-presence information.

Use:

- valid target/presence information;
- moving target information;
- static target information;
- distance/gate information;
- configurable thresholds and filtering.

Important rule:

> **No movement does not mean no person.**

A person can remain stationary while the output must stay ON.

### PIR — optional sensor

PIR is only a **supporting motion/event sensor**.

It can be enabled or disabled:

```text
PIR_ENABLE = 0 → ignore PIR
PIR_ENABLE = 1 → use PIR as supporting input
```

PIR can help with:

- detecting a movement event quickly;
- supporting initial detection;
- providing an independent motion indication.

PIR must not be the main source for determining that a person has left.

Important rule:

> **PIR = 0 must never directly mean “no person”.**

## 4. Main firmware flow

```text
START
  │
  ├─ Initialize MCU
  ├─ Initialize LD2410 UART
  ├─ Initialize PIR GPIO if enabled
  ├─ Initialize timer/tick
  └─ Load configuration
  │
  ▼
MAIN LOOP / TASK
  │
  ├─ Receive LD2410 data
  ├─ Parse and validate frame
  ├─ Read PIR if enabled
  ├─ Evaluate presence
  ├─ Run state machine
  ├─ Update timers
  └─ Update output
  │
  └──────────────► repeat
```

The implementation should be deterministic and easy to debug.

## 5. Presence decision logic

The presence engine converts sensor data into one logical result:

```text
presence = TRUE / FALSE
```

### Presence = TRUE

Normally when LD2410 reports a valid target according to the configured criteria.

If `PIR_ENABLE = 1`, a valid PIR event may support the detection, but the final presence decision should still be controlled by the presence logic rather than treating PIR as a standalone presence sensor.

### Presence = FALSE

Only after LD2410 has indicated that valid presence is absent for the configured **loss-confirmation period**.

Do not use these rules:

```text
PIR = 0              → NO PERSON   ❌
LD2410 no movement   → NO PERSON   ❌
No single frame      → NO PERSON   ❌
```

Instead, use validated LD2410 information plus timing/filtering.

## 6. State machine

Use four primary states:

```text
                    valid presence
NO_PERSON ─────────────────────────► DETECTING
   ▲                                    │
   │                              confirmed
   │                                    ▼
   │                                  ACTIVE
   │                                    │
   │                              presence lost
   │                                    ▼
   └──────── timeout ◄──────────── WAIT_OFF
                                      │
                               presence returns
                                      │
                                      └────► ACTIVE
```

### `NO_PERSON`

- Output = OFF.
- Wait for valid presence.

### `DETECTING`

- Confirm that detection is valid.
- Apply optional ON delay/debounce.
- Reject short/noisy detections if required.

### `ACTIVE`

- Output = ON.
- Continue monitoring LD2410.
- A stationary person must keep the system active when LD2410 still reports valid presence.
- Any valid presence cancels/resets the OFF timer.

### `WAIT_OFF`

- Output remains ON.
- Start the configurable `OFF_DELAY` timer.
- If presence returns before timeout → cancel timer → `ACTIVE`.
- If timeout expires with no valid presence → output OFF → `NO_PERSON`.

## 7. Configurable parameters

The firmware should expose a small configuration structure:

| Parameter | Purpose |
|---|---|
| `ON_DELAY` | Confirmation time before turning output ON |
| `OFF_DELAY` | Time to wait after presence is lost |
| `LD2410_MAX_DISTANCE` | Maximum useful detection distance |
| `MOVING_THRESHOLD` | Moving-target threshold |
| `STATIC_THRESHOLD` | Static-target threshold |
| `LD2410_GATE_CONFIG` | Distance-gate configuration |
| `PIR_ENABLE` | Enable/disable PIR support |
| `PRESENCE_LOSS_CONFIRM` | Time used to confirm presence loss |

Example development values only:

```text
ON_DELAY             = 0.5 s
OFF_DELAY            = 30 s
PRESENCE_LOSS_CONFIRM = 1 s
PIR_ENABLE           = 1
```

These values are not final and must be adjusted during testing.

## 8. Core timing logic

Example: `OFF_DELAY = 30 s`.

```text
LD2410 detects person
        │
        ▼
      ACTIVE
        │
        │ valid presence continues
        │ → OFF timer remains cancelled/reset
        │
        ▼
   presence lost
        │
        ▼
    WAIT_OFF
        │
        ├── presence returns ──► ACTIVE
        │
        └── 30 s expires ──────► NO_PERSON / OUTPUT OFF
```

The important behavior is:

```text
valid presence detected
        → cancel OFF timer

presence lost
        → start OFF timer

presence returns before timeout
        → cancel OFF timer
        → keep output ON

OFF timer expires
        → output OFF
```

## 9. Firmware architecture

The project should focus primarily on clean code organization.

```text
firmware/
├── main
├── ld2410_driver
├── pir_driver          # optional
├── presence_engine
├── state_machine
├── timer_manager
├── output_controller
└── config_manager
```

### Module responsibilities

- `ld2410_driver`: UART reception, frame parsing, data validation, LD2410 configuration.
- `pir_driver`: optional PIR GPIO/event handling.
- `presence_engine`: convert sensor data into logical presence.
- `state_machine`: control `NO_PERSON`, `DETECTING`, `ACTIVE`, `WAIT_OFF`.
- `timer_manager`: ON delay, OFF delay, loss-confirmation timer, periodic tick.
- `output_controller`: set output ON/OFF and maintain safe startup behavior.
- `config_manager`: store and validate user parameters.
- `main`: initialization and high-level execution flow.

Avoid putting the entire algorithm inside `main()`.

## 10. Development phases — firmware first

### Phase 1 — Define logic

- [ ] Define presence criteria.
- [ ] Define `NO_PERSON`, `DETECTING`, `ACTIVE`, `WAIT_OFF`.
- [ ] Define ON/OFF timing behavior.
- [ ] Define PIR optional behavior.
- [ ] Define configuration structure.

### Phase 2 — LD2410 driver

- [ ] Implement UART receive.
- [ ] Implement frame synchronization/parsing.
- [ ] Extract moving/static/distance information.
- [ ] Validate frames.
- [ ] Handle communication/data errors.

### Phase 3 — Presence engine

- [ ] Convert LD2410 data to `presence = TRUE/FALSE`.
- [ ] Add filtering and confirmation time.
- [ ] Ensure stationary presence remains valid.
- [ ] Add optional PIR support.
- [ ] Verify `PIR_ENABLE = 0` completely disables PIR influence.

### Phase 4 — State machine + timer

- [ ] Implement all four states.
- [ ] Implement `ON_DELAY`.
- [ ] Implement `OFF_DELAY`.
- [ ] Implement presence-loss confirmation.
- [ ] Reset/cancel OFF timer when presence returns.
- [ ] Prevent rapid ON/OFF oscillation.

### Phase 5 — Configuration

- [ ] Define configuration structure.
- [ ] Validate parameter ranges.
- [ ] Add runtime configuration if required.
- [ ] Add non-volatile storage if required.

### Phase 6 — Output control

- [ ] Implement simple output abstraction.
- [ ] Keep startup output safe.
- [ ] Ensure state-machine decisions are correctly reflected at the output.

### Phase 7 — Logic validation

Test the firmware logic with sensor input scenarios:

1. No person.
2. Person appears.
3. Person moves.
4. Person stays stationary.
5. Person leaves.
6. Person returns before `OFF_DELAY` expires.
7. Person returns after timeout.
8. PIR enabled.
9. PIR disabled.
10. PIR inactive while LD2410 still detects a stationary person.
11. Invalid/noisy LD2410 frame.
12. Temporary LD2410 communication loss.

## 11. Success criteria

The firmware is successful when:

- LD2410 detection turns the output ON reliably.
- Stationary presence does not incorrectly turn the output OFF.
- Presence loss starts the OFF timer only after confirmation.
- Returning before timeout cancels the OFF timer.
- Output turns OFF only after the configured delay.
- `PIR_ENABLE = 0` makes PIR irrelevant to the decision.
- PIR cannot independently force the system OFF.
- Invalid/noisy data does not cause rapid state changes.
- The state machine can be tested independently from the physical output hardware.

## 12. Future extensions

Only add complexity when testing shows a need.

Possible extensions:

- runtime configuration interface;
- multiple outputs;
- configurable LD2410 distance zones;
- event logging;
- diagnostic/debug mode;
- additional sensors if the LD2410 alone proves insufficient.

## 13. Design principle

> **LD2410 provides the primary presence evidence. PIR is optional supporting information. The presence engine decides presence. The state machine decides behavior. The timer decides when OFF is allowed. The output controller applies the final command.**

The project should prioritize **clear, deterministic, testable firmware logic** over unnecessary hardware complexity.