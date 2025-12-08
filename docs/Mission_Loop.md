# Mission Loop – OPEC-in-Space

This document defines the **operational cycle** for the OPEC-in-Space system:
one mothership and **one skimmer**, running a repetitive volatile-harvest loop.

It’s not a vibe description. This is the actual behavior you’d code.

---

## 1. Scope and assumptions

### 1.1 System scope

The mission loop covers:

1. Mothership–skimmer interaction from **pre-departure** through **relaunch**
2. **Phase structure** (what happens when, in what order)
3. **Timers** (heartbeat and death timer)
4. **Mode hooks** for the skimmer (ACTIVE-HARVEST, CRUISE, etc.)
5. Entry points for **abort logic** (ABORT-1/2/3)

Detailed abort behavior is in `Failure_Tree_and_Abort_Modes.md`.  
Throughput and math live in `Performance_and_Scalability.md`.

### 1.2 Core assumptions

- Exactly **one skimmer** is in service at a time.
- The mothership is the **source of truth** for:
  - Mission planning
  - Abort decisions (except emergency ABORT-1 self-abort)
  - Skimmer retirement
- Communication is **intermittent but predictable**.
- The environment (e.g. upper atmosphere of a target body) is:
  - Harvestable
  - Bounded enough that you can define a repeatable sortie envelope

---

## 2. Phase overview

The mission loop is broken into **eight phases**:

1. **Phase 1 – Pre-Departure Prep**  
2. **Phase 2 – Departure Burn**  
3. **Phase 3 – Transit Outbound**  
4. **Phase 4 – Harvest Operations**  
5. **Phase 5 – Transit Return**  
6. **Phase 6 – Docking**  
7. **Phase 7 – Offload & Refuel**  
8. **Phase 8 – Health Check & Relaunch Decision**

Then it either loops back to Phase 1 or the skimmer is retired.

### 2.1 Phase summary table

| Phase | Name                     | Primary owner | Skimmer mode(s)            | Typical duration            |
|-------|--------------------------|---------------|----------------------------|-----------------------------|
| 1     | Pre-Departure Prep       | Mothership    | DOCKED                     | Minutes to hours            |
| 2     | Departure Burn           | Skimmer       | CRUISE                     | Minutes                     |
| 3     | Transit Outbound         | Skimmer       | CRUISE / IDLE/STANDBY      | Minutes to hours            |
| 4     | Harvest Operations       | Skimmer       | ACTIVE-HARVEST             | Dominant fraction of sortie |
| 5     | Transit Return           | Skimmer       | CRUISE / ECLIPSE/SURVIVAL  | Minutes to hours            |
| 6     | Docking                  | Mothership    | DOCKED (entry transition)  | Minutes                     |
| 7     | Offload & Refuel         | Mothership    | DOCKED                     | Minutes to hours            |
| 8     | Health Check & Relaunch  | Mothership    | DOCKED / retired           | Minutes                     |

Durations are system- and environment-dependent. The point is structure, not fantasy numbers.

---

## 3. Timers and control signals

Two timers govern the loop:

1. **Heartbeat timer (`T_HEARTBEAT`)**
   - Regular status packets from skimmer → mothership.
   - Period: chosen so you can detect “we lost it” **well before** fuel/energy margins are gone.
   - Missing N consecutive heartbeats feeds into ABORT-2 or loss declaration.

2. **Death timer (`T_DEATH`)**
   - Set by mothership at **1.5× expected sortie duration**.
   - If `T_DEATH` expires with no confirmed return/dock, the skimmer is **declared lost**.
   - At that point:
     - The skimmer is logically retired in software.
     - Future mission planning disregards it.
     - Mothership may adjust orbit, power, and tank planning accordingly.

Control signals:

- **Mission plan**: sent from mothership to skimmer before departure; includes:
  - Waypoints / altitudes / target region
  - Expected sortie duration
  - Abort thresholds and triggers
- **Abort commands**: ABORT-2 (forced recall) issued by mothership when required.
- **Docking clearance**: mothership explicitly grants clearance for final approach.

