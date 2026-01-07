# Discogs Lakehouse (Local) — End-to-End Project

This is a **local-first, reproducible Discogs lakehouse project** split into two repositories:

1) **Infrastructure layer**: Trino + Hive Metastore + external tables bootstrap  
2) **Pipeline & validation layer**: streaming ingestion → typed Parquet + tests + analytical query pack

The split is intentional: **infrastructure** evolves differently than **pipelines**, and each repo remains clean and focused.

---

## Repositories

### 1) trino-hive-setup (Infrastructure)
- Docker Compose for Trino + Hive Metastore (Postgres-backed)
- Idempotent SQL bootstrap that registers external Parquet tables and views
- Sanity checks runnable in Trino

➡️ Repo: `https://github.com/PabloPablo666/trino-hive-setup`

### 2) discogs_tools_refactor (Pipelines, tests, SQL)
- Streaming XML → Parquet ingestion pipelines
- DuckDB-based tests for correctness and regressions
- SQL query portfolio / showcase queries (Trino-friendly)

➡️ Repo: `https://github.com/PabloPablo666/discogs_tools_refactor`

---

## Architecture (high level)

- **Storage**: external Parquet data lake (outside Docker)
- **Metadata**: Hive Metastore (Postgres-backed)
- **Compute**: Trino (stateless)
- **Pipelines**: Python streaming parsers writing typed datasets (`*_v1_typed`)

You can destroy and rebuild compute + metastore **without touching the data lake**.

---

## Data layout (typed-first)

The lake uses a typed canonical layout, e.g.:

discogs_data_lake/
└── hive-data/
├── artists_v1_typed/
├── artist_aliases_v1_typed/
├── artist_memberships_v1_typed/
├── masters_v1_typed/
├── releases_v6/
├── labels_v10/
├── collection/
└── warehouse_discogs/
├── artist_name_map_v1/
├── release_artists_v1/
├── release_label_xref_v1/
└── ...

The lakehouse exposes stable logical names (`*_v1`) as **VIEWs** over the typed physical datasets.

---

## End-to-end quick start

### 0) Requirements
- Docker + Docker Compose
- Python 3 (for pipelines)
- Discogs dumps (not included in repos)

### 1) Build Parquet datasets (pipelines repo)

Clone pipelines repo:
```bash
git clone https://github.com/PabloPablo666/discogs_tools_refactor.git
cd discogs_tools_refactor
Set the data lake root (example path):

export DISCOGS_DATA_LAKE=/Users/you/discogs_data_lake/hive-data
Run pipeline tests (example):

./tests/run_test_artists_v1.sh /path/to/discogs_YYYYMMDD_artists.xml.gz
Run ingestion pipelines (see discogs_tools_refactor/README.md for the exact commands per dataset).

2) Start Trino + Metastore (infra repo)
Clone infra repo:

git clone https://github.com/PabloPablo666/trino-hive-setup.git
cd trino-hive-setup
Create .env (example):

printf "DISCOGS_DATA_LAKE=%s\n" "/Users/you/discogs_data_lake/hive-data" > .env
Start services:

docker compose up -d
Bootstrap schema + external tables + views:

docker exec -it trino trino --catalog hive --file /etc/trino/bootstrap_discogs.sql
Optional: run sanity checks:

docker exec -it trino trino --catalog hive --file /etc/trino/sanity_checks_trino.sql
Verify:

docker exec -it trino trino --catalog hive --schema discogs --execute "SHOW TABLES"
3) Run portfolio queries
Portfolio / showcase queries live in:

discogs_tools_refactor/sql/

Run them via Trino after bootstrap.

Notes
Discogs data is subject to Discogs licensing and terms.

These repositories do not distribute Discogs datasets.

This project focuses on reproducible infrastructure, typed storage, and verifiable pipelines.

---
