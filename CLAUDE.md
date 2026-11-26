# CLAUDE.md

Context for Claude Code working in this repo. Read this before doing anything.

## Project

A sensorless FOC (field-oriented control) motor controller, written from scratch, for a small drone motor.
It is a learning and portfolio project aimed at embedded firmware, BMS and firmware-validation roles
(e.g. Tesla BMS firmware, automotive validation). Quality of understanding matters more than speed.

Full plan: `docs/project-plan.md` (roadmap, phase steps, requirements with IDs, parts, risks).
Requirement IDs (P1-x, FR-x, PR-x, SR-x, IR-x, BR-x, VR-x) are defined there; refer to them in
commits, PRs and tests.

## Hardware

- Motor controller: ST **B-G431B-ESC1** (STM32G431CB, L6387 gate drivers, STL180N6F7 MOSFETs,
  3-shunt current sensing, TCAN330 CAN transceiver on J1, detachable ST-LINK with virtual COM port)
- Motor: iFlight XING2 1404 4600KV (rated 2-4S), sensorless only (no encoder)
- NUCLEO-G474RE: processor-in-the-loop target, CAN partner, bench controller (FreeRTOS)
- STM32F411RE Nucleo: INA228 bus V/I logger (no CAN peripheral)
- Bench supply with adjustable current limit; 12 V, 0.5-2 A limit during development

## Repo layout

```
docs/       plan, requirements, notes, images (link vendor PDFs, don't commit them)
firmware/   C firmware (STM32CubeIDE projects; commit the .ioc and generated code)
tools/      Python: sim/ (motor models), sil/ (software-in-the-loop), bench/, can/, reports/
hardware/   wiring notes, photos, later PCB files
data/       small logs and summaries only (no large raw logs)
```

## Current status

- Phase 1 (baseline on ST firmware): parts in delivery; toolchain setup in progress
- Phase 2a (theory + simulation): started. `tools/sim/pmsm_abc_openloop.py` is the step 0b
  open-loop abc PMSM model with placeholder parameters (R, L, pole pairs, J, B are placeholders
  until Motor Profiler measures them; KV = 4600 is from the motor label)

## How we work

**Learning rule:** the owner writes the code first; Claude explains concepts, reviews, debugs and
suggests. Do not write complete solutions for FOC, observer or driver code unless explicitly asked.
Before running a simulation or test, ask the owner to predict the result.

**Architecture rule:** motor control runs as C on the ESC1 (current loop in the ADC interrupt,
25-50 us). Python never sits inside the control loop; it simulates (SIL/PIL) or supervises bench
tests (setpoints, telemetry, logging). Keep FOC algorithm code hardware-free (`foc_step()`,
`observer_update()`) so the same C runs in SIL on the PC and on the target.

**Testing:** pytest for Python and SIL tests. Tag each test with its requirement:
`@pytest.mark.req("PR-2")`. Every requirement should end up with at least one test (VR-1).

## Git workflow

- Never commit to `main`; every change goes through a branch and a pull request
- Branch names: `feat/...`, `fix/...`, `test/...`, `docs/...`, `chore/...`
- Conventional Commits: `feat(sim): add dq motor model`, `fix(foc): clamp iq integrator`
- One logical change per commit; explain the why in the body when not obvious
- PR description: what changed, how it was tested, which requirement IDs it covers
- Tag firmware milestones (`v0.1.0`) and record the version in test logs
- Don't commit build outputs, `__pycache__`, virtual environments, vendor PDFs or large raw logs

## Safety rules (hardware)

- Current limit set before every power-up (0.5 A first power-up, 1-2 A for testing); fuse in line
- No propeller until Phase 3, and then only behind a shield
- Never run the owner's own PWM on the ESC1 until dead time has been checked on a scope or
  logic analyser with bus power off
- Never power the ESC1 from a LiPo until SR-1 to SR-6 pass
- Scope ground clip only on ESC1 ground
