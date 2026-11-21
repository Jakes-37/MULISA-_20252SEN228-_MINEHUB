# MINEHUB: Oracle PL/SQL Database Data Dictionary & Schema Specification

## 1. System Overview & Data Architecture

The **MINEHUB Data Dictionary** provides a comprehensive technical specification of the relational schema, constraints, data types, and PL/SQL integration interfaces for the MINEHUB Mining Site Operations & Heavy Equipment Safety Tracking System. 

Designed for implementation in an **Oracle 19c/21c Database Engine**, the architecture supports automated Risk-Based Inspection (RBI) scoring, real-time overdue inspection tracking, and live Grafana dashboard alerting.

### 1.1 Global Database Naming Conventions
* **Table Names**: UPPERCASE, pluralized nouns (`MINES`, `EQUIPMENT`, `EQUIPMENT_INSPECTIONS`, `SAFETY_INCIDENTS`, `GRAFANA_ALERT_LOGS`).
* **Column Names**: lowercase with underscores (`mine_id`, `equipment_code`, `next_inspection_due`).
* **Primary Key Constraint Naming**: `PK_<TABLE_NAME>` (e.g., `PK_MINES`).
* **Foreign Key Constraint Naming**: `FK_<CHILD_TABLE>_<PARENT_TABLE>` (e.g., `FK_EQUIPMENT_MINES`).
* **Check Constraint Naming**: `CK_<TABLE_NAME>_<COLUMN_NAME>` (e.g., `CK_EQUIPMENT_STATUS`).
* **Unique Constraint Naming**: `UK_<TABLE_NAME>_<COLUMN_NAME>` (e.g., `UK_EQUIPMENT_CODE`).

---

## 2. Global Entity Inventory

The MINEHUB database consists of five primary relational tables and four supporting database objects (views, functions, procedures, and triggers).

| Table / Object Name | Entity Type | Primary Function & Business Purpose | Record Volume Range |
|---|---|---|---|
| `MINES` | Master Entity | Stores active and inactive mining site locations, environment severity factors, and regional classifications. | 10 – 50 sites |
| `EQUIPMENT` | Master Entity | Tracks heavy machinery assets, dynamic RBI risk scores, inspection cadences, and operational availability states. | 200 – 1,000 assets |
| `EQUIPMENT_INSPECTIONS` | Transactional Entity | Records completed physical field inspections, diagnostic ratings, inspector codes, and maintenance notes. | 5,000 – 50,000 logs |
| `SAFETY_INCIDENTS` | Transactional Entity | Logs workplace accidents, mechanical failures, hazard severity levels, and injury metrics. | 500 – 5,000 reports |
| `GRAFANA_ALERT_LOGS` | System Log Entity | Captures real-time overdue inspection alerts and severity states generated for Grafana dashboard visualization. | 1,000 – 20,000 alerts |
| `VW_GRAFANA_OVERDUE_ALERTS` | Database View | Calculates real-time date math (`SYSDATE - next_inspection_due`) for active visual monitoring panels. | Dynamic (Read-Only) |
| `FN_CALCULATE_RBI_SCORE` | PL/SQL Stored Function | Computes dynamic numerical risk scores ($1\text{--}100$) and assigns inspection intervals ($7\text{--}30$ days). | Executed on demand/trigger |
| `SP_PROCESS_OVERDUE_INSPECTIONS` | PL/SQL Stored Procedure | Batch scans active machinery for passed due dates, updates statuses, and writes alert log entries. | Scheduled daily / batch |
| `TRG_INCIDENT_OVERRIDE` | PL/SQL Trigger | Intercepts high-severity incident entries to force fleet-wide re-inspections across affected mine units. | Event-driven (AFTER INSERT) |

---

## 3. Detailed Data Dictionary by Entity

### 3.1 Entity: `MINES`
* **Description**: Holds master records for mining concessions, operating sites, geographic regions, and mining environment types. The environment type determines the baseline severity factor used in RBI risk calculations.
* **Primary Key**: `mine_id`

