# Orbit Arsenal — Project Plan

## 1. Project Vision

Orbit Arsenal will be an on-premise Earth-observation intelligence platform that lets an analyst:

- search satellite imagery with natural-language descriptions;
- find visually similar imagery using an example image or selected map region;
- filter results by location, date, satellite, sensor, cloud cover, and other metadata;
- compare imagery of the same area across multiple dates;
- detect, rank, and explain meaningful changes;
- distinguish real surface changes from clouds, seasons, atmospheric effects, viewing geometry, sensor differences, and registration errors;
- trace every result back to its source imagery, processing history, model version, and confidence evidence.

The system should support analyst discovery. An analyst should not need to know every location of interest before beginning a search.

## 2. Planning Assumptions

This plan currently assumes:

- the product name is **Orbit Arsenal**;
- the source problem is semantic retrieval and multi-temporal change analysis of satellite imagery;
- the first release is a research-grade MVP, followed by an operational hardening phase;
- only public, properly licensed Earth-observation data will be used;
- the initial imagery sources will be Sentinel-2 and Landsat Collection 2;
- Sentinel-1 SAR and ISRO/Bhuvan data will be added after the optical workflow is stable;
- deployment must be possible on-premise, without depending on a third-party cloud API at runtime;
- model outputs will assist analysts and will not be presented as unquestionable ground truth.

These assumptions must be checked against the current repository and final project requirements before implementation starts.

## 3. MVP Scope

### Included

1. Area-of-interest selection on a map.
2. STAC-compatible imagery catalogue and metadata filters.
3. Incremental ingestion of newly available scenes.
4. Preprocessing and tiling of optical satellite imagery.
5. Text-to-image semantic search.
6. Image-to-image similarity search.
7. Two-date and multi-date scene comparison.
8. Change candidates with masks, confidence, and before/after evidence.
9. Analyst review, confirmation, rejection, and annotation.
10. Provenance records for source data and derived products.
11. An on-premise deployment path using containers.

### Deferred until after the MVP

- real-time satellite tasking;
- classified or operational military data;
- automatic operational decisions;
- unrestricted object identification or tracking;
- full national-scale ingestion from day one;
- training a large remote-sensing foundation model from scratch;
- mobile applications;
- blockchain-based provenance.

## 4. Primary Users and Workflows

### Imagery Analyst

1. Draws an area of interest or searches for a place.
2. Applies time, sensor, resolution, and quality filters.
3. Searches with a phrase such as “new construction near a road” or supplies an example image.
4. Reviews ranked tiles and their metadata.
5. Selects dates for comparison.
6. Inspects highlighted changes with confidence and possible artefact warnings.
7. Confirms, rejects, or annotates the result.
8. Exports an evidence package or report.

### Data Administrator

1. Registers a dataset or STAC source.
2. Starts or schedules ingestion.
3. Monitors failed scenes and quality-control checks.
4. Reprocesses data when a pipeline or model version changes.

### System Administrator

1. Deploys the platform on local infrastructure.
2. Manages users, roles, storage, model versions, backups, and audit logs.
3. Monitors resource usage and service health.

## 5. Proposed System Architecture

```text
Public EO Sources / Local Files
             |
             v
     Catalogue + Ingestion
     (STAC, metadata, jobs)
             |
             v
  Geospatial Preprocessing Pipeline
  (validate, reproject, align, mask,
   normalize, tile, quality metrics)
             |
       +-----+------------------+
       |                        |
       v                        v
 Image/Object Storage      Metadata Database
 raw + COG + tiles         PostgreSQL + PostGIS
       |                        |
       +-----------+------------+
                   |
          +--------+---------+
          |                  |
          v                  v
 Semantic Embeddings    Change Analysis
 + Vector Index         + Time-Series Evidence
          |                  |
          +--------+---------+
                   v
              Backend API
                   |
                   v
      Web Map + Search + Review UI
```

### Suggested Components

- **Frontend:** React and TypeScript with MapLibre GL or OpenLayers.
- **Backend API:** Django and Django REST Framework, unless the existing repository establishes a different stack.
- **Spatial database:** PostgreSQL with PostGIS.
- **Vector retrieval:** pgvector for the MVP; evaluate a dedicated vector engine only if scale requires it.
- **Job execution:** Celery with Redis or RabbitMQ.
- **Object storage:** S3-compatible storage such as MinIO for on-premise deployment.
- **Geospatial processing:** GDAL, Rasterio, GeoPandas, xarray/rioxarray, and pyproj.
- **Catalogue standard:** STAC items, collections, and asset links.
- **Model runtime:** PyTorch with locally hosted, versioned model weights.
- **Packaging:** Docker Compose for development and the initial demonstrator; Kubernetes can remain an optional scale-out path.

