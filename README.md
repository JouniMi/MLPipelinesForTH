# ML-Assisted Threat Hunting Reference Implementation

## Purpose

This repository provides practical examples demonstrating how machine learning and deep learning can assist threat hunters, but it requires significant adaptation including data source configuration changes, hyperparameter tuning based on your specific environment, and understanding that this is not a production-ready solution.

**Note:** This work is based on thesis research. For detailed methodology, experimental results, and theoretical foundations, refer to the thesis document: https://www.theseus.fi/handle/10024/913789.

The evaluation and confusion matrix code is left at the bottom of the each notebook. These were specific to the evaluated environment and WILL NOT work elsewhere. I left them there just for reference.

**BEWARE!** I cleaned up the notebooks with help of AI for publification. All of them were technically tested and work as they should, however, some of the comments may have mistakes in them. I did read them through but might have missed things. As everyone using Jupyter Notebooks knows, the files tend to get messy while you are experimenting so I tried to ensure that they are relatively easy to follow.

## Architecture Overview

```
┌──────────────────────┐
│   OpenSearch Cluster │
│   (Raw Security Logs)│
└──────────┬───────────┘
           │
           │ Scroll API (batch extraction)
           ▼
┌─────────────────────────────────────┐
│   Dockerized Python Service         │
│   - Pulls documents in batches      │
│   - Static PyArrow schema conversion│
│   - Flattens nested structures      │
│   - Normalizes timestamps           │
│   - Full dumps + incremental updates│
│   - 24-hour schedule                │
└──────────┬──────────────────────────┘
           │
           │ Column-wise Parquet files
           │ (compressed, columnar format)
           ▼
┌─────────────────────────────────────┐
│   MinIO Object Storage Bucket       │
│   (Efficient column-based analytics)│
└──────────┬──────────────────────────┘
           │
           │ Spark / PySpark reads
           ▼
┌─────────────────────────────────────┐
│   Jupyter Notebooks                 │
│   (ML/DL threat hunting pipelines)  │
└─────────────────────────────────────┘
```

**Data Origin:** The data was generated in a simulated testing environment with 11 virtual Windows devices producing Sysmon logs. Simulated threats were injected into the environment and detected using the pipelines described in these notebooks.

## Repository Layout

```
notebooks/              # Jupyter notebooks for ML-assisted threat hunting
  01_ad_enumeration_kmeans.ipynb      # K-Means clustering for AD enumeration detection
  02_ad_enumeration_dl.ipynb          # Deep learning autoencoder for AD enumeration
  03_c2_detection.ipynb               # C2 beaconing detection (timing + context)
  04_process_anomaly.ipynb            # Process anomaly detection (Seq2Seq + Isolation Forest)
  05_widespread_attack.ipynb          # Widespread attack detection (autoencoder)
  .env.example                        # ENV variables for Spark cluster, Minio and hyperparameters
.gitignore              # Excludes credentials, models, cache files
```

## Quick Start

There is no one-click quick start. These notebooks require significant refinement for your specific environment:

1. **Configure your data infrastructure:** Set up MinIO/S3-compatible storage with your security logs
2. **Adapt data ingestion:** Modify the data loading cells to match your schema and data source
3. **Tune hyperparameters:** Adjust model parameters based on your environment's baseline behavior
4. **Set environment variables:** Configure MinIO credentials, Spark cluster settings, and detection thresholds
5. **Run notebooks selectively:** Start with one notebook matching your threat hunting objective

Each notebook contains detailed comments explaining where changes are needed. The examples use Parquet files from MinIO, but you must adapt this to your specific data source.

## Configuration – Required Variables

### Environment-Based Configuration

All notebooks use environment variables for configuration. Create a `.env` file in the repository root:

```bash
# Required
MINIO_SECRET_KEY=your-secret-key-here

# MinIO Configuration (S3-compatible storage)
MINIO_ACCESS_KEY=r00t
MINIO_ENDPOINT=http://your-minio-server:9000
MINIO_PATH_STYLE_ACCESS=true

# Data Paths
DATA_PATH=s3a://winlogbeat/winlogbeat/*.parquet
BASE_S3_PATH=s3a://temp/anomaly_detection

# Spark Cluster Configuration
SPARK_MASTER=spark://your-spark-master:7077
SPARK_DRIVER_HOST=your-hostname
SPARK_DRIVER_PORT=7771
SPARK_BLOCK_MANAGER_PORT=7772
SPARK_UI_PORT=8088
SPARK_EXECUTOR_CORES=4
SPARK_EXECUTOR_MEMORY=6G
SPARK_DRIVER_MEMORY=2g

# GPU Configuration (optional)
USE_GPU=true
SPARK_EXECUTOR_GPU_AMOUNT=1
SPARK_TASK_GPU_AMOUNT=1
```

