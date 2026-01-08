# Observability Task Design – Take-Home Challenge

**Candidate:** Kosha Rajesh Gohil  
**System:** Serverless File Upload & Malware Scanning Pipeline (AWS S3 + Lambda + ClamAV)

---

## System Overview

This system scans user-uploaded files for malware using a fully serverless AWS architecture:

1. Users upload files to **uploads-bucket**
2. S3 triggers **clamav-scan-lambda**
3. Lambda downloads the file to `/tmp` and scans it using **ClamAV**
4. Files are moved to:
   - **clean-bucket** if clean  
   - **quarantine-bucket** if infected
5. A scan result JSON record is written to **results-bucket**

This pipeline processes security-critical data and requires strong observability to ensure reliability, correctness, and security.

---
## Dataset Conventions (used across all tasks)

The following datasets are used across all observability tasks.

### 1) Lambda Application Logs (JSON Lines)
Each Lambda invocation emits multiple structured log events (start, download, scan, move, end).

**Schema**
- `timestamp` (ISO8601)
- `level` (INFO, WARN, ERROR)
- `service` (`clamav-scan-lambda`)
- `function_version`
- `request_id`
- `cold_start` (true/false)
- `source_bucket`
- `object_key`
- `object_size_bytes`
- `download_ms`
- `scan_ms`
- `move_ms`
- `total_ms`
- `clamav_db_version`
- `clamav_result` (CLEAN, INFECTED, ERROR)
- `signature_name` (if infected)
- `destination_bucket`
- `error_type` (timeout, tmp_full, access_denied, kms, clam_error)
- `error_message`
- `retry_attempt`

---

### 2) Lambda Metrics (CSV, 1-minute resolution)

**Schema**
- `timestamp`
- `function_name`
- `metric_name` (Invocations, Errors, DurationP95Ms, DurationP99Ms)
- `value`

---

### 3) Scan Result Records (JSON, written to results-bucket)

**Schema**
- `scan_id`
- `timestamp`
- `request_id`
- `source_bucket`
- `object_key`
- `sha256`
- `object_size_bytes`
- `clamav_result`
- `signature_name`
- `destination_bucket`
- `destination_key`
- `move_status` (SUCCESS, FAILURE)
- `attempt_number`

---

### 4) S3 Object Inventory (CSV, daily snapshots)

**Schema**
- `snapshot_date`
- `bucket`
- `key`
- `size_bytes`
- `last_modified`
- `etag`

---

### 5) Change Events (CSV)

**Schema**
- `timestamp`
- `component` (S3, Lambda, IAM, ClamAV-layer)
- `change_type` (deploy, config_change, policy_change)
- `old_value`
- `new_value`
- `actor`


## Task 1 — Files uploaded but not scanned

### Problem
Users upload files successfully, but some files never appear in either the clean or quarantine bucket.

### What the intern must do
- Detect that uploads are not being processed  
- Verify Lambda is being triggered  
- Identify S3 event notification misconfiguration  
- Fix the trigger  
- Add monitoring to detect this condition  

### Datasets
- Lambda invocation metrics  
- Lambda logs  
- S3 inventory for all buckets  
- S3 event configuration change logs  

### Log Schema
Uses the shared **Lambda Application Logs**, **Lambda Metrics**, **Scan Result Records**, and **S3 Inventory** datasets defined above.


### Data Characteristics
- S3 prefix or suffix filter prevents events from firing  
- No errors appear; scans simply do not start
  
### Dataset Size & Time Span
- 7 days of data
- ~200,000 S3 inventory rows
- ~50,000 Lambda log entries
- ~10,000 scan result records
- Metrics at 1-minute resolution

---

## Task 2 — Scan latency and timeouts

### Problem
Large file uploads cause Lambda timeouts and slow scans.

### What the intern must do
- Detect increase in scan duration and errors  
- Identify file size as the root cause  
- Tune Lambda memory and timeout  
- Add alerting for scan latency  

### Datasets
- Lambda duration metrics  
- Lambda logs with file size and scan time  
- Scan result records  

### Log Schema
Uses Lambda Application Logs with emphasis on:
- `object_size_bytes`
- `scan_ms`
- `total_ms`
- `clamav_result`
- `error_type`

and Lambda Metrics:
- `DurationP95Ms`
- `DurationP99Ms`
- `Errors`

### Data Characteristics
- Few very large files cause most failures  
- Cold starts increase latency
  
### Dataset Size & Time Span
- 48 hours of data
- ~1–3 million Lambda log entries
- ~10,000 metric points
- Heavy-tailed file size distribution

---

## Task 3 — Infected files not quarantined

### Problem
Some infected files appear in the clean bucket.

### What the intern must do
- Compare scan results vs file locations  
- Identify mismatches  
- Find race conditions or retries  
- Fix logic and add verification  

### Datasets
- Scan result records  
- S3 inventory for clean and quarantine buckets  
- Lambda move/copy logs  

### Result Schema
Uses **Scan Result Records** and **S3 Object Inventory** to compare:
- `clamav_result`
- `destination_bucket`
- `actual bucket location from inventory`
  
### Dataset Size & Time Span
- 14 days of data
- One scan result record per uploaded file
- Daily S3 inventory snapshots

---

## Task 4 — AccessDenied and KMS failures

### Problem
Some scans fail with permission or encryption errors.

### What the intern must do
- Identify error spikes  
- Find affected object prefixes  
- Trace to IAM or bucket policy change  
- Fix permissions and add alerts  

### Datasets
- Lambda logs with AWS error codes  
- Lambda error metrics  
- Policy change logs  
- S3 inventory  

### Log Fields
Uses Lambda Application Logs filtered by:
- `error_type` = access_denied or kms
- `object_key`
- `destination_bucket`
and Change Events to correlate failures with IAM or bucket policy updates.

### Dataset Size & Time Span
- 7 days of data
- ~500,000 Lambda log entries
- Error spike concentrated in a 2-hour window

---

## Task 5 — Alerting and SLO design

### Problem
Alerts are noisy and true failures are missed.

### What the intern must do
- Define pipeline health metrics  
- Create dashboards  
- Define SLOs  
- Build actionable alerts  

### Key Metrics
- Uploads per minute  
- Scans completed per minute  
- Scan success rate  
- p95 scan duration  
- Infected file rate  

### SLO Examples
- 99% of uploads scanned within 2 minutes  
- 99.9% scan success rate  
- 100% infected files quarantined
  
### Dataset Size & Time Span
- 30 days of metrics
- Per-minute aggregates (~43,200 rows per metric)
- Alert history including both false positives and real incidents

---

## Design Summary

| Area | Approach |
|------|---------|
| Logs vs Metrics | Both used in every task |
| Data Sources | Lambda logs, Lambda metrics, S3 inventory, scan results, config changes |
| Time Range | 48 hours to 30 days |
| Challenges | Bursty traffic, retries, missing events, race conditions, permission drift |
| Goal | Train engineers to detect, debug, and prevent production failures |

## Why this is a good observability training system

This serverless malware scanning pipeline is a strong observability training system because it combines security, performance, reliability, and cloud configuration failures in a single workflow.
