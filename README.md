# GCP Resource Label Remover

Scans GCP resources for labels, removes only the label keys you specify
(or all labels if no filter is given), and produces an audit CSV report.

---

## Prerequisites

- Python 3.10+
- Google Cloud SDK installed and authenticated
- Service Account or user credentials with the IAM roles listed below

### Required IAM Roles

| Service Flag | Required Role |
|---|---|
| `compute` | `roles/compute.instanceAdmin.v1` |
| `networking` | `roles/compute.networkAdmin` |
| `gcs` | `roles/storage.admin` |
| `bigquery` | `roles/bigquery.dataEditor` |
| `gke` | `roles/container.clusterAdmin` |
| `pubsub` | `roles/pubsub.admin` |
| `run` | `roles/run.admin` |
| `functions` | `roles/cloudfunctions.admin` |
| `sql` | `roles/cloudsql.admin` |
| `kms` | `roles/cloudkms.admin` |
| `secretmanager` | `roles/secretmanager.admin` |
| `monitoring` | `roles/monitoring.alertPolicyEditor` |
| `dns` | `roles/dns.admin` |
| `artifactregistry` | `roles/artifactregistry.admin` |
| `vertexai` | `roles/aiplatform.admin` |
| `vertexaisearch` | `roles/discoveryengine.admin` |
| `dataplex` | `roles/dataplex.admin` |
| `notebooks` | `roles/notebooks.admin` |
| `workstations` | `roles/workstations.admin` |
| `healthcare` | `roles/healthcare.datasetAdmin` |
| `scheduler` | `roles/cloudscheduler.admin` |
| `datacatalog` | `roles/datacatalog.admin` |

---

## Setup

### 1. Authenticate

```bash
gcloud auth application-default login
```

### 2. Install dependencies

```bash
pip install -r requirements_gcp_labels.txt
```

---

## Usage

### Dry-run — safe preview (default)

Scans all resources and shows what would be changed. **No labels are removed.**

```bash
python gcp_label_remover.py --project YOUR_PROJECT_ID --dry-run
```

### Filter by specific label keys (recommended)

Only removes labels whose **key** matches the supplied names.
All other labels on the resource are left untouched.

```bash
# Preview — show which costcentre and apmid labels would be removed
python gcp_label_remover.py --project YOUR_PROJECT_ID --dry-run \
  --label-keys costcentre,apmid

# Execute — remove only costcentre and apmid labels
python gcp_label_remover.py --project YOUR_PROJECT_ID --execute \
  --label-keys costcentre,apmid
```

### Remove ALL labels (no key filter)

Omit `--label-keys` to remove every label from every matched resource.

```bash
python gcp_label_remover.py --project YOUR_PROJECT_ID --execute
```

### Scan specific services only

```bash
python gcp_label_remover.py --project YOUR_PROJECT_ID --dry-run \
  --label-keys costcentre,apmid \
  --services compute,gcs,bigquery,kms,secretmanager
```

### Custom CSV output path

```bash
python gcp_label_remover.py --project YOUR_PROJECT_ID --execute \
  --label-keys costcentre,apmid \
  --output my_audit_2026.csv
```

---

## Options

| Flag | Default | Description |
|---|---|---|
| `--project` | *(required)* | GCP project ID |
| `--dry-run` / `--execute` | `--dry-run` | Preview or apply changes |
| `--label-keys` | *(none — removes all)* | Comma-separated label key names to target (e.g. `costcentre,apmid`) |
| `--regions` | *(none — all regions)* | Comma-separated GCP regions to scan (e.g. `us-central1,us-east1`) |
| `--fuzzy-threshold` | `0.0` (disabled) | Similarity ratio 0.0–1.0 for fuzzy key matching |
| `--services` | all 22 services | Comma-separated list of services to scan |
| `--output` | `gcp_label_audit_<timestamp>.csv` | CSV output file path |

---

## Limiting the scan to specific regions