### Notebook-Specific Hyperparameters

Each notebook has model-specific hyperparameters that can be overridden:

```bash
# Example: AD Enumeration K-Means
AD_KMEANS_K=4                    # Number of clusters
AD_TFIDF_NUM_FEATURES=2000       # TF-IDF feature dimension
AD_ANOMALY_THRESHOLD=0.01        # Anomaly detection threshold

# Example: C2 Detection
C2_KMEANS_K=50                   # Timing clusters
C2_BEACON_THRESHOLD_HIGH=8.0     # Upper beacon interval
C2_BEACON_THRESHOLD_LOW=0.8      # Lower beacon interval

# Example: Process Anomaly
PROCESS_MAX_SEQ_LEN=128          # Max sequence length
PROCESS_IF_CONTAMINATION=0.05     # Isolation Forest contamination
```

**Important:** All values have defaults hardcoded in the notebooks, so they work without `.env` file. Override these values to experiment with your data.

### How MinIO Is Used

MinIO (or any S3-compatible storage) serves as the central data repository:

1. **Data Source:** Notebooks read Parquet files from MinIO using Spark's `read.parquet()` with S3a protocol
2. **Model Storage:** Trained models and intermediate results are saved to MinIO for persistence
3. **Output Storage:** Detection results, features, and artifacts are written back to MinIO

The Parquet format provides efficient column-based querying, enabling large-scale security log analysis without loading entire datasets into memory.

### Hyperparameter Tuning

Tuning is **environment-specific** and requires domain knowledge:

1. **Understand your baseline:** Collect normal traffic patterns from your environment
2. **Start with defaults:** Use the provided hyperparameters as a starting point
3. **Iterate incrementally:** Change one parameter at a time and evaluate impact
4. **Validate with labeled data:** If available, use known threat instances to calibrate thresholds
5. **Monitor false positives:** Adjust thresholds to balance detection rate vs. alert fatigue

**Example tuning workflow:**
```python
# Start with defaults (k=4 clusters)
# Run notebook → examine results → many false positives?

# Increase clusters to capture more nuanced patterns
AD_KMEANS_K=8

# Run again → better separation? → adjust detection threshold
AD_ANOMALY_THRESHOLD=0.02
```

Tuning is an ongoing process. What works in one environment may not work in another.

## Adapting to a Different Data Source (API-based SIEM/XDR)

These notebooks assume data from MinIO (Parquet format). To use API-based SIEM/XDR platforms:

### Step 1: Locate Data Loading Cell

Modify **Cell 7: Load Data** in each notebook. Replace the Spark Parquet read with your API client. The following logic is AI created as examples.

### Step 2: Replace Data Loading Logic

**Current approach (MinIO/Parquet) - Cell 7:**
```python
df_raw = spark.read.parquet(DATA_PATH)
```

**Alternative 1: API with Pandas (for smaller datasets):**
```python
import requests
import pandas as pd

# Query SIEM API
response = requests.get(
    "https://your-siem/api/v1/logs",
    headers={"Authorization": "Bearer YOUR_TOKEN"},
    params={"query": "event_code:1644", "limit": 10000}
)

# Convert to Pandas DataFrame
df = pd.DataFrame(response.json()["results"])

# Convert to Spark DataFrame for processing
df_spark = spark.createDataFrame(df)
```

**Alternative 2: API with pagination (for large datasets):**
```python
import requests
from datetime import datetime, timedelta

def fetch_logs_with_pagination(start_time, end_time):
    all_logs = []
    page = 0

    while True:
        response = requests.get(
            "https://your-siem/api/v1/logs",
            headers={
                "Authorization": "Bearer YOUR_TOKEN",
                "Content-Type": "application/json"
            },
            json={
                "query": "event_code:1644",
                "time_range": {
                    "start": start_time.isoformat(),
                    "end": end_time.isoformat()
                },
                "page": page,
                "page_size": 10000
            }
        )

        data = response.json()
        if not data["results"]:
            break

        all_logs.extend(data["results"])

        # Check rate limiting
        if "Retry-After" in response.headers:
            time.sleep(int(response.headers["Retry-After"]))

        page += 1

    return pd.DataFrame(all_logs)

df = fetch_logs_with_pagination(
    start_time=datetime.now() - timedelta(days=7),
    end_time=datetime.now()
)
df_spark = spark.createDataFrame(df)
```

