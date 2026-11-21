# Dataset Card: MINEHUB Mining Safety & Equipment Inspection Dataset

## 1. Dataset Overview & Metadata

* **Dataset Name:** MINEHUB Mining Safety & Equipment Inspection Dataset
* **Project Name:** MINEHUB — Mining Site Operations & Equipment Safety Tracking System
* **Version:** 1.0.0
* **Date of Release:** November 2026
* **Author / Maintainer:** MULISA Israel Jakes (Student ID: 20252SEN228)
* **Domain:** Mining Site Operations, Equipment Predictive Maintenance, Industrial Risk Management, PL/SQL Database Engineering
* **Primary License:** Public Domain / Creative Commons CC BY 4.0 (Educational Use)
* **Repository Path:** `data/msha_safety_dataset.csv` & `sql/seed_data.sql`

---

## 2. Dataset Summary & Purpose

### 2.1 Purpose
The **MINEHUB Mining Safety & Equipment Inspection Dataset** provides structured, relational data capturing heavy machinery telemetry, physical safety inspection histories, operational hazard incidents, and real-time alert logs across active mining sites. The dataset is designed specifically to test and evaluate automated database routines, including:
1. Dynamic **Risk-Based Inspection (RBI)** scoring functions in PL/SQL.
2. Automated overdue inspection detection and asset status state-transitions.
3. Live visualization of warning and critical alert streams via **Grafana** dashboards.

### 2.2 Operational Context
In heavy industrial operations—such as underground coltan/gold mines and surface quarrying—equipment wear directly impacts workforce safety. This dataset models real-world operational conditions, allowing database systems to simulate how equipment age, operating environment (underground vs. surface), and failure history influence safety inspection cadences and alert triggers.

---

## 3. Data Provenance & Collection Methodology

