# MINEHUB Database Normalization Analysis (1NF to 3NF / BCNF)

## 1. System Context & Normalization Objectives

### 1.1 Project Overview
* **System Name:** MINEHUB — Mining Site Operations & Equipment Safety Tracking System
* **Author / Student:** MULISA Israel Jakes (Student ID: 20252SEN228)
* **Target Architecture:** Oracle PL/SQL Relational Database Engine & Grafana Monitoring Platform

### 1.2 Purpose of Database Normalization
Database normalization is a systematic design technique used to organize relational database schemas by decomposing complex, unnormalized data structures into smaller, well-structured relations. The primary objectives of normalizing the MINEHUB database schema include:

1. **Elimination of Data Redundancies:** Preventing duplicate storage of mine site location details, equipment specification data, and inspector records across multiple transaction logs.
2. **Prevention of Operational Anomalies:** Eliminating Insertion, Update, and Deletion anomalies that degrade data integrity during live site operations.
3. **Enforcement of Referential Integrity:** Structuring primary key ($PK$) and foreign key ($FK$) relationships to support transactional constraints, automated PL/SQL triggers (`TRG_INCIDENT_OVERRIDE`), stored functions (`FN_CALCULATE_RBI_SCORE`), and real-time database views (`VW_GRAFANA_OVERDUE_ALERTS`).
4. **Optimization of Query Performance:** Ensuring storage efficiency and reducing locking contention during concurrent field inspection logging and Grafana polling.

---

## 2. Unnormalized Form (UNF) & Anomaly Diagnosis

### 2.1 Raw Unnormalized Data Relation (UNF)
Prior to normalization, industrial mining data is often ingested as a flat, unnormalized transaction stream from field spreadsheets or legacy reporting logs. Consider the raw relation `UNNORMALIZED_MINING_SAFETY_LOG`:

$$\text{UNNORMALIZED\_MINING\_SAFETY\_LOG}(\text{mine\_code}, \text{mine\_name}, \text{mine\_location}, \text{mine\_type}, \text{env\_factor}, \text{equipment\_code}, \text{equipment\_name}, \text{equipment\_type}, \text{purchase\_date}, \text{baseline\_interval}, \text{risk\_score}, \text{current\_status}, \text{next\_inspection\_due}, \text{inspector\_code}, \text{inspector\_name}, \text{inspection\_date}, \text{result\_status}, \text{defect\_notes}, \text{telemetry\_temp}, \text{telemetry\_vibe}, \text{incident\_id}, \text{incident\_category}, \text{severity\_level}, \text{injuries}, \text{alert\_id}, \text{alert\_level}, \text{alert\_timestamp})$$

### 2.2 Operational Anomaly Analysis

#### 1. Insertion Anomaly
* **Scenario:** A new mining site (`Rutongo Underground Mine`, code `M-006`) is acquired by the mining corporation, but no heavy machinery has been assigned to it yet.
* **Failure:** Under `UNNORMALIZED_MINING_SAFETY_LOG`, a record cannot be inserted without providing an `equipment_code` and an `inspection_date` if those attributes form part of the primary key. Forcing dummy or `NULL` values corrupts entity integrity.

#### 2. Update Anomaly
* **Scenario:** The environmental severity factor (`env_factor`) for `Rwamagana Open Pit` (code `M-002`) is re-evaluated from $10$ to $12$ due to heavy rainy season mudding.
* **Failure:** Because `mine_name`, `mine_location`, and `env_factor` are repeated across thousands of historical inspection rows, updating this value requires updating every single historical row containing `M-002`. If any row is missed due to execution timeouts, the database enters an inconsistent state where the same mine has conflicting environmental risk weights.

#### 3. Deletion Anomaly
* **Scenario:** An old, decommissioned haul truck (`EQ-901`) and all its historical inspection logs are deleted from the flat file to clean up database storage.
* **Failure:** Deleting these rows inadvertently purges the only record of the Mine Site (`M-003`) or the Inspector (`INS-104`) who performed those inspections, leading to permanent loss of organizational audit data.

---

## 3. First Normal Form (1NF) — Atomicity & Primary Key Selection

