# Discogs Lakehouse (Local)

Run-based, Reproducible Data Platform

This project implements a local-first Discogs lakehouse built around run-based versioning, immutable snapshots, and atomic data promotion.

The system is designed to behave like a real production-grade data platform, while remaining fully reproducible on a single machine.

It prioritizes correctness, auditability, and safety over convenience.

===================================================

## Overview

The platform follows three fundamental rules:
	•	Data is immutable
	•	Every pipeline execution is versioned
	•	Only validated data can be published

Instead of overwriting datasets, every run produces a complete snapshot that can be re-queried at any time.

Publishing is performed by switching a single symbolic pointer.

This mirrors how modern lakehouse systems operate in production.

=====================================================

## Project structure

The architecture is intentionally split into two independent repositories, each with a clearly defined responsibility.


### 1) Infrastructure layer

Trino + Hive Metastore + external table bootstrap

Repository:
👉 https://github.com/PabloPablo666/trino-hive-setup

Responsibilities
	•	Trino compute engine (stateless)
	•	Hive Metastore backed by Postgres
	•	External table registration
	•	Stable SQL contract for consumers

This layer owns query execution only.

It does not own data.

It can be destroyed and recreated at any time without affecting stored datasets.


### 2) Pipeline & validation layer

Discogs ingestion, transformation, validation, orchestration

Repository:
👉 https://github.com/PabloPablo666/discogs_tools_refactor

Responsibilities
	•	Download Discogs dumps
	•	Stream-parse large XML files
	•	Write typed Parquet datasets
	•	Build analytical warehouse tables
	•	Run validation checks
	•	Generate audit reports
	•	Promote validated data atomically

This repository owns the entire data lifecycle.

=============================================================

## Core design principles

✅ Run-based architecture

Every pipeline execution produces an immutable snapshot:
```text
hive-data/
└── _runs/
    └── <run_id>/
```

Each run contains:
	•	canonical typed datasets
	•	derived warehouse datasets
	•	validation reports
	•	execution metadata

Nothing is overwritten.
Every run remains queryable forever.


✅ Active pointer (publish layer)

Consumers never read directly from _runs.

Instead, a single symbolic link defines the published dataset:

hive-data/active -> _runs/2026-01__20260117_192144

Publishing consists of switching this pointer atomically.

Benefits:
	•	zero-downtime publishing
	•	instant rollback
	•	stable table paths in Trino
	•	no partial states ever visible


✅ Immutable data, mutable pointer

Data never changes after creation.

Only the pointer moves.

This is the same principle used by:
	•	data warehouses
	•	lakehouse systems
	•	versioned datasets in production environments

===================================================

## Data layout

Physical storage (run snapshot)

```text
hive-data/
└── _runs/
    └── <run_id>/
        ├── artists_v1_typed/
        ├── artist_aliases_v1_typed/
        ├── artist_memberships_v1_typed/
        ├── masters_v1_typed/
        ├── releases_v6/
        ├── labels_v10/
        ├── release_artists_v1/
        ├── release_label_xref_v1/
        ├── label_release_counts_v1/
        ├── genre_style_xref/
        └── _reports/
```


All datasets are stored as Parquet only.

No mutable formats.
No partial overwrites.

==================================================

## Logical access (Trino)

Trino external tables always point to:
file:/data/hive-data/active/...

As a result:
	•	SQL never changes
	•	dashboards never change
	•	notebooks never change

Only the active pointer changes after promotion.

This decouples compute from storage completely.

==================================================

## Pipeline lifecycle

The ingestion pipeline is orchestrated using Digdag and follows a strict execution model.


### 1) Preflight
	•	validate environment variables
	•	verify dump availability
	•	compute deterministic run_id

The run ID is generated once and propagated to all tasks.


### 2) Download (optional)
	•	downloads Discogs dumps by month
	•	idempotent
	•	skips existing files safely


### 3) Ingest
	•	streaming XML parsing
	•	no full-file loading
	•	constant memory usage

Typed canonical datasets are written:
	•	artists
	•	labels
	•	masters
	•	releases
	•	relationships

Each entity is processed independently.


### 4) Build warehouse

Derived analytical datasets are generated:
	•	artist name mappings
	•	release–artist relationships
	•	release–label relationships
	•	label release aggregations
	•	genre and style normalization tables

These tables are optimized for analytics, not raw storage.


### 5) Run-level parquet sanity checks

Before promotion, filesystem-level validations are executed:
	•	required datasets exist
	•	directories are not empty
	•	basic structural integrity

If any check fails, the run is aborted.

Nothing is published.


### 6) Promote

If all validations pass:
active -> _runs/<run_id>

The previous pointer is preserved automatically:
active__prev_<timestamp>

Rollback is a single filesystem operation.


### 7) Post-promotion Trino sanity report

After promotion, Trino-based validations are executed on the active dataset:
	•	row counts
	•	null ratios
	•	orphan foreign keys
	•	duplicate keys
	•	cross-table integrity

Results are exported as CSV:
_runs/<run_id>/_reports/trino_sanity_active_<timestamp>.csv

This creates a permanent audit trail.

=============================================================

## Why this design matters

✔ Reproducibility

Any historical run can be queried exactly as it was produced.

✔ Safe experimentation

New dumps can be ingested without touching production data.

✔ Atomic publishing

Consumers see either old data or new data. Never partial states.

✔ Rollback

One symlink switch.

✔ Auditability

Every run produces structured, timestamped validation reports.

✔ Infrastructure independence

Trino and Hive can be rebuilt freely without data loss.

==========================================================

## What this project is not
	•	not a toy ETL
	•	not overwrite-based ingestion
	•	not a one-off XML parser
	•	not “just some Parquet files”

It behaves like a real lakehouse pipeline.

===========================================================

## Legal note

Discogs data is subject to Discogs licensing terms.

This project:
	•	does not distribute Discogs datasets
	•	does not ship dumps
	•	focuses exclusively on data engineering architecture and patterns