---

## 4. Phase definitions

### 4.1 Phase 1 – Pre-Departure Prep (Mothership-led)

**Goal:** Bring the skimmer from DOCKED idle state to a ready-to-fly state with a fresh mission plan.

**Entry conditions:**

- Skimmer is **docked**.
- Previous mission is:
  - Completed successfully, or
  - Declared aborted but the skimmer is still recoverable.

**Actions:**

1. **Mission planning**
   - Mothership computes:
     - Target region / corridor
     - Expected sortie time `T_SORTIE`
     - Harvest duration budget
     - Fuel and power margins
   - Sets death timer:  
     `T_DEATH = 1.5 × T_SORTIE`.

2. **Resource prep**
   - Top-off skimmer:
     - Propellant to mission-specified margin.
     - Batteries BATT_A and BATT_B to required SOC.
   - Ensure any required data packages/updates are loaded.

3. **Configuration**
   - Skimmer’s **MODE_CTRL** set to pre-launch configuration.
   - Critical sensors and ABORT_SENSORS verified alive.

4. **Go/No-Go check**
   - Mothership runs a simple checklist:
     - Power margins
     - Thermal constraints
     - Comms window viability
     - Tank levels (don’t launch if depot is already maxed and has nowhere to store harvest)
   - If any hard constraint fails, **mission is delayed** and Phase 1 repeats after a wait window.

**Exit conditions:**

- Mission plan uploaded and acknowledged by skimmer.
- All critical systems pass basic checks.
- Mothership issues **launch command**.

**Skimmer mode:**  
`DOCKED` transitioning to `CRUISE` at departure.

---

### 4.2 Phase 2 – Departure Burn (Skimmer-led)

**Goal:** Separate from mothership and commit to outbound trajectory.

**Entry conditions:**

- Launch command received and acknowledged.
- Docking latches and data/power connections are in “ready to undock” configuration.

**Actions:**

1. **Undock sequence**
   - Minimal actuators used to mechanically separate.
   - Verify geometric clearance from mothership.

2. **Initial burn**
   - Controlled thruster firing to insert skimmer into outbound corridor.
   - Attitude control kept as simple as possible (e.g. pre-defined profile).

3. **Mode transition**
   - Skimmer switches from `DOCKED` to `CRUISE`.
   - HEARTBEAT timer activated (if not already).

**Exit conditions:**

- Safe clearance from mothership.
- Skimmer on a trajectory that will naturally lead to the target region under standard maneuver plan.

**Skimmer mode:**  
`CRUISE`.

---

### 4.3 Phase 3 – Transit Outbound

**Goal:** Move from mothership-adjacent orbit to the harvest zone.

**Entry conditions:**

- Departure burn completed.
- No abort conditions triggered.

**Actions:**

1. **Trajectory following**
   - Skimmer executes simple guidance to reach entry conditions for harvest (altitude, velocity, geometry).
   - Adjustments made using THRUSTER_FWD and THRUSTER_ATT under `MODE_CTRL`.

2. **Power management**
   - Default mode: `CRUISE`.
   - If in shadow or power-constrained, system may temporarily fall back toward `ECLIPSE/SURVIVAL` for part of transit.

3. **Health monitoring**
   - ABORT_SENSORS continuously monitored.
   - If a critical fault is detected:
     - **ABORT-1 (self-abort)** may be triggered:
       - Skimmer tries best-effort return or safe disposal.
     - Or ABORT-2 may be triggered from mothership based on telemetry anomalies.

**Exit conditions:**

- Skimmer arrives at the defined entry point for the harvest zone with acceptable:
  - Fuel margins
  - Power margins
  - Thermal state

**Skimmer mode:**  
Primarily `CRUISE`, possibly with short dips into `IDLE/STANDBY` or `ECLIPSE/SURVIVAL`.

---

### 4.4 Phase 4 – Harvest Operations

**Goal:** Harvest as much usable volatile mass as possible without violating safety margins or propellant closure.

