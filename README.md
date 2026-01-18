Discogs Lakehouse (Local) — Run-based, Reproducible Pipeline

This project implements a local-first Discogs lakehouse with run-based versioning, immutable snapshots, and atomic promotion of validated data.

The system is designed to behave like a real production data platform, while remaining fully reproducible on a laptop.

================================================================================


Project structure

The architecture is intentionally split into two repositories, each with a clear responsibility.


1) Infrastructure layer

Trino + Hive Metastore + external table bootstrap

Repo:
👉 https://github.com/PabloPablo666/trino-hive-setup

Responsibilities:
	•	Trino compute engine (stateless)
	•	Hive Metastore backed by Postgres
	•	External table registration
	•	Stable SQL contract for consumers

This layer can be destroyed and recreated at any time without touching the data.


2) Pipeline & validation layer

Discogs ingestion, transformations, tests, orchestration

Repo:
👉 https://github.com/PabloPablo666/discogs_tools_refactor

Responsibilities:
	•	Download Discogs dumps
	•	Stream-parse large XML files
	•	Write typed Parquet datasets
	•	Build analytical warehouse tables
	•	Run validation tests
	•	Produce sanity reports
	•	Promote validated data atomically

This repo owns the data lifecycle.

================================================================================


Core design principles


✅ Run-based architecture

Every pipeline execution creates an immutable snapshot:
hive-data/
└── _runs/
    └── YYYYMMDD_HHMMSS/

Each run contains:
•	base typed datasets
•	derived warehouse datasets
•	reports and logs

Nothing is overwritten.
Every run is fully reproducible.


✅ Active pointer (publish layer)

Consumers never read from _runs.

Instead, a single symbolic link is used:

hive-data/active -> _runs/20260117_192144

Promotion is performed by switching this pointer atomically.

Benefits:
	•	zero-downtime data publishing
	•	instant rollback
	•	stable table locations in Trino


✅ Immutable data, mutable pointer

Data is immutable.
Only the pointer moves.

This is the same principle used by:
	•	data warehouses
	•	lakehouse systems
	•	versioned datasets in production

================================================================================


Data layout

Physical storage (run snapshot)
hive-data/
└── _runs/
    └── <run_id>/
        ├── artists_v1_typed/
        ├── artist_aliases_v1_typed/
        ├── artist_memberships_v1_typed/
        ├── masters_v1_typed/
        ├── releases_v6/
        ├── labels_v10/
        ├── collection/
        ├── warehouse_discogs/
        └── _reports/

Each directory contains Parquet files only.

================================================================================


Logical access (Trino)

Trino external tables always point to:       
file:/data/hive-data/active/...

As a result:
	•	SQL never changes
	•	dashboards never change
	•	notebooks never change

Only the underlying run changes after promotion.

================================================================================


Pipeline stages

The pipeline is orchestrated with Digdag and follows a strict lifecycle.

1) Preflight
	•	validate environment variables
	•	verify dump availability
	•	compute run_id


2) Download (optional)
	•	download Discogs dumps by month
	•	idempotent (skips existing files)


3) Ingest
	•	streaming XML parsing (no full-file loading)
	•	typed canonical datasets written as Parquet
	•	one dataset per entity

Examples:
	•	artists_v1_typed
	•	labels_v10
	•	masters_v1_typed
	•	releases_v6


4) Build warehouse

Derived analytical datasets are generated:
	•	artist_name_map_v1
	•	release_artists_v1
	•	release_label_xref_v1
	•	label_release_counts_v1
	•	genre/style cross-reference tables

These tables are optimized for analytics, not raw storage.


5) Run-level parquet sanity checks

Executed on the current run directory before promotion.

Examples:
	•	required datasets exist
	•	directories not empty
	•	structural sanity

If these fail, the run is aborted.


6) Promote

If all checks pass:

active -> _runs/<run_id>

The previous active pointer is backed up automatically:

active__prev_<timestamp>
Rollback is a single filesystem operation.


7) Post-promotion Trino sanity report

After promotion, Trino is used to validate real query behavior.

Checks include:
	•	row counts
	•	null ratios
	•	orphan foreign keys
	•	duplicate keys
	•	cross-table integrity

Results are exported as CSV.
_runs/<run_id>/_reports/trino_sanity_active_<timestamp>.csv

This creates a permanent audit trail.

================================================================================


Why this design matters

✔ Reproducibility

Any historical run can be re-queried exactly as it was produced.

✔ Safe experimentation

New dumps can be ingested without touching production data.

✔ Atomic publishing

Consumers see either old data or new data, never partial states.

✔ Rollback

One symlink switch.

✔ Auditable

Every run produces structured validation reports.

✔ Infrastructure independence

Trino and Hive can be rebuilt freely.

================================================================================

What this project is not
	•	not a toy ETL
	•	not a one-off parser
	•	not overwrite-based ingestion
	•	not “just some Parquet files”

It behaves like a real lakehouse pipeline.

================================================================================


Legal note

Discogs data is subject to Discogs licensing terms.

This project:
	•	does not ship Discogs datasets
	•	does not redistribute dumps
	•	focuses purely on infrastructure and data engineering patterns
