# FOC Motor Controller: Project Plan

Nov 24, 2026 · @Farhan Sadik

A sensorless FOC controller for the iFlight XING2 1404 on the ST B-G431B-ESC1, built in three phases. Follow the phases in order; Sections 7–9 are reference material to look things up in.

## 1. Roadmap

Three phases, done in order: prove the hardware on ST's firmware, replace it with my own FOC, then measure and validate it properly. Each phase ends with an exit gate; the next phase's hardware work does not start until the gate is passed.

| Phase | Goal | Main steps | Exit gate | Target dates |
| --- | --- | --- | --- | --- |
| 1. Baseline | Prove the ESC1, motor and power setup on ST's firmware; measure the motor | Setup → soldering → first power-up → Motor Profiler → Motor Pilot sweep → schematic study | Motor spins reliably; Rs, Ls, pole pairs, KV measured | Now – 31 Dec 2026 |
| 2. My own FOC | Replace ST's firmware with my own sensorless FOC | 2a theory + simulation → 2b C code + SIL tests in CI → 2c hardware bring-up | FR-1 to FR-5, PR-1 to PR-3, PR-5, SR-1 to SR-6 pass | Jan – May 2027 |
| 3. Bench + validation | Measure thrust and efficiency; full automated validation | Thrust bench → props → efficiency vs. ST → nightly HIL → write-up | PR-4 and all VR requirements pass; project written up | From Jun 2027 |

**What can overlap:** Phase 2a and 2b need only a PC, so they can start while Phase 1 parts are in delivery. Only Phase 2c (my own PWM on the ESC1) waits for the Phase 1 gate and a scope.

**Running alongside all phases:** Plett's BMS courses (evenings), and the job search from about mid-2027 using this project as the portfolio.

## 2. System

The ESC1 runs the motor; everything else either powers it, commands it or measures it.

&#91;embedded content: Bench setup · solid = Phase 1, dashed = joins in Phase 2\]

Power flows left to right through the fuse into the ESC1; the PC programs and commands it over one USB cable. The two Nucleo boards join in Phase 2.

### 2.1 Hardware

| Item | Key specs | Role | Status |
| --- | --- | --- | --- |
| ST B-G431B-ESC1 | STM32G431CB (Cortex-M4F, 170 MHz, FPU, CORDIC), 3× L6387 gate drivers, 6× STL180N6F7 60 V MOSFETs, 3-shunt current sensing, on-chip op-amps and comparators, TCAN330 CAN transceiver on J1, 3S–6S input, 40 A peak with prop airflow, 5 V 0.5 A BEC, detachable ST-LINK/V2-1 with USB and potentiometer | Motor controller: ST firmware in Phase 1, my firmware from Phase 2 | Have |
| iFlight XING2 1404 4600KV | 1 pc, no plug, rated 2–4S | Motor under test | Have |
| Industrial power supply | Adjustable voltage and current limit (confirm true CC/CV) | Bus supply | Have |
| NUCLEO-G474RE | STM32G474RE, FDCAN, all pins broken out | Phase 2: PIL target, CAN partner, bench controller | Have |
| STM32F411RE Nucleo | Cortex-M4F 100 MHz, no CAN | Phase 2: INA228 logger | Have |
| CAN transceiver module | Model to confirm | Phase 2: CAN bus for the Nucleo | Have |
| 8898 soldering station | 750 W, iron + hot air, 220 V EU plug | Soldering | Have (needs earthed EU-to-UK adapter) |
| 22 AWG wire, wire stripper | Wire good for ≤ 2 A | Phase 1 wiring | Have |

**Connections:** supply → fuse → ESC1 battery input (J5/J6) → motor (J7). PC → USB → ST-LINK → SWD for programming and a virtual COM port to the G431's UART. Spare on the ESC1: J1 CAN, J3 PWM input + UART + 5 V, J8 Hall/encoder, J4 SWD pads. The ESC1's logic can run from USB or the CAN 5 V line with the bus supply off.

