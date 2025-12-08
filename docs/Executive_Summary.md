# Executive Summary – OPEC-in-Space

**OPEC-in-Space** is a minimal, single-skimmer orbital volatile-harvesting
architecture designed around three unforgiving constraints:

1. **No moving parts**, unless absolutely unavoidable  
2. **Tank-farm-first design** – the depot is the product  
3. **Single-skimmer baseline** – prove one loop before talking about fleets

The system consists of:

- A **mothership** that acts as refinery, depot, and mission control
- A **single skimmer** that performs repeated sorties to harvest volatiles
- A tightly defined **mission loop** with explicit abort modes and retirement rules
- A **throughput and performance framework** that exposes all hand-waving

The “OPEC” framing is deliberate: the mothership is not a science toy or
demonstrator; it is conceived as an **export node for volatiles and propellant**,
with:

- CO₂ intake and processing → LOX production via solid oxide electrolysis (SOXE)
- Cryogenic and high-pressure storage for O₂, H₂, CH₄, and He-3
- A docking and offload system for skimmers and visiting vehicles
- A slow but capable solar-electric propulsion system for orbit management

This repository captures the **paper architecture** in a form engineers can
actually argue with: text-mode ACS diagrams, mission loops, failure trees,
and explicit performance math.

No claim is made that this is optimal or even sane for flight. The point is to:

- Demonstrate a consistent, end-to-end architecture that can be attacked,
  refined, or repurposed
- Show how much “OPEC in space” capability you can squeeze out if you refuse
  to hide behind buzzwords and magical hardware
- Provide a baseline that others can fork, cannibalize, or trash as needed
  (subject to the license)

Further details are broken down in:

- `Architecture_Overview.md`
- `Mission_Loop.md`
- `Failure_Tree_and_Abort_Modes.md`
- `Performance_and_Scalability.md`