### 3.1 1NF Definition & Requirements
A relation is in **First Normal Form (1NF)** if and only if:
1. All attribute values are **atomic** (indivisible; no multi-valued attributes, set-valued attributes, or nested repeating groups).
2. Every column contains values of a single domain.
3. Each tuple (row) is unique, identified by a defined Primary Key.

### 3.2 1NF Transformation Steps
* **Elimination of Repeating Groups:** Multi-valued fields such as list of diagnostic defects or multiple incident codes recorded in a single cell are expanded into distinct, atomic row entries.
* **Primary Key Assignment:** To uniquely identify every tuple in the flat log, a composite primary key is established:  
  $$\text{Primary Key} = \{\text{equipment\_code}, \text{inspection\_date}, \text{incident\_id}, \text{alert\_id}\}$$

### 3.3 Flattened 1NF Relation & Functional Dependencies ($FDs$)
$$\text{MINING\_LOG\_1NF}(\underline{\text{equipment\_code}}, \underline{\text{inspection\_date}}, \underline{\text{incident\_id}}, \underline{\text{alert\_id}}, \text{mine\_code}, \text{mine\_name}, \text{mine\_location}, \text{mine\_type}, \text{env\_factor}, \text{equipment\_name}, \text{equipment\_type}, \text{purchase\_date}, \text{baseline\_interval}, \text{risk\_score}, \text{current\_status}, \text{next\_inspection\_due}, \text{inspector\_code}, \text{inspector\_name}, \text{result\_status}, \text{defect\_notes}, \text{telemetry\_temp}, \text{telemetry\_vibe}, \text{incident\_category}, \text{severity\_level}, \text{injuries}, \text{alert\_level}, \text{alert\_timestamp})$$

#### Identified Functional Dependencies in 1NF:
* $FD_1: \text{mine\_code} \rightarrow \text{mine\_name}, \text{mine\_location}, \text{mine\_type}, \text{env\_factor}$
* $FD_2: \text{equipment\_code} \rightarrow \text{equipment\_name}, \text{equipment\_type}, \text{purchase\_date}, \text{baseline\_interval}, \text{risk\_score}, \text{current\_status}, \text{next\_inspection\_due}, \text{mine\_code}$
* $FD_3: \text{inspector\_code} \rightarrow \text{inspector\_name}$
* $FD_4: \text{equipment\_code}, \text{inspection\_date} \rightarrow \text{inspector\_code}, \text{result\_status}, \text{defect\_notes}, \text{telemetry\_temp}, \text{telemetry\_vibe}$
* $FD_5: \text{incident\_id} \rightarrow \text{incident\_category}, \text{severity\_level}, \text{injuries}, \text{equipment\_code}, \text{mine\_code}$
* $FD_6: \text{alert\_id} \rightarrow \text{alert\_level}, \text{alert\_timestamp}, \text{equipment\_code}$

---

## 4. Second Normal Form (2NF) — Elimination of Partial Dependencies

### 4.1 2NF Definition & Requirements
A relation is in **Second Normal Form (2NF)** if and only if:
1. It is in **First Normal Form (1NF)**.
2. Every non-prime attribute is **fully functionally dependent** on the primary key (i.e., no non-prime attribute depends on a proper subset of a composite primary key).

### 4.2 Identification of Partial Dependencies in 1NF
The primary key of `MINING_LOG_1NF` is composite: $\{\text{equipment\_code}, \text{inspection\_date}, \text{incident\_id}, \text{alert\_id}\}$.  
Evaluating the non-prime attributes reveals severe partial dependencies:
* **Partial Dependency 1:** Attributes $\text{equipment\_name}, \text{equipment\_type}, \text{purchase\_date}, \text{mine\_code}$ depend *only* on $\text{equipment\_code}$ (part of the composite key).
* **Partial Dependency 2:** Attributes $\text{incident\_category}, \text{severity\_level}, \text{injuries}$ depend *only* on $\text{incident\_id}$ (part of the composite key).
* **Partial Dependency 3:** Attributes $\text{alert\_level}, \text{alert\_timestamp}$ depend *only* on $\text{alert\_id}$ (part of the composite key).

