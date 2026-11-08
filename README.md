# MULISA-_20252SEN228-_MINEHUB
# MINEHUB: Mining Safety Incidents & Equipment Inspections Tracking System with Grafana Overdue Alerts

## Student Information 
* **Full Name:** MULISA Israel Jakes 
* **Student ID:** 20252SEN228
* **Project Short Name:** MINEHUB
* **Course  Domain:** Mining Site Safety & Database System Architecture


##  Problem Statement
Delayed safety inspections and unmonitored equipment wear at active mining sites frequently lead to critical operational failures, preventable workplace incidents, and regulatory non compliance. Manual inspection logging often results in overlooked maintenance deadlines, exposing mining crews and heavy machinery to severe operational hazards.


## Project Objectives 
The objective of **MINEHUB** is to design and deploy an automated industrial tracking platform powered by an **Oracle PL/SQL relational database** and **Grafana real time alerts**. 

Key objectives include: 
1. **Centralized Logging:** Maintain a centralized schema for mine sites, heavy equipment, inspection logs, and incident reports.
2. **Automated Risk Scoring:** Implement a dynamic **Risk Based Inspection (RBI)** scoring engine in PL/SQL that automatically recalculates equipment risk levels and adjusts inspection frequencies.
3. **Real Time Monitoring:** Connect Grafana dashboards to database views to trigger real time visual alerts (Warning/Critical) for overdue equipment inspections.
4. **Regulatory Auditability:** Ensure data integrity and compliance through automated database triggers and audit logs.

## Data Source 
* **Dataset Name:** U.S. Mine Safety and Health Administration (MSHA) Accident & Inspection Open Data
* **Source Portal:** [Data.gov MSHA Accident Injuries Dataset](https://catalog.data.gov/dataset/msha-accident-injuries-data-set)
* **Official Repository:** [MSHA Open Data Resources](https://www.msha.gov/data-and-reports/data-sources-and-calculators/data-resources/mdsrg/accident-injuries-and-illnesses-data-set)
* **License & Compliance:** Public Domain (U.S. Government Open Data). Fully cleared for educational use with zero Personally Identifiable Information (PII).

## System Innovation: Dynamic Risk Based Inspection (RBI) Scoring Rather than relying on static time intervals (e.g., inspecting all machines every 30 days), MINEHUB introduces a **Dynamic Risk-Based Inspection (RBI) Engine** built in PL/SQL:
* **Dynamic Adjustment:** Calculates a risk score ($1\text{--}100$) for each machine based on equipment age, past incident frequency, operational environment, and inspector notes.
* **Proactive Scheduling:** High risk machinery (e.g., underground ventilation fans with prior failures) automatically receives shortened inspection intervals (e.g., 7 days instead of 30), preventing catastrophic failures before they occur.

## Planned Database Architecture & PL/SQL Features 
### Planned Relational Tables
1. `MINES`: Tracks mine site locations, operating regions, types (Underground/Surface/Open Pit), and operational status.
2. `EQUIPMENT`: Manages heavy equipment machinery, inspection intervals, dynamic risk scores, and operational statuses. 
3. `EQUIPMENT_INSPECTIONS`: Logs completed inspection records, inspector codes, pass/fail status, and logging dates. 
4. `SAFETY_INCIDENTS`: Records workplace safety incidents, severity levels (Low to Critical), affected equipment, and injury counts.
5. `GRAFANA_ALERT_LOGS`: Stores alert notifications generated for Grafana dashboards.

### Core PL/SQL Modules
* **`VW_GRAFANA_OVERDUE_ALERTS` (Database View):** Evaluates overdue equipment inspection thresholds for real time Grafana alert visualization.
* **`FN_CALCULATE_RBI_SCORE` (PL/SQL Stored Function):** Computes dynamic risk scores based on equipment age, environment, and incident history.
* **`SP_PROCESS_OVERDUE_INSPECTIONS` (PL/SQL Stored Procedure):** Scans overdue inspections, updates machine statuses, and generates alert entries for Grafana. 
* **`TRG_INCIDENT_OVERRIDE` (PL/SQL Trigger):** Automatically invalidates the inspection status of matching equipment across a site when a critical incident occurs.

##  References & Credible Sources
1. **Research Paper (Scopus/PMC):** *The Future of Mine Safety: A Comprehensive Review of Anti Collision and Inspection Systems in Underground Mines*. PubMed Central. [https://pmc.ncbi.nlm.nih.gov/articles/PMC10181612/](https://pmc.ncbi.nlm.nih.gov/articles/PMC10181612/)
2. **Research Paper (ScienceDirect):** *Efficient Risk Based Inspection Framework: Balancing Safety and Budgetary Constraints*. Reliability Engineering & System Safety. [https://doi.org/10.1016/j.ress.2024.110519](https://doi.org/10.1016/j.ress.2024.110519)
3. **Report / Statistic (NISR):** *Rwanda Statistical Yearbook 2025 (Mining & Quarrying Sector Performance)*. National Institute of Statistics of Rwanda. [https://www.statistics.gov.rw/sites/default/files/documents/2025-12/RWANDA_STATISTICAL_YEARBOOK_2025.pdf](https://www.statistics.gov.rw/sites/default/files/documents/2025-12/RWANDA_STATISTICAL_YEARBOOK_2025.pdf)
4. **Existing System (MSHA):** *Mine Data Retrieval System (MDRS)*. U.S. Mine Safety and Health Administration. [https://www.msha.gov/data-and-reports](https://www.msha.gov/data-and-reports)
5. **Data Source (Data.gov):** *MSHA Mine Inspections & Safety Incidents Open Dataset*. Data.gov. [https://catalog.data.gov/dataset/msha-inspections-dataset](https://catalog.data.gov/dataset/msha-inspections-dataset)
6. **Technical Source (Oracle Docs):** *Oracle Database PL/SQL Language Reference & Grafana Integration*. Oracle Documentation. [https://docs.oracle.com/en/database/oracle/oracle-database/](https://docs.oracle.com/en/database/oracle/oracle-database/)
