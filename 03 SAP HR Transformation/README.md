# SAP HR Transformation — Hands-On HCM Build & SuccessFactors Migration Lab

## What We're Trying to Achieve

Most SAP HCM learning material teaches configuration in isolation — a Personnel Area here, a wage type there — without ever connecting those settings to a system that actually runs. This repository takes the opposite approach: it builds one complete, working SAP ECC / S/4HANA HCM system for a fictional company, end to end, and then puts that system through the exact transformation that real HR Transformation programmes go through — migrating off the on-premise HCM core onto SAP SuccessFactors Employee Central, and standing up the ongoing data replication that keeps both systems in sync afterward.

The reasoning behind this approach is simple: configuration only makes sense in the context of the business outcome it serves. A Personnel Subarea grouping is just a SPRO screen until you've watched it determine which work schedule a real hired employee gets defaulted into. A `NUMKR` feature is an abstract decision tree until you've traced exactly which personnel number range it assigned during a test hire. So every stage in this build is sequenced, tested, and verified against a single fictional company — **KMZP Consulting** (with a second reference company, **BLDP Blood Pact Consulting**, used for comparison scenarios) — so that every configuration decision is tied to a working, demonstrable result rather than abstract theory.

The project is split into two phases that mirror the two halves of a real HR Transformation engagement:

**Phase 1 — build the on-premise system.** Starting from a blank SAP ECC / S/4HANA HCM client, configure the Enterprise Structure, Personnel Structure, Number Ranges, Payroll Area, Features, Pay Scale Structure, Organizational Management, PA/OM Integration, Personnel Administration, and Time Management — in dependency order — until the system can hire a real employee, run that employee through their full lifecycle (promotion, transfer, leave of absence, termination, rehire), capture their time, and run a live payroll simulation that posts to Finance. This is the "build a working company" phase.

**Phase 2 — transform it into a hybrid SuccessFactors landscape.** Once Phase 1 is provably working, the same fictional company is migrated to SAP SuccessFactors Employee Central as the new system of record for HR master data, while SAP S/4HANA HCM is retained for Payroll, Time, and other SAP-side processes. This phase documents the full migration (Infoporter-based extraction, key mapping, cutover/cutoff dates, event reason mapping) and the ongoing replication (Business Integration Builder transformation templates, the Compound Employee API, field-level mapping) that keeps the two systems synchronized after go-live. This is the "Core Hybrid" deployment model, and it is the most common real-world architecture for organizations that adopt SuccessFactors Employee Central but keep SAP Payroll on-premise.

The result is a single, coherent reference: not a checklist of SPRO paths, but a working demonstration of why each configuration decision exists, what it produces downstream, and how it survives the transition to a hybrid cloud/on-premise landscape.

> **Scope note:** This README and repository cover the `HR Data Migration` build and migration content only. A separate `SAP HCM` reference library exists alongside this folder in the wider vault, but it is intentionally out of scope here.

---

## Repository Structure

```
HR Data Migration/
├── 01 End to End Set up Checklist.md       — master Phase 1 checklist, Stage 0–15, T-code & table quick reference
│
├── Phase 1 Organization Set up/
│   ├── 01 Stage 0,1 - Enterprise Structure.md
│   ├── 02 Stage 2 - Number Ranges.md
│   ├── 03 Stage 3 - Payroll Area & Control Record.md
│   ├── 04 Stage 4 - PINCH & Admin Groups.md
│   ├── 05 Stage 5 - Pay Scale Structure.md
│   ├── 06 Stage 6 - Features (TARIF & LGMST).md
│   ├── 07 Stage 7 - Organizational Management (OM).md
│   ├── 08 Stage 8 - PA OM Integration.md
│   ├── 09 Stage 9 - Personnel Administration.md
│   ├── 10 Stage 9.1 - Employee Hiring.md
│   ├── 11 Stage 10 - Time Management.md
│   └── 12 Stage9_Sample_Hire_Data.docx       — worked sample hire data used in test cases
│
└── Phase 2 SF Integration/
    ├── 01 Stage 1 - Migration.md             — S/4HANA HCM → Employee Central
    ├── 02 Stage 2 - Replication.md           — Employee Central → S/4HANA (ongoing sync)
    └── 03 Employee Migration Scenarios.md    — worked examples: active, LOA, and terminated employees
```

*(Screenshot subfolders accompanying each stage are omitted from this structure — they are supporting visual evidence, not standalone content.)*

---

## Phase 1 — Organization Set Up

**Goal:** build a working HCM system end-to-end in a single SAP ECC / S/4HANA on-premise system, so that the system can hire employees, capture time, run payroll, and process the full Hire-to-Retire lifecycle for KMZP Consulting / BLDP Blood Pact Consulting. Each stage depends on the one before it — they are designed to be worked through in order.

Phase 1 applies equally to **SAP ECC 6.0 EhP8** and **SAP HCM for S/4HANA on-premise (H4S4 — compatibility scope)** — the SPRO menus, T-codes, and configuration objects (PA, OM, PT, PY) are functionally identical between the two code lines. Country-specific configuration throughout uses **Canada (MOLGA 07)** as the reference country.