#### Column Specifications
| Column Name | Data Type & Length | Nullable | Key | Default Value | Business Rules & Description |
|---|---|:---:|:---:|---|---|
| `mine_id` | `NUMBER(10,0)` | **NOT NULL** | **PK** | `SEQ_MINES.NEXTVAL` | Unique surrogate primary key identifying each mine site. |
| `mine_code` | `VARCHAR2(20)` | **NOT NULL** | **UK** | None | Unique alphanumeric site identifier (e.g., `MINE-RUT-01`, `MINE-RWA-02`). |
| `mine_name` | `VARCHAR2(100)` | **NOT NULL** | None | None | Full commercial or geographic name of the mining site (e.g., `Rutongo Tantalite Mine`). |
| `location_region` | `VARCHAR2(50)` | **NOT NULL** | None | None | Province or administrative region (e.g., `Northern Province`, `Kigali Sector`). |
| `environment_type` | `VARCHAR2(20)` | **NOT NULL** | None | `'SURFACE'` | Operating environment category. Must be one of: `'UNDERGROUND'`, `'OPEN_PIT'`, `'SURFACE'`. |
| `environment_factor` | `NUMBER(3,0)` | **NOT NULL** | None | `5` | Severity risk weight added to RBI scoring: Underground = `20`, Open Pit = `10`, Surface = `5`. |
| `operational_status` | `VARCHAR2(20)` | **NOT NULL** | None | `'ACTIVE'` | Site operating status. Must be one of: `'ACTIVE'`, `'SUSPENDED'`, `'CLOSED'`. |
| `created_at` | `TIMESTAMP` | **NOT NULL** | None | `CURRENT_TIMESTAMP` | System timestamp when the mine record was initially created. |

---

### 3.2 Entity: `EQUIPMENT`
* **Description**: Central registry for heavy mining machinery (e.g., haul trucks, ventilation fans, crushers, drill rigs). Stores dynamic risk scores, next inspection due dates, and operational status states.
* **Primary Key**: `equipment_id`
* **Foreign Key**: `mine_id` references `MINES(mine_id)`

#### Column Specifications
| Column Name | Data Type & Length | Nullable | Key | Default Value | Business Rules & Description |
|---|---|:---:|:---:|---|---|
| `equipment_id` | `NUMBER(10,0)` | **NOT NULL** | **PK** | `SEQ_EQUIPMENT.NEXTVAL` | Unique surrogate primary key identifying each machinery asset. |
| `equipment_code` | `VARCHAR2(30)` | **NOT NULL** | **UK** | None | Unique serial or asset code (e.g., `EQ-CAT-777D-004`, `EQ-FAN-UG-012`). |
| `equipment_type` | `VARCHAR2(50)` | **NOT NULL** | None | None | Machinery category (e.g., `'HAUL_TRUCK'`, `'VENTILATION_FAN'`, `'EXCAVATOR'`, `'DRILL_RIG'`). |
| `model_number` | `VARCHAR2(50)` | **NOT NULL** | None | None | Manufacturer model designation (e.g., `Caterpillar 777D`, `Joy Mining WX-12`). |
| `manufacture_year` | `NUMBER(4,0)` | **NOT NULL** | None | None | Year of manufacture (must be between 1980 and current year). Used in age calculation. |
| `mine_id` | `NUMBER(10,0)` | **NOT NULL** | **FK** | None | Foreign key linking the machine to its assigned operating mine site (`MINES.mine_id`). |
| `risk_score` | `NUMBER(5,2)` | **NOT NULL** | None | `20.00` | Dynamic RBI Risk Score ($1.00\text{--}100.00$) calculated by `FN_CALCULATE_RBI_SCORE`. |
| `inspection_interval_days` | `NUMBER(3,0)` | **NOT NULL** | None | `30` | Calculated inspection cadence in days ($7$, $15$, or $30$ days based on risk score). |
| `last_inspection_date` | `DATE` | NULL | None | NULL | Date of the most recently completed physical field inspection. |
| `next_inspection_due` | `DATE` | **NOT NULL** | None | `SYSDATE + 30` | Target deadline for the next mandatory physical safety check. |
| `current_status` | `VARCHAR2(25)` | **NOT NULL** | None | `'OPERATIONAL'` | Current asset state: `'OPERATIONAL'`, `'INSPECTION_OVERDUE'`, `'UNDER_REPAIR'`, `'OUT_OF_SERVICE'`. |
| `updated_at` | `TIMESTAMP` | **NOT NULL** | None | `CURRENT_TIMESTAMP` | System timestamp of the latest state or risk score update. |