No technology choice becomes final until the existing codebase and deployment constraints are reviewed.

## 6. Data and Processing Design

### Initial Datasets

1. Sentinel-2 Level-2A optical imagery.
2. Landsat Collection 2 Level-2 imagery.
3. A small, geographically bounded pilot area with known changes.
4. Public labelled change-detection datasets for evaluation.

Sentinel-1 SAR should be introduced in a separate milestone because SAR preprocessing and interpretation differ substantially from optical imagery.

### Ingestion Record

Each scene should retain:

- provider and collection;
- immutable source identifier and asset URL;
- acquisition and ingestion timestamps;
- geometry, footprint, CRS, and spatial resolution;
- platform, instrument, bands, processing level, and orbit information when available;
- cloud/quality metadata;
- checksum and licence information;
- preprocessing pipeline and model versions;
- parent-child links between raw and derived assets.

### Preprocessing Pipeline

1. Validate metadata, bands, checksums, and spatial footprint.
2. Preserve the original asset or a verifiable source reference.
3. Convert working imagery to Cloud Optimized GeoTIFF where appropriate.
4. Reproject to a consistent analysis grid for the selected area.
5. Apply cloud, shadow, snow, no-data, and saturation masks.
6. Normalize sensor-specific bands without erasing meaningful spectral differences.
7. Co-register dates and calculate alignment-quality metrics.
8. Generate fixed-size, overlapping analysis tiles.
9. Produce thumbnails and map-display assets.
10. Calculate embeddings and index searchable metadata.

All derived products must be reproducible from their recorded source and pipeline version.

## 7. Semantic and Multimodal Retrieval

### Search Modes

- natural-language query to imagery;
- example image to similar imagery;
- selected map tile to similar imagery;
- optional metadata-only search;
- hybrid search combining semantic similarity with spatial, temporal, sensor, and quality constraints.

### Ranking Strategy

Start with a transparent hybrid score:

```text
retrieval_score =
    semantic_weight × semantic_similarity
  + metadata_weight × metadata_relevance
  + quality_weight × image_quality
  + recency_weight × temporal_relevance
```

Weights must be configurable and evaluated on a documented query set. Filters such as geographic intersection and allowed date range should be applied as constraints rather than hidden ranking preferences.

### Retrieval Deliverables

- ranked results with similarity score;
- map and gallery views;
- matching tile and parent-scene context;
- metadata and provenance panel;
- saved searches and analyst relevance feedback;
- explanation of which filters and score components affected ranking.

## 8. Multi-Temporal Change Analysis

### Processing Stages

1. Select two or more comparable acquisitions over the same area.
2. verify geometric alignment and compatible ground sampling distance.
3. exclude invalid pixels using quality masks.
4. normalize radiometry and account for sensor differences.
5. generate one or more change signals:
   - spectral/index difference;
   - feature-embedding difference;
   - segmentation or object-level difference;
   - time-series deviation when enough dates exist.
6. combine signals into candidate change regions.
7. suppress or flag likely nuisance changes.
8. assign confidence and retain the evidence behind it.
9. present before/after views and masks for analyst review.

### False-Change Suppression

The system must explicitly model or flag:

- clouds and cloud shadows;
- snow and water-level variation;
- seasonal vegetation changes;
- atmospheric and illumination differences;
- viewing-angle and terrain effects;
- imperfect co-registration;
- different sensors or processing levels;
- low-data or no-data regions.

### Explainable Change Evidence

For each detected region, show:

- before and after image chips;
- acquisition dates and sensors;
- change mask or polygon;
- area and relevant spectral or feature deltas;
- alignment and valid-pixel quality;
- possible nuisance factors;
- model and threshold version;
- analyst decision and notes.

The displayed confidence should be calibrated on validation data. It must not be described as probability unless calibration supports that interpretation.

## 9. API and Domain Modules

Proposed modules:

- `catalogue`: collections, scenes, assets, footprints, and licences;
- `ingestion`: source adapters, jobs, validation, and retries;
- `processing`: preprocessing runs and derived artefacts;
- `search`: text/image queries, embeddings, ranking, and feedback;
- `change`: comparisons, candidates, masks, evidence, and status;
- `annotations`: analyst labels and comments;
- `provenance`: lineage, model versions, checksums, and audit trail;
- `accounts`: users, roles, access policy, and sessions;
- `exports`: reports and evidence packages.