### 3.1 Primary Sources
* **Official Data Source:** U.S. Mine Safety and Health Administration (MSHA) Accident Injuries & Mine Inspection Datasets via [Data.gov](https://catalog.data.gov/dataset/msha-accident-injuries-data-set).
* **Domain Benchmark:** National Institute of Statistics of Rwanda (NISR) Mining Sector Performance Statistics.
* **Standards Alignment:** Formatted according to federal mine inspection categories (Code of Federal Regulations CFR Title 30).

### 3.2 Collection & Synthesis Protocol
The dataset integrates verified real-world incident categories from public MSHA repositories combined with realistic site-specific operational parameters (e.g., regional mine locations in Rwanda: Rutongo, Rwamagana, Gatumba, Kivu, Muhanga). Data generation follows a deterministic PL/SQL seed generator script to ensure full relational integrity, realistic date distributions, and constraint compliance.

---

## 4. Relational Schema & Table Definitions

The dataset is structured across **5 interconnected relational tables** maintaining strict referential integrity.

```
+------------------+       +-------------------+       +-----------------------+
|      MINES       |1    * |     EQUIPMENT     |1    * | EQUIPMENT_INSPECTIONS |
|------------------|-------|-------------------|-------|-----------------------|
| PK  mine_id      |       | PK  equipment_id  |       | PK  inspection_id     |
|     mine_name    |       | FK  mine_id       |       | FK  equipment_id      |
|     location     |       |     equipment_code|       |     inspection_date   |
|     mine_type    |       |     equipment_type|       |     result_status     |
|     status       |       |     last_insp_date|       |     comments          |
+------------------+       |     next_due_date |       +-----------------------+
                           |     current_status|
                           +-------------------+
                                   | 1     | 1
                                   | *     | *
                           +-------v---+ +-v---------------------+
                           |  INCIDENTS| |  GRAFANA_ALERT_LOGS   |
                           +-----------+ +-----------------------+
```

### 4.1 Table: `MINES`
Stores site-level geographic and operational metadata.

| Column Name | Data Type | Constraints | Description / Allowed Values |
|---|---|---|---|
| `mine_id` | `NUMBER` | Primary Key, Identity | Unique internal identifier for each mining site. |
| `mine_name` | `VARCHAR2(100)` | NOT NULL | Official name of the mining operation (e.g., Rutongo Tin Mine). |
| `location_region` | `VARCHAR2(100)` | NOT NULL | Province/Region of the mine site (e.g., Northern Province, Rwanda). |
| `mine_type` | `VARCHAR2(50)` | CHECK | Mine classification: `'Underground'`, `'Surface'`, `'Open Pit'`. |
| `status` | `VARCHAR2(20)` | CHECK | Site operational state: `'ACTIVE'`, `'MAINTENANCE'`, `'INACTIVE'`. |

### 4.2 Table: `EQUIPMENT`
Master record for heavy machinery and inspection schedules.

| Column Name | Data Type | Constraints | Description / Allowed Values |
|---|---|---|---|
| `equipment_id` | `NUMBER` | Primary Key, Identity | Unique internal identifier for the machinery. |
| `mine_id` | `NUMBER` | Foreign Key (`MINES`) | Associated mine site where equipment is deployed. |
| `equipment_code` | `VARCHAR2(50)` | UNIQUE, NOT NULL | Unique serial tag (e.g., `EQ-101`, `EQ-204`). |
| `equipment_type` | `VARCHAR2(50)` | CHECK | Machinery type: `'Haul Truck'`, `'Excavator'`, `'Ventilation Fan'`, `'Crusher'`, `'Drill Rig'`, `'Conveyor Belt'`. |
| `model_number` | `VARCHAR2(50)` | NULLABLE | Manufacturer model identifier (e.g., `CAT-2022-X`). |
| `last_inspection_date`| `DATE` | NOT NULL | Date when the physical inspection was last completed. |
| `inspection_interval_days` | `NUMBER` | DEFAULT 30 | Scheduled days between inspections (dynamically updated by RBI engine). |
| `next_inspection_due` | `DATE` | NOT NULL | Target deadline for the upcoming safety inspection. |
| `current_status` | `VARCHAR2(20)` | CHECK | Operational state: `'OPERATIONAL'`, `'INSPECTION_OVERDUE'`, `'UNDER_REPAIR'`, `'OUT_OF_SERVICE'`. |

### 4.3 Table: `EQUIPMENT_INSPECTIONS`
Transactional history of physical field examinations.

| Column Name | Data Type | Constraints | Description / Allowed Values |
|---|---|---|---|
| `inspection_id` | `NUMBER` | Primary Key, Identity | Unique transactional log ID. |
| `equipment_id` | `NUMBER` | Foreign Key (`EQUIPMENT`) | Associated piece of machinery inspected. |
| `inspection_date` | `DATE` | NOT NULL | Timestamp when inspection occurred. |
| `inspector_code` | `VARCHAR2(30)` | NOT NULL | Anonymous inspector identification badge (e.g., `INSP-104`). |
| `result_status` | `VARCHAR2(30)` | CHECK | Diagnostic result: `'PASSED'`, `'MINOR_DEFECT'`, `'CRITICAL_FAIL'`, `'REPAIR_REQUIRED'`. |
| `overdue_days_logged`| `NUMBER` | DEFAULT 0 | Number of days overdue at the time of inspection. |
| `comments` | `VARCHAR2(255)` | NULLABLE | Field observations and inspector notes. |

### 4.4 Table: `SAFETY_INCIDENTS`
Operational safety incidents, equipment failures, and workplace hazards.

| Column Name | Data Type | Constraints | Description / Allowed Values |
|---|---|---|---|
| `incident_id` | `NUMBER` | Primary Key, Identity | Unique incident log ID. |
| `mine_id` | `NUMBER` | Foreign Key (`MINES`) | Mine site where the incident occurred. |
| `equipment_id` | `NUMBER` | Foreign Key (`EQUIPMENT`) | Machinery directly involved in the incident. |
| `incident_date` | `DATE` | NOT NULL | Date and time of incident occurrence. |
| `severity_level` | `VARCHAR2(20)` | CHECK | Risk rating: `'LOW'`, `'MEDIUM'`, `'HIGH'`, `'CRITICAL'`. |
| `incident_category` | `VARCHAR2(50)` | CHECK | Failure mode: `'Mechanical Failure'`, `'Electrical Hazard'`, `'Structural Defect'`, `'Operator Error'`, `'Overheating'`. |
| `description` | `VARCHAR2(255)` | NOT NULL | Detailed narrative of the incident event. |
| `injuries_reported` | `NUMBER` | DEFAULT 0 | Count of personnel injuries associated with the incident. |

### 4.5 Table: `GRAFANA_ALERT_LOGS`
Alert event history generated for Grafana dashboard monitoring.

| Column Name | Data Type | Constraints | Description / Allowed Values |
|---|---|---|---|
| `alert_id` | `NUMBER` | Primary Key, Identity | Unique alert log entry ID. |
| `equipment_id` | `NUMBER` | Foreign Key (`EQUIPMENT`) | Overdue equipment triggering the alert. |
| `alert_timestamp` | `TIMESTAMP` | DEFAULT CURRENT_TIMESTAMP | Timestamp when alert was logged by PL/SQL engine. |
| `alert_level` | `VARCHAR2(20)` | CHECK | Escalation tier: `'WARNING'` (1–14d overdue), `'CRITICAL'` (>14d overdue). |
| `days_overdue` | `NUMBER` | NOT NULL | Days elapsed past `next_inspection_due`. |
| `alert_status` | `VARCHAR2(20)` | CHECK | Management workflow state: `'ACTIVE'`, `'ACKNOWLEDGED'`, `'RESOLVED'`. |

---

## 5. Data Distributions & Statistical Summary

### 5.1 Volume Overview
* **Total Records:** 350+ relational records
* **`MINES` Records:** 5 active mining sites
* **`EQUIPMENT` Records:** 50 heavy machinery units
* **`EQUIPMENT_INSPECTIONS` Records:** 200 historical inspection logs
* **`SAFETY_INCIDENTS` Records:** 40 incident reports
* **`GRAFANA_ALERT_LOGS` Records:** 60+ generated alert records

### 5.2 Key Attribute Distributions

#### Equipment Breakdown by Type
* Haul Truck: 20%
* Excavator: 20%
* Drill Rig: 16.7%
* Ventilation Fan: 16.7%
* Crusher: 13.3%
* Conveyor Belt: 13.3%

#### Inspection Outcomes (`EQUIPMENT_INSPECTIONS`)
* `PASSED`: 65%
* `MINOR_DEFECT`: 22.5%
* `CRITICAL_FAIL`: 12.5%

#### Incident Severity (`SAFETY_INCIDENTS`)
* `LOW`: 40%
* `MEDIUM`: 35%
* `HIGH`: 17.5%
* `CRITICAL`: 7.5%

---

## 6. Ethical Considerations & Privacy (PII)

* **Zero Personally Identifiable Information (PII):** All human actors are represented using anonymized alphanumeric badges (e.g., `INSP-101`, `INSP-102`). No real names, national IDs, contact numbers, or home addresses are present.
* **Open Data Compliance:** Derived directly from public domain U.S. Government data (MSHA Data.gov) and open educational frameworks.
* **Safety & Security:** Machinery serial numbers and location titles are stylized for academic evaluation and do not expose sensitive commercial operations.

---

## 7. Intended Use & Applications

### 7.1 Primary Use Cases
1. **PL/SQL Algorithm Validation:** Testing the `FN_CALCULATE_RBI_SCORE` function to verify that high-incident equipment receives shorter inspection windows.
2. **Automated State Machine Execution:** Testing `SP_PROCESS_OVERDUE_INSPECTIONS` to ensure `EQUIPMENT.current_status` updates seamlessly when `next_inspection_due < SYSDATE`.
3. **Database Trigger Testing:** Testing `TRG_INCIDENT_OVERRIDE` to confirm that high-severity incidents force fleet-wide re-inspection triggers.
4. **Grafana Dashboard Integration:** Connecting `VW_GRAFANA_OVERDUE_ALERTS` to Grafana panels to verify warning and critical alert visual thresholds.

### 7.2 System Limitations
* The dataset is tailored for database architecture evaluation and real-time alert testing; it should not be used as live clinical/safety advice for actual mining operations without localized calibration.

---

## 8. Versioning & Maintenance

| Version | Date | Summary of Changes |
|---|---|---|
| `v1.0.0` | 2026-11-07 | Initial release with 5 tables, 350+ seed records, and MSHA alignment. |
