# Performance and Scalability – OPEC-in-Space

This document defines the **throughput equations**, **cycle-time structure**, and **scaling logic** for the OPEC-in-Space architecture.

The mothership and skimmer form a closed loop.  
Throughput depends on only three things:

1. **Mass harvested per sortie**  
2. **Number of sorties per year**  
3. **How many skimmers you run in parallel**

Everything else is margin management and thermal survival.

No fake optimism appears here.

---

# 1. Definitions and Symbols

We define the following:

- `m_h`  
  Mass of harvested volatile per sortie (kg)

- `m_out`  
  Mass actually delivered to mothership per sortie (kg), after:
  - internal tank inefficiency  
  - routing losses  
  - venting  
  - pressure stabilization

  Typically:  
  `m_out = η_route · m_h`  
  where routing efficiency `η_route` is 0.85–0.97 depending on embodiment.

- `T_sortie`  
  Total time for one full mission cycle (hours):

T_sortie = T_prep + T_out + T_harvest + T_return + T_dock + T_offload


- `N_cycles`  
Number of sorties per year the skimmer can physically perform:

N_cycles = 8760 / T_sortie

- `R_rel`  
**Reliability factor**, accounting for:
- occasional aborts  
- health downgrades  
- downtime  
- thermal lockout windows

Typical realistic value for first-generation hardware: **0.60–0.85**.

- `Y_single`  
Yearly yield for **one skimmer**:

Y_single = m_out · N_cycles · R_rel

- `S`  
Number of skimmers (1, 3, or 6 in the baseline scaling model).

- `U`  
**Utilization factor** for multiple skimmers sharing docking, tank load, and processing time.  
Ranges:
- 1 skimmer → U = 1  
- 3 skimmers → U ≈ 0.85  
- 6 skimmers → U ≈ 0.70–0.80 depending on architecture

- `Y_total`  
Total annual harvest volume for S skimmers:
Y_total = S · U · Y_single

yaml
Copy code

Everything else is commentary.

---

# 2. Cycle-Time Structure

The sortie cycle time is composed of six segments:

T_sortie = T_prep + T_out + T_harvest + T_return + T_dock + T_offload

diff
Copy code

Where:

- **T_prep**  
  Mission planning + resource top-off + health check  

- **T_out**  
  Outbound transit to harvest corridor  

- **T_harvest**  
  Dominant phase. Efficiency here defines everything.  

- **T_return**  
  Transit back to mothership  

- **T_dock**  
  Approach, capture, and mechanical mating  

- **T_offload**  
  Dumping volatiles + refueling + thermal settling  

In general, OPEC harvest regimes produce:

T_harvest >> T_out, T_return, T_dock, T_prep

yaml
Copy code

Meaning: most of the time is spent harvesting.  
So if you want real performance, you optimize T_harvest and the margins around it.

---

# 3. Mass Per Sortie

Harvested mass per sortie is:

m_h = ρ_env · V_sweep · η_intake

markdown
Copy code

Where:

- `ρ_env` = density of target volatile region  
- `V_sweep` = volume swept by the skimmer  
- `η_intake` = intake efficiency (geometry + partition fill performance)

After routing losses:

m_out = η_route · m_h

yaml
Copy code

Where:

- `η_route` captures:
  - manifold losses  
  - temperature normalization  
  - depressurization effects  
  - any emergency venting  

Thus, per sortie delivered mass is always lower than theoretical harvest mass.  
This is non-negotiable.

---

# 4. Annual Throughput for a Single Skimmer

Combine:

N_cycles = 8760 / T_sortie

Copy code
Y_single = m_out · N_cycles · R_rel

sql
Copy code

Substituting cycle structure:

Y_single = η_route · m_h · (8760 / T_sortie) · R_rel

yaml
Copy code

This is the **definitive equation** of the OPEC architecture.

Everything the raccoons would argue about flows out of this one line.

---

# 5. Reliability-Weighted Cycles

A perfect system would run N_cycles every year without fail.  
Reality says otherwise.

### Causes of reliability loss:

- ABORT-1 events (self-abort)
- ABORT-2 events (forced recall)
- Harvest cutoffs due to orbital shadow or thermal swing  
- Skimmer derating in Phase 8  
- Transitional downtime  
- Docking delays due to thermal instability  
- Health-degradation drift over many sorties  