**Entry conditions:**

- Skimmer within defined harvest corridor.
- Power and propellant margins above abort thresholds.

**Actions:**

1. **Mode switch**
   - `MODE_CTRL` transitions skimmer to **`ACTIVE-HARVEST`**.
   - INTAKE and internal routing prioritized.
   - Nonessential loads can stay on if power allows; otherwise, they’re shed.

2. **Harvest execution**
   - Follow a simple, bounded pattern:
     - Passes / dips / loops through the target region.
     - Internal **CO2_STORE** fills progressively.
   - Mothership keeps track of:
     - Cumulative time in harvest
     - Expected vs. actual harvest mass (if estimators exist)

3. **Margin enforcement**
   - Mothership and skimmer both track:
     - Minimum fuel required to return.
     - Minimum power to survive transit + docking.
   - If margins are approached:
     - Harvest is **terminated early**.
     - Phase 5 (return) is triggered.

**Exit conditions:**

- At least one of:
  - Harvest storage volume is near planned capacity.
  - Time budget for harvest is reached.
  - Safety margins (fuel, power, thermal) are close to limits.
  - An abort is commanded (ABORT-1/2/3 hooks).

**Skimmer mode:**  
`ACTIVE-HARVEST` (dominant), possibly back to `CRUISE` for repositioning segments.

---

### 4.5 Phase 5 – Transit Return

**Goal:** Get the skimmer back to the mothership with enough fuel and power to dock safely.

**Entry conditions:**

- Harvest phase ended by:
  - Planned time/volume completion, or
  - Safety-margin trigger, or
  - Abort command.

**Actions:**

1. **Trajectory**
   - `MODE_CTRL` transitions back to `CRUISE`.
   - Skimmer flies a pre-planned return trajectory; minimal guidance complexity.

2. **Energy management**
   - Aggressive power management:
     - Shed secondary loads first.
     - If power drops below thresholds, move into `ECLIPSE/SURVIVAL`.
   - Under `ECLIPSE/SURVIVAL`:
     - Only critical avionics, thermal control, and basic comms allowed.

3. **Heartbeat & aborts**
   - HEARTBEAT continues.
   - Mothership monitors:
     - If the skimmer is trending toward missing `T_DEATH`.
     - If an ABORT-2 forced recall is still meaningful (e.g. cut harvest early, prioritize return).

**Exit conditions:**

- Skimmer reaches pre-docking corridor around mothership.
- Mothership confirms geometry and relative motion are acceptable for docking.

**Skimmer mode:**  
`CRUISE`, with potential temporary `ECLIPSE/SURVIVAL` if power is poor.

---

### 4.6 Phase 6 – Docking

**Goal:** Bring the skimmer back into physical contact with the mothership safely.

**Entry conditions:**

- Skimmer in mothership vicinity, within docking corridor.
- Relative motion and attitude are within limits.

**Actions:**

1. **Approach**
   - Mothership grants explicit **docking clearance**.
   - Skimmer executes a predefined, conservative docking trajectory.
   - Minimal new logic here: keep it as dumb and robust as possible.

2. **Capture**
   - Docking cone and structural features guide the skimmer into place.
   - Limited actuators secure the fit.
   - Once mechanically secured, vehicle transitions to `DOCKED` mode.

3. **Interface verification**
   - Confirm:
     - Structural locks
     - Fluid connections (for volatiles and fuel)
     - Data and power connections

**Exit conditions:**

- Skimmer is fully **docked** and stable.
- All required interfaces pass basic checks.

**Skimmer mode:**  
Transition from `CRUISE` to `DOCKED`.

---

### 4.7 Phase 7 – Offload & Refuel

**Goal:** Move harvested volatiles into the mothership’s tank farm, refill the skimmer, and stabilize both systems.

**Entry conditions:**

- Docking complete and verified.

**Actions:**

