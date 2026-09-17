# 📡 Ingestion, Response & Dispatch Payload Schemas

This document defines the interface contracts for the **Autonomous 6-DOF Robotic Manipulator Prognostics & Safety Guardrail Agent** in n8n.

---

## 1. Synchronous Telemetry Ingestion Endpoint
- **HTTP Method:** `POST`
- **Path:** `/webhook/robotics-telemetry-triage`
- **Content-Type:** `application/json`
- **Authentication:** Standard header / token (configurable in n8n Webhook node options)

### Option A: Preset Industrial Scenario Ingestion
Ideal for automated QA test runs, CI/CD healthchecks, and SCADA simulation:
```json
{
  "scenario": "critical_collision"
}
```
*Supported preset scenarios:* `critical_collision`, `critical_harmonic_wear`, `warning_thermal_overload`, `nominal`.

### Option B: Real-Time Multi-Axis Telemetry Stream (ROS 2 / DDS / OPC-UA)
```json
{
  "robot_id": "ROBOT-UR10E-CELL-04",
  "joint_data": [
    { "joint": 1, "angle_cmd_deg": 45.0, "angle_meas_deg": 45.02, "torque_meas_nm": 24.5, "current_a": 5.2, "temp_c": 44.1, "imu_accel_g": 0.18 },
    { "joint": 2, "angle_cmd_deg": -80.0, "angle_meas_deg": -79.94, "torque_meas_nm": 68.2, "current_a": 11.4, "temp_c": 56.3, "imu_accel_g": 0.32 },
    { "joint": 3, "angle_cmd_deg": 115.0, "angle_meas_deg": 111.80, "torque_meas_nm": 112.4, "current_a": 19.8, "temp_c": 62.0, "imu_accel_g": 0.85 },
    { "joint": 4, "angle_cmd_deg": -35.0, "angle_meas_deg": -34.98, "torque_meas_nm": 18.0, "current_a": 4.1, "temp_c": 42.5, "imu_accel_g": 0.22 },
    { "joint": 5, "angle_cmd_deg": 90.0, "angle_meas_deg": 86.40, "torque_meas_nm": 89.6, "current_a": 16.5, "temp_c": 59.8, "imu_accel_g": 0.74 },
    { "joint": 6, "angle_cmd_deg": 0.0, "angle_meas_deg": 0.05, "torque_meas_nm": 8.5, "current_a": 2.1, "temp_c": 38.2, "imu_accel_g": 0.15 }
  ],
  "expected_model_torques_nm": [23.0, 65.0, 38.0, 16.0, 14.0, 7.5],
  "tcp_speed_m_s": 0.02,
  "vibration_window_g": [0.35, 0.42, 0.95, 1.25, 0.88]
}
```

### Ingestion Field Specifications:
| Field | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `robot_id` | `string` | Optional | Unique fleet asset tag (defaults to `ROBOT-UR10E-CELL-04`). |
| `scenario` | `string` | Optional | Name of benchmark scenario to execute. |
| `joint_data` | `array[object]` | Optional | Array of 6 joint telemetry readings. |
| `joint_data[i].joint` | `integer` | Required | Joint index ($1 \dots 6$). |
| `joint_data[i].angle_cmd_deg` | `number` | Required | Target commanded position in degrees. |
| `joint_data[i].angle_meas_deg`| `number` | Required | Measured position from joint optical encoder. |
| `joint_data[i].torque_meas_nm`| `number` | Required | Joint torque measured via strain-gauge / current sensor. |
| `joint_data[i].current_a` | `number` | Required | Stator phase current in Amperes. |
| `joint_data[i].temp_c` | `number` | Required | Motor winding RTD temperature in °C. |
| `joint_data[i].imu_accel_g` | `number` | Required | Accelerometer vibration reading in $g$. |
| `expected_model_torques_nm` | `array[number]`| Optional | Rigid-body dynamic model torque prediction (defaults to standard 6-DOF baseline). |
| `tcp_speed_m_s` | `number` | Optional | Cartesian Tool Center Point linear velocity. |
| `vibration_window_g` | `array[number]`| Optional | Rolling window of acceleration samples for RMS calculation. |

---

## 2. Synchronous Webhook HTTP Response Schema
When the webhook finishes evaluation, n8n immediately returns HTTP `200 OK` with the complete structured diagnostic payload:

```json
{
  "status": "success",
  "timestamp": "2026-09-17T15:13:39.507Z",
  "robot_asset_id": "ROBOT-UR10E-CELL-04",
  "severity": "Critical",
  "iso_risk_tier": "Tier-1-Hazard",
  "stop_category": "Cat-0-EStop",
  "root_cause_hypothesis": "Newton-Euler torque residual exceedance (75.6 Nm) on Joint 5...",
  "metrics": {
    "max_tracking_error_deg": 3.6,
    "max_torque_residual_nm": 75.6,
    "worst_affected_joint": 5,
    "max_joint_temp_c": 62.0,
    "max_joint_current_a": 19.8,
    "vibration_rms_g": 0.84,
    "tcp_speed_m_s": 0.02
  },
  "dispatch_payload": {
    "incident_id": "ESTOP-MU5O6D8A-239",
    "safety_action": "EMERGENCY_STOP_ACTUATION",
    "stop_category": "Cat-0-EStop",
    "affected_joint": 5,
    "actuation_commands": [
      { "target": "FIELDBUS_SAFETY_BUS", "command": "TRIP_STO_SAFE_TORQUE_OFF", "latency_ms": 12 },
      { "target": "JOINT_5_BRAKE", "command": "CLAMP_MECHANICAL_FRICTION_BRAKE", "latency_ms": 28 },
      { "target": "ROS2_TOPIC_SAFETY_ESTOP", "command": "PUBLISH_COLLISION_STATE" },
      { "target": "CELL_BEACON", "command": "ACTUATE_RED_STROBE_AND_SIREN" }
    ],
    "emergency_brake_sequence": [ ... ]
  },
  "markdown_report": "# 🤖 AUTONOMOUS ROBOTIC MANIPULATOR DIAGNOSTICS & SAFETY REPORT\n..."
}
```

---

## 3. Critical Safety Dispatch Payload (Branch 0: Cat 0/1 Stop)
Dispatched via HTTP POST to the cell safety PLC / ROS 2 safety bridge:
```json
{
  "incident_id": "ESTOP-MU5O6D8A-239",
  "timestamp": "2026-09-17T15:13:38.938Z",
  "robot_asset_id": "ROBOT-UR10E-CELL-04",
  "safety_action": "EMERGENCY_STOP_ACTUATION",
  "stop_category": "Cat-0-EStop",
  "affected_joint": 5,
  "functional_safety": {
    "category": "ISO 13849-1 Category 4 / PL-e",
    "safety_integrity_level": "IEC 62061 SIL-3",
    "brake_actuation_time_budget_ms": 35
  },
  "root_cause": "Newton-Euler torque residual exceedance (75.6 Nm) on Joint 5...",
  "telemetry_snapshot": {
    "max_tracking_error_deg": 3.6,
    "max_torque_residual_nm": 75.6,
    "worst_affected_joint": 5,
    "max_joint_temp_c": 62.0,
    "max_joint_current_a": 19.8,
    "vibration_rms_g": 0.84,
    "tcp_speed_m_s": 0.02
  },
  "actuation_commands": [
    { "target": "FIELDBUS_SAFETY_BUS", "command": "TRIP_STO_SAFE_TORQUE_OFF", "latency_ms": 12 },
    { "target": "JOINT_5_BRAKE", "command": "CLAMP_MECHANICAL_FRICTION_BRAKE", "latency_ms": 28 },
    { "target": "ROS2_TOPIC_SAFETY_ESTOP", "command": "PUBLISH_COLLISION_STATE", "payload": { "joint": 5, "status": "TRIPPED" } },
    { "target": "CELL_BEACON", "command": "ACTUATE_RED_STROBE_AND_SIREN", "mode": "CONTINUOUS" }
  ],
  "emergency_brake_sequence": [
    "Send hardware E-Stop interrupt via Safety Fieldbus (PROFIsafe / CIP Safety) within 15ms.",
    "Actuate electromechanical fail-safe friction brakes on Joints 1 through 6 (response time <= 35ms).",
    "De-energize servomotor PWM inverters (Safe Torque Off - STO, Cat 4 / PL-e).",
    "Broadcast ROS 2 message on topic /safety/emergency_stop with collision joint vector.",
    "Flash cell red beacon, sound audible 85dB alarm, and latch safety interlock requiring manual key reset."
  ],
  "pagerduty_incident": {
    "summary": "[CRITICAL TIER-1] Robot Manipulator Safety E-Stop Triggered (ESTOP-MU5O6D8A-239)",
    "urgency": "high",
    "assigned_team": "Robotics Safety & Cell Maintenance Operations"
  }
}
```

---

## 4. Operational Derating & CMMS Payload (Branch 1: Warning)
Dispatched to the robot trajectory controller and maintenance management system:
```json
{
  "ticket_id": "CMMS-ROB-MU5O6KEO",
  "timestamp": "2026-09-17T15:13:48.240Z",
  "robot_asset_id": "ROBOT-UR10E-CELL-04",
  "safety_action": "DYNAMIC_TRAJECTORY_DERATING",
  "stop_category": "Dynamic-Velocity-Derating",
  "affected_joint": 2,
  "functional_safety": {
    "classification": "ISO 13849-1 Category 2 / PL-c",
    "safety_level": "Operational Speed Limiting Mode"
  },
  "root_cause": "Servomotor thermal overload and stator saturation due to sustained I²t Joule heating...",
  "robot_controller_reconfiguration": {
    "topic": "/robot_controller/trajectory_override",
    "speed_factor": 0.60,
    "acceleration_derate_pct": 40,
    "dwell_time_compensation_s": 1.8,
    "forced_cooling_fan_override": "ON_MAX_FLOW"
  },
  "cmms_maintenance_tasks": [
    "Inspect Joint 2 servomotor stator winding thermistor and thermal paste.",
    "Verify heat sink cooling duct airflow and clear particulate intake filter.",
    "Conduct cycle time vs thermal dissipation telemetry review on cell controller."
  ]
}
```
