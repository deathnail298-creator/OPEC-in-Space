# Architecture Overview

This document outlines the architecture of the **OPEC-in-Space** system:
mothership, skimmer, tank farm, power, and thermal.

---

## 1. Mothership

### 1.1 Roles

The mothership is responsible for:

- **Volatile intake and processing**
- **Storage** of CO₂, O₂, H₂, CH₄, He-3
- **Docking and offload** for the skimmer
- **Power generation and distribution**
- **Propulsion and attitude control**
- **Mission management and abort logic**

### 1.2 Tank farm

Core design rules:

- No duplicated “identical” tanks purely for redundancy
- Redundancy handled by **partitions and manifolds**, not hardware cloning
- Each tank connects to its manifold via **at least two independent routing paths**
- Critical manifolds are implemented as **dual-lumen** (or equivalent) trunks

Tank rings:

- **CO₂_TANK_RING** – inner ring, bulk CO₂
- **O2_TANK_RING** – LOX
- **H2_TANK_RING** – LH₂ / high-pressure hydrogen
- **CH4_TANK_RING** – methane
- **HE3_TANKS** – small, isolated high-pressure tanks

Backbone manifolds:

- **CO2_M** – CO₂ backbone
- **O2_M** – O₂ backbone
- **FUEL_M** – H₂/CH₄/He-3 backbone
- **VAC_M** – vacuum/pressure management backbone

CO₂ flows from docking intake → **CO2_M** → **CO₂_TANK_RING** and **SOXE**.
O₂ from SOXE flows → **O2_M** → **O₂_TANK_RING**, → engine feed, → depot export.

See `acs2-diagrams/Mothership_ACS2.txt` for the text-mode topology.

### 1.3 Processing (SOXE)

The SOXE module:

- Takes CO₂ from **CO2_M**
- Produces O₂ to **O2_M**
- Dumps carbon byproduct into either:
  - Dedicated waste storage
  - Future materials feed (not detailed)

Thermal handling ensures the SOXE “hot zone” is tightly isolated from cryogenic
tanks and routed to radiators.

### 1.4 Power system

- **Primary source:** solar arrays deployed once and left fixed
- **Energy storage:** battery banks with tiered loads:
  - **Primary bus:** processing, docking, core avionics
  - **Secondary bus:** comms, diagnostics, non-critical payloads
  - **Survival bus:** bare minimum for thermal control, critical sensors, and
    basic attitude control

The mothership does not try to be clever. It should be painfully obvious what
is on which bus and what gets cut first during power crunch.

### 1.5 Thermal system

Thermal design is split into zones:

- **Hot zone:** SOXE and associated electronics
- **Cold zone:** cryogenic tanks and sensitive instruments
- **Ambient zone:** structure, docking cone, manifolds

Tools:

- Passive insulation and multi-layer insulation (MLI)
- Heat pipes or equivalent to dedicated radiators from the hot zone
- Thermal isolation and baffling around cryo tanks
- Eclipse/survival modes that throttle processing to keep the vehicle alive

---

## 2. Skimmer

### 2.1 Roles

The skimmer is designed to:

- Skim volatiles from the environment
- Store them in internal tanks
- Return to the mothership for offload
- Survive long enough to justify building it

It is not meant to be smart or heroic. It’s meant to be **disposable but reusable**.

### 2.2 Topology

Core features:

- **INTAKE** → internal CO₂ or volatile **STORAGE_PARTITIONS**
- **FUEL_LOBES** for the skimmer’s own propulsion
- **HOT_SPINE** (if used) to dump waste heat
- Battery banks **A** and **B**, plus a **reserve core**
- Docking interface matching the mothership’s docking cone and manifolds

Operating modes and their impact on power/thermal/routing are detailed in
`Mission_Loop.md` and the skimmer ACS diagram.

See `acs2-diagrams/Skimmer_ACS2.txt` for the topology.

### 2.3 Power & modes (summary)

- Two main battery banks (**A**, **B**) run alternately or in combination
- One small **reserve bank** is kept as a last-ditch survival buffer
- **Survival mode** is brutally minimal:
  - Only critical avionics, thermal management, and beacon functions

Modes:

- **ACTIVE-HARVEST:** high draw, maximum intake
- **CRUISE:** transit to/from mothership
- **IDLE/STANDBY:** waiting for next command
- **ECLIPSE/SURVIVAL:** all nonessential systems shed
- **DOCKED:** power and thermal largely slaved to the mothership

---

## 3. Docking and transfer

Docking is designed to be as **passive** as possible:

- Funnel geometry to pull the skimmer into alignment
- Poppet / check valves for fluid transfer with minimal moving complexity
- Structural latch system that can be implemented with the fewest actuators
  the laws of physics will tolerate

Docked state:

- Offload harvested volatiles from skimmer → mothership tank farm
- Refuel skimmer propellant and top off its batteries
- Perform health checks and decide:
  - Relaunch
  - Downgrade to reduced-capability mode
  - Retire the skimmer entirely

---

For more detail, see:

- `Mission_Loop.md` – temporal behavior
- `Failure_Tree_and_Abort_Modes.md` – what happens when things break
- `Performance_and_Scalability.md` – whether this is even worth doing