**Alternative 3: Write to Parquet first (for repeated analysis):**
```python
# Fetch from API once, write to local Parquet
df = fetch_logs_with_pagination(...)
df.to_parquet("/tmp/siem_data.parquet", index=False)

# Now notebooks can read efficiently
df_spark = spark.read.parquet("/tmp/siem_data.parquet")
```

### Step 3: Schema Normalization

After fetching data, you may need to normalize the schema. Use this helper function:

```python
def normalize_and_cast(df, spark):
    """
    Flatten nested structures and cast types for Spark compatibility.

    Args:
        df: Pandas DataFrame from API
        spark: SparkSession

    Returns:
        Spark DataFrame with normalized schema
    """
    # Flatten nested JSON if needed
    df_flat = pd.json_normalize(df.to_dict(orient='records'))

    # Cast timestamps
    timestamp_cols = [c for c in df_flat.columns if 'time' in c.lower() or 'date' in c.lower()]
    for col in timestamp_cols:
        df_flat[col] = pd.to_datetime(df_flat[col])

    # Convert to Spark DataFrame
    df_spark = spark.createDataFrame(df_flat)

    return df_spark

df_normalized = normalize_and_cast(df, spark)
```

### Step 4: Update Filters and Column Names

Each notebook uses specific column names in filtering logic. You must map your schema to these expected names.

**Widespread Attack Example (05_widespread_attack.ipynb):**

**Cell 8: Filter Data** expects:
- `winlog_channel` == "Microsoft-Windows-Sysmon/Operational"
- `winlog_event_id` (Event IDs: 1, 3, 11, 19, 20, 21)
- `winlog_event_data_processguid` (must be non-null)

**Cell 10: Blast Radius Analysis** uses:
- `winlog_event_id` == 3 (network events)
- `winlog_event_data_sourceip` (source IP address)
- `winlog_event_data_destinationip` (destination IP address)
- `winlog_event_data_destinationhostname` (target hostname)

**Cell 11: Correlate Attack Activity** uses:
- `winlog_computer_name` (must match destination hostname)
- `winlog_event_data_processguid`
- `winlog_event_data_image`
- `winlog_event_data_commandline`
- `winlog_event_data_parentimage`
- `winlog_event_data_parentcommandline`
- `winlog_event_data_targetfilename`

**Example adaptation if your SIEM uses different names:**
```python
# Add column mapping after loading data (in Cell 8)
df_mapped = df.withColumnRenamed("SourceIP", "winlog_event_data_sourceip") \
              .withColumnRenamed("DestinationIP", "winlog_event_data_destinationip") \
              .withColumnRenamed("EventID", "winlog_event_id")
```

**Warning:** Large-scale changes may be needed depending on your data model. Review filtering cells in each notebook carefully. The first notebook (01) is the simplest one to start with, after understanding the logic it gets much easier.

## Renaming / Mapping Columns

Notebook filters expect specific column names based on the OpenSearch/Winlogbeat schema:

### Expected Schema Fields

| Category | Expected Column Name | Purpose |
|----------|---------------------|---------|
| Event Channel | `winlog_channel` | Log channel (e.g., "Microsoft-Windows-Sysmon/Operational") |
| Event ID | `winlog_event_id` | Sysmon event ID (1=process, 3=network, 11=file, 19-21=WMI) |
| Timestamp | `event_created` | Event timestamp in epoch (nanoseconds) |
| Process ID | `winlog_event_data_processguid` | Unique process session identifier |
| Network Fields | `winlog_event_data_sourceip` | Source IP address |
| Network Fields | `winlog_event_data_destinationip` | Destination IP address |
| Hostname | `winlog_event_data_destinationhostname` | Target hostname |
| Hostname | `winlog_computer_name` | Computer name |
| Process Fields | `winlog_event_data_image` | Process image path |
| Process Fields | `winlog_event_data_commandline` | Command line arguments |
| Process Fields | `winlog_event_data_parentimage` | Parent process image |
| Process Fields | `winlog_event_data_parentcommandline` | Parent command line |
| File Fields | `winlog_event_data_targetfilename` | Target file name |

