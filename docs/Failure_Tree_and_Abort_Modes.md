# Failure Tree and Abort Modes – OPEC-in-Space

This document defines the **three abort classes** in the OPEC mission architecture:
1. **ABORT-1 — Skimmer Self-Abort**  
2. **ABORT-2 — Mothership-Forced Recall**  
3. **ABORT-3 — TOTAL ABORT (Catastrophic Mothership Failure)**  

Abort modes are not optional behavior — they are **hard divergence paths** in mission logic.  
They override the normal mission loop immediately.

---

# 1. Abort Philosophy

The OPEC system assumes **hardware will fail** and **telemetry will be intermittent**.  
Abort logic must therefore:

- Be **simple enough** to implement without complex autonomy.
- Be **deterministic** — no probabilistic magic.
- **Protect the mothership first**, skimmer second.
- Always err on the side of **propellant closure** and **thermal survival**.
- Trigger **early**, not late.

The skimmer is disposable if required.  
The mothership is not.

---

# 2. Abort Overview Table

| Abort Mode | Triggered By | Primary Goal | Skimmer Behavior | Mothership Behavior |
|------------|--------------|--------------|------------------|---------------------|
| **ABORT-1** | Skimmer detects its own critical fault | Save itself *if possible*; avoid becoming debris | Immediate safe-turn return or safe disposal | Monitor; accept skimmer loss if necessary |
| **ABORT-2** | Mothership detects abnormal telemetry/state | Yank the skimmer back before it dies | Stop harvest & return now | Issue command, prep docking / adjust power |
| **ABORT-3** | Catastrophic mothership fault | Save the mothership at all costs | Skimmer abandoned unless already docked | Isolate damage, drop to survival bus, return-to-origin burn |

---

# 3. ABORT-1 – Skimmer Self-Abort

ABORT-1 exists because sometimes the skimmer will know it is dying before the mothership does.

This is **the most common abort** and the one requiring the least sophistication to detect.

---

## 3.1 ABORT-1 Trigger Conditions

The skimmer enters ABORT-1 if **any** of the following occur:

### **A. Propulsion-critical**
- Loss of majority of THRUSTER_FWD capability  
- Loss of attitude control authority (THRUSTER_ATT failure cluster)  
- Fuel leak trending below **minimum return delta-v margin**

### **B. Power-critical**
- BATT_A or BATT_B failure *and* remaining bank < required return margin  
- Inability to maintain minimum avionics power  
- Entering **ECLIPSE/SURVIVAL** mode *without* ability to maintain thermal survival

### **C. Thermal-critical**
- Hot zone (HOT_SPINE or equivalent) exceeds hard thermal max  
- Cryogenic mass nearing structural overpressure due to thermal creep  
- Failure of thermal control routing in a way that risks tank integrity  

### **D. Storage-critical**
- CO2_STORE integrity breach detected  
- Internal partitions indicate leak or failure propagating across multiple tanks  

### **E. Navigation / Sensor-critical**
- Loss of inertial reference OR loss of attitude sensing  
- Navigation drift exceeding corridor boundaries  
- Failure of ABORT_SENSORS themselves  

---

## 3.2 ABORT-1 Behavior (State Machine)

Once triggered, the skimmer executes:

### **Step 1 — Immediate Mode Change**
`MODE_CTRL → SURVIVAL`  
Cuts to:
- Minimum avionics  
- Minimum thermal  
- Minimal comms  
All secondary loads are dropped instantly.

### **Step 2 — Decide Return Feasibility**
Based on remaining fuel and power:

#### **Case A — Return is feasible**
- Compute direct-return trajectory (no repositioning, no optimization).  
- Fire minimal thruster pulses to align with mothership vector.  
- Transmit continuous heartbeat (low-power mode).

#### **Case B — Return is *not* feasible**
- Attempt to:
  - Dump volatile mass if it helps (optional design choice).
  - Execute disposal trajectory (safe orbit raising/lowering).
- Send emergency telemetry until power exhaustion.

### **Step 3 — Docking Attempt (If feasible)**
If the skimmer reaches the mothership:
- Mothership grants emergency docking clearance.
- Docking procedure is the minimal-actuator version.
- No offload; no refuel; immediate health assessment.
- Skimmer is almost always **retired** after an ABORT-1.

### **Step 4 — Declared Lost**
If:
- Telemetry stops, or
- T_DEATH expires,

the mothership marks the skimmer **lost** and reallocates resources.

---

## 3.3 ABORT-1 Effects on Mission Loop

- Phases 3, 4, or 5 are **immediately terminated**.
- Mission loop jumps straight to:
  - **Phase 6 (Docking)** if return is possible  
  - Or **Loss path** if not  
- Phase 7 (offload) and Phase 8 (relaunch decision) are skipped unless the skimmer actually docks.

---

# 4. ABORT-2 – Mothership-Forced Recall

ABORT-2 is triggered when the **mothership sees something bad** before the skimmer does.

ABORT-2 pulls rank.  
The skimmer obeys immediately.

---

## 4.1 ABORT-2 Trigger Conditions

### **A. Telemetry anomalies**
- Broken or intermittent heartbeats  
- Drift in nav/attitude metrics  
- Irregular current draw trends  
- Unexpected thermal signatures  

### **B. Environmental hazards**
- Dust, plasma, or high-density region detected by mothership sensors  
- Inbound debris or conjunction risk  
- Solar flare / CME requiring immediate withdraw

### **C. Tank farm constraints**
- Depot nearing max CO₂ or fuel storage  
- Cryogenic tank thermal approaching unsafe thresholds  
- Internal manifold behavior suggesting pressure anomalies