### 4.3 2NF Decomposition Step
To achieve 2NF, we decompose `MINING_LOG_1NF` into distinct relations where every non-key attribute depends on the full primary key of its respective table:

1. **`EQUIPMENT_2NF`** $(\underline{\text{equipment\_code}}, \text{equipment\_name}, \text{equipment\_type}, \text{purchase\_date}, \text{baseline\_interval}, \text{risk\_score}, \text{current\_status}, \text{next\_inspection\_due}, \text{mine\_code}, \text{mine\_name}, \text{mine\_location}, \text{mine\_type}, \text{env\_factor})$
2. **`INSPECTION_2NF`** $(\underline{\text{equipment\_code}}, \underline{\text{inspection\_date}}, \text{inspector\_code}, \text{inspector\_name}, \text{result\_status}, \text{defect\_notes}, \text{telemetry\_temp}, \text{telemetry\_vibe})$
3. **`INCIDENT_2NF`** $(\underline{\text{incident\_id}}, \text{equipment\_code}, \text{mine\_code}, \text{incident\_category}, \text{severity\_level}, \text{injuries})$
4. **`ALERT_2NF`** $(\underline{\text{alert\_id}}, \text{equipment\_code}, \text{alert\_level}, \text{alert\_timestamp})$

---

## 5. Third Normal Form (3NF) — Elimination of Transitive Dependencies

### 5.1 3NF Definition & Requirements
A relation is in **Third Normal Form (3NF)** if and only if:
1. It is in **Second Normal Form (2NF)**.
2. No non-prime attribute is **transitively dependent** on the primary key.  
   *(Formally: For every non-trivial functional dependency $X \rightarrow Y$, either $X$ is a superkey or $Y$ is a prime attribute).*

### 5.2 Identification of Transitive Dependencies in 2NF Relations

#### Transitive Dependency in `EQUIPMENT_2NF`:
In `EQUIPMENT_2NF`, we have:
$$\text{equipment\_code} \rightarrow \text{mine\_code}$$
$$\text{mine\_code} \rightarrow \text{mine\_name}, \text{mine\_location}, \text{mine\_type}, \text{env\_factor}$$
Therefore:
$$\text{equipment\_code} \rightarrow \text{mine\_name}, \text{mine\_location}, \text{mine\_type}, \text{env\_factor}$$
Here, non-key attributes ($\text{mine\_name}, \text{mine\_location}, \text{mine\_type}, \text{env\_factor}$) depend on $\text{mine\_code}$, which is a non-key attribute in `EQUIPMENT_2NF`. This is a classic transitive dependency $A \rightarrow B \rightarrow C$.

#### Transitive Dependency in `INSPECTION_2NF`:
In `INSPECTION_2NF`, we have:
$$\text{equipment\_code}, \text{inspection\_date} \rightarrow \text{inspector\_code}$$
$$\text{inspector\_code} \rightarrow \text{inspector\_name}$$
Therefore, $\text{inspector\_name}$ depends transitively on the composite inspection key through $\text{inspector\_code}$.

### 5.3 3NF Decomposition & Final Relational Schema
To eliminate transitive dependencies, we extract the determinant attributes ($\text{mine\_code}$ and $\text{inspector\_code}$) into independent master entities and replace functional codes with system-generated surrogate keys (`ID` sequence identifiers) for optimal Oracle PL/SQL indexing.

This yields the final **5 Normalized 3NF Relational Entities** matching the `data_dictionary.md` specification:

#### 1. `MINES` (Master Site Entity)
$$\text{MINES}(\underline{\text{mine\_id}}, \text{mine\_code}, \text{mine\_name}, \text{location}, \text{mine\_type}, \text{env\_severity\_factor}, \text{status})$$
* **Primary Key:** `mine_id`
* **Alternate (Candidate) Key:** `mine_code`
* **Functional Dependencies:**
  * $FD_{\text{MINES}}: \text{mine\_id} \rightarrow \text{mine\_code}, \text{mine\_name}, \text{location}, \text{mine\_type}, \text{env\_severity\_factor}, \text{status}$
  * $\text{mine\_code} \rightarrow \text{mine\_id}$