### Column Mapping Strategy

If your data source uses different names, add a mapping step after data loading:

```python
# === COLUMN MAPPING ===
# Adapt your source schema to expected column names

# Define mapping: {source_column: target_column}
column_mapping = {
    "EventID": "event_code",
    "UserID": "winlog_user_name",
    "Computer": "host_name",
    "TimeCreated": "event_created",
    "DataParam2": "winlog_event_data_param2"
}

# Rename columns
for source_col, target_col in column_mapping.items():
    if source_col in df.columns:
        df = df.withColumnRenamed(source_col, target_col)

# Verify mapping
print("Mapped columns:")
for target_col in column_mapping.values():
    if target_col in df.columns:
        print(f"  ✅ {target_col}")
    else:
        print(f"  ❌ {target_col} NOT FOUND")
```

### Filter-Specific Requirements

Each notebook's filters rely on specific columns. Refer to the following cells for detailed requirements:

**Widespread Attack (`05_widespread_attack.ipynb`):**
- **Cell 8: Filter Data** - Filters for Sysmon Operational channel, specific event IDs (1, 3, 11, 19, 20, 21), and non-null processguid
- **Cell 10: Blast Radius Analysis** - Filters for network events (Event ID 3) with internal-to-internal IP connections
- **Cell 11: Correlate Attack Activity** - Joins blast radius results with process/file events on targeted hosts

If columns are missing, filters will return empty DataFrames. 

### Adding Custom Filters

If your data requires different filtering logic:

1. **Locate Cell 8: Filter Data** - Modify the filtering logic for your data source
2. **Locate Cell 10: Blast Radius Analysis** - Adjust IP address filtering if needed
3. Add your custom filters while preserving the original logic as comments
4. Test with a small sample before running on full dataset

Example for Cell 8 adaptation:
```python
# Original filter (OpenSearch/Winlogbeat)
# CleanData = df_with_time.where(F.col("winlog_channel") == "Microsoft-Windows-Sysmon/Operational") \
#     .where(F.col("winlog_event_id").isin(WS_RELEVANT_EVENTS)) \
#     .where(F.col("winlog_event_data_processguid").isNotNull())

# Custom filter (your SIEM schema)
CleanData = df_with_time.where(F.col("log_source") == "Sysmon") \
    .where(F.col("event_type").isin(["process_create", "network_connect", "file_create"])) \
    .where(F.col("process_id").isNotNull())
```

## Notes and Limitations

### What This Repository Provides

- **Practical examples** of ML/DL techniques applied to security endpoint data
- **Working code** that can be adapted to your environment
- **Documentation** of data pipelines and model approaches
- **Demonstrated threat detection** scenarios with experimental results

### What This Repository Does NOT Provide

- **Production-ready deployments** Requires tuning, changes and testing, iterating what works for you
- **Complete data pipelines** You must provide your own data infrastructure

### Evaluation Results

The notebooks were evaluated on simulated data with known threat scenarios:

- **AD Enumeration Detection:** Detected slow LDAP enumeration with 85%+ F1-score
- **C2 Beaconing:** Identified periodic beaconing patterns with clear timing signatures
- **Process Anomaly:** Detected anomalous process sequences using Seq2Seq autoencoder
- **Widespread Attack:** Identified multi-host attacks using temporal clustering

These are not adapted directly to any other environment. The purpose of publishing these is to provide concrete examples of how Machine Learning can be used in threat hunting.

### Threat Hunting Workflow

These notebooks follow this threat hunting workflow:

1. **Hypothesis Generation:** Identify suspicious patterns to investigate
2. **Data Collection:** Gather relevant logs from your SIEM/XDR
3. **Feature Engineering:** Transform raw logs into ML-compatible features
4. **Model Training:** Train unsupervised models on baseline data
5. **Anomaly Detection:** Identify deviations from normal behavior
6. **Investigation:** Manually review anomalies for threat confirmation
7. **Feedback:** Incorporate findings into hypothesis refinement

This is an iterative, human-in-the-loop process. ML assists, but does not replace query based hunting.

## License & Contributing

Licensed under the Apache License 2.0. You are free to use, modify, and distribute this code for commercial or non-commercial purposes.

## Support and Questions

For detailed methodology, theoretical foundations, and experimental analysis, refer to the thesis [document](https://www.theseus.fi/handle/10024/913789.).

Remember: These are reference implementations for educational purposes. No support whatsoever is provided for getting this to work. 