### 2.2 Who does what

The motor-control algorithm always runs as C on the ESC1. Python on the PC only supervises tests; USB delay and jitter are far too large for a current loop that runs every 25–50 µs.

| Layer | Runs on | Rate | Job |
| --- | --- | --- | --- |
| Current loop (FOC) | ESC1, C in the ADC interrupt | Every 25–50 µs | Clarke/Park, PI current control, SVPWM |
| Speed loop, state machine, protections | ESC1, C | About every 1 ms | Speed PI, arm/disarm, faults, watchdog |
| Test supervision | PC, Python over USB/UART or CAN | 10 ms to seconds | Setpoints, telemetry, sweeps, logs, pass/fail |

In Phase 1, ST's Motor Pilot is the test supervisor. Python meets my algorithm only off the bench, in SIL and PIL tests against a simulated motor.

## 3. Safety (every phase)

The supply's current limit is the main protection: set it before every session, and never power the ESC1 from a LiPo until my own firmware has passed SR-1 to SR-6.

### 3.1 Bench rules

- Current limit set before switching on: 0.5 A for a first power-up, 1–2 A for profiling and spinning.
- Inline fuse (3–5 A) in the positive supply lead at all times.
- No propeller until Phase 3, and then only behind a shield.
- Motor screwed to a mount (M2 holes), mount clamped to the bench.
- Safety glasses on whenever the motor can spin.
- One hand near the supply's output switch when trying new settings.
- Soldering station on an earthed, fused EU-to-UK adapter.
- Stop and investigate if the supply hits its limit unexpectedly, anything smells hot, or the motor sounds rough.
- My own PWM never runs on the ESC1 until dead time has been checked on a scope or logic analyser.

### 3.2 First power-up procedure

Use this for the first power-up of any new hardware or new firmware.

1. **Inspect:** solder joints for bridges; supply lead polarity with the multimeter; no short between bus + and − on the ESC1.
2. **Set the supply:** 12 V, 0.5 A limit, output off, ESC1 disconnected. Check 12 V at the lead ends.
3. **USB only:** connect the ESC1 by USB with bus power off. Confirm STM32CubeProgrammer connects.
4. **Bus power on:** idle current should be tens of mA. If the supply hits its limit immediately, switch off and go back to step 1.
5. **Raise the limit to 1–2 A** for the test.
6. **Shut down in order:** stop the motor, output off, then unplug USB.

### 3.3 Hazards to plan for

- **Regeneration:** the supply cannot absorb current, so hard braking pushes bus voltage up. Add a bulk low-ESR capacitor at the ESC1 input before prop tests.
- **Scope grounding:** a bench scope's ground clip is tied to mains earth. Clip it only to ESC1 ground; use two probes and A − B math for anything else.
- **Wire gauge:** 22 AWG is for ≤ 2 A. Use 16–18 AWG once props go on.
- **Voltage:** the ESC1 takes 3S–6S but the motor is rated 2–4S; the motor sets the limit.

## 4. Phase 1: Baseline on ST firmware (now – 31 Dec 2026)

Phase 1 proves the hardware with ST's known-good firmware and measures the motor, so that later problems can be traced to my own code. No code of mine drives the motor in this phase.

**Out of scope:** my own FOC firmware, propellers, thrust measurement, CAN.

### Step 1 — Review the requirements (week 1, no hardware)

- [x] Read Section 7; add, cut or reword requirements
- [x] Note which measurement will fill each TBD

### Step 2 — Toolchain and repo (week 1, no hardware)

- [x] Install STM32CubeIDE, STM32CubeMX, STM32CubeProgrammer
- [x] Install the ST Motor Control SDK (Workbench, Motor Profiler, Motor Pilot)
- [x] Python with pyserial, numpy, pandas, matplotlib, pytest
- [x] Git repo: `docs/`, `firmware/`, `tools/`, `data/`, `hardware/`
- [ ] Download UM2516, the ESC1 schematic and the STM32G4 reference manual into `docs/`

