# Off-Grid Microgrid Multi-Agent Sizing & BOM Optimizer

An autonomous 22-node multi-agent engineering workflow built in **n8n** (`@n8n/n8n-nodes-langchain`) that automates electrical sizing, seasonal autonomy simulation, thermodynamic derating checks, and dynamic Bill of Materials (BOM) compilation for off-grid solar-plus-storage microgrids.

*(Originally developed as a hackathon engineering project to automate complex off-grid power system design).*

---

## 1. Project Overview & Problem Statement

Sizing a standalone off-grid microgrid (such as a 400A 120/240V split-phase residential or light-commercial service) requires solving tightly coupled electrical, thermal, and economic constraints simultaneously:
- **Seasonal Irradiance vs. Load Mismatch:** High-latitude coastal regions experience up to an 81% drop in winter Peak Sun Hours (e.g., 6.02 PSH in summer down to 1.17 PSH in winter) while heating loads peak (+30% winter demand).
- **Cold-Weather String Voltage Limits:** Low ambient temperatures ($-15^\circ	ext{C}$) increase open-circuit PV module voltage ($V_{oc}$), requiring strict string length verification to prevent overvoltage damage on 600 VDC inverter inputs.
- **Generator Operating Economics:** Backup generator costs are dominated by maintenance intervals rather than fuel alone ($1.65/kWh fuel + $300 service every 100 kWh = $4.65/kWh true cost, plus 2% cumulative efficiency loss per service cycle).
- **Multi-Day Outage Resilience:** Battery banks and smart load-shedding contactors must sustain critical loads through 72-hour blackout conditions while accounting for high-discharge rate derating (5%) and monthly capacity fade (3%/month).

Instead of relying on static spreadsheets, this workflow decomposes the engineering problem across **8 specialized AI agents** running in a two-stage parallel-to-sequential Directed Acyclic Graph (DAG), culminating in a deterministic JavaScript BOM and lifecycle cost compiler.

---

## 2. Automation Architecture & Execution Pipeline

```
[Start Simulation] (manualTrigger)
        │
        ▼
[Load System Parameters] (n8n-nodes-base.set — 32 System Constants)
        │
        ├──────────────────┬──────────────────┬──────────────────┬──────────────────┬──────────────────┐
        ▼                  ▼                  ▼                  ▼                  ▼                  ▼
  [Agent 1]          [Agent 2a]         [Agent 2b]          [Agent 3]          [Agent 4]          [Agent 6]
  Consumption        Solar Yield        Gen Dispatch       Unit Costing       25-Yr Maint       72h Autonomy
  & Load Split       & Cold Voc         & O&M Curve        & LCOE Rates       & Fade Model      & Shedding
        │                  │                  │                  │                  │                  │
        └──────────────────┴──────────────────┴────────┬─────────┴──────────────────┴──────────────────┘
                                                       ▼
                                       [Merge Parallel Calculations] (6-Input Synchronization Barrier)
                                                       │
                                                       ▼
                                       [Consolidate Agent Outputs] (JS State Aggregator)
                                                       │
                                        ┌──────────────┴──────────────┐
                                        ▼                             ▼
                                  [Agent 5]                     [Agent 7]
                           Management & Optimization     Zero-Gen Contingency
                           (Pareto Lifecycle Solver)     (100% Renewable Bound)
                                        │                             │
                                        └──────────────┬──────────────┘
                                                       ▼
                                       [Merge Final Optimizations] (2-Input Barrier)
                                                       │
                                                       ▼
                                    [Executive Synthesis & BOM Report] (Dynamic JS BOM Engine)
```

### Stage 0: Parameter Injection (`Load System Parameters`)
A centralized `Set` node defines 32 hardware, environmental, and financial parameters passed downstream to all calculation branches and referenced directly by the terminal BOM compiler. Modifying site conditions or equipment prices in this single node updates the entire pipeline automatically.

### Stage 1: Concurrent Domain Analysis (6 Parallel Agents)
Six independent `@n8n/n8n-nodes-langchain.agent` nodes execute in parallel, each paired with a dedicated OpenAI language model sub-node (`gpt-5-mini`):

1. **`Agent 1 - Home Consumption`**
   - Computes seasonal daily demand (Spring: 30 kWh/d, Winter: 39 kWh/d, Summer: 25.5 kWh/d, Fall: 34.5 kWh/d) and annual energy consumption.
   - Partitions the 400A service into **Emergency/Critical circuits (~40% demand)** and **Non-Emergency/Sheddable circuits (~60% demand)** for smart contactor load shedding.