#### 2. `EQUIPMENT` (Master Machinery Entity)
$$\text{EQUIPMENT}(\underline{\text{equipment\_id}}, \text{equipment\_code}, \text{mine\_id}^*, \text{equipment\_name}, \text{equipment\_type}, \text{model\_number}, \text{purchase\_date}, \text{baseline\_interval\_days}, \text{risk\_score}, \text{current\_status}, \text{last\_inspection\_date}, \text{next\_inspection\_due})$$
* **Primary Key:** `equipment_id`
* **Foreign Key:** `mine_id` $\rightarrow$ `MINES(mine_id)`
* **Functional Dependencies:**
  * $FD_{\text{EQUIPMENT}}: \text{equipment\_id} \rightarrow \text{equipment\_code}, \text{mine\_id}, \text{equipment\_name}, \text{equipment\_type}, \text{model\_number}, \text{purchase\_date}, \text{baseline\_interval\_days}, \text{risk\_score}, \text{current\_status}, \text{last\_inspection\_date}, \text{next\_inspection\_due}$

#### 3. `EQUIPMENT_INSPECTIONS` (Transactional Inspection Log Entity)
$$\text{EQUIPMENT\_INSPECTIONS}(\underline{\text{inspection\_id}}, \text{equipment\_id}^*, \text{inspection\_date}, \text{inspector\_code}, \text{result\_status}, \text{defect\_notes}, \text{telemetry\_temperature}, \text{telemetry\_vibration})$$
* **Primary Key:** `inspection_id`
* **Foreign Key:** `equipment_id` $\rightarrow$ `EQUIPMENT(equipment_id)`
* **Functional Dependencies:**
  * $FD_{\text{INSPECTIONS}}: \text{inspection\_id} \rightarrow \text{equipment\_id}, \text{inspection\_date}, \text{inspector\_code}, \text{result\_status}, \text{defect\_notes}, \text{telemetry\_temperature}, \text{telemetry\_vibration}$

#### 4. `SAFETY_INCIDENTS` (Incident Log Entity)
$$\text{SAFETY\_INCIDENTS}(\underline{\text{incident\_id}}, \text{mine\_id}^*, \text{equipment\_id}^*, \text{incident\_date}, \text{severity\_level}, \text{incident\_category}, \text{injuries\_count}, \text{description})$$
* **Primary Key:** `incident_id`
* **Foreign Keys:**
  * `mine_id` $\rightarrow$ `MINES(mine_id)`
  * `equipment_id` $\rightarrow$ `EQUIPMENT(equipment_id)`
* **Functional Dependencies:**
  * $FD_{\text{INCIDENTS}}: \text{incident\_id} \rightarrow \text{mine\_id}, \text{equipment\_id}, \text{incident\_date}, \text{severity\_level}, \text{incident\_category}, \text{injuries\_count}, \text{description}$

#### 5. `GRAFANA_ALERT_LOGS` (Alert Staging Entity)
$$\text{GRAFANA\_ALERT\_LOGS}(\underline{\text{alert\_id}}, \text{equipment\_id}^*, \text{alert\_timestamp}, \text{alert\_level}, \text{days\_overdue}, \text{alert\_status})$$
* **Primary Key:** `alert_id`
* **Foreign Key:** `equipment_id` $\rightarrow$ `EQUIPMENT(equipment_id)`
* **Functional Dependencies:**
  * $FD_{\text{ALERTS}}: \text{alert\_id} \rightarrow \text{equipment\_id}, \text{alert\_timestamp}, \text{alert\_level}, \text{days\_overdue}, \text{alert\_status}$

---

## 6. Boyce-Codd Normal Form (BCNF) & Higher Form Evaluation

### 6.1 BCNF Evaluation
A relation is in **Boyce-Codd Normal Form (BCNF)** if and only if for every non-trivial functional dependency $X \rightarrow Y$, $X$ is a **superkey**.

