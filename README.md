# OPEC-in-Space# OPEC-in-Space

Possible NASA SBIR:This Phase Zero architecture could be applied to NASA SBIR topics involving passive or low-power resource storage, thermal management, and distributed power routing in space environments, especially for cryogenic or volatile propellant handling.

**OPEC-in-Space** is a theoretical architecture for a simple, low-maintenance
orbital volatile-harvesting system built around a **mothership + single skimmer**
loop.

It is **not** a flight-ready design. This is a thought-experiment engineering
paper in repo form: the entire point is to see how far you can get if you are
ruthless about:

- **No moving parts** wherever possible  
- **Mechanical simplicity first**  
- **Tank-farm and manifold architecture that can actually be debugged**  
- **Single-skimmer prototype** as the baseline, with multi-skimmer fleets only as a later scaling layer

The goal: show a credible path for an “OPEC in space”–style volatile depot
(“OPEC” here as shorthand, not a legal or organizational claim) that can
harvest CO₂ and fuels, crack CO₂ to O₂, store everything in a passive tank farm,
and sell/export propellants or life-support oxidizer.

---

## Status

- **Phase:** 0 (paper architecture / GitHub whitepaper)
- **Hardware:** None. No hardware is implied, promised, or warranted.
- **Scope:** Mothership + single skimmer + mission loop + performance framework.
- **Assumption:** Utopian lab conditions and idealized components.
  If you try to build this in the real world, that's on you.

---

## High-level concept

### Mothership

The mothership is a **static depot and processor** with:

- A **tank farm** partitioned for:
  - CO₂ bulk storage
  - LOX (O₂)
  - Hydrogen (H₂)
  - Methane (CH₄)
  - Helium-3 (He-3) in small, high-pressure tanks
- A **CO₂ → O₂ solid oxide electrolysis (SOXE)** module
- A **passive/semi-passive cryogenic storage system**  
- **Solar-electric propulsion** (with one-shot solar deployment) for slow but
  steady orbit changes
- **RCS-only attitude control**
- Simple **avionics + autonomy** sufficient for:
  - Skimmer docking
  - Mission-cycle timing
  - Abort & return-to-origin decisions

Design rule: **no moving parts unless you absolutely cannot avoid them.**  
Redundancy is implemented architecturally (partitions, multi-path routing),
not by copy/pasting hardware everywhere.

### Skimmer

The skimmer is a **single, reusable harvester** that:

- Skims CO₂ and/or fuel-bearing material from a target environment (e.g. upper atmosphere)
- Stores volatiles in internal **partitioned tanks**
- Returns to the mothership, docks, offloads, refuels, and goes again

Canonical features:

- Fixed intake geometry and internal partitions
- Dual battery banks + protected reserve bank
- Fixed-body solar for trickle power
- Three-tier power bus: **primary / secondary / survival**
- Defined operating modes:
  - ACTIVE-HARVEST
  - CRUISE
  - IDLE/STANDBY
  - ECLIPSE/SURVIVAL
  - DOCKED

We deliberately start with **one skimmer only** because:

- It’s much easier to prototype and debug
- It’s far easier for external engineers to replicate
- It proves the full OPEC loop without multi-vehicle scheduling or traffic control

Later docs show how a 1-skimmer system scales to 3 or 6 skimmers.

---

## Design pillars

1. **Mechanical Simplicity**
   - Fewer moving parts → fewer single-point mechanical failures
   - Tank farm redundancy via internal partitions + dual-path manifolds, not copy-pasted tanks

2. **No-Moving-Parts Bias**
   - Flow control via passive routing, check/isolation valves, and pressure differentials
   - One-shot deployments where possible (e.g. solar arrays)

3. **Single-Skimmer Baseline**
   - Mothership + 1 harvester is the reference system
   - Multi-skimmer fleets are a scaling layer, not a requirement

4. **Propellant Closure**
   - System is only “valid” if the mothership + skimmer loop can close its own propellant budget,
     or at least approach closure on realistic timescales

---

## Repo structure

```text
.
├─ README.md                     # This file
├─ LICENSE.md                    # OPEC-in-Space license (3% royalty on commercial use)
├─ docs/
│  ├─ Executive_Summary.md       # Narrative overview of the whole concept
│  ├─ Architecture_Overview.md   # Mothership, skimmer, tank farm, power, thermal
│  ├─ Mission_Loop.md            # 8-phase mission cycle, timing, health checks
│  ├─ Failure_Tree_and_Abort_Modes.md   # ABORT-1/2/3 logic & triggers
│  └─ Performance_and_Scalability.md    # Cycle-time math, throughput, fleet scaling
└─ acs2-diagrams/
   ├─ Mothership_ACS2.txt        # Text-mode ACS2 diagram for mothership topology
   ├─ Skimmer_ACS2.txt           # Text-mode ACS2 diagram for skimmer topology
   └─ Mission_Loop_ACS2.txt      # Text-mode ACS2 diagram for mission-level loop
