# datasus-conversational-agent

This repository contains the artifacts of the paper "A Conversational Agent for Natural Language Access to Public Health Data", submitted to CBMS 2026.
To the best of our knowledge, we present the first Text-to-SQL system for the Hospital Information System in Reduced Data format (SIH-RD) from DATASUS, enabling queries over 18.7 million hospitalization records from the states of Rio Grande do Sul and Maranhão between 2008 and 2023. 
To address this, we derive fifteen domain-specific SQL generation rules from systematic SIH-RD schema analysis and embed them in a 9-stage LangGraph pipeline with query routing, automatic table selection, chain-of-thought planning, SQL generation, static validation, and bounded self-repair, requiring no model fine-tuning.
We also construct a benchmark of 120 Portuguese healthcare queries stratified into Easy, Medium, and Hard tiers (40 each) with gold-standard SQL over records from the two Brazilian states (2008--2023), comprising the first formal Text-to-SQL evaluation on SIH-RD.
The agent achieves 93.3% execution accuracy (112/120) with 100% pipeline completion; a controlled single-shot baseline sharing identical model, domain rules, and prompts achieves 90.0% (108/120), with the advantage concentrated in Hard queries (+10.0~pp), isolating the contribution of graph orchestration for complex multi-table reasoning.

---

## Database


### Data Source
Data was collected from SIH-RD via the PySUS library, in the Reduced Data format (RD) — the contracted version of DATASUS that enables processing on commodity local hardware. The data consists of public microdata containing no personally identifiable information (PII), ensuring compliance with the Brazilian General Data Protection Law (LGPD).

### Schema

The database is modeled as a Snowflake Schema, optimized for OLAP analytical workloads, comprising:

| Prefix | Type       | DATASUS Name    | FHIR Name               |
|:-------|:-----------|:----------------|:------------------------|
| `TF_`  | Fact       | `internacoes`   | `Encounter`             |
| `RL_`  | Bridge     | `atendimentos`  | `Procedure`             |
| `TD_`  | Dimension  | `hospital`      | `Organization`          |
| `TD_`  | Dimension  | `municipios`    | `Location`              |
| `TD_`  | Dimension  | `cid`           | `ICDCode`               |
| `TD_`  | Dimension  | `procedimentos` | `ProcedureCode`         |
| `TD_`  | Dimension  | `especialidade` | `SpecialtyCode`         |
| `TD_`  | Dimension  | `sexo`          | `AdministrativeSex`     |
| `TD_`  | Dimension  | `raca_cor`      | `RaceCode`              |
| `TD_`  | Dimension  | `etnia`         | `EthnicityCode`         |
| `TD_`  | Dimension  | `nacionalidade` | `NationalityCode`       |
| `TD_`  | Dimension  | `instrucao`     | `EducationCode`         |
| `TD_`  | Dimension  | `vincprev`      | `SocialSecurityCode`    |
| `TD_`  | Dimension  | `contraceptivos`| `ContraceptiveCode`     |
| `TD_`  | Dimension  | `tempo`         | `TimeDimension`         |
| `TD_`  | Analytical | `socioeconomico`| `SocioeconomicIndicator`|

![Database Schema](assets/schema.png)


---

## Benchmark

The benchmark comprises 120 Portuguese-language queries with gold-standard SQL, stratified into three difficulty tiers:

| Tier   | N  | Structural Criteria                                                                |
|:-------|:---|:-----------------------------------------------------------------------------------|
| Easy   | 40 | Single table; basic filter or simple aggregation; no joins                         |
| Medium | 40 | Two-table FK join, or single-table temporal/grouped aggregation                    |
| Hard   | 40 | ≥2 joins, multi-step reasoning, or computed metrics across dimensional hierarchies |

Queries span four analytical dimensions: **aggregation**, **temporal filtering**, **geographic analysis**, and **multi-table joins**.

The `ground_truth.json` file contains for each query: `id`, `difficulty`, `question`, `query` (gold-standard SQL), `tables`, and `notes` with relevant modeling decisions.

---

## Agent Pipeline

The agent is implemented in **LangGraph** across 9 sequential stages:

1. **Query Classification** — routing: DATABASE, conversational, schema, or ambiguous
2. **Table Selection** — identification of relevant tables via regex fast-path or LLM
3. **Schema Retrieval** — fetches columns, types, FKs, and value ranges exclusively for selected tables
4. **CoT Planning** — chain-of-thought planning for complex queries (per-group top-N, global vs. local averages, temporal comparisons)
5. **SQL Generation** — synthesis using RULES A–O, SUS-specific value mappings, and per-table few-shot examples
6. **SQL Validation** — static syntax and schema alignment check
7. **SQL Execution** — query execution against DuckDB
8. **Self-Repair** — error recovery: schema-level (rerouting to table selection) or syntax-level (targeted regeneration), up to 2 retries
9. **Response Generation** — natural-language formatting; guara

### Results

| System              | Easy (n=40) | Medium (n=40) | Hard (n=40)      | Overall (n=120) |
|:--------------------|:------------|:--------------|:-----------------|:----------------|
| Prompt Baseline     | 97.5%       | 100.0%        | 72.5%            | 90.0%           |
| LangGraph Agent     | 100.0%      | 97.5%         | 82.5%            | **93.3%**       |
| Δ (Agent − Baseline)| +2.5 pp     | −2.5 pp       | **+10.0 pp**     | +3.3 pp         |

McNemar's exact test: b=6, c=2, p=0.289. Wilson 95% CI: agent [87.4%, 96.6%], baseline [83.3%, 94.2%].


## Code Availability

> **The agent and data pipeline source code will be made publicly available in this repository upon paper acceptance.**

Artifacts currently available:
- `ground_truth.json` — full benchmark with 120 gold-standard queries
- evaluation reports
- Data dictionary and database schema


# About the Authors
We are members of the Data Visualization and Interaction Lab (DaVInt) at PUCRS:

- Isabel H. Manssour -- Professor Coordinator of DaVInt -- 2017-current.
- Isadora Ferraz e Figueiredo -- Undergraduate Research Student -- 2025-current.
- Maicon Kevyn Moraes da Silva -- Undergraduate Research Student -- 2025
- Victória Cavalheiro Marques -- Undergraduate Research Student -- 2025-current.
