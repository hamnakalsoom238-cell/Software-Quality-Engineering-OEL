# Task 2: Change Requests & Traceability Matrix (RTM)

### 1. Version-Controlled Change Log

| Req. ID | Old Text (Baseline) | New Text (Updated) | Reason for Change | Status |
| :--- | :--- | :--- | :--- | :--- |
| **FR-05 / SAF-01** | The robot shall enter Safe Mode when a critical battery or thermal condition is detected. | The robot shall enter Safe Mode within **3 seconds** when chamber or battery temperature exceeds the critical threshold or battery capacity falls below the emergency level. | **Measurability & Performance:** Replaced vague triggering conditions and unbounded response times with a strict **3-second threshold** and explicit environmental triggers. | **Modified** |
| **FR-06 / NFR-02** | The system shall support communication with multiple robots simultaneously. | The system shall support **at least 20 simultaneously connected robots** across all active deployment zones. | **Scalability & Scope Alignment:** Aligned with the pilot release target of serving 20 concurrent clinics, turning an ambiguous requirement into a verifiable capacity metric. | **Modified** |
| **FR-03 / SEC-01** | Only authenticated operators shall be permitted to issue robot commands. | The system shall require **authenticated and role-authorised operators** before accepting and executing any remote robot command. | **Security & Access Control:** Clarified that identification alone is insufficient; explicit role-based permissions must be evaluated before command dispatch. | **Modified** |

---

### 2. Justification of Measurability and Testability

* **CR-01 (Safe Mode Latency):** The original wording used an unspecified timeline ("when... detected"), making it impossible to write an objective pass/fail test. The updated version introduces a strict **3-second execution window** and explicit parameter bounds (critical temperature and emergency battery levels), allowing automated test rigs to inject threshold-breaching inputs and measure response latency down to the millisecond.
* **CR-02 (Concurrent Robot Capacity):** Saying "multiple robots" is subjective and dependent on system load limits. Specifying **at least 20 simultaneously connected robots** provides a concrete load-testing criterion, enabling test engineers to spin up 20 virtual robot telemetry streams and verify that heartbeat, command routing, and telemetry logs function without packet loss or timeout.
* **CR-03 (Role-Based Authorization):** The original requirement only checked authentication (identity verification). The updated text enforces **role authorization**, distinguishing between a standard operator and a supervisory administrator. This is testable by attempting to dispatch high-privilege commands (such as overriding a safety lock) using a standard operator role and verifying that the system rejects the command.

---

### 3. Multi-Layer Traceability Matrix (RTM)

| Requirement ID | State / Transition (Task 3) | Operation ID / Name (Task 4) | Formal Constraint ID (Task 5) | Test Case ID | Risk Level |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **FR-05** (Safe Mode) | `NAVIGATING` $\rightarrow$ `SAFE_MODE` | `OP-03: TriggerSafeMode` | `C-02` | `TC-01`, `TC-07` | **Critical** |
| **FR-06** (Scalability) | `IDLE` / `NAVIGATING` | `OP-09: BroadcastTelemetry` | `C-08` | `TC-02`, `TC-08` | **High** |
| **FR-03** (Command Auth) | `IDLE` $\rightarrow$ `NAVIGATING` | `OP-01: AuthenticateOperator` | `C-08` | `TC-03` | **High** |
| **FR-01** (Mission Start) | `IDLE` $\rightarrow$ `NAVIGATING` | `OP-02: LoadDeliveryProfile` | `C-01` | `TC-04` | **Medium** |
| **FR-02** (Obstacle Handling)| `NAVIGATING` $\rightarrow$ `AVOIDING` | `OP-05: HandleObstacle` | `C-01` | `TC-05` | **High** |
| **FR-04** (Railway Crossing) | `NAVIGATING` $\rightarrow$ `CROSSING_RAIL`| `OP-08: VerifyLevelCrossing`| `C-04` | `TC-06` | **Critical** |

---

### 4. Traceable Test Cases & Impact Analysis

#### End-to-End Test Suite
* **`TC-01` (Safe Mode Thermal Trigger):** Inject a chamber temperature of $42^\circ\text{C}$ (exceeding critical limits). *Expected Result:* Robot enters `SAFE_MODE` within $2.4\text{ seconds}$, halts actuators, triggers an audible alert, and writes an emergency log entry.
* **`TC-02` (Multi-Robot Load Test):** Connect 20 simulated autonomous robots streaming telemetry concurrently to the Mission Control Centre. *Expected Result:* All 20 connection streams remain active with packet round-trip times under $500\text{ ms}$.
* **`TC-03` (Role-Based Command Rejection):** Issue a warehouse-return command using an operator profile restricted to read-only telemetry. *Expected Result:* Command is rejected with an `UnauthorizedRole` error code and logged as a security event.
* **`TC-04` (Valid Mission Dispatch):** Transmit a verified delivery profile and valid dispatch command to an `IDLE` robot. *Expected Result:* Robot transitions to `NAVIGATING`, loads environmental parameters, and acknowledges via telemetry.
* **`TC-05` (Obstacle Interruption & Resumption):** Simulate a dynamic pedestrian obstruction while the robot is in `NAVIGATING`. *Expected Result:* Robot suspends travel, enters `AVOIDING_OBSTACLE`, clears the path after $12\text{ seconds}$, and seamlessly resumes `NAVIGATING`.
* **`TC-06` (Railway Level-Crossing Gate Check):** Approach a railway crossing while `Train_Present = TRUE`. *Expected Result:* Robot brakes to a complete stop before the safety perimeter while the barrier remains closed (`Barrier_Open = FALSE`).
* **`TC-07` (Battery Threshold Breach):** Draw battery capacity down to $9\%$ (below emergency threshold during navigation). *Expected Result:* Robot aborts delivery, triggers `SAFE_MODE`, and recalculates an emergency return path to the warehouse within the $3$-second window.
* **`TC-08` (Telemetry Event Logging):** Verify that every state transition (`IDLE` $\rightarrow$ `NAVIGATING` $\rightarrow$ `DELIVERING`) creates an immutable database record with timestamp and operator signature. *Expected Result:* 100% audit log completeness verified against system event counters.

#### Change Impact Analysis (Regression Tests Required)
1. **Re-run `TC-01` (CR-01 Impact):** Validate that the 3-second deadline for Safe Mode is strictly maintained under heavy thermal/battery load.
2. **Re-run `TC-02` (CR-02 Impact):** Verify 20-robot concurrent connections and telemetry stability under peak load.
3. **Re-run `TC-03` (CR-03 Impact):** Test command permissions across role matrices to ensure strict authorization enforcement.
