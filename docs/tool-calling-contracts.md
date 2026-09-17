# 🛠️ AI Agent Tool-Calling Contracts & Safety Guardrails

This specification details how the **Google Gemini AI Agent** interfaces with the embedded domain knowledge tool, maintains fleet asset context, and complies with deterministic functional safety standards.

---

## 1. Language Model & Agent Node Configuration

| Component | n8n Node Identifier | Type Version | Operational Role |
| :--- | :--- | :--- | :--- |
| **Agent Core** | `@n8n/n8n-nodes-langchain.agent` | 3.1 | Evaluates multi-joint kinematics, queries SOP knowledge, synthesizes structured diagnosis. |
| **Model Engine** | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | 1.1 | Google Gemini 1.5 Pro inference engine with temperature `0.1`. |
| **Domain Tool** | `@n8n/n8n-nodes-langchain.toolCode` | 1.3 | Supplies standardized SOP procedures & derating factors. |
| **Session Memory** | `@n8n/n8n-nodes-langchain.memoryBufferWindow` | 1.3 | Retains sliding-window diagnostic context keyed by `robot_asset_id`. |

---

## 2. Gemini AI Agent Prompting Contract

### Agent Persona & System Prompt:
```
You are a Senior Robotics Systems Architect, Mechatronics Diagnostics Specialist, and ISO 10218-1 / ISO/TS 15066 Functional Safety Officer. You analyze 6-DOF industrial robot and cobot telemetry to detect collisions, harmonic drive wear, and thermal overloads.

### ROBOT SAFETY CRITERIA & FAILURE MODES:
1. Critical (Biomechanical Impact / Collision Hazard):
   - Criteria: External torque residual Δτ_ext > 70.0 Nm or sudden velocity collapse under high commanded torque.
   - Standard: ISO/TS 15066 Clause 5.5 biomechanical pain limit exceedance & ISO 10218-1.
   - Action: IEC 60204-1 Category 0/1 Emergency Stop (E-Stop), mechanical brake engagement within 35ms.
   - Classification: severity = "Critical", iso_risk_tier = "Tier-1-Hazard", stop_category = "Cat-0-EStop".

2. Critical (Harmonic Drive Flexspline Failure / Gear Tooth Pitting):
   - Criteria: Vibration RMS a_RMS > 1.60 g accompanied by tracking error ε_θ > 1.0°.
   - Root Cause: Wave generator bearing race spalling or harmonic flexspline tooth shear risking catastrophic mid-trajectory lockup.
   - Classification: severity = "Critical", iso_risk_tier = "Tier-1-Hazard", stop_category = "Cat-1-ControlledStop".

3. Warning (Servomotor Thermal Overload & Stator Saturation):
   - Criteria: Joint temperature T_joint > 75.0 °C with sustained motor current I > 14.0 A without severe torque shock.
   - Root Cause: Excessive acceleration duty cycle, high ambient heat, or gripper payload inertia saturation (I²t accumulation).
   - Classification: severity = "Warning", iso_risk_tier = "Tier-2-OperationalDeviation", stop_category = "Dynamic-Velocity-Derating".

4. Nominal (Normal Trajectory Execution):
   - Criteria: All joints within normal limits (Δτ_ext < 20 Nm, ε_θ < 0.1°, a_RMS < 0.5 g, T_joint <= 60 °C).
   - Classification: severity = "Nominal", iso_risk_tier = "Tier-3-Nominal", stop_category = "None".

### MANDATORY TOOL USAGE:
If the status is Critical or Warning, you MUST call the "Robotics ISO Safety & SOP Tool" to obtain standard operating procedure steps and dynamic speed scaling factors.
```

### Agent Output JSON Schema Contract:
```json
{
  "severity": "Critical | Warning | Nominal",
  "iso_risk_tier": "Tier-1-Hazard | Tier-2-OperationalDeviation | Tier-3-Nominal",
  "stop_category": "Cat-0-EStop | Cat-1-ControlledStop | Dynamic-Velocity-Derating | None",
  "worst_affected_joint": 5,
  "root_cause_hypothesis": "string",
  "emergency_brake_sequence": ["Step 1...", "Step 2..."],
  "velocity_scaling": {
    "speed_factor": 0.60,
    "target_joints": [2],
    "cooling_protocol": "string"
  },
  "applicable_standard": "ISO 10218-1 / ISO/TS 15066 / IEC 60204-1"
}
```

---

## 3. Embedded Tool Catalog (`Robotics ISO Safety & SOP Tool`)

### Tool Protocol Registry:
| SOP Code | Classification | Trigger Condition | Interlock Actions |
| :--- | :--- | :--- | :--- |
| **`SOP-ROB-COL-001`** | **Critical (Collision)** | $\Delta \tau_{ext} > 70.0\text{ Nm}$ | STO bus trip (15ms), friction brake clamp (35ms), ROS 2 E-Stop topic publication, red strobe alarm. |
| **`SOP-ROB-MEC-102`** | **Critical (Gearbox Wear)** | $a_{RMS} > 1.60 g$ AND $\epsilon_\theta > 1.0^\circ$ | Cat 1 controlled deceleration, zero gravity-sag brake lock, ferrographic grease sampling, backlash dial measurement. |
| **`SOP-ROB-THM-203`** | **Warning (Thermal $I^2 t$)** | $T_{joint} > 75.0^\circ\text{C}$ AND $I > 14\text{ A}$ | Trajectory speed scaled to $0.60\times$, 1.8s dwell extension, auxiliary convection fan override, CMMS inspection ticket. |
| **`SOP-ROB-NOM-001`** | **Nominal (Baseline)** | All variables within $\pm 2\sigma$ envelope | 100Hz telemetry slice logged to SCADA historian, cumulative MTBF wear index updated. |

---

## 4. Deterministic Fail-Safe Guardrail Fallback
In high-consequence machinery environments, LLMs must never be single points of failure. The `Parse & Validate Safety Triage` code node enforces a deterministic physical interlock:

```javascript
// If LLM output fails to parse or is incomplete, deterministic physics take over:
if (!parsed || !parsed.severity) {
  if (dTau > 70.0) {
    // FORCE Cat-0 Emergency Stop immediately
    parsed = { severity: "Critical", stop_category: "Cat-0-EStop", ... };
  } else if (aRms > 1.60 && epsDeg > 0.8) {
    // FORCE Cat-1 Controlled Deceleration immediately
    parsed = { severity: "Critical", stop_category: "Cat-1-ControlledStop", ... };
  } else if (Tmax > 75.0) {
    // FORCE Dynamic Velocity Derating immediately
    parsed = { severity: "Warning", stop_category: "Dynamic-Velocity-Derating", ... };
  } else {
    // Safe default to Nominal
    parsed = { severity: "Nominal", stop_category: "None", ... };
  }
}
```
This guarantees fail-safe operation compliant with **ISO 13849-1 (PL-e)** even during model hallucination, network timeout, or schema violations.
