# Smart Human Detect — Project Plan

## 1. Project objective

Build a configurable human-presence controller using **Hi-Link LD2410 + PIR + MCU** to automatically control an external switch/load.

Core behavior:

> Detect human presence → turn output ON → keep output ON while presence remains → when presence is lost, wait for a user-configured OFF delay → turn output OFF.

The project focuses on **firmware/algorithm design**, especially sensor fusion, presence validation, state management, and configurable timing.

## 2. System architecture

```text
                  ┌──────────────┐
                  │     PIR      │
                  │  Movement    │
                  └──────┬───────┘
                         │ GPIO
                         ▼
┌──────────────┐   ┌───────────────┐
│    LD2410    │──►│      MCU      │
│              │UART│               │
│ Presence     │   │ Sensor Fusion │
│ Moving       │   │ Validation    │
│ Static       │   │ State Machine │
│ Distance/Gate│   │ Timer         │
└──────────────┘   └───────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │ Output Driver│
                    │ Relay/MOSFET │
                    └──────┬───────┘
                           │
                           ▼
                         LOAD
```

## 3. Sensor responsibilities

### LD2410 — primary presence sensor

Use LD2410 as the main source for human-presence information. Its moving/static target information and distance/gate data are used by the MCU to determine whether a valid presence remains.

Important principle:

- LD2410 reporting no movement does **not** automatically mean no person.
- A stationary person can still be considered present.
- LD2410 data must be filtered/validated before changing the output state.

### PIR — motion/event support sensor

Use PIR as a complementary motion detector.

PIR is useful for:

- detecting a new movement event quickly;
- supporting initial presence confirmation;
- providing an independent motion signal.

Important principle:

> PIR = 0 does not mean that no person is present.

Therefore PIR must not be used alone as the OFF condition.

## 4. Main firmware flow

```text
START
  │
  ├─ Initialize MCU peripherals
  ├─ Initialize LD2410 UART
  ├─ Initialize PIR input
  ├─ Initialize timer/tick
  └─ Load user configuration
  │
  ▼
MAIN LOOP
  │
  ├─ Receive and parse LD2410 frames
  ├─ Read PIR state/event
  ├─ Validate LD2410 data
  ├─ Evaluate presence
  ├─ Run state machine
  ├─ Update ON/OFF timers
  └─ Control output
  │
  └──────────────► repeat
```

## 5. Presence decision logic

The first version should keep the logic deterministic and easy to debug.

### Presence = TRUE when

- LD2410 reports a valid target/presence according to configured detection criteria; or
- a valid PIR event is used as a supporting trigger and LD2410 confirms presence.

### Presence = FALSE when

- LD2410 has no valid presence for the configured loss-confirmation period.

PIR alone must not force Presence = FALSE.

## 6. State machine

Use four primary states:

```text
             valid presence
NO_PERSON ───────────────────► DETECTING
   ▲                              │
   │                         confirmation OK
   │                              ▼
   │                           ACTIVE
   │                              │
   │                         presence lost
   │                              ▼
   └──── timeout ◄────────── WAIT_OFF
                                  │
                         presence returns
                                  │
                                  └──────► ACTIVE
```

### NO_PERSON

- Output = OFF
- Waiting for a valid presence event.

### DETECTING

- Confirm that the detection is valid.
- Optional ON delay/debounce can be applied.
- Avoid reacting to a single noisy sample.

### ACTIVE

- Output = ON.
- Continue monitoring LD2410.
- PIR may be inactive while a stationary person remains detected.
- Any valid presence resets/cancels the OFF timer.

### WAIT_OFF

- Output remains ON.
- Start the user-configured OFF timer.
- If presence returns before timeout: cancel timer and return to ACTIVE.
- If timeout expires with no valid presence: turn output OFF and return to NO_PERSON.

## 7. Configurable parameters

Initial configuration should include:

| Parameter | Purpose |
|---|---|
| `ON_DELAY` | Time used to confirm presence before turning output ON |
| `OFF_DELAY` | Time without valid presence before turning output OFF |
| `LD2410_MAX_DISTANCE` | Maximum useful detection distance |
| `LD2410_GATE_CONFIG` | LD2410 distance-gate configuration |
| `MOVING_THRESHOLD` | Moving-target sensitivity/threshold |
| `STATIC_THRESHOLD` | Static-target sensitivity/threshold |
| `PIR_ENABLE` | Enable/disable PIR support |

Default example for development only:

```text
ON_DELAY  = 0.5 s
OFF_DELAY = 30 s
```

Defaults are not final product values and must be validated experimentally.

## 8. Important timing behavior