If most of your resources are in one or two regions, use `--regions` to avoid
scanning every GCP location. This has two effects:

1. **Services that iterate locations** (KMS, Healthcare, Cloud Scheduler,
   Vertex AI, Data Catalog, Vertex AI Search) skip every API call outside
   the specified regions — the biggest speed and quota saving.
2. **Services that use a `locations/-` wildcard** (Compute, GCS, GKE,
   Cloud Run, etc.) still make a single global list call, but any resource
   returned in a different region is discarded before being recorded.

**Global resources** (Secret Manager secrets, DNS zones, Pub/Sub topics,
Cloud Monitoring alert policies) are *always* included regardless of `--regions`
because they have no meaningful geographic location.

**Zone mapping** — if your resources are in zones like `us-central1-a`,
just specify the parent region `us-central1`; zones are automatically
matched to their region.

```bash
# Scan only us-central1 (saves significant time on large projects)
python gcp_label_remover.py --project my-project --dry-run \
  --label-keys costcentre,apmid \
  --regions us-central1

# Scan two regions
python gcp_label_remover.py --project my-project --execute \
  --label-keys costcentre,apmid \
  --regions us-central1,us-east1
```

---

## Handling spelling variants of the same key

In large organisations with many projects and teams, the same label key is
often entered with different spellings across resources. The tool handles this
with three matching layers applied in order:

| Layer | What it matches | Example |
|---|---|---|
| **Exact** (always on) | Case-insensitive equality | `Costcentre` → `costcentre` |
| **Normalized** (always on) | Strip `-` `_` and spaces, then compare | `cost-centre`, `cost_centre` → `costcentre` |
| **Fuzzy** (`--fuzzy-threshold`) | Edit-distance similarity ratio | `costcenter`, `costecentre` → `costcentre` |

### Choosing a fuzzy threshold

| Threshold | Ratio | What it catches |
|---|---|---|
| `0.82` (recommended) | ≥ 0.82 | US/UK spelling (`costcenter` = 0.90), extra chars (`costecentre` = 0.95) |
| `0.80` | ≥ 0.80 | Also catches severe typos (`costcanter` = 0.80) — slightly more false-positive risk |

Every non-exact match is logged as a **WARNING** and written to the
`key_matches` column in the audit CSV so you can review every variant
that was automatically matched.

### Example

```bash
python gcp_label_remover.py --project my-project --dry-run \
  --label-keys costcentre,apmid --fuzzy-threshold 0.82
```

Resource with mixed variants:

```json
{ "costcentre": "cc-123", "cost-centre": "cc-456",
  "costcenter":  "cc-789", "apmId": "42", "env": "prod" }
```

Result:

```
labels_removed : {"costcentre": "cc-123", "cost-centre": "cc-456",
                  "costcenter":  "cc-789", "apmId": "42"}
labels_kept    : {"env": "prod"}
key_matches    : {"cost-centre": "costcentre",   ← normalised
                  "costcenter":  "costcentre",   ← fuzzy (0.90)
                  "apmId":       "apmid"}        ← case
```

---

## How label key filtering works

When `--label-keys` is supplied:

- Only labels whose key matches a target (by any of the three layers) are removed.
- All other labels on the resource are preserved.
- Resources that carry **none** of the target keys are skipped entirely and do not appear in the CSV.
- During removal the tool performs a **live GET** of each resource to read its current labels, then writes back only the surviving keys. This prevents accidentally wiping labels added between the scan and the delete.

---

## Reliability features

### Fingerprint conflict retry (Compute Engine, GKE, Networking)

Compute Engine and GKE use a label fingerprint to detect concurrent
modifications. If a resource is changed between the GET and the SET, the
API returns an error and the write is automatically retried up to 3 times
with exponential backoff (1 s → 2 s → 4 s).

Two error codes are handled:
- `FailedPrecondition` (HTTP 412) — returned by Compute Engine
- `Aborted` (HTTP 409) — returned by GKE