Thus:

Effective_cycles = N_cycles · R_rel

markdown
Copy code

Where:
- `R_rel = 0.60` → early prototype  
- `R_rel = 0.75` → stable operational Phase II  
- `R_rel = 0.85+` → mature system

---

# 6. Tank Farm & Propellant Closure Constraints

OPEC is only viable if:

1. The **mothership’s tank farm** can accept all harvest deliveries  
2. The **skimmer’s round-trip fuel** does not exceed allowed margins  
3. The **mothership** does not spend more propellant on station-keeping and orbital conditioning than it gains in delivered O₂/H₂/CH₄/etc.

This produces two constraints:

### 6.1 Mothership storage constraint:
Y_single ≤ C_tank / τ_drain

markdown
Copy code
Where:

- `C_tank` = usable tank capacity  
- `τ_drain` = turnover period (how often propellant is exported or processed)

### 6.2 Propellant closure constraint:
m_prop_skim + m_prop_depot < m_out · η_use

markdown
Copy code
Where:

- `m_prop_skim` = propellant used by skimmer per sortie  
- `m_prop_depot` = propellant expended by mothership per year  
- `η_use` = fraction of harvested mass usefully convertible into deliverable propellant  

If either constraint breaks, the system stops being a depot and becomes a toy.

---

# 7. Multi-Skimmer Scaling Logic

We do **not** treat scaling as linear multiplication, because:

- Docking is sequential  
- Tank farm throughput is finite  
- SOXE processing throttles secondary flows  
- Thermal settling is not parallelizable  
- One mothership’s attitude/RCS budget sees increased disturbance

Thus we apply a **utilization factor U** for S skimmers:

| Skimmers (S) | Utilization (U) | Notes |
|--------------|------------------|-------|
| **1** | 1.00 | Baseline reference system |
| **3** | 0.80–0.90 | Occasional docking queueing, thermal bottlenecks |
| **6** | 0.70–0.80 | Mothership becomes throughput-limited |

The total yield is:

Y_total = S · U · Y_single

yaml
Copy code

Meaning:  
By 6 skimmers, the depot—not the skimmers—is the bottleneck.

---

# 8. Yearly Throughput Summary Equations

Here is the entire system in clean mathematical form:

m_out = η_route · m_h
N_cycles = 8760 / T_sortie
Y_single = m_out · N_cycles · R_rel
Y_total = S · U · Y_single

yaml
Copy code

Plug-and-play.  
This is the architecture’s performance backbone.

Anyone arguing with this is arguing with math, not opinion.

---

# 9. Scaling Envelope (Interpretation)

### Small systems (1 skimmer)
- Highest stability  
- Simplest debugging  
- Depot is underutilized  
- Ideal as a demonstrator or early prototype

### Medium systems (3 skimmers)
- Depot begins to actually work as a depot  
- Throughput meaningful for propellant export  
- SOXE processing becomes the next bottleneck  
- RCS usage on the mothership must be carefully managed

### Large systems (6 skimmers)
- You approach the natural limit of one mothership  
- Docking, offload, and tank routing saturate  
- Propellant closure becomes delicate  
- Thermal modes must be extremely well tuned  
- Represents the final “reasonable” scale before needing:
  - A second mothership  
  - Or a distributed depot network

---

# 10. Interpretation: When Is OPEC Worth It?

OPEC becomes economically meaningful when:

Y_single * η_use > annual station-keeping + processing cost

csharp
Copy code

And OPEC becomes strategically meaningful when:

Y_total reaches depot export rates stable enough to replenish
lunar/cislunar operations or refuel returning vehicles.

yaml
Copy code

The key takeaway:

> **OPEC isn’t about one sortie. It’s about cycle count.  
> The system wins by not dying for thousands of hours.**

---

# 11. Summary

The OPEC performance envelope is defined by:

- **m_out** — the mass you actually deliver  
- **T_sortie** — how often you can run the loop  
- **R_rel** — how often it actually works without fouling  
- **S, U** — how well the architecture scales  

Everything else is commentary.

This document closes the architecture by showing how much O₂/CO₂/CH₄/H₂/He-3 you can **actually** export yearly, how fast you can grow the fleet, and where the bottlenecks appear as you scale from 1 → 3 → 6 skimmers.


