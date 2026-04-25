# DataVisSUS TXT2SQL Agent

[![Python Version](https://img.shields.io/badge/python-3.11+-blue.svg)](https://www.python.org/downloads/)
[![OpenAI](https://img.shields.io/badge/OpenAI-GPT--4o--mini-412991.svg)](https://openai.com/)
[![LangChain](https://img.shields.io/badge/LangChain-0.3.x-orange.svg)](https://github.com/langchain-ai/langchain)
[![LangGraph](https://img.shields.io/badge/LangGraph-0.6.6-purple.svg)](https://github.com/langchain-ai/langgraph)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.115.13-009688.svg)](https://fastapi.tiangolo.com/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-sihrd5-336791.svg)](https://www.postgresql.org/)

> An AI-powered Text-to-SQL system for Brazilian public healthcare data (DATASUS SIH-RD/SUS), built with LangGraph, OpenAI, PostgreSQL, and a multi-interface architecture.

## Table of Contents
- [Overview](#overview)
- [Features](#features)
- [Architecture](#architecture)
- [Technology Stack](#technology-stack)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Running the Application](#running-the-application)
- [Usage](#usage)
- [Evaluation](#evaluation)
- [Database Context](#database-context)
- [System Design](#system-design)
- [Observability](#observability)
- [Contributing](#contributing)
- [License](#license)
- [Acknowledgments](#acknowledgments)

## Overview
**DataVisSUS TXT2SQL Agent** is an intelligent assistant that translates natural-language questions in Portuguese into executable SQL over a PostgreSQL database containing SIH-RD/SUS hospital data. The project is designed for healthcare analytics, public health exploration, and SUS-oriented data access without requiring users to know the schema or write SQL manually.

### Key Capabilities
- **Natural-Language Querying in Portuguese**: Accepts user questions about SUS hospital data and converts them into SQL.
- **Query Routing**: Distinguishes between database questions, conversational prompts, schema inspection, and clarification flows.
- **Healthcare-Aware Table Selection**: Combines heuristics, schema descriptions, and semantic ranking to choose the right tables.
- **SQL Safety and Validation**: Blocks unsafe SQL and validates generated queries before execution.
- **Multi-Interface Access**: Supports CLI, FastAPI, and a separate web frontend.
- **Operational Observability**: Includes structured logs and LangSmith tracing hooks.

## Features
- **LangGraph-Orchestrated Pipeline**: Stateful workflow for classification, planning, SQL generation, validation, execution, and response formatting.
- **Domain-Specific Prompting**: Table descriptions and templates tailored to SIH-RD/SUS data semantics.
- **Semantic Table Selection**: Embedding-based ranking via `sentence-transformers` to improve table retrieval for ambiguous queries.
- **Model Switching Support**: OpenAI model can be changed at runtime through orchestrator methods and API exposure.
- **REST API**: FastAPI service exposing query, schema, model, and health endpoints.
- **Web Interface**: Node/Express frontend that proxies requests to the agent API.
- **Evaluation Harness**: Includes benchmark runners, ground-truth datasets, and artifact generation for agent vs baseline comparisons.

## Architecture
```mermaid
flowchart TD
    U[User Question] --> C[classify_query]
    C -->|DATABASE| T[list_tables]
    C -->|SCHEMA| S[get_schema]
    C -->|CONVERSATIONAL| R[generate_response]
    C -->|CLARIFICATION| CL[clarification]

    T --> G[get_schema]
    G --> P[query_planner / reasoning]
    P --> Q[generate_sql]
    Q --> V[validate_sql]
    V -->|valid| E[execute_sql]
    V -->|invalid| RS[repair_sql]
    RS --> V
    E --> R

    DB[(PostgreSQL<br/>sihrd5)]
    DB --> E

    LLM[OpenAI Chat Model]
    LLM --> C
    LLM --> T
    LLM --> Q
    LLM --> RS
    LLM --> R
```

## Technology Stack
| Component | Technology | Purpose |
|-----------|-----------|---------|
| **LLM Framework** | LangChain | Prompting, tool integration, SQL utilities |
| **Graph Workflow** | LangGraph 0.6.6 | Stateful orchestration for the agent pipeline |
| **Language Model** | OpenAI GPT-4o-mini | Query understanding, SQL generation, repair, responses |
| **API Layer** | FastAPI 0.115.13 | REST API for query processing and health endpoints |
| **Database** | PostgreSQL | Storage for SIH-RD/SUS relational data |
| **Database Access** | SQLAlchemy + psycopg2 | Connection and execution layer |
| **Semantic Retrieval** | sentence-transformers 2.6.1 | Table-selection embedding cascade |
| **Frontend** | Node.js + Express | Independent web UI server |
| **Observability** | LangSmith | Tracing and experiment logging |

## Project Structure
```bash
kevyn/
├── src/                                # Main application code
│   ├── agent/                          # LangGraph nodes, orchestrator, SQL generation, validation
│   ├── application/config/             # Table descriptions, templates, runtime config
│   ├── infrastructure/database/        # Database connection services
│   ├── interfaces/api/                 # FastAPI server
│   ├── interfaces/cli/                 # CLI interface
│   ├── memory/                         # Semantic table-selection assets
│   └── utils/                          # Logging, SQL safety, utility helpers
├── baselines/                          # Baseline implementations for evaluation
│   └── rich_prompt_baseline/
├── evaluation/                         # Benchmark runners, metrics, reports, visualizations
├── frontend/                           # Independent web interface
├── tests/                              # Unit and integration-style tests
├── docs/                               # Project documentation and supporting material
├── logs/                               # Runtime logs
├── requirements.txt                    # Python dependencies
├── .env.example                        # Environment variable example
└── README.md                           # This file
```

## Getting Started
### Prerequisites
- **Python**: 3.11 or higher
- **PostgreSQL**: Accessible instance containing the `sihrd5` database
- **OpenAI API Key**: Required for query classification and SQL generation
- **Node.js**: 16+ if you want to run the optional web interface

### Installation
1. **Enter the project directory**
```bash
cd /home/maiconkevyn/PycharmProjects/DataVisSUS/kevyn
```

2. **Set up Python environment**
```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

3. **Optional: install frontend dependencies**
```bash
cd frontend
npm install
cd ..
```

### Configuration
Create a `.env` file from the provided example:

```bash
cp .env.example .env
```

Required variables:

```env
OPENAI_API_KEY=sk-your_openai_key_here

LANGSMITH_TRACING=true
LANGSMITH_API_KEY=your_langsmith_api_key_here
LANGCHAIN_PROJECT=txt2sql

DATABASE_PATH=postgresql+psycopg2://postgres:your_password@localhost:5432/sihrd5
```

Notes:
- `DATABASE_PATH` is read from `DATABASE_URL` or `DATABASE_PATH`.
- The code normalizes `postgresql+psycopg2://` to `postgresql://` internally before creating the SQLAlchemy connection.
- Default application model is `gpt-4o-mini`.

### Running the Application
1. **Run the CLI**
```bash
python src/interfaces/cli/agent.py
```

2. **Run the API**
```bash
python src/interfaces/api/main.py
```

The API will be available at [http://localhost:8000/docs](http://localhost:8000/docs).

3. **Run the web interface** (optional)
```bash
cd frontend
npm run dev
```

The web interface will be available at [http://localhost:3000](http://localhost:3000).

## Usage
### CLI Examples
**1. Single query**
```bash
python src/interfaces/cli/agent.py --query "Quantas mortes ocorreram em 2022?"
```

**2. Interactive mode**
```bash
python src/interfaces/cli/agent.py
```

**3. Debug workflow mode**
```bash
python src/interfaces/cli/agent.py --query "Quantos hospitais existem?" --debug-steps
```

**4. Override the model**
```bash
python src/interfaces/cli/agent.py --model gpt-4o --query "Quais municípios tiveram mais internações?"
```

### API Endpoints
- `POST /api/v1/query` or `POST /query`: process a natural-language query
- `GET /api/v1/schema` or `GET /schema`: inspect table descriptions
- `GET /api/v1/models` or `GET /models`: inspect available/current models
- `GET /api/v1/health` or `GET /health`: health check

### Example Questions
- `Quantas mortes ocorreram em 2022?`
- `Quais municípios tiveram mais internações?`
- `Qual é a idade média das mulheres que morreram?`
- `Mostre a estrutura da tabela internacoes`

## Evaluation
The repository includes an evaluation suite for measuring agent quality over healthcare query datasets, plus a rich-prompt baseline for comparison.

### Main Components
- `evaluation/run_dag_evaluation.py`: evaluates the LangGraph agent
- `evaluation/run_rich_prompt_baseline.py`: evaluates the single-shot baseline
- `evaluation/ground_truth.json`: benchmark dataset
- `evaluation/results/`: generated reports, JSON summaries, and plots

### Running Evaluation
```bash
python evaluation/run_dag_evaluation.py
python evaluation/run_rich_prompt_baseline.py
```

Artifacts are written to:
- `evaluation/results/`
- `baselines/rich_prompt_baseline/artifacts/`

## Database Context
This project targets the **`sihrd5` PostgreSQL database**, which contains SIH-RD/SUS hospital data and support tables. The agent relies heavily on curated metadata in:

- `src/application/config/table_descriptions.py`
- `src/application/config/table_templates.py`

The database used by the project includes 16 public tables:
- `atendimentos`
- `cid`
- `contraceptivos`
- `especialidade`
- `etnia`
- `hospital`
- `instrucao`
- `internacoes`
- `municipios`
- `nacionalidade`
- `procedimentos`
- `raca_cor`
- `sexo`
- `socioeconomico`
- `tempo`
- `vincprev`

## System Design
### Query Routing
The agent first decides whether the user request is:
- a database query
- a conversational request
- a schema request
- a clarification case

This prevents unnecessary SQL generation for non-database prompts.

### Healthcare-Aware SQL Generation
SQL generation is grounded in:
- curated table descriptions
- table-specific prompt templates
- SUS-specific value mappings
- database schema fetched at runtime

This is where most domain precision comes from. The quality of `table_descriptions.py` and `table_templates.py` directly affects table selection and SQL accuracy.

### Safety and Repair Loop
Before execution, SQL is validated and sanitized. Unsafe statements are blocked by `src/utils/sql_safety.py`, and invalid SQL can be repaired using error context and schema feedback instead of failing immediately.

### Model Management
The orchestrator exposes:
- available models
- current model metadata
- runtime switching support
- model-level health information

## Observability
The project includes:
- **LangSmith tracing** through `.env` configuration
- **Structured logs** in `logs/`
- **Health endpoints** in the API and web interface
- **Model and orchestrator health reports** from runtime services

Useful log files include:
- `logs/txt2sql_orchestrator.log`
- `logs/txt2sql_nodes.log`
- `logs/txt2sql_llm_manager.log`
- `logs/txt2sql_api.log`
- `logs/txt2sql_cli.log`

## Contributing
Contributions should focus on one of the following high-leverage areas:
- improving table descriptions and prompt templates
- expanding test coverage for routing, validation, and SQL safety
- refining evaluation datasets and failure analysis
- improving API and frontend ergonomics

When changing agent behavior, update both the evaluation artifacts and the documentation that explains the intended routing/template logic.

## License
This project is licensed under the **MIT License** and maintained in the context of **DaVInt Lab (Data Visualization and Interaction), PUCRS**.

From the [LICENSE](LICENSE) file:

> Copyright (c) 2026 DaVInt Lab (Data Visualization and Interaction), PUCRS
>
> Permission is hereby granted, free of charge, to any person obtaining a copy
> of this software and associated documentation files (the "Software"), to deal
> in the Software without restriction, including without limitation the rights
> to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
> copies of the Software, and to permit persons to whom the Software is
> furnished to do so, subject to the following conditions.