| Stage | Covers |
|---|---|
| **Stage 0** | Pre-requisites: SAP GUI access, user parameters (`MOL`, `UGR`, `PLOG`), client customizing settings, key tables to know |
| **Stage 1 — Enterprise Structure** | Company & Company Code, Personnel Area & Personnel Subarea, Employee Group & Employee Subgroup, and the ESG groupings that unlock downstream payroll/time tables |
| **Stage 2 — Number Ranges** | PERNR (Personnel Number) ranges via PA04, the `NUMKR` feature, and OM object number ranges (Org Unit, Position, Job, Person, Cost Center) via OONR |
| **Stage 3 — Payroll Area & Control Record** | Payroll Areas, Period Parameters, Date Modifiers, Payroll Periods, Cumulation Calendar, and the Payroll Control Record that drives period status |
| **Stage 4 — Features (PE03)** | The decision-tree features that default infotype values at hiring: `NUMKR`, `ABKRS`, `PINCH`, `SCHKZ`, `TARIF`, `LGMST`, `TMSTA`, `WRKHR`, `VTART`, `IGMOD`, `PFREQ` |
| **Stage 5 — Pay Scale Structure** | Pay Scale Type & Area, assignment to Personnel Area/Subarea, Pay Scale Groups/Levels with indirect valuation, and Pay Grades for management positions |
| **Stage 6 — Wage Types** | Wage Type Catalogue (OH11), Wage Type Characteristics, Processing/Cumulation/Evaluation classes, permissibility per PSA/ESG, and wage type valuation modules |
| **Stage 7 — Organizational Management** | OM foundation settings (plan version, number ranges, object types, relationships), building the org hierarchy (PPOCE/PPOME/PO13/PO03), and Account Assignment, Cost Center Assignment, Planned Compensation |
| **Stage 8 — PA/OM Integration** | The `T77S0 PLOGI` switches that link Personnel Administration to Organizational Management, the `RHINTE00–30` consistency programs, and five worked test-hire scenarios (HR Manager, Plant Manager, Union Operator, Part-time Sales Rep, Contractor) |
| **Stage 9 — Personnel Administration** | Infotype menu configuration (V_T588B), Infogroups (Hire, Org Reassignment, Change of Pay, Termination, LOA, Return to Work, Transfer), Personnel Action Types, and screen modifications |
| **Stage 9.1 — Employee Hiring** | Running the BH Hire action end-to-end through the infogroup sequence, using the worked sample hire data |
| **Stage 10 — Time Management** | Calendars (public holiday, factory), Daily/Period Work Schedules and Work Schedule Rules, Absence/Attendance Types and Quotas, Time Evaluation schemas and PCRs, and running Time Evaluation (PT60) |

The master checklist also documents **Stages 11–15** (Payroll Configuration including Canadian tax/statutory deductions and FI/CO posting, Master Data Maintenance, Authorization Setup, day-to-day business process execution, and Reports & Verification) as the full path to a working, postable payroll run — the proof point that closes out Phase 1.

**Final Checkpoint for Phase 1** is a 10-point demonstration: hire an employee, see them appear correctly in the org chart, enter and evaluate their time, run a payroll simulation and review the payslip, post that payroll to FI, then run a termination, an org reassignment, and a rehire, and reconcile it all against the standard reports. Only once all ten run cleanly does the system move into Phase 2.

---

## Phase 2 — SuccessFactors Integration

**Goal:** transform the now-working on-premise system into a **Core Hybrid** landscape — SAP SuccessFactors Employee Central as the system of record for HR master data, with SAP S/4HANA HCM retained for Payroll, Time Management, and other on-premise processes.

### Stage 1 — Migration (S/4HANA HCM → Employee Central)

Covers the one-time data migration that establishes Employee Central as the new master:

- **Architecture & integration flow** for the S4 → EC migration, including the Infoporter programs used for extraction and the inventory/staging tables that track migration status
- **Migration cutoff date vs. FTSD (First Time Stored Data)** — the two key dates that govern what data moves and why FTSD cannot be changed after go-live
- **Employee scope** — who is included in the migration, with explicit handling for terminated employees and employees on leave of absence at the moment of cutover
- **Migration event patterns** — how active, terminated, and on-leave employees should each appear in Employee Central's Job Information history and in `IT0000` on go-live day
- **Organizational data migration** — mapping S/4HANA OM object types (Org Unit, Position, Job, Cost Center) to Employee Central Foundation Objects (Business Unit, Division, Department, Job Code, Position), with the full migration sequence and transformation templates
- **Employee data replication via the Compound Employee API**, including the employee identifier strategy and key mapping table (`PAOCFEC_EEKEYMAP`)
- **Infotype-to-EC-entity mapping** for Canada, including Canada-specific National ID (SIN) handling
- **EC Event Reason setup** required before migration, with the S4 → EC event reason value mapping
- **Execution steps, validation, and the go-live readiness gate** that confirms the migration succeeded

### Stage 2 — Replication (Employee Central → S/4HANA, ongoing sync)

Covers the continuous, post-go-live replication that keeps S/4HANA HCM in sync with Employee Central as the new system of record:

- **Position Management and PA/PD integration behavior** under the Core Hybrid model
- **Employee Class, Employment Type, Personnel Area/Subarea** mapping rules between EC and S4
- **FTE handling** and its downstream impact on `IT0007-EMPCT` and `IT0008-BSGRD`
- **Wage type scope** — what is migrated once vs. what is replicated on every change
- **Address, phone, and email replication** — the three address templates and two phone number types supported
- **Rehire with new employment** — the special handling this scenario requires
- **Organizational data replication** (SF → S4) for ongoing org changes after go-live
- **Employee data replication** via the Compound Employee API, including the precise difference between Person-level and Employment-level data
- **Business Integration Builder (BIB)** transformation templates, with exact field mappings taken from the SAP Replication Workbook, value mapping, secondary mapping, and conversion rules
- **BAdI extensibility points** for custom replication logic
- **Key tables and key programs/transactions reference** for troubleshooting replication issues

### Employee Migration Scenarios

Three fully worked, side-by-side scenarios that trace a single employee's record through all three migration phases (before migration in S4, during migration via Infoporter, and after migration via BIB replication):

- **Scenario 1 — Active Employee** (John Smith)
- **Scenario 2 — Leave of Absence Employee** (Sarah Johnson)
- **Scenario 3 — Terminated Employee** (Mark Chen)

Each scenario ends with a summary comparison covering EC Job Information slices at migration, `IT0000` behavior in S4 post-migration, and BIB replication behavior — making explicit exactly how each employee category should look at every point in the transformation.

---

## Why This Structure

Real HR Transformation programmes fail more often on sequencing and integration points than on any individual configuration step. A Personnel Subarea created before its grouping decisions are made has to be reworked. A migration run before Event Reasons exist in Employee Central produces orphaned records. This repository is deliberately built in dependency order — Phase 1 stages are numbered because they must be done in that order, and Phase 2 cannot begin until Phase 1's ten-point checkpoint passes — so that the structure of the repository itself teaches the structure of a real transformation programme.

---

## Reference Sources

| Topic | Link |
|---|---|
| HCM Local Enterprise Structure — Configuration Guide | [help.sap.com — PDF](https://help.sap.com/doc/3449018718a1497a94f73ff42d7a70d1/2311/en-US/HCM_Local_Enterprise_Structure_Configuration_Guide_10M.pdf) |
| HCM Global Personnel Administration — Configuration Guide | [help.sap.com — PDF](https://help.sap.com/doc/0c77e45a7117450b8123b5bc4fcc2414/2305/en-US/HCM_Global_Personnel_Administration_Configuration_Guide.pdf) |
| Local Payroll Administration — Configuration Guide | [help.sap.com — PDF](https://help.sap.com/doc/0019b42ad1414b3eb9a9383591dc643e/2305/en-US/Local_Payroll_Administration_Configuration_Guide.pdf) |
| Migrating Data from ERP HCM to EC — Configuration Guide | [help.sap.com — PDF](https://help.sap.com/doc/39b5eeb5766d4a76bf585ef81ef153ab/2311/en-US/Migrating-Data-from-ERP-HCM-to-EC-Config-Guide-4BE.pdf) |
| Personnel Subarea — SAP ERP HCM Help | [help.sap.com](https://help.sap.com/docs/ERP_HCM/73275d5d354843ecad62cbe3e4c243e8/adc2dc53b5ef424de10000000a174cb4.html) |
| What is SAP HCM for S/4HANA On-Premise (H4S4)? | [SAP Community blog](https://community.sap.com/t5/enterprise-resource-planning-blog-posts-by-members/what-is-sap-hcm-for-s-4hana-on-premise-h4s4/ba-p/13493317) |
| The 5 Commonly Used SAP HCM Features (NUMKR, ABKRS, PINCH, SCHKZ, TARIF, LGMST) | [SAP Community blog](https://community.sap.com/t5/enterprise-resource-planning-blog-posts-by-members/the-5-commonly-used-sap-hcm-features/ba-p/13441668) |
| Beginner's Guide to Payroll Personnel Calculation Rules (PCR) | [Part I](https://community.sap.com/t5/human-capital-management-blog-posts-by-members/beginner-s-guide-to-payroll-personnel-calculation-rules-pcr-i/ba-p/14304844) · [Part II](https://community.sap.com/t5/human-capital-management-blog-posts-by-members/beginner-s-guide-to-payroll-personnel-calculation-rules-pcr-ii/ba-p/14370343) |
| Master Data Configuration in SAP HCM for S/4HANA | [SAP Learning course](https://learning.sap.com/courses/master-data-configuration-in-sap-hcm-for-s-4hana) |
| Get Started with SAP HCM Payroll — Running Payroll | [SAP Learning journey](https://learning.sap.com/learning-journeys/get-started-with-sap-hcm-payroll/running-payroll_e1629f8b-5236-4a89-af07-6531cb316d9b) |

---

*Scope: SAP ECC / S/4HANA HCM on-premise build → SuccessFactors Employee Central Core Hybrid migration & replication. Reference companies: KMZP Consulting, BLDP Blood Pact Consulting.*