---

### 3.3 Entity: `EQUIPMENT_INSPECTIONS`
* **Description**: Transactional table capturing every completed physical field inspection. Logs diagnostic ratings, inspector credentials, diagnostic measurements, and inspector comments.
* **Primary Key**: `inspection_id`
* **Foreign Key**: `equipment_id` references `EQUIPMENT(equipment_id)`

#### Column Specifications
| Column Name | Data Type & Length | Nullable | Key | Default Value | Business Rules & Description |
|---|---|:---:|:---:|---|---|
| `inspection_id` | `NUMBER(12,0)` | **NOT NULL** | **PK** | `SEQ_INSPECTIONS.NEXTVAL` | Unique surrogate primary key identifying each inspection log. |
| `equipment_id` | `NUMBER(10,0)` | **NOT NULL** | **FK** | None | Foreign key linking the inspection record to the specific asset (`EQUIPMENT.equipment_id`). |
| `inspection_date` | `TIMESTAMP` | **NOT NULL** | None | `CURRENT_TIMESTAMP` | Exact timestamp when the physical inspection was performed. |
| `inspector_code` | `VARCHAR2(20)` | **NOT NULL** | None | None | Badge or employee code of the certified Field Safety Inspector (e.g., `INSP-7842`). |
| `result_status` | `VARCHAR2(20)` | **NOT NULL** | None | None | Diagnostic outcome: `'PASSED'`, `'MINOR_DEFECT'`, or `'CRITICAL_FAIL'`. |
| `operating_hours` | `NUMBER(8,1)` | **NOT NULL** | None | `0.0` | Accumulated engine operating hours logged at the time of inspection. |
| `temperature_reading_c` | `NUMBER(5,1)` | NULL | None | NULL | Sensor/thermal camera reading in Celsius (used for overheating diagnostics). |
| `vibration_score` | `NUMBER(4,2)` | NULL | None | NULL | Mechanical vibration amplitude index ($0.00\text{--}10.00$). |
| `inspector_comments` | `VARCHAR2(500)` | NULL | None | NULL | Detailed technical observations, defect descriptions, or maintenance recommendations. |

---

### 3.4 Entity: `SAFETY_INCIDENTS`
* **Description**: Records workplace safety incidents, mechanical breakdowns, structural failures, and near-miss hazards occurring at mine sites. Used to calculate historical failure rates and trigger emergency cascade overrides.
* **Primary Key**: `incident_id`
* **Foreign Keys**: `mine_id` references `MINES(mine_id)`, `equipment_id` references `EQUIPMENT(equipment_id)`

#### Column Specifications
| Column Name | Data Type & Length | Nullable | Key | Default Value | Business Rules & Description |
|---|---|:---:|:---:|---|---|
| `incident_id` | `NUMBER(10,0)` | **NOT NULL** | **PK** | `SEQ_INCIDENTS.NEXTVAL` | Unique surrogate primary key identifying each safety incident report. |
| `mine_id` | `NUMBER(10,0)` | **NOT NULL** | **FK** | None | Foreign key linking the incident to the mine site (`MINES.mine_id`). |
| `equipment_id` | `NUMBER(10,0)` | NULL | **FK** | NULL | Optional foreign key linking the incident to a specific machine (`EQUIPMENT.equipment_id`). |
| `incident_date` | `TIMESTAMP` | **NOT NULL** | None | `CURRENT_TIMESTAMP` | Exact date and time when the incident occurred or was reported. |
| `severity_level` | `VARCHAR2(15)` | **NOT NULL** | None | None | Classification level: `'LOW'`, `'MEDIUM'`, `'HIGH'`, or `'CRITICAL'`. |
| `incident_category` | `VARCHAR2(50)` | **NOT NULL** | None | None | Category (e.g., `'MECHANICAL_FAILURE'`, `'COLLISION'`, `'VENTILATION_LOSS'`, `'NEAR_MISS'`). |
| `injuries_count` | `NUMBER(3,0)` | **NOT NULL** | None | `0` | Number of personnel injuries resulting from the incident. Must be $\ge 0$. |
| `incident_description` | `VARCHAR2(1000)` | **NOT NULL** | None | None | Complete narrative of the event, immediate root causes, and safety actions taken. |