1. **Offload**
   - Volatiles from **CO2_STORE** → mothership via DOCK_PORT → appropriate manifolds:
     - CO₂: DOCK_CO2 → CO2_M → CO2_TANK_RING and/or SOXE.
     - Fuel (if skimmer also harvests fuels): DOCK_FUEL → FUEL_M → relevant tank rings.

2. **Refuel skimmer**
   - Propellant from appropriate mothership tanks → FUEL_LOBES.
   - Battery charge from mothership POWER_BUS → BATT_A and BATT_B.

3. **Thermal settling**
   - Both vehicles may remain docked for a while to:
     - Equalize thermal conditions.
     - Avoid stressing cryogenic tanks with rapid state changes.

**Exit conditions:**

- Harvested volatiles offloaded to planned levels.
- Skimmer propellant and batteries restored to required margins for another sortie (unless intentionally retiring).
- No major anomalies flagged.

**Skimmer mode:**  
`DOCKED`.

---

### 4.8 Phase 8 – Health Check & Relaunch Decision

**Goal:** Decide whether to send the skimmer back out, downgrade it, or retire it.

**Entry conditions:**

- Offload and refuel completed.
- Skimmer is in `DOCKED` state.

**Actions:**

1. **Hardware health review**
   - Automated and/or operator-in-the-loop review of:
     - Propulsion performance metrics from last sortie.
     - Power system behavior (deep discharges, current spikes, etc.).
     - Thermal excursions.
     - Any logged anomalies or near-misses.
   - Evaluate against predefined **retirement criteria**:
     - Max cycle count
     - Repeated thermal over-stress
     - Structural or seal degradation indicators

2. **Mission-logic decision**
   - If health is acceptable and mission demand remains:
     - Mark skimmer as **ACTIVE**.
     - Loop back to **Phase 1** for another sortie.
   - If health is marginal but usable:
     - Potentially **derate** skimmer (reduced payload, shorter sorties).
     - Update mission planning logic accordingly.
   - If health is unacceptable:
     - Mark skimmer as **RETIRED**.
     - It remains docked as dead mass or is jettisoned per mission rules.

3. **Timer reset**
   - If relaunching:
     - Reset HEARTBEAT expectations and `T_DEATH` for next mission.
   - If retiring:
     - All skimmer-related mission timers are effectively frozen.

**Exit conditions:**

- One of:
  - Skimmer flagged for **relaunch** → return to Phase 1.
  - Skimmer flagged as **derated** but still usable → return to Phase 1 with updated constraints.
  - Skimmer **retired** → mission loop ends for this vehicle.

**Skimmer mode:**  
Remains `DOCKED` or transitions to “retired” logical state.

---

## 5. Integration with abort modes

The mission loop is not optimistic; it assumes things will break. Abort modes are threaded through:

- **ABORT-1 – Skimmer Self-Abort**
  - May trigger during:
    - Phase 3 (Transit Outbound)
    - Phase 4 (Harvest)
    - Phase 5 (Transit Return)
  - Skimmer attempts safest possible behavior with whatever is left.

- **ABORT-2 – Mothership Forced Recall**
  - May trigger during:
    - Phase 3, 4, or 5, if the mothership sees something the skimmer doesn’t.
  - Command: “stop what you’re doing and come back now.”

- **ABORT-3 – TOTAL ABORT**
  - Mothership catastrophic fault:
    - Can trigger in any phase.
    - Mothership executes immediate **TOTAL ABORT**:
      - Isolate damage
      - Drop to survival bus
      - Begin **return-to-origin burn**.
      - Skimmer is abandoned if not already docked.

Detailed trees and behaviors are defined in `Failure_Tree_and_Abort_Modes.md`.

---

## 6. Summary

The mission loop is:

1. **Eight phases**, no magic.
2. **Two timers** (heartbeat and death timer) that enforce reality.
3. **Five skimmer modes** threaded across the phases.
4. **Three abort paths** that can cut in at any point where physics stops playing nice.

If someone wants to implement this, they don’t need to guess the flow; they just need to argue with the numbers and thresholds in the other docs.