### Step 3 — Order the missing Phase 1 parts (week 1)

- [ ] Everything in Section 8.1 not yet owned

### Step 4 — Study the ESC1 schematic (weeks 1–4, no hardware)

- [ ] Current-sense path per phase: shunt → op-amp gain → ADC channel
- [ ] Overcurrent path: comparator → TIM1 break input; note the threshold
- [ ] TIM1 channels → gate-driver inputs; dead-time handling in the L6387
- [ ] Bus-voltage divider and temperature-sensor channels
- [ ] Whether J1 has a CAN termination resistor fitted
- [ ] Fill table 4.2; sketch the current-sense and overcurrent path from memory

### Step 5 — Prepare the hardware (when parts arrive)

- [ ] Earthed EU-to-UK adapter for the soldering station; test the station
- [ ] Motor mount: scrap block, M2 screws, clamp
- [ ] Solder the three motor leads to the ESC1 phase pads; heat-shrink
- [ ] Supply lead: 22 AWG, inline fuse, polarity marked
- [ ] Photograph the wiring into `hardware/`

### Step 6 — First power-up

- [ ] Follow Section 3.2 steps 1–5
- [ ] Record the idle current (P1-5)

### Step 7 — Measure the motor

- [ ] Run Motor Profiler (board B-G431B-ESC1, test current below the supply limit, no prop); repeat 2–3 times and note the spread
- [ ] Four-wire Rs check: supply in current limit at 0.5–1 A through two leads, multimeter on the leads; Rs = (V / I) ÷ 2. Keep it short so the winding stays cool
- [ ] Pole pairs: from the motor's slot/magnet spec (magnets ÷ 2) or by counting magnets
- [ ] Note which units Motor Profiler uses for Ke
- [ ] Fill table 4.1

### Step 8 — Characterize on ST firmware

- [ ] Generate an MCSDK project with the measured parameters; flash it
- [ ] Start/stop 10 times (P1-1)
- [ ] Sweep from minimum to maximum speed in at least 6 steps; fill table 4.3 (P1-2, P1-4)
- [ ] Record the MCSDK settings (PWM frequency, current-loop gains, observer) as a reference for Phase 2

### Step 9 — Phase 1 review

- [ ] Tick the exit criteria below; update TBDs in Section 7
- [ ] Short results note with photos
- [ ] Scope access arranged and logic analyser in hand for Phase 2c

### Exit criteria

- [ ] Motor spins reliably on ST firmware from minimum to maximum speed at 12 V, no prop
- [ ] Rs, Ls, pole pairs and KV recorded in table 4.1, Rs cross-checked by the four-wire method
- [ ] No-load sweep recorded (≥ 6 points)
- [ ] ESC1 current-sense and overcurrent path understood and sketched
- [ ] No hardware damaged

### 4.1 Motor parameters

| Parameter | Symbol | Unit | How measured | Value | Notes |
| --- | --- | --- | --- | --- | --- |
| Phase resistance | Rs | mΩ | Motor Profiler + four-wire check |  |  |
| Phase inductance | Ls (Ld, Lq) | µH | Motor Profiler (LCR meter or scope step later) |  | Expect very low for a 1404 |
| Pole pairs | p | — | Spec sheet or magnet count ÷ 2; Motor Profiler |  | Must be exact |
| Velocity constant | KV | rpm/V | Motor Profiler; compare with 4600 |  |  |
| Back-EMF constant | Ke | V/krpm | Motor Profiler (scope check in Phase 2) |  | Note the unit convention |
| No-load bus current | I0 | A | Supply display at a fixed speed |  | Record speed and voltage |
| Electrical time constant | τe = Ls / Rs | µs | Calculated |  | Short τe means large ripple |
| Max electrical frequency | fe,max | Hz | Max speed × p ÷ 60 |  | PWM well above this |