---

### 3.5 Entity: `GRAFANA_ALERT_LOGS`
* **Description**: System log table populated by automated PL/SQL procedures. Serves as the data store for historical and active Grafana monitoring alerts generated when machinery inspections become overdue.
* **Primary Key**: `alert_id`
* **Foreign Key**: `equipment_id` references `EQUIPMENT(equipment_id)`

#### Column Specifications
| Column Name | Data Type & Length | Nullable | Key | Default Value | Business Rules & Description |
|---|---|:---:|:---:|---|---|
| `alert_id` | `NUMBER(12,0)` | **NOT NULL** | **PK** | `SEQ_ALERTS.NEXTVAL` | Unique surrogate primary key for each generated alert event. |
| `equipment_id` | `NUMBER(10,0)` | **NOT NULL** | **FK** | None | Foreign key identifying the overdue machine (`EQUIPMENT.equipment_id`). |
| `alert_timestamp` | `TIMESTAMP` | **NOT NULL** | None | `CURRENT_TIMESTAMP` | Timestamp when the alert was triggered by the system. |
| `days_overdue` | `NUMBER(4,0)` | **NOT NULL** | None | None | Number of elapsed days past `next_inspection_due` (`TRUNC(SYSDATE) - TRUNC(due_date)`). |
| `alert_level` | `VARCHAR2(15)` | **NOT NULL** | None | None | Visual alert classification: `'WARNING'` ($1\text{--}14$ days) or `'CRITICAL'` ($>14$ days). |
| `alert_status` | `VARCHAR2(15)` | **NOT NULL** | None | `'ACTIVE'` | Current state of the alert: `'ACTIVE'`, `'ACKNOWLEDGED'`, or `'RESOLVED'`. |
| `resolved_at` | `TIMESTAMP` | NULL | None | NULL | Timestamp when the overdue inspection was completed and the alert resolved. |

---

## 4. Referential Integrity & Relationship Matrix

The table below outlines all foreign key constraints, parent-child dependencies, cardinality, and delete enforcement behavior across the schema.

| Parent Table | Parent Key | Child Table | Foreign Key Column | Cardinality | Delete Action / Rule | Business Relationship Description |
|---|---|---|---|:---:|---|---|
| `MINES` | `mine_id` | `EQUIPMENT` | `mine_id` | 1 : N | `ON DELETE RESTRICT` | A mine site hosts many equipment assets. A mine cannot be deleted if active equipment exists. |
| `EQUIPMENT` | `equipment_id` | `EQUIPMENT_INSPECTIONS` | `equipment_id` | 1 : N | `ON DELETE CASCADE` | Equipment has many historical inspection logs. Deleting an asset removes its inspection history. |
| `MINES` | `mine_id` | `SAFETY_INCIDENTS` | `mine_id` | 1 : N | `ON DELETE RESTRICT` | A mine site logs many safety incidents. Mines with historical incidents cannot be purged. |
| `EQUIPMENT` | `equipment_id` | `SAFETY_INCIDENTS` | `equipment_id` | 0..1 : N | `ON DELETE SET NULL` | Incidents may link to a specific asset. If asset is removed, incident log preserves site record. |
| `EQUIPMENT` | `equipment_id` | `GRAFANA_ALERT_LOGS` | `equipment_id` | 1 : N | `ON DELETE CASCADE` | Equipment generates many overdue alert logs over time. Alerts cascade delete with asset. |

---

## 5. Summary of Database Constraints & Domain Validations