2. **`Agent 2a - Solar Production`**
   - Models daily and annual yield across candidate PV array sizes (48, 60, 72 modules of 440W rating) using seasonal PSH values and 96.5% CEC inverter efficiency.
   - Performs cold-weather $V_{oc}$ verification at $-15^\circ	ext{C}$ ($+10\%$ voltage rise to $42.79	ext{ V/module}$), confirming that 12 modules per string ($513.5	ext{ VDC}$) maintains a 14.4% safety margin below the $600	ext{ VDC}$ MPPT limit.
3. **`Agent 2b - Generator Production`**
   - Models the 6 kW continuous / 7.5 kW surge backup generator under a **5.0 kW bulk-charging setpoint** (~1.8 kW feeding household loads + ~3.2 kW charging the battery bank).
   - Quantifies true operating cost ($1.65/kWh fuel + $3.00/kWh service overhead = $4.65/kWh initial rate) and compounds the 2% thermodynamic efficiency penalty after every 100 kWh service interval.
4. **`Agent 3 - Unit Cost & Sourcing`**
   - Computes levelized unit economics across all candidate components: installed solar CapEx ($0.80/Wdc), battery storage ($384/kWh nominal), hybrid inverter capacity ($458/kW), and generator LCOE ($4.65–$4.97/kWh).
5. **`Agent 4 - Long-Term Maintenance`**
   - Evaluates 25-year asset lifecycle schedules, including LiFePO4 cycle life (>8,000 cycles), mid-life inverter refurbishment (Years 12–15), 500 kcmil / 4/0 AWG conductor and breaker inspections, and battery derating (5% full-power derate + 3%/month capacity fade).
6. **`Agent 6 - 3-Day Seasonal Autonomy`**
   - Simulates 72-hour blackout survival across all four seasons (90 kWh Spring, 117 kWh Winter, 76.5 kWh Summer, 103.5 kWh Fall).
   - Incorporates residual winter blackout solar generation (1.17 PSH/day) and automated load shedding via `EG4 GridBOSS` Smart Ports 1–4.

### Stage 1.5: Barrier Synchronization & State Consolidation
- **`Merge Parallel Calculations`**: A 6-input `Merge` barrier that waits for all six parallel domain agents to finish before advancing execution.
- **`Consolidate Agent Outputs`**: A JavaScript `Code` node that maps the six raw outputs into a unified JSON state object (`agent1_consumption` through `agent6_three_day`) with an execution timestamp.

### Stage 2: Multi-Objective Synthesis & Contingency Modeling
Two downstream synthesis agents ingest the consolidated Stage-1 state in parallel:

- **`Agent 5 - Management & Optimization` (Primary Solver)**
  - Synthesizes all six engineering analyses to minimize **10-Year Total Lifecycle Cost (CapEx + 10-Year Generator O&M)** while satisfying 3-day backup autonomy, 400A service splitting, and zero utility export.
  - Outputs the complete engineering rationale followed by a strict machine-readable JSON block containing the exact hardware quantities and annual generator operating metrics.
- **`Agent 7 - Zero-Gen Contingency Optimizer` (Boundary Analysis)**
  - Evaluates an alternative boundary condition where the backup generator is removed entirely and load is restricted to critical 3-day profiles, quantifying the capital premium required to achieve 100% winter autonomy purely through PV and BESS overbuilding.

