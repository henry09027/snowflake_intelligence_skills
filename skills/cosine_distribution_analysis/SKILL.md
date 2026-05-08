# Cosine Similarity Distribution Analysis Skill

## Metadata
- **Skill Name**: Cosine Distribution Analysis
- **Version**: 1.0.0
- **Created**: 2026-05-08
- **Language**: SQL (Snowflake View)
- **Type**: Analytical View

## Objective
Analyzes the distribution of cosine similarity scores by bucketing similarities into predefined ranges and computing aggregate statistics (count, min, max, average) for each bucket. Results are returned as a tabular SQL result set compatible with Snowflake Intelligence charting and visualization tools.

## Output Schema
Returns a table with the following columns:

| Column | Type | Description |
|--------|------|-------------|
| similarity_bucket | STRING | Ordered bucket category (e.g., '1. Below 0.90', '2. 0.90 to 0.95', etc.) |
| filing_count | INT | Number of filings in this bucket |
| min_cosine_similarity | FLOAT | Minimum cosine similarity value in bucket |
| max_cosine_similarity | FLOAT | Maximum cosine similarity value in bucket |
| avg_cosine_similarity | FLOAT | Average cosine similarity value in bucket |
| min_next_perioddate | DATE | Earliest filing period date in bucket |
| max_next_perioddate | DATE | Latest filing period date in bucket |

## View Definition

```sql
CREATE OR REPLACE VIEW QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION AS
SELECT
  CASE
    WHEN lm_cosine_similarity < 0.90
    THEN '1. Below 0.90'
    WHEN lm_cosine_similarity >= 0.90 AND lm_cosine_similarity < 0.95
    THEN '2. 0.90 to 0.95'
    WHEN lm_cosine_similarity >= 0.95 AND lm_cosine_similarity < 0.97
    THEN '3. 0.95 to 0.97'
    WHEN lm_cosine_similarity >= 0.97 AND lm_cosine_similarity < 0.98
    THEN '4. 0.97 to 0.98'
    WHEN lm_cosine_similarity >= 0.98 AND lm_cosine_similarity < 0.99
    THEN '5. 0.98 to 0.99'
    WHEN lm_cosine_similarity >= 0.99 AND lm_cosine_similarity < 0.995
    THEN '6. 0.99 to 0.995'
    WHEN lm_cosine_similarity >= 0.995 AND lm_cosine_similarity < 0.999
    THEN '7. 0.995 to 0.999'
    WHEN lm_cosine_similarity >= 0.999 AND lm_cosine_similarity <= 1.0
    THEN '8. 0.999 to 1.0'
    ELSE '9. Other'
  END AS similarity_bucket,
  COUNT(*) AS filing_count,
  MIN(lm_cosine_similarity) AS min_cosine_similarity,
  MAX(lm_cosine_similarity) AS max_cosine_similarity,
  AVG(lm_cosine_similarity) AS avg_cosine_similarity,
  MIN(next_perioddate) AS min_next_perioddate,
  MAX(next_perioddate) AS max_next_perioddate
FROM QRSLLM_POC_DB.HENRY_SCHEMA.master_fin_period_filingid_yoy_risk_sp500_lmcountdict_sector_cleaned
WHERE
  lm_cosine_similarity IS NOT NULL
GROUP BY
  similarity_bucket
ORDER BY
  similarity_bucket ASC;
```

## Usage Examples

### Query the View Directly
```sql
SELECT * FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION;
```

### Filter by Year
```sql
SELECT * 
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
WHERE YEAR(max_next_perioddate) = 2025;
```

### Get Total Filing Count
```sql
SELECT 
  SUM(filing_count) as total_filings,
  COUNT(*) as num_buckets
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION;
```

### Analyze High Similarity Filings
```sql
SELECT 
  SUM(filing_count) as high_similarity_count,
  AVG(avg_cosine_similarity) as avg_similarity
FROM QRSLLM_POC_DB.HENRY_SCHEMA.V_GET_COSINE_SIMILARITY_DISTRIBUTION
WHERE similarity_bucket >= '5. 0.98 to 0.99';
```

## Similarity Buckets Reference

| Bucket | Range | Description | Interpretation |
|--------|-------|-------------|-----------------|
| 1. Below 0.90 | < 0.90 | Very Low | Significant changes in risk language |
| 2. 0.90 to 0.95 | [0.90, 0.95) | Low | Substantial language differences |
| 3. 0.95 to 0.97 | [0.95, 0.97) | Medium-Low | Moderate language changes |
| 4. 0.97 to 0.98 | [0.97, 0.98) | Medium | Some language variation |
| 5. 0.98 to 0.99 | [0.98, 0.99) | Medium-High | Minor language changes |
| 6. 0.99 to 0.995 | [0.99, 0.995) | High | Minimal language differences |
| 7. 0.995 to 0.999 | [0.995, 0.999) | Very High | Nearly identical language |
| 8. 0.999 to 1.0 | [0.999, 1.0] | Extremely High | Virtually no language change |
| 9. Other | > 1.0 | Out of Range | Edge cases/data anomalies |

## Key Metrics

- **similarity_bucket**: Categorizes cosine similarity into 9 discrete ranges for distribution analysis with numeric prefix for proper ordering
- **filing_count**: Number of filings that fall within each similarity bucket
- **min_cosine_similarity**: Minimum cosine similarity value observed in the bucket
- **max_cosine_similarity**: Maximum cosine similarity value observed in the bucket
- **avg_cosine_similarity**: Average (mean) cosine similarity value in the bucket
- **min_next_perioddate**: Earliest filing date in bucket for temporal analysis
- **max_next_perioddate**: Latest filing date in bucket for temporal analysis

## Data Requirements

### Source Table
- Database: `QRSLLM_POC_DB`
- Schema: `HENRY_SCHEMA`
- Table: `master_fin_period_filingid_yoy_risk_sp500_lmcountdict_sector_cleaned`

### Required Columns
- `next_perioddate` (DATE/TIMESTAMP): Filing period date for temporal analysis
- `lm_cosine_similarity` (FLOAT): Cosine similarity score (typically between 0 and 1)

## Key Features

1. **Tabular Result Set**: Returns results as a standard SQL table compatible with Snowflake Intelligence charting
2. **Null Handling**: Automatically filters out rows where `lm_cosine_similarity` is NULL
3. **Ordered Buckets**: Numeric prefixes (1-9) ensure proper sorting in visualizations
4. **Aggregated Statistics**: Provides min, max, average, and count for each bucket
5. **Temporal Data**: Includes min/max date ranges for each bucket for time-series analysis
6. **Performance**: View-based approach provides efficient querying and caching benefits
7. **Easy Integration**: Direct SQL view that works seamlessly with BI tools and Snowflake Intelligence

## Notes

1. The numeric prefixes in bucket names ensure correct alphabetical sorting
2. All buckets are grouped and aggregated regardless of year (use WHERE clause filters for specific years)
3. The view is read-only and automatically updates as source data changes
4. Results are directly compatible with charting tools without additional transformation
5. For year-specific analysis, apply WHERE clause to `min_next_perioddate` or `max_next_perioddate`
6. Consider adding indexes on `next_perioddate` and `lm_cosine_similarity` in source table for better performance with large datasets