### 4.2 Board parameters

| Parameter | Unit | How measured | Value |
| --- | --- | --- | --- |
| Current-sense ADC offsets (3 phases) | counts | Read with PWM off |  |
| Current-sense gain | A per count | UM2516 shunt value and op-amp gain |  |
| Bus-voltage divider ratio | V per count | Schematic, checked against the supply |  |
| Overcurrent comparator threshold | A | Schematic / MCSDK config |  |
| Idle current, motor stopped | mA | Supply display |  |

### 4.3 No-load sweep log (ST firmware, 12 V, no prop)

| Speed setpoint (rpm) | Bus voltage (V) | Bus current (A) | Motor temp after 1 min | Notes |
| --- | --- | --- | --- | --- |
|  |  |  |  |  |
|  |  |  |  |  |
|  |  |  |  |  |
|  |  |  |  |  |
|  |  |  |  |  |
|  |  |  |  |  |

## 5. Phase 2: My own FOC (Jan – May 2027)

Phase 2 replaces ST's firmware with my own sensorless FOC in three stages: design it in simulation, write and test the C on the PC, then bring it up on the ESC1. Stages 2a and 2b need only a PC and can start now; 2c waits for the Phase 1 gate and a scope.

**How to learn:** one concept at a time, built into the simulation straight away. I write the code first and get it reviewed; I predict each result before running it. Check equations against one fixed reference (my FOC course, Microchip AN1078 / AN1292, or Hughes' *Electric Motors and Drives*).

### 2a — Theory and simulation (Python, can start now)

| Step | Learn | Build in the simulation | Done when |
| --- | --- | --- | --- |
| 0a | What a PMSM is: rotor magnets, three stator phases, torque from the 90° field angle | — | I can explain why FOC holds the stator field at 90° to the rotor |
| 0b | Each phase as R + L + back-EMF | abc motor model, spun up open-loop (`pmsm_abc_openloop.py`) | Rotor follows the drive; ia + ib + ic ≈ 0 |
| 0c | Clarke and Park transforms | Transform the logged currents; plot id and iq | Sinusoids become flat lines at steady speed |
| 1 | The dq motor model | dq version of the model | Matches the abc model's speed and torque |
| 2 | SVPWM | Open-loop spin through transforms + SVPWM | Phase voltages look right |
| 3 | PI current control in dq; tuning Kp = Ls·ωc, Ki = Rs·ωc | id/iq loops with a known angle | iq step settles < 1 ms, id ≈ 0 (PR-2) |
| 4 | Cascaded speed control | Speed PI around the current loops | Speed steps track cleanly (PR-5) |
| 5 | Sensorless: back-EMF observer (sliding-mode or flux) + PLL | Replace the known angle with the estimate | Estimated angle tracks the true one above a minimum speed |
| 6 | Startup: align → I/f ramp → handoff | Full startup state machine | Reliable start from standstill in simulation (PR-3) |

After Phase 1, replace the placeholder parameters with the measured ones and re-run every step. Step 5 is the hardest; budget the most time for it.

### 2b — C code and software-in-the-loop (PC, from step 3 of 2a)

- [ ] **Test framework skeleton:** `tools/` with `sim/`, `sil/`, `bench/`, `can/`, `reports/`; pytest config, shared fixtures, JUnit XML + HTML reports (VR-1, VR-2)
- [ ] **Traceability:** tag every test with its requirement, e.g. `@pytest.mark.req("PR-2")`; script writes the requirement → test → result matrix and flags gaps (VR-1)
- [ ] **Port each proven block to C** as hardware-free functions (`foc_step()`, `observer_update()`), with unit tests
- [ ] **SIL harness:** C built as a shared library, called from Python against the motor model; results compared with the Python reference
- [ ] **CI:** GitHub Actions runs the unit and SIL tests on every push and publishes the reports (VR-2)
- [ ] **PIL on the NUCLEO-G474RE:** same C on the real chip, stepping in lockstep with the Python model over UART; measure cycles per step to check PR-1 early

### 2c — Hardware bring-up on the ESC1 (after the Phase 1 gate)

Gate: Phase 1 exit criteria met, scope available, logic analyser in hand.

1. **PWM and ADC drivers:** TIM1 centre-aligned complementary PWM with dead time; ADC injected conversions triggered at the PWM centre. Check gate signals and dead time on the scope **with bus power off**.
2. **Switching check with bus power:** switch node + low-side gate on the scope; no shoot-through.
3. **Current-sense offset calibration** with PWM off; check op-amp outputs on the scope.
4. **Open-loop voltage spin** at a low current limit; check phase order and current shape.
5. **Current loop** with a forced angle; tune on the scope at the op-amp outputs (PR-2).
6. **Observer and startup** (FR-1, PR-3), then **speed loop** (PR-5).
7. **Protections:** overcurrent trip, command loss, voltage limits, overtemperature, stall, watchdog (SR-1 to SR-6), each with an automated fault-injection test (VR-6).
8. **UART command protocol** and Python test supervision: commands, telemetry, state machine (IR-1, FR-2 to FR-4).
9. **Self-commissioning:** firmware measures Rs, Ls and λ; compared with Motor Profiler and the four-wire Rs (FR-5).
10. **Bus power logging:** INA228 on the F411, streamed to the PC (BR-2).
11. **CAN:** message set in a DBC file; Python restbus script plays the flight controller; timing, negative and bus-off tests (IR-3, VR-3 to VR-5).
12. **Bench controller on the NUCLEO-G474RE** under FreeRTOS: task periods, jitter, stack high-water marks (VR-8).

### Exit criteria

- [ ] FR-1 to FR-5 pass
- [ ] PR-1, PR-2, PR-3 and PR-5 pass
- [ ] SR-1 to SR-6 pass, each by an automated test
- [ ] IR-1 and IR-3 pass
- [ ] SIL suite green in CI; traceability matrix shows no untested Phase 2 requirement

## 6. Phase 3: Thrust bench and full validation (from Jun 2027)

Phase 3 measures what my controller achieves under real load and makes the whole test suite run unattended every night. Its output is the portfolio: benchmark results against ST's firmware and a validation record traced to every requirement.

### Steps

1. **Build the thrust stand:** rigid motor mount bolted to a 1–2 kg load cell with an HX711; polycarbonate shield; 16–18 AWG wiring; bulk capacitor at the ESC1 input. Calibrate the load cell with known weights (BR-1).
2. **First prop runs:** 3020 prop, 3S, ST firmware first, current limit raised in steps. Check bus voltage during deceleration on the scope.
3. **Automated sweeps:** the bench controller steps the throttle, logs thrust, bus V/I and ESC1 telemetry with timestamps (BR-3).
4. **Benchmark:** the same sweep on ST's firmware and on mine; compare g/W and thrust curves (PR-4).
5. **Nightly HIL:** a PC wired permanently to the bench runs a self-hosted GitHub Actions runner; each night it flashes the latest firmware, runs the bench suite and archives results. The suite aborts and powers down on any unexpected reading (VR-7).
6. **Complete the fault-injection suite** under prop load (VR-6).
7. **Optional: UDS diagnostics over CAN** — ISO-TP plus read data, read fault codes and clear fault codes, tested with udsoncan (VR-9).
8. **Write-up:** README, plots, scope captures, traceability matrix, safety notes, what I would do differently.

### Exit criteria

- [ ] BR-1 and BR-3 pass; PR-4 measured and reported
- [ ] Nightly HIL has run for at least two weeks without a false failure
- [ ] Traceability matrix shows every requirement tested
- [ ] Write-up published in the repo

### After Phase 3

Options, decided at the time: a second ESC1 and motor as spares, my own ESC PCB, the drone build (sub-250 g or endurance), and the BMS track on the same bench.

## 7. Requirements (reference)

Every requirement has an ID, a measurable target, a verification method and the phase where it is first verified; the phase sections above point here by ID. TBDs are filled from Phase 1 measurements.

**Prefixes:** P1 Phase 1 baseline · FR functional · PR performance · SR safety · IR interface · BR test bench · VR validation.

### 7.1 Phase 1 baseline (ST firmware)

| ID | Requirement | Target | Verification | Phase |
| --- | --- | --- | --- | --- |
| P1-1 | Starts on ST sensorless FOC, no prop | 10 of 10 starts | Motor Pilot start/stop | 1 |
| P1-2 | Runs across its speed range at 12 V | Min to max, no desync | Motor Pilot sweep | 1 |
| P1-3 | Motor parameters measured | Rs, Ls, p, KV recorded | Motor Profiler + four-wire Rs | 1 |
| P1-4 | No-load bus current vs. speed | ≥ 6 points | Supply display, table 4.3 | 1 |
| P1-5 | Idle current, motor stopped | TBD mA | Supply display | 1 |

### 7.2 FOC firmware

| ID | Requirement | Target | Verification | Phase |
| --- | --- | --- | --- | --- |
| FR-1 | Spin sensorless from standstill to max speed | No prop, then 3020 prop | Bench run | 2 |
| FR-2 | Accept a throttle/speed command | UART, then PWM and CAN | Scripted command sweep | 2 |
| FR-3 | Report telemetry | Speed, iq, bus V, temperature, faults | Logged stream | 2 |
| FR-4 | Arm/disarm state machine | IDLE → ALIGN → OPEN\_LOOP → CLOSED\_LOOP → FAULT | State trace | 2 |
| FR-5 | Self-commissioning: measure Rs, Ls, λ; check pole pairs | Within TBD % of reference values | Report vs. Motor Profiler and four-wire Rs | 2 |
| PR-1 | Current-loop rate and CPU load | ≥ 20 kHz; ISR ≤ 50% of period | GPIO toggle; PIL cycle count | 2 |
| PR-2 | Current-loop step response | Settles < 1 ms, no sustained oscillation | iq step, logged + scope | 2 |
| PR-3 | Startup reliability | Closed loop < 500 ms; ≥ 99 of 100 starts | Automated start/stop loop | 2 |
| PR-4 | Efficiency vs. ST firmware | Within TBD % g/W at matched throttle | Same sweep on both firmwares | 3 |
| PR-5 | Speed tracking | Steady-state error < TBD % | Speed step, logged | 2 |
| SR-1 | Hardware overcurrent cutoff | Trip at TBD A via comparator → TIM1 break | Fault injection under current limit | 2 |
| SR-2 | Command loss | Motor stops < 100 ms after last command | Disconnect command mid-run | 2 |
| SR-3 | Bus voltage limits | Shutdown below TBD V, above TBD V | Supply sweep | 2 |
| SR-4 | Overtemperature | Shutdown at TBD °C | Heat test | 2 |
| SR-5 | Stall / desync detection | Stopped within TBD ms | Hold rotor, no prop | 2 |
| SR-6 | Watchdog | Reset within TBD ms if the main loop stalls | Deliberate hang in a test build | 2 |

### 7.3 Interfaces

| ID | Requirement | Target | Verification | Phase |
| --- | --- | --- | --- | --- |
| IR-1 | UART command protocol over the ST-LINK virtual COM port | Documented frame format with CRC | Python round-trip test | 2 |
| IR-2 | PWM throttle input on J3 | 1000–2000 µs | Signal from a Nucleo timer | 2 |
| IR-3 | CAN on J1 | Message set in a DBC; 120 Ω at both ends | Frames exchanged with the NUCLEO-G474RE | 2 |

### 7.4 Test bench

| ID | Requirement | Target | Verification | Phase |
| --- | --- | --- | --- | --- |
| BR-1 | Thrust measurement | 0–1 kg, ≤ 1 g resolution, ≥ 10 Hz | Calibration weights | 3 |
| BR-2 | Input power measurement | V and I up to TBD A, ±1% | INA228 vs. a reference meter | 2 |
| BR-3 | Automated throttle sweep | Time-synced CSV log | Script output | 3 |
| BR-4 | Supply protection | Fuse fitted; limit set before every session | Inspection | 1 |
| BR-5 | Emergency stop | Motor power off in < 1 s by anyone | Test | 1 |
| BR-6 | Mechanical restraint | Motor bolted and clamped; shield before any prop | Inspection | 1, 3 |

### 7.5 Validation

| ID | Requirement | Target | Verification | Phase |
| --- | --- | --- | --- | --- |
| VR-1 | Traceability | Every requirement has ≥ 1 test; matrix generated automatically | Generated report in CI | 2 |
| VR-2 | SIL regression in CI | Runs on every commit; failure blocks merge | CI history | 2 |
| VR-3 | One source for the CAN message set | One DBC used by firmware and Python | Review + decode test | 2 |
| VR-4 | CAN timing | Period within ±TBD %; jitter < TBD ms | python-can capture | 2 |
| VR-5 | CAN robustness | Bad length, value and ID rejected; bus-off recovery < TBD ms | Restbus negative tests | 2 |
| VR-6 | Automated fault injection | SR-1 to SR-6 each automated with pass/fail | Bench suite | 2, 3 |
| VR-7 | Nightly bench HIL | Runs nightly on latest firmware; results archived | Self-hosted runner history | 3 |
| VR-8 | RTOS behaviour on the bench controller | Task periods within TBD; stack margins above TBD bytes | FreeRTOS run-time stats | 2 |
| VR-9 | Optional: UDS diagnostics over CAN | Read data, read and clear fault codes over ISO-TP | udsoncan tests | 3 |

### 7.6 Test layers

| Layer | Runs on | When |
| --- | --- | --- |
| Unit tests (C) | PC | Every commit |
| SIL | PC: my C against the Python motor model | Every commit |
| PIL | NUCLEO-G474RE + PC, lockstep over UART | Weekly or on demand |
| Bench HIL | ESC1 + motor + Nucleo + PC | Nightly (Phase 3) |
| Exploratory | Bench + scope | As needed |

## 8. Parts and tools by phase

Buy each item when its phase is about to start. Costs are rough estimates.

### 8.1 Phase 1

| Item | Why | Approx. cost | Status |
| --- | --- | --- | --- |
| Multimeter (if not owned) | Polarity, shorts, four-wire Rs | S$15–40 |  |
| Inline fuse holder + 3 A and 5 A fuses | Backup to the current limit | S$3–5 |  |
| Earthed, fused EU-to-UK adapter | Soldering station has an EU plug | S$5–15 |  |
| Safety glasses | Flying solder and parts | S$5 |  |
| Solder, flux, heat shrink | Check the 8898 kit first | S$5–15 |  |
| M2 screws + scrap block | Motor mount | \~S$0–5 |  |

### 8.2 Phase 2 (order during Phase 1)

| Item | Why | Approx. cost | Status |
| --- | --- | --- | --- |
| USB logic analyser (8 ch, 24 MHz) | PWM, dead time, ISR timing, UART/CAN decode | S$10–20 |  |
| Oscilloscope | Switch nodes, phase currents, current-loop tuning (needed before step 2c.2) | Borrow first; else Rigol DHO804 (US$439 list, 4 ch, 12-bit) or Hantek DSO2C10 (US$110–160, 2 ch, deep memory) |  |
| INA228 breakout + milliohm shunt | Bus V/I logging (BR-2); later cell testing | S$10–25 |  |
| CAN transceiver for the Nucleo (if current module is unsuitable) | Second CAN node | S$5–10 |  |

### 8.3 Phase 3

| Item | Why |
| --- | --- |
| 1–2 kg load cell + HX711 | Thrust (BR-1) |
| HQProp 3020 / 3030 props | Load testing; stay on 3S with the 4600KV motor |
| 16–18 AWG silicone wire, XT30/XT60 connectors | Prop-level currents |
| Low-ESR bulk capacitor (35 V, 470–1000 µF) | Regeneration and ripple at the ESC1 input |
| Polycarbonate shield | Before any prop spins |
| Spare ESC1 + motor | Once my firmware works |

### 8.4 Software

| Tool | Use | Phase |
| --- | --- | --- |
| STM32CubeIDE, CubeMX, CubeProgrammer | Build, configure, flash, debug | 1 |
| ST Motor Control SDK | Baseline firmware, Motor Profiler, Motor Pilot | 1 |
| Python: numpy, pandas, matplotlib, pyserial | Simulation, logging, plots | 1–2 |
| pytest, GitHub Actions | Unit, SIL and bench tests; CI | 2 |
| python-can, cantools | CAN tests, DBC decoding | 2 |
| udsoncan, can-isotp | Optional diagnostics | 3 |

**Learning resources:** *Python Testing with pytest* (Okken), *Test-Driven Development for Embedded C* (Grenning), the Memfault Interrupt blog, Microchip AN1078 and AN1292, Plett's BMS courses.

## 9. Risks, open questions and references

### 9.1 Risks

| Risk | Impact | Mitigation |
| --- | --- | --- |
| Motor Profiler struggles with the 1404's very low inductance | Wrong Ls, weak baseline | Profile at lower current; repeat runs; check Ls later on the scope |
| Supply limit is overload protection, not true constant current | ESC1 damage on a fault | Confirm CC/CV behaviour in Phase 1; keep the fuse in line |
| Solder bridge or wrong polarity at first power-up | Board damage | Section 3.2 inspection; 0.5 A limit at first switch-on |
| Motor overheats on long no-load runs | Winding or magnet damage | Short runs; log temperature in table 4.3 |
| No scope by the start of Phase 2c | Hardware bring-up stalls | Ask at work / NUS during Phase 1; logic analyser as a stopgap |
| Observer (2a step 5) takes much longer than planned | Phase 2 slips | Budget extra time; keep 2b and 2c steps 1–5 moving with a forced angle meanwhile |
| Scope creep (CAN extras, UDS, drone) | Phases never finish | Each phase has an exit gate; optional items stay optional |

### 9.2 Open questions

- [ ] Is my supply true CC/CV with a settable limit?
- [ ] Which CAN transceiver module do I have; is it CAN FD capable?
- [ ] Is a 120 Ω CAN termination fitted on the ESC1's J1?
- [ ] How many pole pairs does the XING2 1404 have?
- [ ] Can I borrow a scope at work or NUS?
- [ ] Do I already own a multimeter?

### 9.3 References

- [ST B-G431B-ESC1 product page](https://st.com/en/evaluation-tools/b-g431b-esc1.html)
- [UM2516 — B-G431B-ESC1 user manual](https://www.mouser.lt/datasheet/2/389/en.DM00564746-1624778.pdf)
- [ST community thread on the ESC1 input voltage and 26 V TVS](https://community.st.com/s/question/0D50X0000CCp5LUSQZ/bg431besc1-20vdc-max)
- [Rigol DHO800 series price list](https://www.rigolna.com/products/digital-oscilloscopes/dho800/)
- STM32G4 reference manual (RM0440): TIM1, ADC, OPAMP, COMP, FDCAN, CORDIC
- Simulation script: `tools/sim/pmsm_abc_openloop.py` (2a step 0b)

### 9.4 Change log

| Date | Change |
| --- | --- |
| 2026-11-25 | Restructured into one linear plan: roadmap, then Phases 1–3 in order, with requirements, parts and risks as reference sections. Added FR-5, the validation requirements (VR), and the who-does-what table |
| 2026-11-24 | First version (Phase 1 only) |