### Stage 3: Dynamic BOM & Lifecycle Cost Engine (`Executive Synthesis & BOM Report`)
Rather than returning hardcoded cost tables, the terminal JavaScript `Code` node dynamically compiles the Bill of Materials:
1. **Regex JSON Extraction:** Parses the fenced ```json block emitted by `Agent 5` (with secondary regex fallback against the text body if JSON formatting is omitted).
2. **Live Parameter Lookup:** Reads unit hardware prices and Balance of System (BOS) costs directly from the upstream `Load System Parameters` node (`$('Load System Parameters').first().json`).
3. **Deterministic Cost Compilation:** Multiplies `Agent 5`'s selected quantities by live unit prices to compute equipment subtotals, electrical BOS (breakers, 500 kcmil / 4/0 AWG wiring, conduit), total installed CapEx, annual generator O&M, and 10-year lifecycle cost.

---

## 3. Input Parameter Schema (`Load System Parameters`)

```json
{
  "service_rating": "400A 120/240V Split-Phase",
  "location": "Victoria, BC, Canada",
  "spring_baseline_kwh_day": 30,
  "winter_kwh_day": 39,
  "summer_kwh_day": 25.5,
  "fall_kwh_day": 34.5,
  "solar_panel_model": "JA Solar JAM54D41-440/LB (440W)",
  "panel_cost_with_racking": 350,
  "inverter_model": "EG4 FlexBOSS21 (12kW batt / 16kW PV / 21kWdc input)",
  "inverter_cost": 5500,
  "battery_model": "EG4 280Ah All-Weather (51.2V 280Ah = 14.336 kWh)",
  "battery_cost": 5500,
  "gateway_model": "EG4 GridBOSS 200A MID",
  "gridboss_cost": 2500,
  "splitter_cost": 4500,
  "panel_200a_cost": 1500,
  "generator_model": "BE7500ID (6kW continuous / 7.5kW surge)",
  "generator_fuel_rate_per_kwh": 1.65,
  "generator_service_interval_kwh": 100,
  "generator_service_cost": 300,
  "generator_efficiency_loss_per_service": 0.02,
  "psh_winter": 1.17,
  "psh_spring": 4.25,
  "psh_summer": 6.02,
  "psh_fall": 2.41,
  "winter_blackout_psh": 1.17,
  "battery_full_power_derate": 0.05,
  "battery_monthly_degradation_rate": 0.03,
  "bos_wiring_insulation_breakers_cost": 7510,
  "generator_operating_kw": 5,
  "generator_to_load_kw": 1.8,
  "generator_to_battery_kw": 3.2
}
```

---

## 4. Agent 5 Output Schema & Terminal Payload

`Agent 5` outputs the following structured JSON block at the end of its reasoning trace:

```json
{
  "solar_panels": 72,
  "flexboss21_inverters": 2,
  "gridboss_units": 1,
  "eg4_batteries": 8,
  "distribution_splitters": 1,
  "distribution_panels_200a": 2,
  "fused_disconnects_400a": 1,
  "annual_generator_kwh": 330.0,
  "annual_generator_services": 3,
  "annual_generator_cost": 1457.50
}
```

The terminal node (`Executive Synthesis & BOM Report`) transforms this into the final structured output payload:

```json
{
  "workflow_title": "Off-Grid Microgrid 8-Agent Autonomous Design & BOM Optimizer",
  "status": "OPTIMIZATION_COMPLETE",
  "bom_generation_mode": "FULLY_DYNAMIC_FROM_AGENT_5_JSON",
  "agent_5_selected_quantities": { "...": "..." },
  "dynamic_bom_summary": {
    "solar_panels": "72 x JA Solar 440W (31.68 kWdc) @ $350 = $25,200",
    "hybrid_inverters": "2 x EG4 FlexBOSS21 @ $5500 = $11,000",
    "system_controller": "1 x EG4 GridBOSS MID @ $2500 = $2,500",
    "battery_storage": "8 x EG4 280Ah (114.7 kWh nom) @ $5500 = $44,000",
    "service_splitter": "1 x 400A Distribution Splitter @ $4500 = $4,500",
    "distribution_panels": "2 x 200A Electrical Panels @ $1500 = $3,000",
    "fused_disconnect": "1 x 400A Fused Service Disconnect @ $1500 = $1,500",
    "electrical_bos_breakers_wiring_insulation": "Breakers, 500kcmil/4/0AWG wiring, conduit, insulation = $7,510",
    "equipment_subtotal": "$91,700 CAD",
    "total_capital_cost": "$99,210 CAD (Equipment + BOS)",
    "annual_generator_om": "$1,457.5 CAD / year (330 kWh/yr, 3 services/yr)",
    "lifecycle_10yr_total": "$113,785 CAD"
  },
  "primary_design_recommendation_agent5": "<Full engineering report from Agent 5>",
  "zero_generator_contingency_agent7": "<Full contingency analysis from Agent 7>"
}
```

---

## 5. Setup & Usage

### Prerequisites
- **n8n** (`v1.20+` self-hosted or cloud) with `@n8n/n8n-nodes-langchain` enabled.
- An **OpenAI API credential** (or compatible LLM provider) connected to the 7 `LLM Agent *` nodes.

### Import & Run
1. Download `microgrid_8_agent_optimizer.json`.
2. In the n8n editor, go to **Workflows** → **Import from File...** and select `microgrid_8_agent_optimizer.json`.
3. Assign your OpenAI credential to `LLM Agent 1`, `LLM Agent 2a`, `LLM Agent 2b`, `LLM Agent 3`, `LLM Agent 4`, `LLM Agent 5`, `LLM Agent 6`, and `LLM Agent 7`.
4. *(Optional)* Adjust site parameters, load profiles, or hardware unit costs in the `Load System Parameters` node.
5. Click **Execute Workflow** to run the parallel multi-agent simulation and generate the synthesized BOM report.

### CLI Import
```bash
n8n import:workflow --input=microgrid_8_agent_optimizer.json
```