### **D. SOXE / processing faults**
- SOXE overheat  
- O₂ manifold overpress  
- Power bus instability caused by processing load

---

## 4.2 ABORT-2 Behavior (State Machine)

The instant a forced recall is issued:

### **Step 1 — Command Issued**
Mothership broadcasts:
ABORT-2: RETURN IMMEDIATELY. CEASE HARVEST.

### **Step 2 — Skimmer Acknowledges**
If skimmer fails to acknowledge after N attempts:
- Upgrade to **ABORT-2B**
- Mothership prepares for possible loss scenario

### **Step 3 — Skimmer Mode Change**
Skimmer transitions to:
`ACTIVE-HARVEST → CRUISE → SURVIVAL (if required)`

Harvest stops instantly.

### **Step 4 — Direct Return**
- Use simplest safe return trajectory  
- Fuel margins override harvest completion  
- Abort return takes precedence over efficiency

### **Step 5 — Docking**
If skimmer returns:
- Docking priority is **medium** (not as high as ABORT-1, but urgent).  
- Offload *may* be skipped if depot is in unstable condition.

### **Step 6 — Health Assessment**
Mothership determines:
- Whether the skimmer caused the anomaly  
- Whether skimmer is allowed another sortie  
- Whether depot itself is the problem  

ABORT-2 does *not automatically* retire the skimmer, but it often does.

---

## 4.3 ABORT-2 Effects on Mission Loop

Mission loop shortcuts:

- Phase 4 (Harvest) → terminated  
- Phase 5 (Return) → forced  
- Phase 6 (Docking) → immediate on arrival  
- Phases 7–8 may be heavily modified depending on depot conditions

---

# 5. ABORT-3 — TOTAL ABORT  
### (Catastrophic Mothership Fault)

This is the **nightmare scenario**.  
This is the only abort where **the skimmer is expendable by design**.

TOTAL ABORT protects the mothership, because it’s the depot, processor, refueler, and mission brain.

If the mothership dies, the system dies.

---

## 5.1 ABORT-3 Trigger Conditions

ABORT-3 triggers when the mothership detects:

### **A. Structural-critical**
- Tank ring rupture (any of CO₂, O₂, H₂, CH₄, He-3)
- Manifold breach with uncontrolled depressurization
- Hull fracture propagating toward critical systems

### **B. Thermal-critical**
- SOXE runaway beyond containment  
- Cryogenic tank boiloff escalating toward overpressure  
- Radiator or thermal pipe failure causing cascading thermal divergence

### **C. Power-critical**
- Primary bus collapse  
- Battery bank short or runaway  
- Solar array structural failure threatening power stability

### **D. Guidance/Control-critical**
- Loss of majority of RCS authority  
- Attitude instability trending toward tumble  
- Avionics failure undermining safe operations

### **E. External threats**
- Missile/impact debris (realistic for militarized scenarios)  
- Catastrophic conjunction event  
- CME / solar event exceeding survivable radiation thresholds

Bottom line: **If the mothership cannot guarantee its own survival while continuing the mission**, ABORT-3 triggers.

---

## 5.2 ABORT-3 Behavior (State Machine)

This is the most violent branch in the entire architecture.

### **Step 1 — Immediate Broadcast**
Mothership transmits:
ABORT-3 TOTAL ABORT. MOTHERSHIP CRITICAL. RETURN-TO-ORIGIN INITIATED.

This message is *best-effort only* — the skimmer may never receive it.

### **Step 2 — Isolate Damage**
- Close isolation valves (if any remain functional)  
- Electrically isolate failing power segments  
- Shut down SOXE immediately  
- Cut all secondary loads  
- Switch to **SURVIVAL BUS ONLY**

### **Step 3 — Return-to-Origin Burn**
Mothership initiates:
- Safe, slow, solar-electric return trajectory  
- Low thrust but long duration  
- Goal: preserve depot and its volatile inventory

### **Step 4 — Skimmer Handling**
If **docked**:
- Skimmer rides along; mission loop is terminated permanently.

If **not docked**:
- **Skimmer is abandoned.**  
- Mothership will *not* wait and *not* attempt recovery.
- Any skimmer telemetry that arrives is ignored except for logging.

### **Step 5 — Stabilization**
During return:
- Mothership prioritizes:
  - Maintaining tank pressures  
  - Maintaining thermal stability  
  - Maintaining minimal avionics  
- Mission planning system transitions into “shutdown mode.”

---

## 5.3 ABORT-3 Effects on Mission Loop

- Mission loop **terminates irreversibly**.  
- No new sorties occur.  
- Skimmer is either:
  - Carried home (if docked), or  
  - Lost forever (if not docked)  

The depot becomes a damaged but salvageable asset returning to origin.

---

# 6. Failure Tree Overview

Below is a compressed text-mode failure tree showing the decision flow.

FAILURE EVENT
│
├── Skimmer detects critical fault → ABORT-1
│ ├── Return feasible → emergency docking → retire skimmer
│ └── Return infeasible → disposal or loss → skimmer retired
│
├── Mothership detects anomaly in skimmer → ABORT-2
│ ├── Recall acknowledged → forced return → docking → health eval
│ └── No acknowledgement → treat as drifting asset → possible loss
│
└── Mothership detects catastrophic self-fault → ABORT-3
├── Skimmer docked → ride home
└── Skimmer not docked → abandoned


---

# 7. Summary

- **ABORT-1 protects the skimmer** when it notices its own death spiral.  
- **ABORT-2 protects the skimmer and the depot** by yanking it home.  
- **ABORT-3 protects the depot** *even if it means killing the skimmer.*  

This structure keeps the system simple, survivable, and predictable — exactly what a one-skimmer architecture requires before scaling to fleets.