Initial API groups:

```text
/api/catalogue/
/api/ingestion/
/api/search/
/api/comparisons/
/api/change-candidates/
/api/annotations/
/api/exports/
/api/system/health/
```

## 10. Delivery Phases

### Phase 0 — Repository and Requirement Audit

- inspect the current code, dependencies, data model, tests, and deployment files;
- recover the exact problem statement and six required capabilities;
- document existing versus missing functionality;
- select one pilot geography and a small date range;
- define measurable acceptance criteria;
- create a risk register and data-licensing record.

**Exit condition:** approved MVP scope, architecture decision record, and pilot dataset.

### Phase 1 — Catalogue and Map Foundation

- set up spatial database and object storage;
- implement STAC-compatible scene metadata;
- ingest a small Sentinel-2 sample incrementally;
- provide map browsing and conventional metadata filters;
- record source provenance.

**Exit condition:** an analyst can find and display ingested scenes by area, date, and sensor.

### Phase 2 — Preprocessing and Quality Control

- implement reproducible preprocessing jobs;
- generate aligned analysis tiles and thumbnails;
- store cloud/valid-pixel and registration metrics;
- add retry, idempotency, and job monitoring;
- validate geospatial correctness with automated tests.

**Exit condition:** the same source and pipeline version reproducibly produce the same registered tiles.

### Phase 3 — Semantic Retrieval

- benchmark suitable remote-sensing vision-language embeddings;
- implement text and image embedding pipelines;
- add vector indexing and hybrid ranking;
- build text-to-image and image-to-image UI flows;
- create a human-labelled retrieval evaluation set.

**Exit condition:** the system meets agreed Recall@K/nDCG targets on the pilot queries and returns traceable results.

### Phase 4 — Two-Date Change Detection

- implement date-pair selection and comparability checks;
- build baseline spectral and feature-difference methods;
- generate masks, polygons, confidence evidence, and nuisance flags;
- add swipe, side-by-side, and overlay visualisations;
- add analyst confirmation and rejection.

**Exit condition:** known pilot changes are detected at the agreed precision/recall and false changes are visibly explained or flagged.

### Phase 5 — Multi-Temporal Analysis

- support more than two observations per area;
- create temporal signatures and persistent-versus-transient classification;
- add trend and anomaly views;
- rank change events by confidence, magnitude, persistence, and analyst relevance.

**Exit condition:** analysts can distinguish persistent changes from temporary or seasonal observations over the pilot area.

### Phase 6 — Operational Hardening

- implement role-based access, audit logging, backups, and retention controls;
- add offline model packaging and on-premise installation documentation;
- conduct performance, security, failure-recovery, and usability tests;
- add monitoring for ingestion, storage, queues, models, and API health;
- document model limitations and analyst-review policy.

**Exit condition:** a clean on-premise environment can deploy, operate, back up, and restore the platform using documented procedures.

### Phase 7 — Sensor and Scale Expansion

- add Sentinel-1 SAR through a sensor-specific preprocessing pipeline;
- evaluate Landsat/Sentinel cross-sensor comparisons;
- add Bhuvan sources where licensing and interfaces permit;
- introduce distributed processing only after profiling shows it is necessary;
- use analyst feedback to improve ranking and change models.

## 11. Evaluation Plan

### Retrieval Metrics

- Recall@K;
- Precision@K;
- mean reciprocal rank;
- nDCG;
- query latency;
- analyst relevance rating;
- performance by query type, geography, season, and sensor.

### Change-Detection Metrics

- pixel- and object-level precision, recall, F1, and IoU;
- false alarms per square kilometre;
- minimum reliably detected change size;
- detection performance by change type;
- performance under cloud, seasonal, alignment, and cross-sensor conditions;
- confidence calibration error;
- analyst acceptance/rejection rate.

### System Metrics

- scenes processed per hour;
- ingestion-to-searchable latency;
- failed-job and retry rates;
- storage per scene and per derived product;
- API latency and concurrent-user capacity;
- reproducibility across repeated runs;
- completeness of provenance records.

## 12. Testing Strategy

- unit tests for spatial calculations, score composition, and metadata validation;
- contract tests for source adapters and APIs;
- integration tests for ingestion-to-search and comparison-to-review flows;
- geospatial tests for CRS, bounds, resolution, nodata, and alignment;
- golden-data tests for deterministic derived products;
- model evaluation tests against frozen validation sets;
- UI tests for core analyst workflows;
- load tests for vector search, tile delivery, and worker queues;
- security tests for access control, uploads, secrets, and audit records;
- deployment and restore tests in an isolated on-premise environment.