* **Analysis of `MINES`:** The functional dependencies are $\text{mine\_id} \rightarrow \text{all attributes}$ and $\text{mine\_code} \rightarrow \text{all attributes}$. Both $\text{mine\_id}$ and $\text{mine\_code}$ are candidate keys (superkeys). Hence, `MINES` is in BCNF.
* **Analysis of `EQUIPMENT`:** The determinant is $\text{equipment\_id}$, which is the primary key. Hence, `EQUIPMENT` is in BCNF.
* **Analysis of `EQUIPMENT_INSPECTIONS`:** The determinant is $\text{inspection\_id}$, which is the primary key. Hence, `EQUIPMENT_INSPECTIONS` is in BCNF.
* **Analysis of `SAFETY_INCIDENTS`:** The determinant is $\text{incident\_id}$, which is the primary key. Hence, `SAFETY_INCIDENTS` is in BCNF.
* **Analysis of `GRAFANA_ALERT_LOGS`:** The determinant is $\text{alert\_id}$, which is the primary key. Hence, `GRAFANA_ALERT_LOGS` is in BCNF.

**Conclusion:** All 5 relations in the MINEHUB database schema satisfy **BCNF**.

### 6.2 Fourth Normal Form (4NF) Evaluation
A relation is in **4NF** if it is in BCNF and contains no non-trivial **multi-valued dependencies** ($MVD: X \twoheadrightarrow Y$).  
In MINEHUB, no entity contains independent multi-valued facts. For instance, an inspection record associates a single inspection date with specific telemetry readings and a single result status. There are no independent multi-valued relationships (such as an equipment having multiple independent certifications and multiple independent operating locations simultaneously in the same table). Therefore, the schema naturally satisfies **4NF**.

---

## 7. Normalization Dependency Matrix & Verification Proofs

### 7.1 Attribute Mapping Matrix (UNF to 3NF)

| Original UNF Attribute | 1NF Assignment | 2NF Relation | Final 3NF Entity | Role / Key Constraint |
|---|---|---|---|---|
| `mine_code` | `MINING_LOG_1NF` | `EQUIPMENT_2NF` | `MINES` | Alternate Key (`UNIQUE`) |
| `mine_name` | `MINING_LOG_1NF` | `EQUIPMENT_2NF` | `MINES` | Non-Key Attribute |
| `mine_location` | `MINING_LOG_1NF` | `EQUIPMENT_2NF` | `MINES` | Non-Key Attribute |
| `mine_type` | `MINING_LOG_1NF` | `EQUIPMENT_2NF` | `MINES` | Check Constraint (`UNDERGROUND`/`OPEN_PIT`/`SURFACE`) |
| `env_factor` | `MINING_LOG_1NF` | `EQUIPMENT_2NF` | `MINES` | Non-Key Attribute (`env_severity_factor`) |
| `equipment_code` | `MINING_LOG_1NF` | `EQUIPMENT_2NF` | `EQUIPMENT` | Alternate Key (`UNIQUE`) |
| `equipment_name` | `MINING_LOG_1NF` | `EQUIPMENT_2NF` | `EQUIPMENT` | Non-Key Attribute |
| `equipment_type` | `MINING_LOG_1NF` | `EQUIPMENT_2NF` | `EQUIPMENT` | Non-Key Attribute |
| `purchase_date` | `MINING_LOG_1NF` | `EQUIPMENT_2NF` | `EQUIPMENT` | Non-Key Attribute |
| `baseline_interval` | `MINING_LOG_1NF` | `EQUIPMENT_2NF` | `EQUIPMENT` | Non-Key Attribute (`baseline_interval_days`) |
| `risk_score` | `MINING_LOG_1NF` | `EQUIPMENT_2NF` | `EQUIPMENT` | Non-Key Attribute (Computed via `FN_CALCULATE_RBI_SCORE`) |
| `current_status` | `MINING_LOG_1NF` | `EQUIPMENT_2NF` | `EQUIPMENT` | Check Constraint (`OPERATIONAL`/`INSPECTION_OVERDUE`/`OUT_OF_SERVICE`) |
| `next_inspection_due`| `MINING_LOG_1NF` | `EQUIPMENT_2NF` | `EQUIPMENT` | Non-Key Attribute (Queried by `VW_GRAFANA_OVERDUE_ALERTS`) |
| `inspector_code` | `MINING_LOG_1NF` | `INSPECTION_2NF` | `EQUIPMENT_INSPECTIONS`| Non-Key Attribute |
| `inspection_date` | `MINING_LOG_1NF` | `INSPECTION_2NF` | `EQUIPMENT_INSPECTIONS`| Non-Key Attribute |
| `result_status` | `MINING_LOG_1NF` | `INSPECTION_2NF` | `EQUIPMENT_INSPECTIONS`| Check Constraint (`PASSED`/`MINOR_DEFECT`/`CRITICAL_FAIL`) |
| `defect_notes` | `MINING_LOG_1NF` | `INSPECTION_2NF` | `EQUIPMENT_INSPECTIONS`| Non-Key Attribute |
| `telemetry_temp` | `MINING_LOG_1NF` | `INSPECTION_2NF` | `EQUIPMENT_INSPECTIONS`| Non-Key Attribute (`telemetry_temperature`) |
| `telemetry_vibe` | `MINING_LOG_1NF` | `INSPECTION_2NF` | `EQUIPMENT_INSPECTIONS`| Non-Key Attribute (`telemetry_vibration`) |
| `incident_id` | `MINING_LOG_1NF` | `INCIDENT_2NF` | `SAFETY_INCIDENTS` | Primary Key |
| `severity_level` | `MINING_LOG_1NF` | `INCIDENT_2NF` | `SAFETY_INCIDENTS` | Check Constraint (`LOW`/`MEDIUM`/`HIGH`/`CRITICAL`) |
| `alert_id` | `MINING_LOG_1NF` | `ALERT_2NF` | `GRAFANA_ALERT_LOGS` | Primary Key |
| `alert_level` | `MINING_LOG_1NF` | `ALERT_2NF` | `GRAFANA_ALERT_LOGS` | Check Constraint (`WARNING`/`CRITICAL`) |