### 5.1 Primary & Unique Key Constraints
* `PK_MINES`: Primary Key on `MINES(mine_id)`
* `UK_MINES_CODE`: Unique Key on `MINES(mine_code)`
* `PK_EQUIPMENT`: Primary Key on `EQUIPMENT(equipment_id)`
* `UK_EQUIPMENT_CODE`: Unique Key on `EQUIPMENT(equipment_code)`
* `PK_INSPECTIONS`: Primary Key on `EQUIPMENT_INSPECTIONS(inspection_id)`
* `PK_INCIDENTS`: Primary Key on `SAFETY_INCIDENTS(incident_id)`
* `PK_ALERT_LOGS`: Primary Key on `GRAFANA_ALERT_LOGS(alert_id)`

### 5.2 Check Constraints & Permitted Value Enums
* `CK_MINES_ENV_TYPE`: `environment_type IN ('UNDERGROUND', 'OPEN_PIT', 'SURFACE')`
* `CK_MINES_STATUS`: `operational_status IN ('ACTIVE', 'SUSPENDED', 'CLOSED')`
* `CK_EQUIPMENT_STATUS`: `current_status IN ('OPERATIONAL', 'INSPECTION_OVERDUE', 'UNDER_REPAIR', 'OUT_OF_SERVICE')`
* `CK_EQUIPMENT_INTERVAL`: `inspection_interval_days IN (7, 15, 30)`
* `CK_INSPECTION_RESULT`: `result_status IN ('PASSED', 'MINOR_DEFECT', 'CRITICAL_FAIL')`
* `CK_INCIDENT_SEVERITY`: `severity_level IN ('LOW', 'MEDIUM', 'HIGH', 'CRITICAL')`
* `CK_INCIDENT_INJURIES`: `injuries_count >= 0`
* `CK_ALERT_LEVEL`: `alert_level IN ('WARNING', 'CRITICAL')`
* `CK_ALERT_STATUS`: `alert_status IN ('ACTIVE', 'ACKNOWLEDGED', 'RESOLVED')`

---

## 6. PL/SQL Supporting Objects Specification

### 6.1 Database View: `VW_GRAFANA_OVERDUE_ALERTS`
* **Purpose**: Serves as the primary SQL data source for Grafana dashboard panels. Evaluates date arithmetic dynamically upon read.
* **SQL Query Structure**:
  ```sql
  CREATE OR REPLACE VIEW VW_GRAFANA_OVERDUE_ALERTS AS
  SELECT 
      e.equipment_id,
      e.equipment_code,
      e.equipment_type,
      m.mine_name,
      m.location_region,
      e.risk_score,
      e.next_inspection_due,
      TRUNC(SYSDATE) - TRUNC(e.next_inspection_due) AS days_overdue,
      CASE 
          WHEN (TRUNC(SYSDATE) - TRUNC(e.next_inspection_due)) > 14 THEN 'CRITICAL'
          WHEN (TRUNC(SYSDATE) - TRUNC(e.next_inspection_due)) > 0 THEN 'WARNING'
          ELSE 'OK'
      END AS alert_severity,
      e.current_status
  FROM EQUIPMENT e
  JOIN MINES m ON e.mine_id = m.mine_id
  WHERE e.next_inspection_due < SYSDATE
    AND e.current_status != 'OUT_OF_SERVICE';
  ```

### 6.2 Stored Function: `FN_CALCULATE_RBI_SCORE`
* **Purpose**: Calculates dynamic risk score ($1\text{--}100$) and returns recommended inspection cadence ($7$, $15$, or $30$ days).
* **Input Parameters**: `p_equipment_id NUMBER`
* **Return Type**: `NUMBER` (Risk Score)

### 6.3 Stored Procedure: `SP_PROCESS_OVERDUE_INSPECTIONS`
* **Purpose**: Executes daily batch scan. Identifies overdue machinery, updates `EQUIPMENT.current_status` to `'INSPECTION_OVERDUE'`, and inserts records into `GRAFANA_ALERT_LOGS`.

### 6.4 Database Trigger: `TRG_INCIDENT_OVERRIDE`
* **Event**: `AFTER INSERT ON SAFETY_INCIDENTS`
* **Purpose**: When an incident with `severity_level IN ('HIGH', 'CRITICAL')` is recorded, automatically updates `EQUIPMENT.current_status = 'INSPECTION_OVERDUE'` for all identical equipment types operating at the same mine site.
