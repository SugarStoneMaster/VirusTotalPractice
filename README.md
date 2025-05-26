# VirusTotal APK Analysis

This is a general overview of the VirusTotal APK analysis project, detailing the steps taken from data ingestion to MongoDB indexing and query optimization.
The code and data are available in the repository, and the project is designed to be run in a Jupyter Notebook environment. A pdf export of the notebook is also provided for easy sharing and review.

## 1. Project Setup & Dependencies
- Installed and configured necessary Python packages (pandas, numpy, matplotlib, seaborn, pymongo 4.12.0, scipy) and Jupyter exporters (nbconvert, playwright) for PDF generation.
- MongoDB integration using PyMongo client for data storage and retrieval.

## 2. Data Ingestion & Initial Exploration
- Loaded raw VirusTotal JSON exports into a pandas DataFrame, preserving nested scan results.
- Conducted preliminary data inspection (df.info(), histograms) to understand column structure and distribution of APK sizes.
- Dataset contains 157 unique Android APK samples with comprehensive VirusTotal scan results.

## 3. Handling Missing & Duplicate Records
- Quantified missing data percentages; visualized a heatmap of null values.
- Dropped columns with >80% nulls and imputed remaining missing numeric and categorical fields (median/fill with 'UNKNOWN').
- Verified uniqueness of samples via SHA-256 and removed duplicates if any.

## 4. Feature Engineering & Selection
- Created derived metrics:
  - `size_mb` (file size in MB)
  - `detection_rate` (positives/total)
  - `days_since_first_seen` (lag between first seen and scan date)
  - `num_children` (embedded file count)
- Evaluated correlations among selected features (heatmap), noting weak threat signal correlations but strong submission history relationships.

## 5. Exploratory Visualizations
- Boxplots and histograms to illustrate:
  - Distribution of detection rates across APK types (median ~0.32).
  - Relationship between submission frequency and embedded-file complexity.
- Identified and reported outlier counts using IQR method (13 outliers for size_mb, 10 for num_children).

## 6. Data Export & Persistence
- Trimmed DataFrame to essential columns; exported as line-delimited JSON (vt_clean.jsonl).
- Loaded cleaned data into MongoDB; converted date strings to native datetime objects.

## 7. Indexing Strategy in MongoDB
- Created single-field indexes for frequent range filters (sha256, size_mb, detection_rate, num_children, days_since_first_seen).
- Defined a text index on tags for full-text search and a TTL index on scan_date to auto-expire old records.
- Ensured compatibility with PyMongo 4.12.0 by properly invoking the explain() method.

## 8. Query Patterns & Aggregations
- Demonstrated simple finds and covered queries (e.g., files with detection rate > 0.3).
- Built aggregation pipelines with $bucketAuto, $facet, and $unwind to compute:
  - Average detection rates by size or submission buckets.
  - Top tags by frequency.
  - Statistical summaries (min, max, avg, stdDev) for size, detection rate, and complexity.

## 9. Index Performance Benchmarking
- Implemented a benchmarking function using MongoDB's explain() and Python timing to compare performance:
  - On the original small dataset (n=157) and an augmented dataset (~6,400).
  - Used db.command() with 'explain' and verbosity='executionStats' for detailed query analysis.
  - Observed that single-field indexes reduce docsExamined but often add lookup overhead, yielding negligible time gains at small scale.
  - Identified compound indexes (e.g., (size_mb, detection_rate)) as the most effective for multi-predicate filters, delivering clear runtime improvements.

## 10. Final Index Recommendations
- Dropped non-essential indexes and retained only:
  1. sha256 for direct hash lookups.
  2. Compound (size_mb, detection_rate) for combined range queries.
  3. days_since_first_seen for freshness filters.
- Confirmed final index set for optimal balance between storage overhead and query performance.