### 7.2 Lossless-Join Decomposition Proof
A decomposition of relation $R$ into $R_1$ and $R_2$ is **lossless-join** with respect to a set of functional dependencies $F$ if and only if at least one of the following functional dependencies is in $F^+$:
1. $R_1 \cap R_2 \rightarrow R_1$ (The common attributes functionally determine $R_1$)
2. $R_1 \cap R_2 \rightarrow R_2$ (The common attributes functionally determine $R_2$)

#### Proof for `MINES` and `EQUIPMENT` Decomposition:
* $R_{\text{EQUIPMENT\_FULL}} = \text{EQUIPMENT} \cup \text{MINES}$
* $R_1 = \text{EQUIPMENT}(\text{equipment\_id}, \dots, \text{mine\_id})$
* $R_2 = \text{MINES}(\text{mine\_id}, \text{mine\_code}, \text{mine\_name}, \dots)$
* Intersection: $R_1 \cap R_2 = \{\text{mine\_id}\}$
* Since $\text{mine\_id}$ is the primary key of $\text{MINES}$, we have:
  $$\text{mine\_id} \rightarrow \text{MINES}$$
* Because $R_1 \cap R_2 \rightarrow R_2$ holds, the join $R_1 \bowtie R_2$ reconstructs the exact original relation without producing spurious tuples.
* **Conclusion:** The decomposition is strictly **lossless-join**.

### 7.3 Dependency Preservation Verification
A decomposition is **dependency preserving** if the union of the functional dependencies of the decomposed relations $(F_1 \cup F_2 \cup \dots \cup F_n)^+$ is equivalent to the original set of functional dependencies $F^+$.

* $FD_1 (\text{Mine attributes})$ is preserved in `MINES`.
* $FD_2 (\text{Equipment attributes})$ is preserved in `EQUIPMENT`.
* $FD_4 (\text{Inspection telemetry & results})$ is preserved in `EQUIPMENT_INSPECTIONS`.
* $FD_5 (\text{Safety incidents})$ is preserved in `SAFETY_INCIDENTS`.
* $FD_6 (\text{Grafana alert logs})$ is preserved in `GRAFANA_ALERT_LOGS`.

Since every functional dependency in $F$ can be enforced within a single table without requiring expensive cross-table joins, the decomposition is **100% dependency preserving**. This allows Oracle PL/SQL triggers and constraints to enforce business rules with maximum efficiency.