### Quota / rate-limit retry

`ResourceExhausted` (HTTP 429) and `ServiceUnavailable` (HTTP 503) errors
trigger automatic retries with exponential backoff (up to 5 attempts,
capped at 60 s). These errors are **not** silently skipped.

### Pagination

All REST-based list calls (Cloud SQL, Cloud DNS, Cloud Healthcare) follow
`nextPageToken` until all pages are exhausted. Earlier versions silently
returned only the first page.

---

## Supported Services and Resources

### High confidence (well-tested APIs)

| Service Flag | Resources Scanned | Confidence |
|---|---|---|
| `gcs` | Cloud Storage Buckets | 88% |
| `bigquery` | Datasets, Tables | 88% |
| `kms` | Cloud KMS Crypto Keys | 88% |
| `secretmanager` | Secret Manager Secrets | 90% |
| `monitoring` | Alert Policies | 85% |
| `artifactregistry` | Artifact Registry Repositories | 82% |
| `dns` | Cloud DNS Managed Zones | 82% |
| `compute` | Instances, Disks, Snapshots, Images | 83% |
| `networking` | Forwarding Rules (regional + global), VPN Gateways, External VPN Gateways | 83% |
| `gke` | GKE Clusters | 84% |

### Medium confidence

| Service Flag | Resources Scanned | Confidence |
|---|---|---|
| `pubsub` | Topics, Subscriptions | 80% |
| `sql` | Cloud SQL Instances | 82% |
| `scheduler` | Cloud Scheduler Jobs | 80% |
| `run` | Cloud Run Services | 82% |
| `functions` | Cloud Functions (gen2) | 82% |
| `healthcare` | Healthcare Datasets, DICOM Stores, FHIR Stores, HL7v2 Stores | 75% |
| `dataplex` | Dataplex Lakes, Zones, Assets | 72% |
| `notebooks` | Vertex AI Workbench Instances | 72% |
| `workstations` | Workstation Clusters, Configs, Workstations | 72% |
| `vertexai` | Vertex AI Datasets, Models, Endpoints | 72% |

### Lower confidence (newer / less-documented APIs)

| Service Flag | Resources Scanned | Confidence | Known limitation |
|---|---|---|---|
| `vertexaisearch` | Discovery Engine Data Stores (`default_collection`) | 65% | Custom collections not scanned |
| `datacatalog` | Data Catalog Entry Groups | 60% | `EntryGroup.labels` field not guaranteed in all API versions |

### Services without label support (noted for reference)

| Service | Reason |
|---|---|
| Cloud Logging | `LogBucket` / `LogSink` have no `labels` field in the API |
| App Engine | Service / Version resources have no labels API |
| Cloud Tasks | Queue (v2 API) has no `labels` field |
| VM Manager | Patch deployments have no `labels` field |
| Velostrata / Click-to-Deploy | Resources are standard Compute VMs/images — use `compute` |
| BigQuery Storage API | Covered by the `bigquery` service flag |

---

## Audit CSV

A CSV file is generated after every run.

| Column | Description |
|---|---|
| `service` | Service flag (e.g. `kms`, `gcs`) |
| `resource_type` | Resource kind (e.g. `cryptokey`, `bucket`) |
| `name` | Short resource name |
| `resource_id` | Full GCP resource path used in API calls |
| `location` | Zone, region, or `global` |
| `labels_removed_count` | Number of label keys removed |
| `labels_removed` | JSON of the label key-value pairs that were deleted |
| `labels_kept` | JSON of the label key-value pairs preserved on the resource |
| `key_matches` | JSON mapping each variant key to the canonical target it matched (empty for exact matches) |
| `status` | `removed`, `dry-run`, `failed`, or `skipped` |
| `error` | Error message if `status = failed` |
| `timestamp` | UTC timestamp of the operation |

### Example output

