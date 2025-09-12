# From Natural Language to Standards‑Aligned 5G Core Configuration

*All of the code and materials supporting my Master’s thesis project.*

> **Goal:** Convert human‑readable intent into standards‑aligned 5G Core (Open5GS) configuration using an ontology/knowledge‑graph + LLM/RAG pipeline grounded in 3GPP specifications.

---

## Table of contents

* [Overview](#overview)
* [Repository structure](#repository-structure)
* [Core ideas & features](#core-ideas--features)
* [Environment setup](#environment-setup)
* [Configuration (.env)](#configuration-env)
* [Pipelines](#pipelines)
* [Data & artifacts](#data--artifacts)
* [Usage: end‑to‑end run](#usage-end-to-end-run)
* [Design & architecture](#design--architecture)
* [Results & evaluation](#results--evaluation)
* [Roadmap / TODO](#roadmap--todo)
* [Contributing](#contributing)
* [License](#license)
* [Acknowledgements](#acknowledgements)

---

## Overview

This repository contains code, notebooks, and datasets used to build a domain‑aware RAG system that translates *natural language intent* into a **5G Core configuration** consistent with **3GPP** standards. The system combines:

* A **5G ontology & knowledge graph (KG)** capturing entities (NFs, interfaces, parameters) and their relationships
* **Document ingestion** of standards PDFs into a graph database for retrieval
* **Prompting & reasoning** with an LLM constrained by the ontology and KG
* **Config synthesis** targeting **Open5GS** YAMLs and validation against constraints

Although the artifacts are thesis‑oriented, the code and structure aim to be reusable for practitioners who want standards‑grounded, explainable config generation.

---

## Repository structure

> *Paths reflect the current repo; each item explains **what it is** and **why it’s in the repo**.*

```
MS_Thesis-From_Natural_Language_to_Standards_Aligned_5G_Core_Configuration/
├─ 5g_standards_pdf/                  
│   └─ <3GPP_23-5xx_*.pdf> ...        # Source standards PDFs
├─ artefacts/                         # Generated outputs (YAML, CSV, reports)
│   └─ .gitkeep                       # Keeps folder under version control when empty
├─ "gathered data"/                   # Curated structured data used by notebooks
│   └─ yang_ontology.ttl              # Working ontology in Turtle
└─ pipelines/                         # Reproducible notebooks and helper data
    ├─ 01_training_pipeline.ipynb     # Builds helpers/lexicons/models from CSVs
    ├─ 02_ingest_pdf_to_auradb.ipynb  # Parses PDFs and loads KG into Neo4j/AuraDB
    ├─ MGR_*                          # Prototype notebooks/utilities (experiments)
    └─ data/
        ├─ decompose.csv              # Seed decomposition dataset (intent → parts)
        └─ taxonomy.csv               # Domain taxonomy used for validation/prompts
        └─ classes.csv                # Hierarchy of classes used for KG
        └─ reltype.csv                # Identified relationships between data
        └─ triples.csv                # Fact to triplet mappings
```

### Why these are included

* **`5g_standards_pdf/`** – *Needed for ingestion*. The `02_ingest_pdf_to_auradb.ipynb` notebook reads from this folder. If it’s empty, the KG will be empty and retrieval will return nothing. If licensing prevents committing full PDFs, keep the folder with a README placeholder.
* **`artefacts/`** – *Needed for outputs*. Notebooks write exports (Open5GS YAML, CSV slices of the KG, figures). We keep the folder (with a `.gitkeep`) so paths resolve in fresh clones.
* **`"gathered data"/yang_ontology.ttl`** – *Needed for ontology‑first flow*. The TTL file encodes classes/relations/constraints. Several notebooks read or export from this file; without it, ontology checks and KG alignment are incomplete.
* **`pipelines/`** – *The heart of the workflow*. Keeping notebooks versioned enables reproducibility and lets others run the exact steps.
* **`pipelines/data/`** – *Small but essential seed data*. The training and prompt‑building cells expect these CSVs on relative paths. Removing this folder breaks notebook cells or changes results. Keeping them in‑repo ensures deterministic runs.
---

## Core ideas & features

* **Ontology‑first modeling:** Core NFs (AMF/SMF/UPF/AUSF/UDM/PCF/NSSF/NRF), interfaces (N1–Nn), and constraints encoded as triples.
* **KG‑assisted retrieval:** Standards text ingested into Neo4j/AuraDB; queries pull precise, citable context for the LLM.
* **Intent → config mapping:** Natural‑language requests (e.g., PLMN, slice/SST/SD, DNN/APN, TAC lists) are mapped to Open5GS config stanzas.
* **Validation layer:** Simple rule checks (e.g., range/domain checks, cross‑NF dependencies) before emitting YAML.
* **Reproducible notebooks:** Training & ingestion steps captured in Jupyter for transparency.

---

## Environment setup

Tested on Python **3.10–3.11**.

### Windows (PowerShell)

```powershell
# Clone
git clone https://github.com/sofiia2002/MS_Thesis-From_Natural_Language_to_Standards_Aligned_5G_Core_Configuration.git
cd MS_Thesis-From_Natural_Language_to_Standards_Aligned_5G_Core_Configuration

# Virtual env
python -m venv .venv
.\.venv\Scripts\activate

# Install deps (if a requirements or environment file is present)
pip install -r requirements.txt  # if available
# otherwise install the packages referenced in notebooks (Neo4j driver, pandas, numpy,
# scikit-learn, networkx/rdflib, ipykernel, jupyterlab, pyyaml, tiktoken/openai/azure libs, etc.)
```

### Mac / Linux (bash)

```bash
git clone https://github.com/sofiia2002/MS_Thesis-From_Natural_Language_to_Standards_Aligned_5G_Core_Configuration.git
cd MS_Thesis-From_Natural_Language_to_Standards_Aligned_5G_Core_Configuration
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt  # if available
```

> If you plan to version large model files or `trained_models.zip`, install **Git LFS** and track them:
>
> ```bash
> git lfs install
> git lfs track "*.zip"
> git add .gitattributes
> ```

---

## Configuration (.env)

Create a `.env` file in the repo root with credentials/paths (edit as needed):

```dotenv
# LLM provider(s)
OPENAI_API_KEY=your_key_here
# or AZURE_OPENAI_API_KEY=...

# Neo4j / AuraDB
NEO4J_URI=neo4j+s://<host>:<port>
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=your_password
NEO4J_DATABASE=neo4j

# Local paths
DATA_DIR=./5g_standards_pdf
ARTEFACTS_DIR=./artefacts
```

> Tip: Use a library like `python-dotenv` in notebooks to load these automatically.

---

## Pipelines

### 1) Training pipeline – `pipelines/01_training_pipeline.ipynb`

* Preprocesses curated CSVs (`pipelines/data/*.csv`)
* Trains light classifiers or keyword/NER components used to anchor prompts
* Saves artifacts into `artefacts/` or an LFS‑tracked bundle

### 2) Standards ingestion – `pipelines/02_ingest_pdf_to_auradb.ipynb`

* Parses PDFs under `5g_standards_pdf/`
* Chunks and embeds text; writes nodes/relationships to Neo4j/AuraDB
* Produces indexes and example Cypher queries for retrieval

### 3) Prototype utilities – `pipelines/MGR_*`

* Experiments for ontology → Neo4j (Turtle/TTL to graph), OpenAI‑assisted extraction
* Ad‑hoc scripts to validate triples and export for RAG

---

## Data & artifacts

* **`5g_standards_pdf/`** – repository of the standards text to ingest. *Why keep it?* The graph is built from here; having at least a few representative PDFs allows end‑to‑end runs. *Note:* verify redistribution rights; if unsure, commit only file names and a placeholder.
* **`"gathered data"/yang_ontology.ttl`** – ontology source of truth. *Why keep it?* It defines the vocabulary used by prompts/validators and ensures generated configs are standards‑aligned.
* **`pipelines/data/decompose.csv`** – maps user intents to configuration components. *Why keep it?* Training and rule‑based steps depend on these examples for stable behavior.
* **`pipelines/data/taxonomy.csv`** – domain taxonomy (NFs, interfaces, parameters, allowed values). *Why keep it?* Enables validation (e.g., checking SST/SD ranges, NF dependencies) and consistent prompts.
* **`artefacts/`** – destination for generated outputs. *Why keep it (even empty)?* Ensures all relative paths used in notebooks resolve; add a `.gitkeep` so the folder stays in Git.
  
---

## Usage: end‑to‑end run

1. **Prepare services**

   * Obtain Neo4j/AuraDB credentials and create a database.
   * Set `OPENAI_API_KEY` (or provider of choice) in `.env`.
2. **Load standards**

   * Place PDFs into `5g_standards_pdf/`.
   * Run `02_ingest_pdf_to_auradb.ipynb` to build the graph.
3. **Train helpers**

   * Run `01_training_pipeline.ipynb` to build light models/dictionaries.
4. **Query & generate**

   * Use provided Cypher examples in the notebooks to retrieve context.
   * Run the generation cells to produce **Open5GS** YAML fragments.
5. **Validate & export**

   * Run validation checks; export final YAML into `artefacts/`.

---

## Design & architecture

* **Ontology & KG** provide guardrails: only allowed entities/relationships are emitted.
* **Retrieval** pulls exact standards passages for traceability.
* **LLM prompts** cite ontology concepts + retrieved snippets for deterministic structure.
* **Validation** enforces constraints (value ranges, cross‑NF references, required fields).

> This architecture favors safety and auditability over fully free‑form generation.

---

## Results & evaluation

* Measure: config **correctness** (schema/constraint checks), **coverage** (intents supported), and **traceability** (links to standards nodes/passages).
* Baselines: template‑only generation vs RAG‑guided generation.
* See `artefacts/` for example outputs once generated.

---

## Contributing

PRs and issues are welcome. Please:

1. Open an issue describing the change
2. Keep notebooks tidy (restart & run all before committing)
3. Avoid committing secrets or large binaries (use LFS)

---

## License

*No license specified yet.* If you plan to use or share this work, consider adding a license (e.g., MIT for code) and clarifying the status of included standards documents.

---

## Acknowledgements

* 3GPP working groups and specifications that inform the ontology and constraints
* Open5GS community
* Neo4j / AuraDB ecosystem
* Everyone who provided feedback on the thesis direction