## 13. Security, Privacy, and Responsible Use

- process only authorised, properly licensed datasets;
- apply least-privilege role-based access;
- encrypt external connections and protect stored credentials;
- maintain immutable or tamper-evident audit events for sensitive actions;
- validate all uploads and remote-source metadata;
- avoid exposing internal storage paths or signed asset URLs unnecessarily;
- retain model/version information for every inference;
- require analyst confirmation before reporting high-impact conclusions;
- document known blind spots, confidence limits, and prohibited uses;
- establish data retention, deletion, backup, and incident-response procedures.

## 14. Key Risks and Mitigations

| Risk | Mitigation |
|---|---|
| False changes caused by cloud, season, or misalignment | Quality masks, registration metrics, temporal context, nuisance flags, and analyst review |
| Weak text-to-image retrieval in specialised domains | Benchmark remote-sensing models, curate domain queries, use hybrid ranking, and collect relevance feedback |
| Cross-sensor inconsistency | Begin with same-sensor comparisons and add sensor-specific normalization/evaluation later |
| Very large storage and compute requirements | Use a bounded pilot, COGs, tiling, incremental processing, retention policies, and profiling |
| Poorly calibrated confidence | Evaluate calibration explicitly and display evidence components instead of a misleading single probability |
| Dataset leakage or optimistic evaluation | Separate geography/time splits and preserve a frozen blind test set |
| Model or dataset licensing restrictions | Maintain a licence register before downloading or distributing artefacts |
| Scope becoming too broad | Enforce phase exit criteria and defer SAR, national scale, and advanced models until the optical MVP works |
| On-premise hardware limitations | Provide small/medium deployment profiles and benchmark CPU/GPU/storage needs early |

## 15. Initial Backlog

### P0 — Start Here

- [ ] Audit the existing Orbit Arsenal repository.
- [ ] Copy the authoritative problem statement into project documentation.
- [ ] Identify and document all required capabilities.
- [ ] Select the pilot area, dates, and expected change categories.
- [ ] Confirm imagery licences and download methods.
- [ ] Record the available CPU, GPU, RAM, and storage.
- [ ] Create architecture and data-model decision records.
- [ ] Define the first retrieval and change-detection evaluation sets.

### P1 — Platform Foundation

- [ ] Create the scene, asset, processing-run, tile, embedding, comparison, candidate, and annotation models.
- [ ] Configure PostGIS, vector search, object storage, workers, and health checks.
- [ ] Add the first Sentinel-2 ingestion adapter.
- [ ] Implement idempotent jobs, provenance, checksums, and retries.
- [ ] Build the base map and metadata filtering workflow.

### P2 — Intelligence Features

- [ ] Establish keyword and metadata baselines before ML retrieval.
- [ ] Benchmark text/image embedding models.
- [ ] Implement hybrid semantic retrieval.
- [ ] Establish a simple image-difference baseline before advanced change models.
- [ ] Add false-change suppression and explainable evidence.
- [ ] Build analyst feedback and annotation workflows.

### P3 — Demonstration and Validation

- [ ] Prepare reproducible before/after demonstration cases.
- [ ] Run quantitative evaluation and record limitations.
- [ ] Measure end-to-end latency and resource use.
- [ ] Package an on-premise demonstration deployment.
- [ ] Create technical, user, and evaluation documentation.

## 16. MVP Definition of Done

The MVP is complete when an analyst can:

1. ingest new public Sentinel-2 scenes without rebuilding the catalogue;
2. locate scenes using map, metadata, natural language, or an example image;
3. compare valid acquisitions over the same area;
4. view ranked change regions with before/after evidence and quality warnings;
5. confirm, reject, and annotate results;
6. reproduce every result from recorded data, configuration, and model versions;
7. run the complete workflow on documented on-premise infrastructure;
8. review measured retrieval and change-detection performance on a held-out pilot dataset.

## 17. Decisions Required Before Coding

1. What functionality already exists in the current repository?
2. What are the exact six capabilities in the authoritative problem statement?
3. Which pilot location and change types will demonstrate the system?
4. Is the MVP limited to Sentinel-2 optical imagery?
5. What hardware and storage are available for on-premise deployment?
6. What response-time and accuracy targets are realistic for the submission?
7. Which users and roles are needed in the first release?
8. Is offline installation mandatory, or is internet allowed only during dataset acquisition?
9. Which export formats and report contents are required?
10. What repository conventions, frameworks, and tests must new work follow?

The answers should be captured as architecture decision records before major implementation begins.