```
service,resource_type,name,...,labels_removed_count,labels_removed,labels_kept,key_matches,status
compute,instance,web-1,...,2,"{""costcentre"":""cc-123"",""apmid"":""42""}","{""env"":""prod""}","",removed
gcs,bucket,my-bucket,...,1,"{""cost-centre"":""cc-456""}","{""team"":""ops""}","{""cost-centre"":""costcentre""}",removed
kms,cryptokey,my-key,...,1,"{""apmid"":""99""}","{}","",dry-run
```

---

## Running Tests

Unit tests mock every GCP library — **no real GCP connection or credentials needed**.

```bash
# Run all 99 tests
python3 -m unittest test_gcp_label_remover -v

# Or with pytest
pip install pytest
pytest test_gcp_label_remover.py -v
```

### What is tested (99 tests)

| Test class | Coverage |
|---|---|
| `TestNewLabels` | Core filter logic: matching, case-insensitivity, no-filter, empty labels |
| `TestMatchedTarget` | All three matching layers: exact / normalised / fuzzy, priority, threshold boundary |
| `TestRecordVariantKeys` | Variant keys populate `key_matches`; exact matches excluded; CSV correct |
| `TestRecord` | Label splitting, skipping non-matching resources, field population |
| `TestDryRun` | Status = `dry-run`, zero mutation API calls for GCS / KMS / BigQuery |
| `TestFetchCompute` | Instance fetch: recorded / skipped / no-labels / multi-resource |
| `TestFetchGcs` | Bucket fetch: match / no-match / no-filter |
| `TestFetchSecretManager` | Secret fetch with and without target keys |
| `TestFetchKms` | Crypto key fetch across locations |
| `TestRemoveGcs` | Live GET before PATCH; single / multi-key removal; no-filter clears all |
| `TestRemoveBigQuery` | Dataset and table — `labels` attribute directly verified |
| `TestRemoveCompute` | `set_labels` called with live fingerprint and surviving labels only |
| `TestRemoveKms` | Live `get_crypto_key` + correct `CryptoKey(labels=…)` payload |
| `TestRemoveSecretManager` | Live `get_secret` + correct `Secret(labels=…)` payload |
| `TestRemoveMonitoring` | `user_labels` updated correctly on alert policies |
| `TestRemoveCloudRun` | Minimal `Service` object constructed with surviving labels only |
| `TestCsvOutput` | All columns, JSON fields, counts, statuses, multi-row output |
| `TestFilterBehaviour` | End-to-end: only matching resources collected; full dry-run run |
| `TestRegionFiltering` | `_region_of()`, `_in_regions()`, `_record()` gating, `_list_locations()` short-circuit |

---

## Recommended Go-Live Sequence

Label removal is irreversible. Follow this sequence to validate before running
at full scope.

```bash
# Step 1: Preview — scoped to your primary region, high-confidence services only
python gcp_label_remover.py --project my-project --dry-run \
  --label-keys costcentre,apmid \
  --regions us-central1 \
  --services compute,gcs,bigquery,kms,secretmanager \
  --fuzzy-threshold 0.82

# Step 2: Review the CSV — verify every resource and label variant looks correct
open gcp_label_audit_*.csv

# Step 3: Execute on the narrowest possible scope first
python gcp_label_remover.py --project my-project --execute \
  --label-keys costcentre,apmid \
  --regions us-central1 \
  --services gcs

# Step 4: Expand service by service, checking logs for errors after each run
python gcp_label_remover.py --project my-project --execute \
  --label-keys costcentre,apmid \
  --regions us-central1 \
  --services compute,gcs,bigquery,kms,secretmanager

# Step 5: Add lower-confidence services only after the above succeed cleanly
# Exclude vertexaisearch and datacatalog until manually verified in your project
python gcp_label_remover.py --project my-project --execute \
  --label-keys costcentre,apmid \
  --regions us-central1
```

> **Warning:** Always keep the audit CSV as a permanent record of what was
> removed and what was preserved. There is no undo.