Example with `OFF_DELAY = 30 s`:

```text
Person present
      │
      ▼
OUTPUT ON
      │
      │ person remains
      │
      ▼
OUTPUT stays ON
      │
      │ presence lost
      ▼
WAIT_OFF
      │
      ├── person returns ──► ACTIVE
      │
      └── 30 s expires ───► OUTPUT OFF
```

The timer should be restarted/reset whenever valid presence is detected.

## 9. Firmware architecture

Separate the firmware into modules rather than putting all logic in `main()`.

```text
firmware/
├── main
├── ld2410_driver
├── pir_driver
├── sensor_fusion
├── presence_engine
├── state_machine
├── timer_manager
├── output_controller
└── config_manager
```

Suggested responsibilities:

- `ld2410_driver`: UART receive, frame parser, LD2410 configuration.
- `pir_driver`: PIR GPIO/event handling.
- `sensor_fusion`: combine LD2410 and PIR information.
- `presence_engine`: validate presence and apply filtering.
- `state_machine`: manage NO_PERSON/DETECTING/ACTIVE/WAIT_OFF.
- `timer_manager`: ON delay, OFF delay, periodic tick.
- `output_controller`: relay/MOSFET/output state.
- `config_manager`: load/save user settings.

Interrupts should be kept short. UART/PIR/timer interrupts should capture events or data; the main loop/task should perform the heavier parsing and decision logic.

## 10. Development phases

### Phase 1 — Hardware bring-up

- [ ] Verify MCU power and I/O.
- [ ] Verify LD2410 UART communication.
- [ ] Verify PIR digital signal.
- [ ] Verify output driver.
- [ ] Confirm electrical levels and safe isolation where required.

### Phase 2 — LD2410 driver

- [ ] Implement UART reception.
- [ ] Implement frame synchronization/parsing.
- [ ] Extract moving/static presence and distance/gate information.
- [ ] Add invalid-frame handling.
- [ ] Log raw and parsed data during testing.

### Phase 3 — PIR driver

- [ ] Read PIR signal reliably.
- [ ] Add debounce/event handling if required by the selected PIR.
- [ ] Verify behavior for entering, moving, and leaving.

### Phase 4 — Presence engine

- [ ] Define valid LD2410 presence criteria.
- [ ] Add filtering/confirmation time.
- [ ] Combine PIR as supporting information.
- [ ] Ensure stationary people remain detected.

### Phase 5 — State machine + timer

- [ ] Implement NO_PERSON.
- [ ] Implement DETECTING.
- [ ] Implement ACTIVE.
- [ ] Implement WAIT_OFF.
- [ ] Implement configurable ON/OFF delays.
- [ ] Cancel/reset OFF timer when presence returns.

### Phase 6 — Configuration

- [ ] Define configuration structure.
- [ ] Add non-volatile storage.
- [ ] Add a configuration interface appropriate to the selected MCU.
- [ ] Validate parameter ranges.

### Phase 7 — Output control

- [ ] Implement relay/MOSFET control.
- [ ] Add startup-safe output state.
- [ ] Verify behavior during reset/reboot.
- [ ] Verify fail-safe behavior.

### Phase 8 — Real-world validation

Test at minimum:

1. No person.
2. Person enters.
3. Person remains moving.
4. Person remains stationary.
5. Person leaves.
6. Person returns before OFF timeout.
7. False PIR trigger without a valid person.
8. LD2410 temporary detection loss.
9. Multiple environmental conditions.
10. MCU reset while output is ON.

## 11. Success criteria

The first stable version is considered successful when:

- A valid person reliably turns the output ON.
- A stationary person does not cause an unwanted OFF.
- Leaving the detection area starts the OFF timer.
- Returning before timeout cancels the OFF action.
- Output turns OFF only after the configured delay has expired.
- PIR loss alone never incorrectly turns the system OFF.
- Invalid/noisy sensor data does not cause rapid ON/OFF oscillation.
- Configuration survives reboot when non-volatile storage is implemented.

## 12. Future extensions

Possible future improvements:

- Multiple output channels.
- Multiple configurable distance zones using LD2410 gates.
- Ambient-light sensor for automatic lighting decisions.
- Local buttons/display or a PC/mobile configuration interface.
- Event logging.
- Low-power operating modes where practical.
- Additional filtering or confidence scoring after real-world test data is collected.

## 13. Design principle

Keep the project modular:

> **Sensors provide evidence. The MCU decides presence. The state machine decides behavior. The timer decides when an action is allowed. The output driver controls the external load.**

Do not add more sensors or complexity until testing demonstrates a real need.
